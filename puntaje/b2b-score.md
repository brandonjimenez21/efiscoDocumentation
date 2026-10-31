# Puntaje EFISCO 1 DT - B2B (riesgo comercial del taller)

Estado: motor y vista del propio taller listos (2026-10-08). Lo consulta el portal de distribuidoras ([b2b-distributor-portal.md](./b2b-distributor-portal.md)).

No es el puntaje de clientes finales (45/30/25, escala 50 a 250, `scoringEngine.js`): mide a un TALLER para que un
distribuidor o almacén de repuestos decida si le abre crédito comercial.

## Dónde vive

Explicación pública para talleres y distribuidoras: `/red-de-confianza?puntaje=b2b` (`frontend/src/components/public/RedConfianzaB2B.jsx`, 2026-10-25). Si cambian las barras, los pesos, los niveles o las reglas de bloqueo, actualizar esa página.

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

## Salvas defensivas y pruebas de estrés (2026-10-20)

Documento "Actualización de estructura y pruebas en los puntajes (B2C - B2B)". Antes y después, con todas las cifras: `nuevos cambios/cambios realizados/Actualización de estructura y pruebas en los puntajes (B2C - B2B) - antes y después.txt`.

- **Contagio sistémico (F_sys)**: `utils/systemicContagion.js` (pura) + `services/b2bContagion.service.js` (arma la red local por lotes: talleres activos de la misma ciudad). W_node = deuda del taller / deuda total de la zona; con W_node > 0.15 y mora > 15 días el cupo sugerido de los satélites dependientes se multiplica por 0.80 durante 72 h. **Dependiente** (decisión 2026-10-20) = al menos el 34% de la deuda del satélite es con distribuidoras que también atienden al nodo en mora (`DEPENDENCIA_MINIMA`; el servicio calcula `deuda_por_acreedor`); compartir una sola distribuidora ya no basta (mora del nodo entre 16 y 18 días, sin guardar estado). El puntaje no cambia, solo el cupo. El resultado trae `contagio_sistemico` y la consulta de la distribuidora `alerta_contagio` (nunca dice qué taller lo disparó). **Simulación**: con el criterio de 34% el 66.6% de los enfriados sigue siendo sano (antes 70.3%) y solo 11% de los nodos que caen lo disparan; con 75% bajaría a ~60% pero la cobertura caería a 7%.
- **Gatillo de caja de 14 días**: B2B, `velocidadCajaB2B` (R_t = ingreso de los últimos 7 días / los 7 anteriores); con R_t < 0.5 el clamping de 24 h deja de frenar las BAJAS. Resultado `velocidad_caja` y `ajustes.clamping_24h_liberado`.
- **Mora dominante**: la barra de deuda (70 pts) castiga de -60 (W = 0.60) a -70 (antes -55 a -70).
- **Escrow de crédito**: el despacho a crédito reserva cupo antes de crearse (`services/creditEscrow.service.js`, migración `2026-10-20_cupo_reservas_escrow.sql`, `SELECT ... FOR UPDATE`); la consulta de la distribuidora resta lo reservado (`cupo_reservado_cop`). Ver `domain/b2b-distributor-portal.md`.
- **Simulación**: `backend/simulaciones/monte_carlo_riesgo.py` (20.000 perfiles, motor real vía `puente_motor.mjs`). En la tormenta B2B los 300 nodos en default quedan bloqueados (cupo $0) pero conservan puntaje de 85 a 201 (mediana 145): la protección real es el bloqueo duro. La cola [50-75] de B2B está vacía.

## Score B2B del distribuidor mayorista (2026-10-21)
Documento "Puntaje B2B distribuidor mayorista". Subido el 2026-10-21 (backend `9f44c1a`, frontend `800af6e`). Motor puro `utils/scoringMayoristaB2B.js`; carga de datos `services/b2bDistribuidorScore.service.js`.

