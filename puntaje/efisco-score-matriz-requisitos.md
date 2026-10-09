# EFISCO Score: matriz de requisitos del documento "Estructura del puntaje global"

Auditoría del 2026-10-04 contra el código vigente. Cada fila trae el requisito del documento, su estado y cómo funciona por dentro. Estados: **OK** = implementado y probado; **OK (matiz)** = implementado con una interpretación que conviene conocer; **Pendiente** = no cubierto.

> **Recalibración 2026-10-07**: la escala 50 a 250 se conserva, pero los pilares pasaron de 60/10/30 a **45/30/25 sobre base 250** (112.5 / 75 / 62.5 pts), el Pilar 2 ahora es IEA + aceleración de deuda + perfil, y el puntaje se suaviza con el Memory Kernel. Donde las filas de abajo hablan de "60/10/30", "50 + aportes" o "base de naturaleza", manda la sección **Recalibración 2026-10-07** al final.

Archivos de referencia: motor `backend/utils/scoringEngine.js`; constantes `backend/utils/riskConstants.js` (espejo `frontend/src/utils/riskConstants.js`); servicio `backend/services/customerScoring.service.js`; congelamiento `backend/utils/scoreFreeze.js`; controlador `backend/controllers/clients.controller.js`. Tests: `backend/tests/scoring/` y `backend/tests/clients/clients.scoreScale.test.js`.

## Bloque 1. Escala propietaria de 50 a 250

| # | Requisito | Estado | Cómo funciona |
|---|---|---|---|
| 1A | Escala 50 a 250 y mapeo `round(50 + base × 2.0)` | OK | `calcSectorScore` calcula la base 0 a 100 (pilares 60/10/30) y `toFinalScore(base)` la convierte. Todo puntaje que sale del motor (API, PDF, Telegram, QR, pantallas) viene en 50 a 250. Los pilares y sub-puntajes se quedan en 0 a 100 (son los componentes de la base). Un cliente sin historial recibe el piso (50, γ = 0; ver el embudo estadístico en reception-and-quotes.md). |
| 1B | Matriz de cupos: <150 → 0.0, 150-189 → 0.8, 190-229 → 1.8, ≥230 → 3.0 | OK | `CREDIT_SCORE_MULTIPLIERS` y `creditMultiplierOf(score)`; las usa `evaluateCreditAssessment`. La matriz 50/70/90 quedó descartada (era borrador sobre base 0-100). |
| 1C | Niveles y colores del documento: 50-129, 130-169, 170-209 y 210-250 (el documento daba rojo / naranja / verde / azul) | OK (colores cambiados a pedido) | `RISK_LEVELS` / `riskLevelOf`. La clave `risk` es `alto` / `medio` / `bajo` / `elite`. Colores vigentes (paleta de colores.jpeg, decisión del usuario del 2026-10-04): 50-129 magenta #C0267F, 130-169 morado #6B2178, 170-209 azul #0E6BB5, 210-250 azul oscuro #00428C. Los usan Recepción, Mi Puntaje, Red de Confianza, el PDF (encabezado y chips de pilares) y Telegram (emojis). |
| E1 | Constantes globales (`riskConstants`) que afecten backend, medidores web y PDF | OK (matiz) | Es `riskConstants.js` (el proyecto es JavaScript, no `.ts`) con un espejo en el frontend; hay que mantenerlos sincronizados. |
| E2 | Respuestas de la API con `min_scale: 50`, `max_scale: 250`, `time_window_months: 3` | OK | `scaleMeta()` se mezcla en todo objeto de puntaje (sector, global combinado y sin historial). Cada objeto también trae `window {days: 90, ...}`. |
| E3 | Las interfaces reflejan botones, redirecciones y textos | OK | Ver bloques 3 y 4. |
| E4 | Desglose "Cómo se compone tu puntaje global" coherente con la escala | OK | En Mi Puntaje, Recepción y el PDF cada pilar muestra los puntos que aporta (hasta 120 pago, 20 estabilidad y 60 fidelidad) sobre una base de 50, de modo que 50 + aportes = puntaje global exacto. Lo calcula pillarPoints (riskConstants), que reparte el puntaje real y ajusta el residuo del redondeo. El índice interno de cada pilar (0 a 100) sigue gobernando el color de la barra. |

