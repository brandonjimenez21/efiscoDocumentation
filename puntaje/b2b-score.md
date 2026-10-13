# Puntaje EFISCO 1 DT - B2B (riesgo comercial del taller)

Estado: motor y vista del propio taller listos (2026-10-08). Lo consulta el portal de distribuidoras ([b2b-distributor-portal.md](./b2b-distributor-portal.md)).

No es el puntaje de clientes finales (45/30/25, escala 50 a 250, `scoringEngine.js`): mide a un TALLER para que un
distribuidor o almacén de repuestos decida si le abre crédito comercial.

## Dónde vive

| Pieza | Archivo |
|---|---|
| Motor (función pura, `evaluarEFISCO1DT_B2B`) | `backend/utils/scoringB2B.js` |
| Entradas desde el ERP + foto diaria | `backend/services/b2bScore.service.js` |
| Endpoint del dueño | `GET /api/b2b/mi-score` (`controllers/b2b.controller.js`) |
| Migración (la corre el usuario) | `backend/scripts/migrations/2026-10-08_b2b_score_snapshots.sql` |
| Pruebas | `backend/tests/scoring/scoringB2B.test.js`, `b2bScore.service.test.js`, `b2b.mi-score.http.test.js` |

## Cómo se calcula

`Score_Raw = 50 + 200 x (W1 P1 + W2 P2 + W3 P3 + W4 P4 + W5 P5)`, escala 50 a 250.

| Pilar | Ventana | Peso joven (< 365 d) | Peso maduro (>= 365 d) | Vectores |
|---|---|---|---|---|
| P1 Liquidez y velocidad de caja | 14 a 30 d | 35% | 30% | R1 EWMA (alfa 2/15) de ingresos diarios contra 150.000 COP (meta calibrada, el documento pedía 500.000); R2 volatilidad (CV) a 30 d |
| P2 Colateral físico y bahía viva | instantánea | 30% | 25% | V1 órdenes abiertas x 0.75 contra 1.000.000 COP (meta calibrada, el documento pedía 3.000.000); V2 permanencia promedio (<= 2 d = 1) |
| P3 Conducta de deuda y red | 14 a 30 d | 20% | 20% | (reemplazado 2026-10-12: barra de 70 pts por mora ponderada, ver la última sección) D1 cobertura de la deuda; D2 concentración de acreedores; D3 mora máxima |
| P4 Inercia y disciplina operativa | 90 d | 15% | 15% | I1 días activos (objetivo 60); I2 órdenes con foto y cotización |
| P5 Trayectoria y crecimiento | 365 d | 0% | 10% | A1 crecimiento interanual; A2 recurrencia de placas (objetivo 40%); A3 carros por bahía (objetivo 120) |

Suavizado y protección: **prior bayesiano** los días 1 a 13 (`(1 - d/14) x 100 + (d/14) x Score_Raw`), **clamping de 24 h** (máximo 15 puntos
contra el último día anterior, usa `b2b_score_snapshots`), estado **CALIBRANDO_RADAR** (0 días de registro, o sin vehículos en bahía y sin
ingresos) y **bloqueo duro** con mora mayor a 15 días (`BLOQUEADO_POR_MORA`, cupo real 0).

Niveles: EXCELENTE 200 a 250 (azul oscuro `#1E3A8A`, factor de cupo 1.2), BUENO 150 a 199.9 (`#3B82F6`, 0.9), MALO 100 a 149.9 (`#8B5CF6`, 0.4),
MUY_MALO 50 a 99.9 (`#EC4899`, 0.0).

## Decisiones donde el documento no definía el caso

- **Cupo sugerido** = factor del nivel x (7 x EWMA + V_neto), redondeado a 10.000 COP (la misma base de capacidad del vector D1). El ejemplo
  del documento del portal no se puede reproducir con ninguna fórmula que use el factor.
- **Semáforo (desde 2026-10-12 con mora >= 1 d):** `BLOQUEADO_POR_MORA` con mora > 15 d; `APROBADO_CON_CUPO` si queda cupo disponible; `RESTRINGIDO` en el resto.
- **Sin flujo de caja** (todos los ingresos en 0), S(R2) = 0; la fórmula literal daría volatilidad 0 = nota perfecta. Sin órdenes, S(I2) = 0. Sin
  permanencia medida, S(V2) = 0.
- **Bahías:** el taller no tiene configurado el número de bahías; se estima como el pico de órdenes simultáneas en 365 días (mínimo 1).
- **Ingresos** = facturación de las órdenes liquidadas (`invoice_total`; si falta, mano de obra + repuestos), por día en hora de Colombia.
- **Deuda en la red:** sale de las cuotas por pagar a proveedores enlazados con una distribuidora (`providers.distribuidor_id`, ver [b2b-distributor-portal.md](./b2b-distributor-portal.md)); sin compras de la red vale 0. Una obligación en disputa no cuenta como mora.
- **No filtra `is_test_mode`**: mide el negocio del propio taller. Los talleres de prueba no deben aparecer en el directorio del portal.