`Score_Raw = 50 + 200 × (0,40 P1 + 0,35 P2 + 0,25 P3)`, escala 50 a 250, mismos cuatro niveles y colores que el Score B2B del taller. **Decisión del usuario:** manda la función maestra, así que cada pilar aporta hasta **80 / 70 / 50** puntos sobre la base de 50 (el documento también los llama 100 / 87,5 / 62,5, que no cuadran con la fórmula).

| Pilar | Subíndice (peso dentro del pilar) | Cálculo |
|---|---|---|
| P1 Salud de la cartera colocada | Índice de mora en red (50%) | M = saldo vencido / saldo por cobrar; S = 1 - M / 20% |
| | Provisión dinámica (50%) | c = 1,5 × M (3% de mora → 4,5%, el ejemplo del documento); provisión = c × saldo; S = 1 - (provisión / recaudo de 30 días) / 50% |
| P2 Concentración y exposición sistémica | Concentración (60%) | Índice Herfindahl (HHI) del saldo por deudor: 0,10 o menos = 1, 0,60 o más = 0 |
| | Resiliencia ante shocks (40%) | Los 3 deudores más grandes frenan pagos 30 días: sus cuotas vencidas y las que vencen en 30 días frente al recaudo mensual promedio de 90 días |
| P3 Liquidez operativa y flujos en la red | Dinámica de caja (60%) | EWMA (alfa 2/15) del recaudo diario de 30 días contra 300.000 COP/día |
| | Endeudamiento interno (40%) | Días de cartera = saldo / (recaudo de la cartera a crédito en 30 días / 30); 30 días o menos = 1, 120 o más = 0 |

- **Cartera que cuenta** (solo lo que la red verifica): despachos a talleres confirmados (cuotas `supplier_installments`), despachos a otras distribuidoras de origen RED confirmados (`cuotas_despachos_distribuidores`) y ventas a clientes de mostrador (`distribuidor_cuotas_clientes`). Lo de contado entra a caja el día de la confirmación o de la venta. Las facturas que otra distribuidora carga **a mano** a nombre del mayorista no cuentan (no las confirmó él). No filtra datos de prueba: mide el negocio propio, como el B2B del taller.
- Sin despachos, ventas ni cartera en 90 días: `CALIBRANDO_RADAR`. El **minorista** sale en calibración hasta su motor propio (documento "Puntaje B2B distribuidor minorista").
- Cada barra trae su acción sugerida según el subíndice más débil; si todos están en su tope dice "Sigue así".
- **Cifras definidas por nosotros (el documento no las da), aprobadas por el usuario el 2026-10-21:** mora tolerable 20%, consumo de provisión 50%, HHI 0,10 / 0,60, cluster de 3 deudores, meta de recaudo 300.000 COP/día y días de cartera 30 / 120. No se definió cupo sugerido ni bloqueo para distribuidoras.
- Pruebas: `tests/b2b/b2bRedDistribuidores.test.js`.

## Score B2B del distribuidor minorista (2026-10-21)
Documento "Puntaje B2B distribuidor minorista". Motor puro `utils/scoringMinoristaB2B.js`; datos en `cargarDatosMinorista` (`services/b2bDistribuidorScore.service.js`). Sin migración (usa las tablas de los documentos anteriores).

`Score_Raw = 50 + 200 × (0,40 P1 + 0,35 P2 + 0,25 P3) + bono de disciplina − castigo por exposición`, escala 50 a 250, mismos niveles que el resto del B2B. Pilares hasta 80 / 70 / 50 pts sobre la base de 50 (misma decisión de la función maestra que el mayorista).

