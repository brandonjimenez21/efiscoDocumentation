# Motor Financiero

> Ver también: [README](README.md) · [BUSINESS_RULES](BUSINESS_RULES.md) · [TESTING](TESTING.md) · [API](API.md)

`backend/utils/financialEngine.js` — núcleo de cálculo inmutable. Constantes 2026: UVT = $50.318, umbral retenciones **servicios = 2 UVT ≈ $100.636, compras = 10 UVT ≈ $503.180** (Decreto 572/2025, revivido el 1-jul-2026 tras un vaivén de suspensiones del Consejo de Estado — sigue en litigio activo, ver nota de riesgo abajo).

> ⚠️ **Topes UVT en litigio, revisar periódicamente**: el Decreto 572/2025 bajó estos topes (servicios 4→2 UVT, compras 27→10 UVT); el Consejo de Estado lo suspendió el 8-may-2026 (volviendo a 4/27) y revocó esa suspensión el 2-jun-2026, reviviendo el decreto desde el 1-jul-2026 (2/10 de nuevo, los valores usados hoy como default). Ambos son solo el **default** — el contador puede sobreescribirlos por taller en `retention_threshold_services_uvt`/`retention_threshold_parts_uvt` (panel "Matriz de Retenciones (UVT)", con una alerta permanente ahí mismo recordando este litigio) — pero si no lo hace, el sistema usa estos valores, así que hay que mantenerlos alineados a la norma vigente.

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
```

`vat_percentage` es la tasa real configurada por el taller (no un 19% fijo — el default solo aplica si el campo no está configurado). El resultado expone esta misma tasa en `financial_summary.general_vat_rate_pct` para que el ítem de mano de obra de la factura Dataico (`billing.controller.js`) la reuse en vez de un 19 hardcodeado — antes divergían para talleres con IVA configurado ≠ 19% (bug corregido en rev. 25, ver `InformeLoQueFalta.txt`).

```
Si clientIsRetainer AND supera umbral UVT:
  ReteFuente = Base × retefuente_rate_declarante   (exento: Simple, Gran Contribuyente)
  ReteICA    = Base × (reteica_rate / 1000)        (exento: Simple, Gran Contribuyente)
  ReteIVA    = IVA  × reteiva_rate                 (exento: solo Gran Contribuyente)

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

Retención de la PASARELA sobre el giro (solo Régimen Ordinario, ver abajo):
  ReteRenta = Base × gateway_reterenta_rate  [default 1.5%]
  ReteIVA   = IVA  × reteiva_rate            [mismo 15% que la retención del cliente]
  ReteICA   = Base × (gateway_reteica_rate / 1000)  [default 0.4‰... 4/1000 = 0.4%]

Net Cash Inflow = Total − ReteFuente − ReteICA − ReteIVA − Comisión − IVA comisión
                  − ReteRenta pasarela − ReteIVA pasarela − ReteICA pasarela