## Pendiente

- Correr la migración `2026-10-08_b2b_score_snapshots.sql` (sin ella el motor calcula igual, pero sin clamping ni foto diaria; avisa en el log).
- (Hecho el 2026-10-08) La deuda de la red y el directorio por contrato >= 1.1.0 ya están conectados en el portal; falta correr la migración del portal.

## Calibración a los talleres reales (2026-10-09)

- Las metas de ingreso diario y de bahía neta bajaron de 500.000 a 150.000 COP y de 3.000.000 a 1.000.000 COP (`INGRESO_OBJETIVO_COP` y `BAHIA_OBJETIVO_COP` en `utils/scoringB2B.js`). Con las originales, los talleres reales quedaban todos entre 100 y 125 puntos sin importar su actividad. Los pesos, las fórmulas y los umbrales no cambiaron.
- "Calibrando radar" ahora exige que no haya ingresos en 90 días (antes 30) y ningún vehículo en bahía. Con ingresos solo entre 31 y 90 días atrás, el taller se califica con liquidez en 0.

## Vista del taller: panel enriquecido (2026-10-09)

- `GET /api/b2b/mi-score` (solo el taller) suma `tendencia_30d` (puntaje de la foto más vieja de los últimos 30 días y la diferencia; `getScoreTrend`) y `actividad_red_30d` (cuotas a distribuidoras pagadas a tiempo, tarde, pendientes y vencidas; `getNetworkActivity30d`). El motor devuelve además `metas` con las metas de cada vector.
- `components/b2b/B2BTallerInsights.jsx`: "Qué movió tu puntaje B2B", "Hábitos comerciales y de red" (semáforo por pilar: 75+ Excelente, 50-74 Bueno, 25-49 Por mejorar, menos de 25 Requiere atención) y "Cómo te ven los distribuidores" con los 3 hábitos de mayor ganancia posible (`200 x peso x (1 - pilar/100)` puntos).
- La consulta de la distribuidora (`evaluar-taller`) no cambia ni recibe estos campos.

## Deudas con proveedores y cobertura de vencimientos (2026-10-09)

- La deuda del Score B2B ya no se limita a la red: `getNetworkDebt` cruza TODAS las cuotas por pagar pendientes (`supplier_installments`) de proveedores comunes y de la red, sin los proveedores de sistema. Devuelve deuda consolidada (`deuda_red_cop` + `deuda_proveedores_cop`), acreedores, mora máxima y `obligaciones_15d_cop` (vencen en 15 días o ya vencidas). Los proveedores fuera de la red salen con nombre genérico.
- Motor: vector R3 de cobertura de vencimientos. `R3 = min(1, EWMA x 15 x 0,5 / obligaciones_15d)` (1 si no hay obligaciones) y `P1 = (0,70 R1 + 0,30 R2) x (1 - 0,5 (1 - R3))`. Constantes en `utils/scoringB2B.js`.
- Evaluación a una fecha (`asOf`): `evaluateWorkshopB2B(..., { now, asOf: true, scoreAnteriorOverride })` solo mira lo que existía ese día. La usa `scripts/recalibrar-b2b-historico.mjs` (la corre el usuario) para reescribir las fotos diarias.

## Score en 5 barras de puntos, deuda ponderada por mora y cartera B2C (2026-10-12)

**Pedido explícito del usuario** (documento "Cambios en la estructura del pilar de deuda y puntos"). Reemplaza la fórmula de pesos de las secciones anteriores: **el score es la suma exacta de 5 barras en puntos**, con piso en 50 y tope en 250 (`utils/scoringB2B.js`).

| Barra | Máximo | Sale de |
|---|---|---|
| Liquidez y velocidad de caja | 50 | pilar 1 (`round(50 x P1)`) |
| Colateral físico (bahía viva) | 50 | pilar 2 |
| Conducta de deuda en la red | 70 | `conductaDeudaB2B` (reglas de abajo) |
| Disciplina operativa en ERP | 50 | pilar 4 |
| Trayectoria anual y recurrencia | 30 | pilar 5 (solo taller de 365 días o más) |