| Pilar | Subíndice (peso) | Cálculo |
|---|---|---|
| P1 Liquidez, rotación y cuentas por pagar | Rotación y velocidad de venta (40%) | EWMA (alfa 2/15) de las ventas diarias de 30 días (despachos a talleres y distribuidoras, ventas de mostrador) contra 150.000 COP/día |
| | Cumplimiento con proveedores (40%) | Mitad días de mora actuales (15 o más = 0) y mitad R_t: lo pagado a tiempo (gracia de 3 días) de lo que venció en los últimos 60 días |
| | Cobertura de pasivos (20%) | Valor del inventario / saldo por pagar; 1,5 veces o más = 1 |
| P2 Solvencia y cartera local | Calidad del portafolio (52%) | 70% puntaje promedio de sus deudores ponderado por saldo (talleres: última foto `b2b_score_snapshots`; clientes de mostrador: EFISCO Score B2C, máximo 20 por evaluación; distribuidoras deudoras no se puntúan) y 30% la parte de la cartera no vencida. Sin crédito otorgado = 1 |
| | Endeudamiento (48%) | Deuda activa con proveedores / capital operativo; 0,5 o menos = 1, 2 o más = 0 |
| P3 Disciplina, perfil tributario y trazabilidad | Consistencia de registro (50%) | 70% días activos en 90 (objetivo 60: ventas, despachos, confirmaciones, pagos, inventario) y 30% despachos con factura o comprobante (adjunta, VD- automático o venta de mostrador) |
| | Estabilidad fiscal (50%) | Perfil tributario completo: régimen, responsable de IVA, declarante y tarifa de ICA (25% cada uno) |

**Decisiones del usuario (2026-10-21):**
- **Capital operativo** = valor del inventario en EFISCO (cantidad × último valor) + recaudo de 30 días (contado y cuotas cobradas).
- **Cupo sugerido** = factor del nivel (1,2 / 0,9 / 0,4 / 0) × 30 días de ventas (EWMA), redondeado a 10.000 COP; con bloqueo, $0.
- **Bloqueo duro** con mora de más de 15 días con un proveedor o con la cartera colapsada (más del 50% de lo que le deben, vencido): semáforo `BLOQUEADO_POR_MORA`, cupo $0 y sin atenuación.
- **Asimetría de flujos y respaldo de red:** si se atrasa (hasta 15 días) pero su cartera está 100% al día y con puntaje promedio de 170 o más, la penalización del cumplimiento baja a la mitad (`analisis_asimetria.caso = RESPALDO_RED`); si no, `SIN_RESPALDO`.
- **Exposición agregada (red P2P):** si el crédito que tiene colocado supera 1,5 veces su capital operativo, resta de forma gradual hasta 20 pts (con 3 veces o más).

**Decisiones nuestras (el documento no las da), aprobadas por el usuario el 2026-10-21:** meta de ventas 150.000 COP/día (la del taller), gracia de pago de 3 días, cobertura sana de 1,5, endeudamiento 0,5 / 2, **bono por disciplina** de hasta 10 pts cuando la consistencia de registro pasa de 80%. La deuda que cuenta es toda la que el minorista tiene por pagar, de la red o cargada a mano.

- **Red P2P:** el minorista ve en el Directorio a otros minoristas (`VISIBILIDAD.minorista = ['minorista']`), los consulta con el mismo flujo y les despacha desde "Nuevo despacho" ("Otro minorista"), con el número automático VD- en una sola serie para talleres y distribuidoras (`siguienteNumeroDocumento`) y comprobante en PDF (`GET /distribuidor/despachos-distribuidoras/:id/documento`). Cada consulta queda en el registro inmutable y cada crédito cruzado suma a su exposición.
- **Fechas de Colombia (2026-10-26):** en los puntajes del mayorista y del minorista, "hoy", las series diarias de 30 días y los días activos se cuentan por día de Bogotá, y los vencimientos (fechas sin hora) empiezan a la medianoche de Bogotá. Antes usaban la fecha UTC: desde las 7 p. m. una cuota que vencía ese día ya contaba como vencida, la mora subía un día, la gracia de 3 días terminaba a las 7 p. m. y lo registrado de noche caía en el día siguiente. Pruebas en `tests/b2b/fechasColombiaB2B.test.js`; contexto en `core/architecture-overview.md` ("Segunda tanda, 2026-10-26").
- Pruebas: `tests/b2b/scoringMinoristaB2B.test.js`. Verificado en staging el 2026-10-21 en Chrome (puntaje y bloqueo del minorista, directorio de minoristas, consulta y despacho P2P con inventario) y por API (comprobante, cuentas por pagar del par y consulta del mayorista).