```

**Bug real corregido en rev. 40 — preview de `ModalLiquidacion.jsx` no incluía el IVA sobre la comisión**: el motor resta `commission + commissionVat` de `netCashInflow` (expuesto como `gateway_impact.total_gateway_cost`), pero el preview del frontend solo calculaba `gatewayFee` (la comisión base) y la restaba de `net` — nunca calculaba el 19% de IVA sobre esa comisión. El cajero veía un "Total Neto a Recibir" más alto que el que `settleOrder` realmente liquidaba al cobrar por Bold/Addi. Corregido agregando `gatewayFeeVat = gatewayFee × 0.19` (mismo 19% fijo que usa el backend para esta línea, no `config.vat_percentage`) al cálculo de `net`; la línea "Comisión Plataforma" del desglose ahora muestra el costo total real (`gatewayFee + gatewayFeeVat`).

### Matriz de retenciones taller × cliente en ventas (rev. 38)

Reescrita dos veces sobre la misma celda:

- **rev. 34**: se descubrió que ReteICA usaba la misma guarda que ReteFuente/
  ReteIVA (`workshopCanWithhold = is_agente_retenedor_renta` del TALLER) —
  pregunta correcta para compras (quien retiene es el comprador), pero
  equivocada para ventas (quien retiene es el CLIENTE que paga). Se agregó
  una rama especial para Régimen Simple basada en el régimen del cliente,
  confirmada contra una imagen ("Matriz Maestra de Aplicación") que el
  usuario proveyó en ese momento.
- **rev. 38**: el usuario trajo un segundo set de imágenes
  (`Taller_cliente1/2.jpg`) que contradecía la anterior en varias celdas. Se
  investigó contra el Estatuto Tributario real (Decreto 1091 de 2020, Art.
  911 y 437-2 ET) antes de tocar código, y se confirmó que la matriz de
  rev. 34 estaba **incompleta**: el Art. 911 exime al Régimen Simple de
  ReteFuente **e ICA** ("mas no de retención a título de IVA" — Decreto
  1091/2020), sin importar el régimen del cliente — la distinción
  Ordinario/Gran Contribuyente del cliente que introdujo rev. 34 no existe
  en la norma real. Además, un Gran Contribuyente es **autorretenedor** —
  nunca le retienen nada al vender (el mismo principio que ya aplicaba
  correctamente cuando es proveedor, ver más abajo), algo que rev. 34 no
  contemplaba (trataba Gran Contribuyente igual que Ordinario). Por último,
  "No Responsable de IVA" no tiene ninguna exención de ReteFuente/ReteICA —
  esa es una condición sobre el IVA que cobra, no sobre renta/ICA.

| Régimen del Taller | ReteFuente | ReteICA | ReteIVA |
|:---|:---:|:---:|:---:|
| No Responsable de IVA | SÍ (si cliente agente retenedor + umbral) | SÍ | $0 natural (no genera IVA) |
| Régimen Simple | **NO** (Art. 911) | **NO** (Art. 911 + Decreto 1091/2020) | SÍ (si cliente agente retenedor + umbral) |
| Régimen Ordinario | SÍ | SÍ | SÍ |
| Gran Contribuyente | **NO** (autorretenedor) | **NO** (autorretenedor) | **NO** (autorretenedor) |

El régimen del **cliente** (Ordinario vs. Gran Contribuyente) ya NO cambia el
resultado en ninguna fila — solo importa si el cliente está marcado como
agente retenedor (`clientIsRetainer`). Por eso `classifyClientRegime` (rev.
34) se eliminó del motor: dejó de tener uso. Reproducido originalmente con
datos reales del taller `77f4ea4b-...` (rev. 34) y revisado exhaustivamente
con tests para las 4 filas × ambos valores de `clientIsRetainer` (rev. 38, ver
[TESTING.md](TESTING.md)). Mismo fix replicado en el preview de
`ModalLiquidacion.jsx` y en el simulador sin caller activo
`useFinancialStore.calculateLiveInvoice` (frontend) — ver también el bug del
IVA de la comisión de pasarela en ese mismo preview, corregido en rev. 40
(arriba, sección de Pasarela).

### Retención de la Pasarela sobre el giro al taller (rev. 38)

Bold/Addi actúan como agentes de retención **independientes del cliente**
sobre el giro que le hacen al taller (verificado contra el decreto de
retención en pagos electrónicos y el Art. 401-4 ET — Wompi lo confirma en su
propio centro de ayuda): 1.5% de renta sobre la base, 15% del IVA generado
(mismo `reteiva_rate` de arriba), e ICA propio de la actividad financiera
(`gateway_reteica_rate`, default 0.4‰ — **distinto** de la tarifa general de
servicios ~0.966‰ usada arriba para el ICA que practica el cliente).

| Régimen del Taller | ¿Sufre retención de pasarela? |
|:---|:---:|
| No Responsable de IVA | NO (Art. 401-4 — exención específica de quien no cobra IVA) |
| Régimen Simple | NO (Art. 911) |
| Régimen Ordinario | **SÍ**, las 3 (ReteRenta + ReteIVA + ReteICA) |
| Gran Contribuyente | NO (autorretenedor) |

Es independiente de `clientIsRetainer` (la practica la pasarela, no el
cliente), así que puede **coexistir** con la matriz de arriba si el cliente
además es agente retenedor y paga por Bold/Addi — ambas retenciones se restan
del `net_cash_inflow`. Se registra en el ledger como 3 tipos nuevos
(`GW_RET_RENTA`/`GW_RET_IVA`/`GW_RET_ICA`, informativos, cuenta PUC
`puc_pasarela_retencion_code` — ver tabla de tipos de movimiento abajo).
Configurable por el contador en `ContadorInventario.jsx` (sección "Tasas de
Pasarelas Externas").

**Venta a crédito**: `INC_GROSS.net_amount = 0` al liquidar (el ingreso operativo, base sin IVA, se reconoce completo en ese momento); ingresos reales de caja entran vía `payInstallment` (backend), que inserta un segundo asiento `INC_GROSS` por cada cuota cobrada con `net_amount = gross_amount` = valor de la cuota **con IVA incluido** — correcto para caja real (`bankBalance`), pero **no** debe sumarse otra vez al ingreso operativo: ya se reconoció al liquidar. El frontend (`FlujoCaja.jsx`) identifica esos asientos de cobro por su concepto fijo (`"Cuota N/M — ..."`, único texto que arma `payInstallment`) y los excluye del cálculo de Utilidad Neta (bug corregido: antes se sumaban también ahí, duplicando la venta e inflándola con IVA). Ver el flujo funcional completo en [BUSINESS_RULES.md — Panel de Cobros](BUSINESS_RULES.md#panel-de-cobros-cobros).

---

## 2. Liquidación de compras (`liquidateSupplierPurchase`)

```
Base          = total_gross_cost / (1 + 0.19)
IVA de Compra = total_gross_cost − Base

