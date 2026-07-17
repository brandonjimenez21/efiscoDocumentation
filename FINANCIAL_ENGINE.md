# Motor Financiero

> Ver también: [README](README.md) · [BUSINESS_RULES](BUSINESS_RULES.md) · [TESTING](TESTING.md) · [API](API.md)

`backend/utils/financialEngine.js` — núcleo de cálculo inmutable. Constantes 2026: UVT = $50.318, umbral retenciones = 27 UVT ≈ $1.358.586.

## Matriz de Decisión de Liquidación

```mermaid
flowchart TD
    Start([Inicio Liquidación]) --> Data[Cargar: Labor + Parts + Config]
    Data --> Tier{Tier del Servicio?}

    Tier -- Premium --> PM[Aplicar Margen Premium: ~10%]
    Tier -- Básico --> BM[Aplicar Margen Básico: ~5%]

    PM & BM --> Base[Base Impositiva]
    Base --> IVA[Cálculo IVA: 19% si aplica]
    IVA --> Total[Total Factura]

    Total --> Gateway{Usa Pasarela?}
    Gateway -- Bold/Addi --> Comm[Calcular Comisión + IVA]
    Gateway -- Efectivo --> NoComm[Cero Comisión]

    Comm & NoComm --> Rets{Agente Retenedor?}
    Rets -- Sí --> CalcRets[ReteIVA 15% / ReteFuente / ReteICA]
    Rets -- No --> ZeroRets[Sin Retenciones]

    CalcRets & ZeroRets --> DB[(Persistencia Ledger Atómico)]
    DB --> Result[Net Cash Inflow + Real Bank Balance]
```

## 1. Liquidación de servicios (`liquidateClientInvoice`)

```
Base Impositiva = Mano de Obra × (1 + margen%) + Repuestos (con margen)
IVA             = Base × vat_percentage            (si is_responsable_iva)
Total Factura   = Base + IVA

Si clientIsRetainer AND supera umbral UVT AND is_agente_retenedor_renta (taller
Ordinario/Gran Contribuyente — Régimen Simple/No Responsable de IVA nunca retienen):
  ReteFuente = Base × retefuente_rate_declarante
  ReteICA    = Base × (reteica_rate / 1000)
  ReteIVA    = IVA  × reteiva_rate

Pasarela Bold presencial:
  tarjeta_nacional     → 2.99% + $300
  tarjeta_internacional → 3.99% + $300
  qr_billetera         → 1.50%

Pasarela Bold online:
  tarjeta_nacional     → 3.49% + $900
  tarjeta_internacional → 4.49% + $900
  qr_billetera         → 1.50%

Pasarela Addi: gateway_addi_rate / 100
IVA sobre comisión = Comisión × 0.19

Net Cash Inflow = Total − ReteFuente − ReteICA − ReteIVA − Comisión − IVA comisión
```