## Score B2B Planta de aseo -> Tendero (2026-10-29, solo motor)
Documento "ESPECIFICACIÓN TÉCNICA Y MATEMÁTICA PUNTAJE EFISCO B2B SECTOR PLANTA DE ASEO TENDERO". Motor puro `utils/scoringTenderoB2B.js` (`evaluarTenderoB2B`), pruebas en `tests/b2b/scoringTenderoB2B.test.js`. **Conectado el 2026-10-30** (documento #6): `services/tenderoB2BScore.service.js` carga los datos del par planta-tienda, hay endpoints `/api/planta/puntaje` y `/api/planta/tiendas/:id/puntaje`, el panel `PuntajePlanta.jsx` y el aviso de cupo al despachar. Detalle del cableado, `I_ver` (decisión del CEO) y el gate del contrato 1.2.0 en [tenderos-planta.md](tenderos-planta.md). El CEO aprobó las propuestas (tolerancia 30 días, ciclo 15, referencia 80.000, aviso sin bloqueo) el 2026-10-30.

`Score = 50 + 200 x (0,45 P1 + 0,30 P2 + 0,25 P3)`, escala 50 a 250, mismos niveles y colores del B2B.

| Pilar | Índice (peso) | Cálculo |
|---|---|---|
| P1 Pago comercial | `I_pun` (50%) | promedio de `max(0, 1 - retraso / 30)` sobre las facturas exigibles |
| | `I_liq` (30%) | facturas pagadas a tiempo / exigibles |
| | `I_car` (20%) | 0 si alguna factura lleva más de 60 días vencida sin pagar |
| P2 Estabilidad | `I_rot` (40%) | `min(1, vendido / entregado a crédito)`, ambos en los últimos 30 días (tienda: ventas de productos de la planta sobre unidades despachadas; 2026-10-30) |
| | `I_acel` (30%) | `1 - abs(último pedido - promedio anterior) / promedio anterior` |
| | `I_ver` (30%) | verificaciones exitosas / requeridas |
| P3 Fidelidad | `I_tkt` (70%) | `min(1, ticket promedio / referencia de la red)`, respaldo 80.000 |
| | `I_ciclo` (30%) | `1 - abs(días de reabastecimiento - ciclo óptimo) / ciclo óptimo` |

**Cupo** = pedido promedio histórico (todos los pedidos ya despachados; corregido en la auditoría del 2026-10-30, antes excluía el último y con un solo pedido daba 0) x (Score / 100) x `Factor_Plazo` (8 d = 0,5; 15 d = 1,0; 30 d = 1,5 con Score mayor a 180, si no 0,8 (ajuste del CEO 2026-10-29, antes 1,0); otro plazo = 0,5), redondeado a 10.000 COP; cupo 0 con mora crítica o nivel MUY_MALO.

**Decisiones mías donde el documento no define (PROPUESTA, validar con el CEO):** tolerancia de 30 días en `I_pun`; sin datos cada índice vale el neutro 0,5 con marca "sin historial" (`I_pun` sin facturas exigibles vale 1); sin facturas ni pedidos el puntaje queda en 50 y el estado es `CALIBRANDO_RADAR`; los "días del último reabastecimiento" son el mayor entre el intervalo de los dos últimos pedidos y los días desde el último (así quien dejó de pedir también baja); ciclo óptimo por defecto 15 días; cupo 0 con MUY_MALO y con mora crítica.

**Corrección 2026-10-30 (P6-5):** sin pedidos, `I_tkt` e `I_ciclo` valían 0 aunque esta misma regla promete el neutro 0,5. Ahora valen 0,5 y salen en `sin_historial` (`ticket` y `ciclo`), igual que el resto de índices. Efecto: una tienda con facturas pero sin pedidos registrados aporta 25 puntos en P3 en vez de 0; el cupo sigue en 0 porque sale del pedido promedio histórico, que sin pedidos vale 0. Con pedidos nada cambia. Pruebas en `tests/b2b/scoringTenderoB2B.test.js`.