GMF (payment_method='banco' AND apply_4x1000=true):
  GMF = total_gross_cost × 0.004

Net Outflow = total_gross_cost − ReteFuente − ReteICA − GMF
```

Si proveedor NO es responsable de IVA (`is_responsible_vat=false`): el total facturado no trae IVA embebido, Base = total_gross_cost (sin dividir por 1.19). Si la factura trae un IVA explícito (OCR o corregido a mano, `invoiceVatAmount`), ese manda sobre el derivado.

### Matriz de retenciones taller-comprador × proveedor

Reescrita 3 veces sobre la misma tabla:

- Antes de esta sesión: todo-o-nada (las 3 retenciones aplicaban juntas o
  ninguna, según solo si ambos eran Gran Contribuyente — `Correcion.txt:40`).
- Fase 2 de la sesión: se separó en `appliesReteFuente`/`appliesReteIcaReteIva`,
  con el criterio "el proveedor Gran Contribuyente nunca se retiene
  (autorretenedor)", y se permitió a un taller Gran Contribuyente retener
  ICA+IVA (sin ReteFuente) a un proveedor Régimen Simple.
- **rev. 38**: investigado contra Decreto 1091/2020 y Art. 437-2 ET, se
  encontraron 2 problemas más finos: (1) ReteICA y ReteIVA NO comparten la
  misma exención — el Art. 911 protege al proveedor Régimen Simple de ReteICA
  **sin importar el régimen del comprador** (la excepción que Fase 2 le daba
  a Gran Contribuyente no existe en la norma real), y (2) un comprador
  simplemente Ordinario NO es agente retenedor de IVA frente a OTRO Ordinario
  (Art. 437-2: esa calidad es de los Grandes Contribuyentes, "sean o no
  responsables del IVA") — el código aplicaba ReteIVA en ese caso por error,
  bundleada con ReteICA. Ahora son 3 flags totalmente independientes:

| Taller comprador ↓ / Proveedor → | Ordinario / Persona Natural | Régimen Simple | Gran Contribuyente |
|:---|:---:|:---:|:---:|
| No responsable / Simple | Ninguna | Ninguna | Ninguna |
| Ordinario | ReteFuente + ReteICA (si supera el tope UVT de compras). ReteIVA **NO** ("entre ordinarios") | ReteFuente/ReteICA **NO** (Art. 911). ReteIVA **SÍ**, obligatoria (Art. 437-2, retención especial a proveedores RST) | Ninguna (autorretenedor) |
| Gran Contribuyente | ReteFuente + ReteICA + ReteIVA (comprador Gran Contribuyente sí es agente retenedor de IVA, Art. 437-2) | ReteFuente/ReteICA **NO** (Art. 911). ReteIVA **SÍ** | Ninguna (autorretenedor) |

```
ReteFuente: declarante → Base × (supplier_retefuente_rate / 100)     [default 2.5%]
            no declarante → Base × (retefuente_rate_no_declarante / 100) [default 3.5%]
ReteICA:   Base × (reteica_rate_supplier / 1000 si el proveedor tiene tasa propia,
                    si no config.supplier_reteica_rate / 1000)       [default 9.66‰]
            — misma exención que ReteFuente (Art. 911 protege ambas)