## Bloque 2. Horizonte de 3 meses (90 días)

| # | Requisito | Estado | Cómo funciona |
|---|---|---|---|
| 2.1 | Ventana móvil de 90 días para el puntaje | OK | `calcSectorScore` filtra por `SCORE_WINDOW_DAYS = 90`: mora, cuotas por orden, pasarela, frecuencia (visitas anualizadas) y ticket salen solo de la ventana. |
| 2.2 | Cuotas con deuda no desaparecen | OK | Las cuotas `pendiente` cuentan siempre, aunque sean viejas. |
| 2.3 | Sin actividad en 90 días | OK (matiz) | Si no hay nada en la ventana pero sí historial, se usa el historial completo (`window.fallback = true`) para no caer al máximo por defecto. |
| 2.4 | "Ajustar las consultas a la base de datos" | OK (matiz) | Las consultas siguen trayendo todo el historial de la cédula y el filtro de 90 días se aplica dentro del motor, porque la antigüedad, la tendencia y el congelamiento necesitan el historial largo. |
| 2.5 | Factor de decaimiento (lambda) | OK | `MORA_DECAY_LAMBDA = 0.15`: la sanción de mora decae con `exp(-0.15 × meses)` desde la mora más reciente. |
| 2.6 | Espera para habilitar el reporte | Cambiado (2026-10-04) | `REPORT_UNLOCK_DAYS` pasó de 30 a 90 días y ahora bloquea también al taller: `consultScore` y `verifyScoreOTP` responden `{verified:true, locked:true, unlockDate}` sin calcular, sin PDF y sin cobro. Recepción muestra "se está construyendo". Desde 2026-10-05 Recepción, el portal `/mi-puntaje` y el bot de Telegram usan la misma función (`getScoreUnlockInfo`), que no cuenta las fichas de prueba como primer registro: mismo día de habilitación en todos lados. |

## Bloque 3. Navegación y botón de consulta

| # | Requisito | Estado | Cómo funciona |
|---|---|---|---|
| 3A | Botón "Consultar Mi Puntaje" en el panel principal | OK | `Dashboard.jsx` tiene una tarjeta destacada que enlaza a `/mi-puntaje`; se retiró de `RedConfianza.jsx`. Verificado en el navegador. |
| 3B | CTA hacia la vista explicativa de Red de Confianza | OK | La vista de resultados de `MiPuntaje.jsx` incluye el botón "Cómo se calcula" hacia `/red-de-confianza?para=cliente`. Red de Confianza conserva toda su explicación (niveles, pilares, métricas). La vista de resultados exige un código por correo, por eso se verificó por código y build, no en pantalla. |

## Bloque 4. Mensajes

| # | Requisito | Estado | Cómo funciona |
|---|---|---|---|
| 4A | Mensaje del QR de registro (escala 50 a 250, 3 meses) | OK | `QR_SCORE_MESSAGE` se muestra siempre en la tarjeta "Tu Puntaje EFISCO 1 DT" del Registro Seguro, junto a la fecha de habilitación. Verificado en el navegador. |
| 4B | Leyenda junto al medidor de la vista de consulta | OK | `SCORE_VIEW_LEGEND` en `MiPuntaje.jsx`; el PDF lleva una versión equivalente. |

## Puntaje para talleres de bicicletas