- **Taller joven (< 365 d):** la trayectoria es `PENDIENTE` con 0 pts, así que su tope pasa de 250 a **220**. Es una consecuencia de la decisión, señalada al usuario; repartir esos 30 pts entre las otras barras permitiría que un taller joven llegue a 250.
- **Diferencia explícita:** el piso de 50, el prior bayesiano (días 1 a 13) y el límite de ±15 puntos en 24 h siguen aplicándose después de sumar; lo que movieron queda en `ajuste_puntos` para que las barras cuadren con el score. Salida nueva del motor: `barras_puntos`, `ajuste_puntos`, `dimensiones` y `conducta_deuda`.
- **Conducta de deuda (70 pts):** `W_mora = deuda en mora / (deuda en mora + deuda al día)`, por MONTOS (un acreedor grande en mora no se enmascara con varios pequeños al día). Sin mora: 70 pts. `W < 20%`: -10 (hasta 3 días de mora máxima, `BUENO (ALERTA MENOR)`) o -20 (`REQUIERE ATENCIÓN`). `20% a 60%`: -30 (`REQUIERE ATENCIÓN`) o -45 (`CRÍTICO`). `60% o más`: -55 a -70 interpolado con W, `CRÍTICO / DEFAULT DETECTADO`. Ya no pesan la cobertura de la deuda (D1) ni la cantidad de acreedores (D2): siguen como información en `vectores_detallados`. Sin monto de mora informado (llamadas viejas que solo traen días), se asume toda la deuda en mora.
- **Cupo congelado desde 1 día de mora** (antes: más de 15). Cualquier mora vigente deja `cupo_disponible_real_cop = 0` y `BLOQUEADO_POR_MORA`. `tiene_mora_critica` de la vista ahora significa "mora de 1 día o más". Las obligaciones en disputa siguen sin contar como mora.
- **Dimensiones para la UI (`dimensiones`):** `puntos_actuales`, `puntos_maximos`, `estado_etiqueta` (>= 85% EXCELENTE/BUENO; 50 a 84% REQUIERE ATENCIÓN; menos de 50% POR MEJORAR/CRÍTICO; `PENDIENTE` para la trayectoria joven) y `accion_prescriptiva` de máximo 15 palabras según el sub-vector más débil. La pinta `B2BScoreCard.jsx` ("Cómo se compone el puntaje") en la consulta de la distribuidora y en `/score-b2b`.
- `b2bNetworkDebt.service.js` entrega `deuda_mora_cop` (montos vencidos sin pagar), `acreedores_mora_count` y `acreedores_al_dia_count`.

### Evaluación de cartera e ingresos (`utils/portfolioB2B.js`, `services/b2bPortfolio.service.js`)
Bloque nuevo en la consulta B2B (últimos 30 días), función pura `evaluarCarteraB2C`:
- Sin actividad (`dias_activos_erp = 0` o ventas 0): "SIN INFORMACIÓN SOBRE SU CARTERA / SIN ACTIVIDAD REGISTRADA". Solo contado: "100% OPERACIÓN A CONTADO (0% Venta a Crédito Registrada)".
- **Crédito a ciegas:** más del 30% de los clientes a crédito sin 3 meses de historial en EFISCO -> `RIESGO MÁXIMO DE CARTERA`.
- Cartera menor a 30%: `EXCELENTE`. Con 30% o más, por puntaje B2C promedio: 230+ `EXCELENTE`, 170-229 `BUENA / SALUDABLE`, 130-169 `MODERADA / NORMAL`, menos de 130 `ALTO RIESGO DE IMPAGO`.
- **Decisiones donde el documento no define el caso:** una venta es a crédito si la orden liquidada tiene cuotas en `client_installments`; el puntaje de cada cliente sale de `calculateGlobalCustomerScore(cedula, { applyFreeze: false })` y "sin historial" es `memory.antiguedad_meses < 3` (o sin cédula); se puntúan como máximo 60 clientes por consulta (los demás cuentan como sin historial); no incluye ventas directas ni abonos parciales. Se importa `customerScoring.service.js` de forma perezosa porque abre el cliente de Supabase al cargarse.
- **Monitor de aceleración de moras:** `historial_moras_30d` se reconstruye día a día con las cuotas vencidas del taller con sus proveedores (`getMoraHistory30d`; no hay tabla histórica). Si pasa de 1 o menos a 5 o más dentro de una semana: `alerta_aceleracion` ("ALERTA DE RIESGO INCREMENTAL...") y el cupo sugerido baja **a la mitad** (el 50% lo definimos nosotros, el documento no da el porcentaje). No se calcula en la recalibración histórica (`asOf`).
- **Habeas Data:** la whitelist de `formatearVistaDistribuidor` solo deja pasar porcentajes, conteos, la etiqueta y el puntaje promedio redondeado (`evaluacion_cartera`), más `dimensiones` y `conducta_deuda`. Nunca nombres, cédulas ni datos individuales de los clientes B2C.
- Tests: `tests/scoring/scoringB2B.test.js`, `tests/scoring/portfolioB2B.test.js`, `tests/b2b/b2bDistributor.service.test.js`.