ReteIVA:   IVA × (config.reteiva_rate / 100)                         [default 15%]
            — aplica si el comprador es Gran Contribuyente O el proveedor es
              Régimen Simple (o ambos); NUNCA si el proveedor es Gran
              Contribuyente (autorretenedor)
```

`supplier_retefuente_rate` y `supplier_reteica_rate` (contador → `ContadorInventario.jsx`) se leen `/100`/`/1000` igual que el resto de tasas del sistema — antes `supplier_retefuente_rate` se leía como fracción cruda, así que guardar "2.5" (como las demás columnas) habría dado una retención del 250% (bug corregido en rev. 25). `reteiva_rate` en compras usaba un `RETEIVA_MULTIPLIER` fijo (15%) ignorando la tasa configurable del contador — bug corregido en rev. 34, ahora se lee igual que en ventas (`config.reteiva_rate`). Todas las tasas configurables de este módulo (incluidas `retefuente_rate_declarante`, `reteica_rate`, `gateway_addi_rate`) comparan contra `null`/`undefined` explícito, no `|| default` — un 0% real configurado por el contador se respeta en vez de caer al default.

> Nota de simplificación: la retención especial de IVA a proveedores del Régimen Simple tiene, en la norma real, sus propios topes UVT (27 UVT bienes / 4 UVT servicios, fijados por el Art. 437-2 y no afectados por el Decreto 572/2025) — EFISCO reutiliza por simplicidad el mismo `retention_threshold_parts_uvt` configurable que ya usa ReteFuente/ReteICA en vez de una columna separada. Si esto genera diferencias prácticas relevantes, es candidato a una columna dedicada en una futura revisión.

**Ojo — Ordinario y Gran Contribuyente (taller comprador) comparten exactamente los mismos 3 booleans** (`is_responsable_iva`, `is_regimen_simple`, `is_agente_retenedor_renta` — ver [BUSINESS_RULES.md](BUSINESS_RULES.md#configuración-del-taller-config)); la fila "Gran Contribuyente" de esta matriz se decide con `config.fiscal_regime === 'gran_contribuyente'` (columna de texto aparte), no con esos booleans. Bug corregido en rev. 35: `POST /api/workshop/finance-settings` escribía esos 3 booleans directamente sin sincronizar `fiscal_regime` (a diferencia de `PUT /api/workshop/:id`, que sí lo hace vía un `REGIME_MAP` compartido) — un taller configurado como Gran Contribuyente por esa vía podía quedar con `fiscal_regime` desactualizado y perder la fila especial (ReteICA+ReteIVA a un proveedor Régimen Simple). Ambos endpoints ahora usan el mismo `REGIME_MAP` (hoisted en `workshop.controller.js`) como única fuente de verdad.

### Compra a plazos — Cuentas por Pagar a proveedores

Espejo exacto de la venta a crédito de clientes (ver más abajo): al registrar una compra con `payment_mode:'credito'` y `num_installments>1`, la cabecera (`supplier_purchases`) queda `status:'pendiente'`, se reparte el pago neto en `supplier_installments` (mismo `installments.js:splitIntoInstallments`, mismas cuotas mensuales desde `first_payment_date`), y el asiento `SUP_PAY` inicial lleva `net_amount:0` (reconocimiento del costo, sin salida de caja aún — el costo/gasto ya se causa con el monto bruto). Cada cuota pagada (`PATCH /api/providers/installments/:id/pay`) inserta un nuevo `SUP_PAY` con `net_amount = amount` de la cuota, la salida de caja real. `SUP_PAY` no está en `NON_CASH_TYPES`, así que `cashValue`/`calculateGlobalHealth` excluyen automáticamente los $0 sin ningún cambio adicional al motor — mismo mecanismo que ya usa `INC_GROSS` para crédito a clientes.

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

El **Capital Inicial** (`OPENING_BALANCE`, el saldo migrado del cuaderno/Excel previo del taller) SÍ es caja real (`isCashEntry` lo cuenta), pero se reporta aparte de "Total Ingresos"/"Total Egresos" vía `isOpeningBalance`/`openingBalanceValue` — no es una venta ni un gasto del período, es el punto de partida. Mismo criterio espejado en `frontend/src/utils/ledgerLabels.js`. Bug real corregido en rev. 39: `finance.controller.js:setOpeningBalance` registraba este asiento contra `puc_otros_ingresos_code` (una cuenta de INGRESO) — contablemente incorrecto, ya que es efectivo que ya existía en el banco, no una venta del período. Ahora usa `puc_bancos_code` (PUC clase 11 "Disponible", default `111005`), configurable por el contador en `ContadorPanel.jsx` (bloque "Control Financiero").

**`getCashFlow` (`/flujo-caja`) reusa este mismo motor para su posición acumulada** (bug corregido): antes calculaba "Saldo Real"/"IVA Pendiente DIAN" solo con los asientos dentro de `[from,to]` — con el rango por defecto (mes en curso), cualquier taller con más de un mes de uso veía un "Capital Inicial" de $0 y un saldo/IVA que ignoraban todo lo acumulado antes, divergiendo de `/finanzas` (que sí usa el histórico completo). Ahora `getCashFlow` corre `calculateGlobalHealth` dos veces: una sobre los asientos anteriores a `from` (`summary.capital_inicial` = su `bankBalance`, `summary.prior_iva_liability` = su `ivaLiability` — el punto de partida real del rango) y otra sobre el histórico completo hasta `to` (`summary.bank_balance`/`summary.iva_pendiente`/`summary.net_balance` — la posición acumulada real a esa fecha, sin clampear el IVA a 0, igual que aquí). `FlujoCaja.jsx` arranca su tabla diaria y sus tarjetas de resumen desde ese saldo/IVA reales en vez de $0.

---

## Caja real vs. informativo/devengo — por qué existe la distinción

Bug de fondo reportado por el usuario: el Flujo de Caja mezclaba "cuánto entró en total" con "cuánto hay realmente en el banco", y no reconocía el costo de un repuesto/mano de obra en el momento correcto. La causa raíz era que `cash_flow_ledger` solo distinguía `CREDIT`/`DEBIT`, pero varios tipos de asiento son **informativos**: registran a dónde se fue la plata (impuestos, retenciones, comisiones, costo de lo vendido) cuyo efecto de caja **ya está incluido** dentro de otro asiento (`net_amount` de `INC_GROSS`/`SUP_PAY`) — sumarlos aparte como entrada/salida los cuenta dos veces.

`NON_CASH_TYPES` (backend `financialEngine.js`, espejado en frontend `ledgerLabels.js`): `TAX_IVA`, `RET_FUENT`, `RET_IVA`, `RET_ICA`, `GW_FEE`, `GW_VAT`, `GW_RET_RENTA`, `GW_RET_IVA`, `GW_RET_ICA`, `TAX_IVA_DEDUC`, `RET_FUENT_COMPRA`, `RET_ICA_COMPRA`, `RET_IVA_COMPRA`, `INV_COGS`, `MECH_COMMISSION`.

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
| `INC_GROSS` | CREDIT | ✅ | Ingreso por servicio (`net_amount = 0` en crédito al liquidar; el cobro posterior de cada cuota genera otro asiento del mismo tipo, ver [venta a crédito](#1-liquidación-de-servicios-liquidateclientinvoice)) |
| `TAX_IVA` | CREDIT | Informativo | IVA generado en la venta (ya dentro del cobro) |
| `RET_FUENT` | DEBIT | Informativo | ReteFuente practicada por el cliente |
| `RET_ICA` | DEBIT | Informativo | ReteICA practicada por el cliente |
| `RET_IVA` | DEBIT | Informativo | ReteIVA practicada por el cliente |
| `GW_FEE` | DEBIT | Informativo | Comisión de pasarela (Bold / Addi) |
| `GW_VAT` | DEBIT | Informativo | IVA sobre comisión de pasarela |
| `GW_RET_RENTA` / `GW_RET_IVA` / `GW_RET_ICA` | DEBIT | Informativo | Retención que practica la PASARELA sobre el giro (rev. 38) — solo Régimen Ordinario, independiente de las retenciones del cliente |
| `TAX_IVA_DEDUC` | DEBIT | Informativo | IVA descontable de compras |
| `RET_FUENT_COMPRA` / `RET_ICA_COMPRA` / `RET_IVA_COMPRA` | DEBIT | Informativo | Retenciones que practicamos a proveedores |
| `SUP_PAY` | DEBIT | ✅ | Pago a proveedor (`net_amount = 0` en compra a plazos al registrarla; cada cuota pagada genera otro asiento del mismo tipo, ver [Compra a plazos](#2-liquidación-de-compras-liquidatesupplierpurchase)) |
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