| # | Requisito | Estado | Cómo funciona |
|---|---|---|---|
| B1 | `getScoringThresholds(vehicleType)` | OK | Devuelve funciones de evaluación de `'car'` o `'bicycle'`. Fórmulas, pilares, bonificaciones y congelamiento son idénticos en ambos. |
| B2 | Pilar 1, amortización: 1 cuota 100, 2 cuotas 50, ≥3 cuotas 20 | OK | `amortizacion(avgCuotas)` del sector bicicletas (autos: ≤2 → 100, ≤4 → 50, resto 20). |
| B3 | Pilar 2, naturaleza: jurídica 100, natural 70 | OK | `tipoBase(clientType)`. El tipo se infiere de las órdenes (`inferClientType`: la orden más reciente con `work_orders.client_type` que empiece por "Empresa"). Antes los controladores nunca lo pasaban; se corrigió en esta auditoría. Autos: 60 / 50. Desde la segunda orden el Pilar 2 pasa a valer lo que el Pilar 1 (regla vigente). |
| B4 | Pilar 3, frecuencia: 6-12 → 100, 3-5 → 50, 1-2 → 20 | OK | `frecuencia(visitasAnio)`; las visitas se anualizan sobre los últimos 90 días. |
| B5 | Pilar 3, ticket: >180.000 → 100, 70.000-180.000 → 50, <70.000 → 20 | OK | `ticket(avgTicket)`, con los bordes exactos del documento (180.000 es 50, 70.000 es 50). |
| B6 | Bonificaciones de Pilar 1 y 3 y congelamiento idénticos | OK | Las bonificaciones dependen de los sub-puntajes (100/20/50 y 100/20/20), así que operan igual en ambos sectores. |
| B7 | Función modular que recibe `historyData` y `vehicleType` y retorna puntaje y recomendación | OK (matiz) | `evaluateSectorHistory({orders, installments}, vehicleType, opts)`. Devuelve el puntaje en 50 a 250 (el documento dice 0-100, pero la decisión oficial fue 50-250 en todo) y la recomendación RED/YELLOW/GREEN. |
| B8 | Tipos Zod/TypeScript | OK (matiz) | El proyecto es JavaScript; los tipos (`Sector`, `Recommendation`, `Trend`, `CustomerRiskProfile`, `ScoringMetrics`) están como JSDoc en `scoringEngine.js`. |

## Puntaje global unificado (auto + bicicletas)

| # | Requisito | Estado | Cómo funciona |
|---|---|---|---|
| G1 | Cédula/NIT como ID global | OK | Todas las consultas filtran por `client_cedula` sin límite de taller. |
| G2 | Un solo sector → Score_Global = Score_Sectorial | OK | `scoreCustomer` clasifica cada orden por su categoría (`sectorOfOrder`) y cada cuota por la orden a la que pertenece. |
| G3 | Ambos sectores → ponderar por facturación | OK | `Σ Score_sector × (V_sector / V_total)`; V = `invoice_total`, o mano de obra + inventario si falta; sin volumen, por número de órdenes. Se pondera el puntaje exacto sin redondear y se redondea una sola vez. |
| G4 | Mora > 5 días en cualquier sector → ROJO | OK (matiz) | Se usa la mora promedio de la ventana por sector (`> GUARDRAIL_MORA_DAYS = 5`), la misma que cancela un congelamiento. Fuerza `recommendation = RED` y `risk = alto`, agrega `ALERTA_MORA_CRITICA`, pero el puntaje numérico no se altera (`score_level` conserva el nivel por puntaje). |
| G5 | Congelamiento 6 meses si Score_Global > 61 (172) por más de 2 años | OK (matiz) | `applyFreezeLogic`: puntaje ≥ 172 y 24 meses de antigüedad; vence a los 6 meses. Dos matices: se usa "mayor o igual" (como el código previo) y "2 años continuos" se aproxima con la antigüedad de la cuenta, no con un historial puntaje por puntaje. Desde esta auditoría tampoco se crea un congelamiento a quien tiene mora promedio > 5 días (antes se creaba y se cancelaba en cada consulta). |
| G6 | `calculateGlobalCustomerScore(customerId)` con JSON | OK | `customerScoring.service.js`: devuelve `scoreGlobal`, `minScale`, `maxScale`, `timeWindowMonths`, `recommendation`, `risk`, `trend`, `trendDeltas`, `breakdownBySector {car, bicycle}`, `volumes`, `weights`, `activeFlags`, `benefits`. |

## Tendencia (DT, estilo FICO 10 T)

