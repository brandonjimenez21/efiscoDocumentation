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
| P3 Conducta de deuda y red | 14 a 30 d | 20% | 20% | D1 cobertura de la deuda; D2 concentración de acreedores; D3 mora máxima |
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
- **Semáforo:** `BLOQUEADO_POR_MORA` con mora > 15 d; `APROBADO_CON_CUPO` si queda cupo disponible; `RESTRINGIDO` en el resto.
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