**Venta a crédito**: `INC_GROSS.net_amount = 0` al liquidar; ingresos reales entran vía `payInstallment`. Ver el flujo funcional completo en [BUSINESS_RULES.md — Panel de Cobros](BUSINESS_RULES.md#panel-de-cobros-cobros).

---

## 2. Liquidación de compras (`liquidateSupplierPurchase`)

```
Base          = total_gross_cost / (1 + 0.19)
IVA de Compra = total_gross_cost − Base

Retenciones (base ≥ 27 UVT AND proveedor ≠ 'simple' AND taller
is_agente_retenedor_renta AND NO (taller Gran Contribuyente Y proveedor Gran
Contribuyente — dos grandes contribuyentes no se retienen entre sí):
  ReteFuente: declarante → Base × supplier_retefuente_rate
              no declarante → Base × retefuente_rate_no_declarante
  ReteICA:   Base × (reteica_rate_supplier ?? config.supplier_reteica_rate / 1000)
  ReteIVA:   IVA × 0.15

Si proveedor NO es responsable de IVA (`is_responsible_vat=false`): el total
facturado no trae IVA embebido, Base = total_gross_cost (sin dividir por 1.19).

GMF (payment_method='banco' AND apply_4x1000=true):
  GMF = total_gross_cost × 0.004

Net Outflow = total_gross_cost − ReteFuente − ReteICA − GMF
```

---

## 3. Salud financiera global (`calculateGlobalHealth`)

Reescrita en rev. 22 alrededor de la distinción **caja real vs. informativo/devengo** (ver sección siguiente) — antes sumaba una lista fija de tipos DEBIT/CREDIT a mano, lo que duplicaba montos ya restados dentro de `net_amount` (ej. `GW_FEE`/`GW_VAT`) y no reconocía egresos manuales fuera de esa lista blanca:

```
Para cada asiento: isCashEntry(move) = !NON_CASH_TYPES.has(move.type)
                    cashValue(move)  = move.net_amount ?? move.amount   (0 en ventas a crédito)

totalInflows  = Σ cashValue(move) donde isCashEntry(move) && impact = CREDIT
totalOutflows = Σ cashValue(move) donde isCashEntry(move) && impact = DEBIT

bankBalance     = totalInflows − totalOutflows
ivaLiability    = Σ TAX_IVA (CREDIT) − Σ RET_IVA (DEBIT) − Σ VAT_REFUND (CREDIT) − Σ TAX_IVA_DEDUC (DEBIT)
realBankBalance = bankBalance − ivaLiability
```

El **Capital Inicial** (`OPENING_BALANCE`, el saldo migrado del cuaderno/Excel previo del taller) SÍ es caja real (`isCashEntry` lo cuenta), pero se reporta aparte de "Total Ingresos"/"Total Egresos" vía `isOpeningBalance`/`openingBalanceValue` — no es una venta ni un gasto del período, es el punto de partida. Mismo criterio espejado en `frontend/src/utils/ledgerLabels.js`.

---

## Caja real vs. informativo/devengo — por qué existe la distinción

Bug de fondo reportado por el usuario: el Flujo de Caja mezclaba "cuánto entró en total" con "cuánto hay realmente en el banco", y no reconocía el costo de un repuesto/mano de obra en el momento correcto. La causa raíz era que `cash_flow_ledger` solo distinguía `CREDIT`/`DEBIT`, pero varios tipos de asiento son **informativos**: registran a dónde se fue la plata (impuestos, retenciones, comisiones, costo de lo vendido) cuyo efecto de caja **ya está incluido** dentro de otro asiento (`net_amount` de `INC_GROSS`/`SUP_PAY`) — sumarlos aparte como entrada/salida los cuenta dos veces.

`NON_CASH_TYPES` (backend `financialEngine.js`, espejado en frontend `ledgerLabels.js`): `TAX_IVA`, `RET_FUENT`, `RET_IVA`, `RET_ICA`, `GW_FEE`, `GW_VAT`, `TAX_IVA_DEDUC`, `RET_FUENT_COMPRA`, `RET_ICA_COMPRA`, `RET_IVA_COMPRA`, `INV_COGS`, `MECH_COMMISSION`.

Los dos últimos son nuevos en rev. 22 y merecen su propia explicación porque, a diferencia de los impuestos/retenciones (que nacen ya "descontados" de otro asiento), representan un **costo real que aún no generó salida de caja**:

- **`INV_COGS`** (Costo de Ventas) — se genera al **liquidar** una orden (`settleOrder`), no al comprar el repuesto: `unit_cost_at_time × cantidad_consumida` de cada ítem que la orden efectivamente usó (via `service_inventory_items`). El efectivo del repuesto ya salió (o nunca salió, si era stock migrado) cuando se COMPRÓ — contarlo otra vez como salida de caja al venderlo sería duplicar el gasto. Por eso es informativo: entra a "Costos" del Flujo de Caja sin tocar `bankBalance`.
- **`MECH_COMMISSION`** — comisión **devengada** por cada mecánico `comision`/`mixto` asignado a la orden (repartida según su participación real en mano de obra + inventario, ver [BUSINESS_RULES.md — Bahías](BUSINESS_RULES.md#bahías-órdenes-de-trabajo)), generada también al liquidar. Es costo reconocido, pero el mecánico todavía no recibió ese dinero en efectivo — por eso tampoco es caja real.

El ciclo se cierra con dos tipos que SÍ son caja real (no están en `NON_CASH_TYPES`), generados por `mechanics.controller.js:payMechanic` cuando el dueño usa el botón "Pagar a Mecánico" en Configuración:

- **`MECH_SALARY_PAY`** — sueldo efectivamente pagado a un mecánico de nómina fija. Es caja real Y costo nuevo (no se reconoció en ningún otro asiento antes).
- **`MECH_COMMISSION_PAY`** — comisión ya devengada (`MECH_COMMISSION`) que se liquida en efectivo. Es caja real, pero **no** se suma otra vez a "Costos" — el costo ya se reconoció en el `MECH_COMMISSION` de la orden; este asiento solo mueve el dinero.

`GET /api/mechanics/:id/pending-payment` calcula lo pendiente por pagar a un mecánico como `Σ MECH_COMMISSION − Σ MECH_COMMISSION_PAY` (ambos filtrados por `mechanic_id`, columna agregada a `cash_flow_ledger` en rev. 22 — ver MIGRACIONES en `InformeLoQueFalta.txt`).

---

## Libro Auxiliar (Cash Flow Ledger)

### Tipos de movimiento

| Tipo | Impacto | Caja real | Descripción |
|:---|:---:|:---:|:---|
| `INC_GROSS` | CREDIT | ✅ | Ingreso por servicio (net_amount = 0 en crédito) |
| `TAX_IVA` | CREDIT | Informativo | IVA generado en la venta (ya dentro del cobro) |
| `RET_FUENT` | DEBIT | Informativo | ReteFuente practicada por el cliente |
| `RET_ICA` | DEBIT | Informativo | ReteICA practicada por el cliente |
| `RET_IVA` | DEBIT | Informativo | ReteIVA practicada por el cliente |
| `GW_FEE` | DEBIT | Informativo | Comisión de pasarela (Bold / Addi) |
| `GW_VAT` | DEBIT | Informativo | IVA sobre comisión de pasarela |
| `TAX_IVA_DEDUC` | DEBIT | Informativo | IVA descontable de compras |
| `RET_FUENT_COMPRA` / `RET_ICA_COMPRA` / `RET_IVA_COMPRA` | DEBIT | Informativo | Retenciones que practicamos a proveedores |
| `SUP_PAY` | DEBIT | ✅ | Pago a proveedor |
| `INV_COGS` | DEBIT | Informativo | Costo de repuestos consumidos, reconocido al liquidar (no al comprar) |
| `MECH_COMMISSION` | DEBIT | Informativo | Comisión de mecánico devengada en la orden, aún no pagada |
| `MECH_SALARY_PAY` | DEBIT | ✅ | Sueldo pagado a un mecánico de nómina fija |
| `MECH_COMMISSION_PAY` | DEBIT | ✅ | Comisión devengada liquidada en efectivo (no se cuenta como costo otra vez) |
| `TAX_GMF` | DEBIT | ✅ | GMF 4×1000 en pagos bancarios |
| `CARD_FEE` | DEBIT | ✅ | Costo de transacción con tarjeta |
| `NON_OP_INC` | CREDIT | ✅ | Ingreso no operacional (manual) |
| `VAT_REFUND` | CREDIT | ✅ | Devolución de IVA (`puc_code = '135520'`) — además disminuye `ivaLiability` |
| `MAN_INC` | CREDIT | ✅ | Ingreso manual registrado por el usuario |
| `MAN_EGR` | DEBIT | ✅ | Egreso manual registrado por el usuario |
| `REFERRAL` | CREDIT | ✅ | Ingreso por comisión de referido |
| `OPENING_BALANCE` | CREDIT/DEBIT | ✅ (aparte) | Saldo migrado del cuaderno/Excel previo — caja real pero reportado como "Capital Inicial", no como ingreso del período |

El detalle de la vista `/flujo-caja` (filtros, agrupación, tarjetas de Capital Inicial/Costos, descarga CSV) está en [BUSINESS_RULES.md — Flujo de Caja](BUSINESS_RULES.md#flujo-de-caja-flujo-caja). La cobertura de tests de este motor está en [TESTING.md](TESTING.md).