| # | Requisito | Estado | Cómo funciona |
|---|---|---|---|
| T1 | Ventana de los últimos 6 a 12 registros de servicio | OK (matiz) | `computeTrend` toma hasta 12 órdenes del sector; con menos de 6 no hay tendencia y los deltas valen 0 (`TREND_MIN_RECORDS`). Hoy casi ningún cliente la tiene activa por tener poco historial. |
| T2 | Pendiente (regresión) de mora, frecuencia y ticket | OK | `linearSlope` por mínimos cuadrados sobre: mora promedio por orden, frecuencia (visitas por mes según el intervalo entre órdenes) y ticket por orden. |
| T3 | `delta_pago`: pendiente de mora < -0.2 → +0.15; > 0.2 → -0.15 | OK | `DELTA_PAGO_IMPROVING` / `DELTA_PAGO_WORSENING`. |
| T4 | `delta_fidelidad`: ticket y frecuencia suben → +0.10 | OK | Ambas pendientes > 0. |
| T5 | `Pilar1 = min(100, P1 × (1 + delta_pago))`, `Pilar3 = min(100, P3 × (1 + delta_fidelidad))` antes de ponderar 60/10/30 | OK | Se aplica en `calcSectorScore`. `pillars_base` guarda los valores antes de la tendencia. |
| T6 | Divergencia: un sector se deteriora y el otro mejora, el deteriorado pesa > 60% → baja VERDE a AMARILLO con `ALERTA_DETERIORO_SECTORIAL` | OK | En `scoreCustomer`, con las pendientes de mora (> 0.2 y < -0.2) y el peso por facturación (`DIVERGENCE_VOLUME_SHARE = 0.6`). Si pesa menos del 60% no baja. |
| T7 | Salida con `scoreGlobal`, `recommendation`, `breakdownBySector`, `trendDeltas`, `activeFlags` | OK | Ver G6. La etiqueta `trend` (UPWARD / STABLE / DETERIORATING) sale de la suma de deltas. |

## Analítica avanzada

| # | Requisito | Estado | Cómo funciona |
|---|---|---|---|
| A1 | Utilización de cupo: ≤50% → 100, 51-80% → 60, >80% → 20 con aviso de sobreendeudamiento | OK (matiz) | `creditUtilizationScore`: `(saldo pendiente con el taller + servicio nuevo) / techo del taller`. Entra al Pilar 1 como cuarto componente solo cuando se evalúa un crédito en un taller (`buildWorkshopCreditAssessment`); los reportes y el PDF no lo incluyen. Bandera `AVISO_SOBREENDEUDAMIENTO`. |
| A2 | Búsqueda intensiva de crédito: ≥2 talleres en 30 días → -15 puntos al Pilar 2 y `FLAG_CREDIT_SEEKING_BEHAVIOR` | OK (matiz) | `detectCreditSeeking` cuenta talleres distintos que abrieron cuotas al cliente (`client_installments.created_at`) en los últimos 30 días, en toda la red. No existe una tabla aparte de "solicitudes de crédito"; las cuotas creadas son el indicador. 1 taller es normal. |
| A3 | Vector de saldo insoluto: pendiente < 0 → +0.05, > 0 → -0.10 al `delta_pago` | OK | Serie mensual del saldo pendiente de los últimos 6 a 12 meses (exige al menos 6 meses de historia). Se suma al `delta_pago`. |
| A4 | Decaimiento temporal de la mora `Base × exp(-0.15 × meses)` | OK | Aplicado a la penalización de mora desde la mora más reciente; una mora en curso no se atenúa. `breakdown.p1.mora` muestra `score_base`, `score` efectivo, `decay_factor` y `meses_desde_mora`. |

## Cupo sugerido de crédito

| # | Requisito | Estado | Cómo funciona |
|---|---|---|---|
| C1 | Multiplicador por puntaje | OK | Ver 1B. Una recomendación ROJA por guardrail fuerza multiplicador 0 y plazo 0. |
| C2 | Modificador tendencial: ascendente 1.15, estable 1.00, deteriorada o búsqueda de crédito 0.70 | OK | `DT_MODIFIER` en `evaluateCreditAssessment`. |
| C3 | `RawLimit = ticket histórico × multiplicador × modificador` | OK | El ticket es el promedio de todas las órdenes del cliente (`orderValue`). |
| C4 | Parámetro `workshop_max_credit_limit_cop` en la configuración del taller | OK | Columna en `workshop_config` (la migración ya está corrida; verificado en la base) y campo en Ajustes > Datos del Taller > "Política de crédito". Vacío = sin techo. También `workshop_credit_terms_days` para el plazo. |
| C5 | `FinalCreditLimit = min(Raw, techo)`, redondeado al mil | OK | `effective_credit_limit_cop`; 452.300 → 452.000. Sin techo configurado se calcula igual y Recepción avisa que falta definirlo. |
| C6 | Estructura de salida | OK | `{ score, trend, risk_level (LOW/MEDIUM/HIGH), credit_assessment { recommended_individual_limit_cop, workshop_policy_ceiling_cop, effective_credit_limit_cop, max_allowed_terms_days } }` más `details`. Va en la respuesta de `consult-score` y `verify-score-otp` como `creditAssessment`, nunca en endpoints públicos. |
| C7 | Cliente nuevo sin ticket → ticket promedio del taller | OK | `buildWorkshopCreditAssessment` calcula el promedio de las últimas 300 órdenes del taller (`details.ticket_source = 'workshop'`). |
| C8 | Un puntaje congelado | OK (matiz) | Si el puntaje global está congelado, el cupo respeta el puntaje congelado. |

