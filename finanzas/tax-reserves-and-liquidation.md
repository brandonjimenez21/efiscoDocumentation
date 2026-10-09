# Reservas y liquidación de impuestos (2026-10-07)

Documento "Puntos pendientes para hacer en la plataforma", pendiente 3 (FEAT-TAX-LIQUIDATION-01). Migración: `backend/scripts/migrations/2026-10-07_tax_liquidation.sql` (la corre el usuario; el código que depende de ella no se sube hasta que lo confirme). Lógica pura en `backend/utils/taxReserves.js`, servicio `services/taxLiquidation.service.js`, controlador `controllers/taxLiquidation.controller.js`, rutas en `routes/finance.routes.js` (`/api/finance/tax/*`), interfaz `frontend/src/components/TaxReservesPanel.jsx` (en Flujo de Caja) y el bloque "Impuestos: Renta e IVA" de `ContadorPanel.jsx`.

## Qué calcula cada impuesto
Todo sale de `cash_flow_ledger` sin modificarlo. Las sumas las da la función SQL `tax_ledger_sums` con los filtros de `TAX_FILTERS` (los mismos conjuntos `COST_TYPES`, `INCOME_TYPES` y `COST_REVERSAL_TYPES` del motor financiero, para no desincronizar la utilidad).

| Impuesto | Fórmula | Periodo |
|---|---|---|
| **IVA** | `TAX_IVA − TAX_IVA_REV − (RET_IVA − RET_IVA_REV) − TAX_IVA_DEDUC − VAT_REFUND`. Positivo = a pagar; negativo = saldo a favor (no reserva) | Bimestral (6) o cuatrimestral (3) según `workshop_config.iva_periodicity` |
| **RETENCIONES** | `RET_FUENT_COMPRA + RET_ICA_COMPRA + RET_IVA_COMPRA` (lado CREDIT: lo que practicamos a proveedores, pasivo con la DIAN) | Mensual |
| **RENTA** | `max(0, ingresos operativos − costos) × tasa efectiva` menos `RET_FUENT − RET_FUENT_REV + GW_RET_RENTA` (anticipo a favor). Ingresos = `INCOME_TYPES` en CREDIT con la base sin IVA (`gross_amount`) y sin los cobros de cuotas (`concept` que empieza por "Cuota "); costos = `COST_TYPES` en DEBIT menos `COST_REVERSAL_TYPES` en CREDIT | Anual, o provisional (del 1 de enero a hoy) |

La base de renta es la utilidad operativa **antes de IVA**: no es el "Utilidad Neta" de la pantalla, que además resta el IVA. Una pérdida no genera provisión. La **tasa efectiva** la fija solo el contador (`workshop_config.income_tax_effective_rate`, 0 a 100; vacío = sin definir, no se provisiona y la pantalla lo avisa).

## Reservas a la fecha
`GET /api/finance/tax/reserves`: por impuesto, lo pendiente de liquidar (desde el inicio) más lo ya liquidado y no pagado. Total = suma por impuesto de `max(0, ...)`. Flujo de Caja lo muestra en la tarjeta "Reservas de impuestos acumulados a la fecha" y, cuando el rango termina hoy, "Caja disponible real después de reservas" = saldo bancario a hoy − reservas. Sin liquidaciones, la parte de IVA coincide con "IVA Pendiente DIAN".

## Liquidación
`POST /api/finance/tax/liquidate` (dueño o contador; el contador sobre la sede de su grupo que esté viendo): crea el corte con la función atómica `liquidate_tax_period`, que en una transacción inserta `tax_liquidations`, marca los asientos en `tax_liquidation_entries` (la marca LIQUIDADAS_PERIODO_X; índice único `(ledger_id, tax_type)`, así un asiento no se liquida dos veces por el mismo impuesto) y devuelve las sumas de **exactamente las filas marcadas**. El monto se calcula con esas sumas. Reglas: solo periodos ya cerrados (la renta provisional llega hasta hoy), un periodo no se liquida dos veces (`UNIQUE (workshop_id, tax_type, period_label)`, 409), y sin movimientos pendientes no hay corte. Fechas en hora de Bogotá. `GET .../preview` muestra el corte sin marcar nada; `GET .../periods` lista los periodos del año con su estado; `GET .../liquidations` el historial; `POST .../liquidations/:id/mark-paid` marca el corte como pagado (deja de contar como reserva; el egreso de caja por el pago a la DIAN lo registra el usuario como egreso manual). Cada liquidación y pago deja un evento (`TAX_LIQUIDATED`, `TAX_LIQUIDATION_PAID`) en `audit_events` (ver [payment-disputes.md](../negocio/payment-disputes.md)) en modo no estricto.

## Pruebas
`backend/tests/tax/` (cálculo, periodos, filtros, servicio y roles). Además se hizo una prueba de equivalencia contra un Postgres 16 desechable: 600 asientos aleatorios de 35 combinaciones tipo e impacto, y las sumas SQL más `computeTaxAmounts` dieron lo mismo que `calculateOperatingProfit` (IVA, base de renta y retenciones) en tres rangos distintos, y un corte solapado solo tomó lo no liquidado. La migración también se probó en esa base (frontera de hora de Bogotá, doble liquidación, constraints).

## Auditoría financiera (2026-10-07)
Revisado con el auditor financiero: fórmulas de IVA, signos, base de renta y control de doble liquidación coinciden con el motor. Se corrigieron antes de entregar: un periodo sin movimientos dejaba un corte vacío que bloqueaba ese periodo (ahora la función SQL lanza `TAX_NO_ENTRIES` y se deshace todo), la escritura con la vista de "todas las sedes" del contador actuaba sobre su sede propia (ahora responde 400), el monto del corte se guardaba fuera de la transacción (las sumas ahora se guardan dentro y el monto se recalcula si hiciera falta), `concept` nulo en el filtro de cuotas y el año por defecto en hora de Bogotá.

## Límites conocidos
- La liquidación no genera el asiento de pago a la DIAN: "marcar pagada" solo quita la reserva.
- La provisión de renta suma todo lo no liquidado desde el inicio de los datos (no separa años no liquidados).
- El umbral de bimestral/cuatrimestral (92.000 UVT de ingresos) no se infiere: el contador elige la periodicidad.
- Una renta provisional y la anual del mismo año comparten movimientos: lo marcado en la provisional sale de la anual, y una pérdida posterior no reajusta la provisión ya liquidada. Conviene hacer una sola provisional abierta por año.
- Las retenciones solo cuentan el lado CREDIT de `RET_*_COMPRA`: una reversión por devolución a un proveedor (si llegara a existir) no resta.