## Pendientes y matices abiertos
- **Congelamiento sostenido**: hoy usa antigüedad de 24 meses y puntaje actual; verificar 24 meses continuos con el historial de rangos (`score_range_log`) sería más fiel al documento.
- **Mora crítica**: se interpreta como mora promedio del sector (no el máximo de una sola cuota).
- **Tendencia**: exige 6 órdenes por sector; con menos no aplica.
- **Espera del reporte**: 90 días desde el primer registro del cliente en la red, para cliente (QR, portal, Telegram) y para el taller (Recepción).

## Recalibración 2026-10-07 (documento "Recalibración en el puntaje crecidito")

Constantes en `backend/utils/riskConstants.js` (espejo `frontend/src/utils/riskConstants.js`); lógica en `backend/utils/scoringEngine.js`; contexto de red en `loadScoringContext` (`customerScoring.service.js`).

| Pilar | Peso | Máx. | Sub-variables (fracción del pilar) |
|---|---|---|---|
| Comportamiento de pago | 45% | 112.5 | Puntualidad 50% (56.25), retorno de caja 30% (33.75: cuotas por orden + pasarela [+ utilización de cupo]), cero cartera castigada 20% (22.5: ninguna cuota pendiente con más de 90 días vencida) |
| Estabilidad y perfil | 30% | 75 | IEA 50% (37.5), aceleración de deuda 30% (22.5), perfil e identidad 20% (15) |
| Fidelidad | 25% | 62.5 | Frecuencia 50% (31.25), ticket 30% (18.75), antigüedad y recurrencia 20% (12.5) |

- **Puntaje**: `(pago + estabilidad + fidelidad) × (1 − penalización de mora) × γ`, acotado a 50 a 250 (ya no es "50 + aportes"). `pillarPoints` reparte el puntaje real entre los tres pilares, así que la suma de los aportes es el puntaje (un cliente sin historial queda en 50 con aportes 0). El congelamiento sigue en 172.
- **IEA** = deuda activa en la red / cupos aprobados en la red (`computeNetworkExposure`, todas las cuotas de la cédula). El cupo de un taller es su `workshop_max_credit_limit_cop`; si no lo tiene, **cupo implícito = 4 × el mayor crédito que ese taller le abrió al cliente** (aproximación: no existe una tabla de cupos aprobados por cliente). Hasta 30%: 100% de los puntos; 30 a 60%: atenuación sigmoide (1 en 30%, 0 en 60%); más de 60%: 0 pts, bandera `ALERTA_ENDEUDAMIENTO_RED`, descuento de 10 puntos al puntaje y cupo sugerido ×0.25 (en 60% ×0.5, 1.0 hasta 30%).
- **Aceleración (ΔD/Δt)**: deuda nueva (cuotas creadas) en 72 horas como % de los cupos aprobados. Umbral K = 25%: por encima, la sub-variable baja linealmente a 0 en 100% y se levanta `FLAG_DEBT_ACCELERATION`. La penalización logarítmica es `P = 4 × ln(1 + max(0, pct − 25))`, tope 30 puntos, y se resta DESPUÉS de mezclar con el histórico.
- **Perfil** (un tercio cada uno): aceptación de autorización (OTP / Habeas Data, `client_legal_acceptances.accepted`), cédula validada (cliente con nombre) y contacto completo (celular y correo). Sin contexto (`profile` nulo) se asume verificado.
- **Memory Kernel**: `S = α × S_instantáneo + (1 − α) × S_histórico − P(ΔD)`. S_histórico = promedio con vida media de 6 meses de los puntajes de cierre de cada mes (hasta 24 meses atrás), reconstruidos con los datos que existían en esa fecha (`asOfInstallments`), sin tablas nuevas. α por antigüedad en la red: <3 meses 0.90, 3 a 11 0.60, 12 a 23 0.35, 24 o más 0.20. Se aplica por sector antes de ponderar por volumen. `evaluateSectorHistory` es instantáneo (solo resta P).
- **Cupo de crédito**: `evaluateCreditAssessment` suma el factor del IEA y, con menos de 3 meses de antigüedad, acota el cupo a 1 × ticket (fase de entrada); desde el mes 4 escala con el puntaje.
- **Interfaces**: Recepción (45/30/25, desglose de los 75 pts y margen IEA, aviso en el cupo), Mi Puntaje (anillo 45/30/25, tarjeta de estabilidad con nota sobre 75 y estado del margen), PDF, Red de Confianza y Landing (datos tendenciales estilo FICO 10T).
- Cambia la calibración de TODOS los puntajes (clientes con poco historial bajan, los de Pilar 2 completo suben). Tests en `tests/scoring/scoringEngine.test.js`.

## Tablero de puntaje: variación, factores y hábitos (2026-10-07, FEAT-UI-SCOREBOARD)

Frontend en `components/ScoreDashboard.jsx` (Mi Puntaje y modal de Recepción); el motor lo alimenta sin tablas nuevas.
- **Semi-arco** 50 a 250 con las cuatro bandas de la paleta y el tramo lleno del nivel del cliente; chip de variación contra el ciclo anterior (`score_delta`: el mismo puntaje con los datos de hace 30 días, `previous_score`; sin ciclo anterior si el cliente tiene menos de 30 días o su puntaje está congelado).
- **Qué movió tu puntaje** (`factors`, últimos 30 días, `computeScoreFactors`): pagos verificados y moras cerradas, pagos y servicios al día, incumplimientos o demoras, servicios y créditos nuevos.
- **Hábitos**: frecuencia de mantenimiento, recurrencia de pago, liquidez directa y margen de endeudamiento, cada barra en el tono de su nivel (Excelente, Estable, Requiere atención, Alerta crítica).
- **Aviso al pie**: "dato comercial privado, tratado conforme a la Ley 1581 de 2012. EFISCO no es una central de riesgo" (decisión del usuario: no se usa "dato crediticio" ni la Ley 1266; el texto lo debe confirmar el CEO).
- **Mi Puntaje, "Tus movimientos"**: lista las cuotas del cliente con botón **Actualizar** (recarga solo los movimientos, sin cerrar la sesión del portal) y el enlace "¿Reportar inconsistencia o pago no reflejado?", ver [payment-disputes.md](../negocio/payment-disputes.md). Los movimientos de talleres de prueba llevan el aviso "Movimiento de prueba, no afecta tu puntaje" y solo los ve el cliente demo y las cédulas de `INTERNAL_TEST_CEDULAS` (`utils/internalCedulas.js`: `DEMO_CLIENT_CEDULA`); el motor ignora todo lo marcado `is_test_mode`.
- Una cuota con reclamo abierto se trata como `en_disputa`: no cuenta como mora, cartera castigada ni aceleración y el puntaje trae la bandera `DISPUTA_ABIERTA`.

## Facturas viejas liquidadas hoy (2026-10-09)

La ventana de 90 días (bloque 2.1) mira la fecha de creación de la orden, así que una factura que llevaba meses en Bahía y se liquida hoy quedaba fuera. Ahora una orden vieja entra a la ventana si se liquidó dentro de ella: `loadSettlementDates` (`services/customerScoring.service.js`) toma el primer asiento del Libro Mayor de cada orden vieja de la cédula (`settledAtByOrderId` en el contexto de scoring) y el motor lo usa como `settled_at` solo para decidir la ventana. La regla de los 90 días no cambia: una factura liquidada hace más de 90 días no cuenta. Las muy antiguas sin asiento en el Libro Mayor se siguen fechando por su creación.
