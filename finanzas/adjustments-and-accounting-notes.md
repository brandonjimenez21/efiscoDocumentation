# Devoluciones, Ajustes, Notas Contables y Bolsas de Gasto

> Implementado el 2026-09-27 a pedido explícito del usuario: Fases 1–6, 8 y 9 terminadas y verificadas en vivo; **Fase 7 pausada** (ver al final). Segunda ronda el 2026-09-28 ("Puntos a corregir notas contables", fases A–D abajo), completa y verificada en vivo. Queda **una pregunta pendiente** (ver [Pendientes](#pendientes)). Este documento es la referencia completa del diseño; los documentos generales ([ledger-and-accounting.md](./ledger-and-accounting.md), api-reference.md, etc.) solo enlazan acá.

| Fase | Qué | Estado |
|:---:|:---|:---|
| 1 | Esquema: tablas, columnas, triggers de sucursal y mes cerrado, prefijos AJV/NTC | ✅ Hecha |
| 2 | Motor puro del contrasiento (`saleAdjustment.js`) + tipos nuevos en el motor financiero y su espejo del frontend | ✅ Hecha |
| 3 | Endpoints de ajustes + funciones atómicas de aplicar/anular (storno) | ✅ Hecha |
| 4 | Pantalla: botón en el Historial, modal, tarjeta en la Ficha de Servicio, Nota PDF | ✅ Hecha |
| 5 | Bolsas de gasto con cuenta PUC + "Registrar Gasto" con Caja/Bancos | ✅ Hecha |
| 6 | Notas Contables del contador (crear/anular, reclasificar desde un comprobante) | ✅ Hecha |
| 7 | Nota Crédito electrónica DIAN vía Dataico | ⏸️ **Pausada** |
| 8 | Libro Mayor: Tipo de Asiento, Documento Afectado, Observaciones, storno en la columna contraria, hoja "Ajustes y Notas" | ✅ Hecha |
| 9 | Documentación | ✅ Hecha |
| A | (2026-09-28) Modal de devolución: M.O. por servicio y mecánico, tipo de descuento, leyenda de impacto, "Anular Factura Completa", comisión ya pagada queda en contra | ✅ Hecha |
| B | (2026-09-28) "Corregir Stock" con motivo merma o daño (531095 / 1435) | ✅ Hecha |
| C | (2026-09-28) Mi Equipo solo consulta: comisión desde el Libro Mayor, pago sin monto manual, saldo en contra visible | ✅ Hecha |
| D | (2026-09-28) Contador: Finanzas/Flujo en solo lectura, N° de comprobante, buscador, numeración visible (NC, RC/CE), Excel con Tipo de Comprobante y "Mayor por Cuenta" | ✅ Hecha |

**Premisa del producto**: la administradora del taller opera de forma ágil **sin manejar códigos PUC**. El contador es quien mapea las cuentas y audita el Libro Mayor. Todo lo de abajo respeta esa división: la administradora devuelve, rebaja o registra gastos eligiendo opciones simples; el contador asigna cuentas, registra notas contables y deja observaciones.

---

## Decisiones confirmadas con el usuario (no revertir sin preguntar)

| # | Decisión | Dónde se aplica |
|:---:|:---|:---|
| 1 | Una devolución sobre una **factura electrónica (CUFE)** exige Nota Crédito electrónica DIAN. Esa emisión es una fase aparte (**Fase 7, PAUSADA** — ver [Fase 7 pausada](#fase-7--nota-crédito-electrónica-dian-pausada)). Mientras tanto el ajuste se registra con `dian_status='pendiente'` y un aviso visible. Los comprobantes sin CUFE se ajustan normalmente. | `saleAdjustment.js` (`hasCufe`), `SaleAdjustmentModal.jsx`, `SaleAdjustmentsPanel.jsx`, PDF |
| 2 | **Venta a crédito**: lo devuelto se descuenta primero de las **cuotas pendientes, de la última hacia atrás**, sin bajar ninguna cuota por debajo de lo ya abonado. Si lo devuelto supera lo pendiente, el excedente sale en dinero y el usuario elige Caja o Bancos. | `planInstallmentReduction` |
| 3 | **Rebaja de mano de obra**: se reversa en proporción la comisión del mecánico del servicio rebajado, **aunque ya se le haya pagado** (cambio del 2026-09-28, antes solo se reversaba lo no pagado). Lo que exceda su saldo pendiente queda **en contra suyo**: el saldo pendiente (`pendingCommissionFromRows`) queda negativo y se descuenta de su próxima liquidación; el ajuste lo avisa y lo devuelve en `header.commission_debts`. Descontar de nómina exige autorización escrita del trabajador (CST art. 149). | `buildSaleAdjustment` paso 5, `utils/mechanicCommission.js` |
| 4 | Las **notas contables** (`ACC_NOTE`) son informativas para el dueño (`NON_CASH_TYPES`, en los dos espejos): afectan el Libro Mayor pero **nunca** la Utilidad ni la Caja que ve el dueño. Excepción natural: el saldo por cuenta de Caja/Bancos (`calculateCashAccountBalance`) sí la refleja, porque suma por `puc_code`. | `financialEngine.js`, `ledgerLabels.js` |
| 5 | Ajustes y notas se fechan **hoy o en un mes que no esté archivado** en `monthly_ledger_books`. Nunca dentro de un mes cerrado. En la práctica solo el mes en curso está abierto (`getMonthlyBooks` archiva perezosamente todo mes terminado). | Trigger `trg_ledger_adjustment_note_guard` |
| — | Checkbox **"¿El repuesto regresa al inventario vendible?"** por repuesto devuelto (`restock`). Sin marcar = repuesto defectuoso: no suma al stock y su costo va a pérdida. | `SaleAdjustmentModal.jsx` |
| — | **Aislamiento por sucursal** en las tablas nuevas (pedido como "RLS por `workshop_id`"). Se resolvió con RLS deny-by-default + triggers de integridad, ver [Seguridad](#seguridad-y-aislamiento-por-sucursal). | Migración de la Fase 1 |
| 6 | (2026-09-28) **Numeración visible**: 4 dígitos (`FV-0038`) solo como máscara; **AJV → NC**; **CC según dirección: entrada → RC, salida → CE** (un gasto nunca se muestra como Recibo de Caja); NTC igual. Los gastos nuevos se numeran directo en la serie **CE**. | `utils/documentDisplay.js` (2 espejos), `payFixedCost`, `addManualMovement` |
| 7 | (2026-09-28) **Contador en Finanzas y Flujo de Caja: 100 % solo lectura**, blindado en el backend y filtrado por sede de su grupo. | `financeReadWorkshopId`, `useContadorScopeStore` |
| 8 | (2026-09-28) **Merma o daño** en Corregir Stock = pérdida 531095 contra 1435 (no Capital); Corregir Stock nunca lleva Caja. | `subtractStock`, `INV_SHRINKAGE` |
| 9 | (2026-09-28) **Mi Equipo no ajusta comisiones a mano**: todo cambio sale de las facturas; el pago de comisión es el pendiente exacto del Libro Mayor. | `payMechanic`, `commissionBreakdownFromLedger` |

---

## Migraciones (las corre el usuario en el SQL Editor)

| Archivo | Qué hace |
|:---|:---|
| `2026-09-27_sales_adjustments_accounting_notes.sql` | Tablas `sales_voucher_adjustments`, `sales_voucher_adjustment_items`, `accounting_notes`, `ledger_observations`, `expense_categories` (+ semilla de 6 bolsas por sede y trigger para sedes nuevas); columnas `cash_flow_ledger.sales_adjustment_id/accounting_note_id/reference_document_number`; `inventory_transactions.sales_adjustment_id` y tipos `sale_return`/`sale_return_defective`; cuentas `puc_devoluciones_ventas_code` (4175) y `puc_perdida_devoluciones_code` (531095); prefijos `AJV`/`NTC` en `claim_document_number`; triggers de aislamiento y de mes cerrado |
| `2026-09-27_sales_adjustments_apply_void.sql` | `installment_plan` + funciones atómicas `apply_sales_adjustment` / `void_sales_adjustment` |
| `2026-09-27_accounting_notes_create_void.sql` | Funciones atómicas `create_accounting_note` / `void_accounting_note` |
| `2026-09-28_ce_sequence_after_cc_egresos.sql` | Adelanta `doc_seq_ce` después de los CC de salida viejos (gastos ahora en la serie CE) |
| `2026-09-28_inventory_shrinkage.sql` | Tipo `shrinkage` en el Kardex + columna `note` (merma en Corregir Stock, ver inventory-and-providers.md) |

---

### Qué se puede ajustar

| Tipo (`kind`) | Qué es | Cuenta del contra-ingreso |
|:---|:---|:---|
| `part_return` | Devolución de un repuesto vendido (cantidad entera, hasta lo que quede sin devolver) | `puc_devoluciones_ventas_code` (**4175**) |
| `labor_adjustment` | Rebaja del valor de la mano de obra (antes de IVA, ya neto del descuento de la venta). Opcionalmente sobre un servicio puntual | `puc_income_code` (**415505**) — es una rebaja de precio, no devolución de mercancía |
| `post_discount` | Descuento posterior a la venta (antes de IVA) | `puc_descuento_ventas_code` (**417595**), igual que el descuento por regateo |

Un mismo ajuste puede combinar los tres. El comprobante original (`FV-`) **nunca se modifica**; el ajuste tiene su propio número `AJV-`.

### Motor puro: `backend/utils/saleAdjustment.js`

Se parte de las **líneas originales del ledger** (el `FV-` que insertó `settleOrder`), no se vuelve a liquidar con `liquidateClientInvoice`: si el taller cambió después sus tasas o cuentas, el contrasiento debe reversar exactamente lo que se causó ese día.

- **`summarizeOriginalSale(lines, soldItems)`**: separa mano de obra y repuestos por el **concepto** fijo de los `INC_GROSS` (no por `puc_code`, que el contador pudo igualar), calcula `discountRatio = base / subtotalBruto`, el IVA total, y deriva la **tasa real de IVA de la mano de obra** = (IVA total − IVA de los repuestos a su tasa congelada) / mano de obra neta. Ignora las cuotas cobradas después (`INC_GROSS` "Cuota N/M").
- **Repuesto**: `base = precio × cantidad × discountRatio`; IVA a la tasa del ítem (`vat_percentage` congelado), 0 si la venta no llevó IVA.
- **Mano de obra**: IVA a la tasa derivada. **Descuento posterior**: IVA a la tasa efectiva de la venta (IVA / base).
- **Topes**: no se devuelven más unidades de las vendidas (contando ajustes vigentes y filas repetidas del mismo ajuste); la mano de obra no baja de 0; el total no supera la base que queda.
- **Retenciones del cliente** (`RET_FUENT`/`RET_ICA` sobre la base, `RET_IVA` sobre el IVA): se reversan **en proporción** contra su misma cuenta de anticipo. El cliente recibe **base + IVA − lo que él había retenido**.
- **Pasarela**: comisión y retenciones de Bold/Addi **no** se reversan (el banco no las devuelve).
- **Costo de ventas**: `restock=true` → DEBIT Inventario (1435) / CREDIT Costo (6135) al costo de la venta original (`unit_cost_at_time`, no el actual). `restock=false` → DEBIT Pérdida (`puc_perdida_devoluciones_code`, 531095) / CREDIT Costo.
- **Crédito**: `planInstallmentReduction` rebaja cuotas de la última hacia atrás (decisión 2) y devuelve el plan `{id, reduce_by, new_amount, settled}`.
- **Validación final**: el contrasiento cuadra al centavo o el motor lanza error (antes de llegar a la base).

### Tipos de asiento nuevos

| Tipo | Impacto | Caja real | Uso |
|:---|:---:|:---:|:---|
| `SALE_RETURN` | DEBIT | ✅ por `net_amount` | Contra-ingreso de cada ítem. `net_amount` = la parte reembolsada en dinero (0 si todo fue a cartera), así `calculateGlobalHealth` resta de caja solo lo que salió. En `COST_TYPES` (resta Utilidad), mismo criterio que `SALES_DISCOUNT` |
| `TAX_IVA_REV` | DEBIT | Informativo | Reversa IVA generado. `calculateOperatingProfit` (`ivaPendiente`) y `calculateGlobalHealth` lo restan |
| `RET_FUENT_REV` / `RET_IVA_REV` / `RET_ICA_REV` | CREDIT | Informativo | Reversan los anticipos de retención. `RET_IVA_REV` suma de vuelta al IVA pendiente |
| `INV_COGS_REV` | DEBIT + CREDIT | Informativo | DEBIT 1435 (o nada, si es defectuoso) / CREDIT 6135. En `COST_REVERSAL_TYPES`: su lado **CREDIT resta de costos** |
| `INV_RETURN_LOSS` | DEBIT | Informativo | Costo del repuesto defectuoso a pérdida. En `COST_TYPES` |
| `MECH_COMMISSION_REV` | DEBIT + CREDIT | Informativo | DEBIT 233525 / CREDIT gasto de comisión. En `COST_REVERSAL_TYPES`. Resta del saldo pendiente del mecánico |
| `AR_REDUCTION` | CREDIT | Informativo | Rebaja de Cartera (130505) |
| `CASH_PAYMENT` (existente) | CREDIT | Informativo | Reembolso desde Caja o Bancos (etiqueta la cuenta real) |
| `ACC_NOTE` | DEBIT/CREDIT | Informativo | Línea de Nota Contable (decisión 4) |

Todos están espejados en `frontend/src/utils/ledgerLabels.js` (`TYPE_LABELS`, `NON_CASH_TYPES`, `COST_TYPES`, `COST_REVERSAL_TYPES`, `signedCostValue`, `ivaLiabilityValue`), que usan Flujo de Caja y Finanzas. **Cualquier tipo nuevo futuro se agrega en los dos lados.**

### Persistencia atómica

`POST /api/billing/orders/:id/adjustments` llama a `apply_sales_adjustment`, que en **una sola transacción**: bloquea la orden (`FOR UPDATE`), vuelve a validar que no se devuelvan más unidades ni más que el total facturado (carrera entre dos usuarios), valida el cuadre, reclama `AJV-N`, inserta encabezado + ítems + líneas del ledger + Kardex, suma al stock solo lo `restock` y rebaja las cuotas en relativo (si una cuota cambió mientras tanto, aborta todo). Errores de negocio con prefijo `AJV:` → 409 legible.

### Anulación = STORNO

`void_sales_adjustment` (y `void_accounting_note`) **copian las líneas con montos negativos, mismo `type` y mismo `impact`**, bajo un número nuevo. Así cada consumidor que filtra por `type`/`impact` (Utilidad, IVA, Caja, comisión pendiente, Equilibrio) se cancela solo, sin reglas especiales. La anulación además saca del stock lo que había vuelto (se rechaza si ese repuesto ya se volvió a usar), devuelve a cada cuota exactamente lo rebajado (`installment_plan`) y marca el original como `anulado`. Una anulación no se anula; un ajuste con Nota Crédito DIAN emitida no se anula desde acá.

Dónde se "desnegativiza" el storno para mostrarlo:
- **Listas de movimientos** (`ledgerLabels.js:getEntryFormatting`, usada por Flujo de Caja, Finanzas y Libros Mensuales): una línea negativa se muestra como su lado contrario con el valor positivo (antes salía "−$-1.000", bug encontrado en la prueba en vivo).
- **Libro Mayor Excel**: el monto va en la **columna contraria**, positivo (SIIGO/Alegra no aceptan negativos).

### Numeración

`documentNumbering.getOrderDocumentNumber` ahora filtra `document_number LIKE 'FV-%'`: las líneas de un ajuste llevan `work_order_id` (para resolver el tercero en el Libro Mayor) y sin ese filtro el cobro de una cuota podía heredar el número `AJV-` en vez del `FV-` de la venta.

### Otros consumidores del ledger actualizados

- **Comisión pendiente del mecánico**: `utils/mechanicCommission.js:pendingCommissionFromRows` = ganada − reversada (`MECH_COMMISSION_REV`) − pagada, usada por `getPendingPayment` y por los ajustes.
- **Panel de Equilibrio** (`getBreakevenPanel`): resta `SALE_RETURN` del ingreso del mes y el CREDIT de `INV_COGS_REV` del costo de repuestos.
- El Dashboard (función SQL agregada) y los libros mensuales pasan por el motor genérico: no necesitaron cambios.
- **Mi Equipo solo consulta** (2026-09-28): la "comisión del mes" (`getMechanicMetrics`) y el desglose (`getMechanicCommissionBreakdown`, `mechanicCommission.js:commissionBreakdownFromLedger`) salen del ledger: lo causado en cada venta (repartido entre los servicios del mecánico) y, en negativo, cada `MECH_COMMISSION_REV` con su comprobante y la factura afectada. `payMechanic` paga **exactamente** el pendiente (`pay_commission: true`); un `commission_amount` distinto se rechaza (409) y con saldo en contra no hay comisión que pagar (se descuenta de las próximas comisiones; el sueldo sí se paga aparte). El % y el sueldo configurados siguen editables (son la tarifa, no un ajuste).

### Pantalla (Fase 4)

- **Historial de Órdenes** (`/ordenes`): ícono de devolución por fila (dueño/admin) → `SaleAdjustmentModal.jsx`.
- **Ficha de Servicio**: tarjeta "Devoluciones y Ajustes" (`SaleAdjustmentsPanel.jsx`) con historial, "Nota PDF", "Anular" (motivo obligatorio, confirmación en la misma tarjeta, sin `window.confirm`) y "Generar Devolución / Ajuste". El contador la ve en solo lectura.
- **Modal**: 3 pestañas (Repuestos con +/− y checkbox de inventario vendible, Mano de obra, Descuento), vista previa en vivo contra `POST .../preview`, Caja/Bancos solo si hay reembolso en dinero, "Ver asiento contable" plegado (para el contador), motivo (5 opciones) + explicación obligatorios.
- **Nota Crédito en PDF** (`frontend/src/utils/saleAdjustmentPdf.js`, html2pdf, todo pasa por `escapeHtml`): membrete del taller (`/adjustable` devuelve `workshop` con nombre/NIT/ciudad/teléfono, **nunca** credenciales Dataico), cliente, `FV-` afectado, detalle, totales, cómo se devolvió el dinero y firmas. **Desde 2026-09-29** (pedido explícito del usuario, "debe coincidir" con el número NC) el PDF se titula "Nota Crédito" / "Anulación de Nota Crédito", el archivo es `Nota_Credito_NC-0001.pdf` y el pie dice "Nota crédito interna"; el modal ("Nota crédito NC-0003 registrada"), la tarjeta ("Nota crédito PDF") y el mensaje de anulación del backend (que antes mostraba el número guardado `AJV-2`) usan el mismo nombre. Sigue siendo un soporte interno: la Nota Crédito **electrónica** DIAN es la Fase 7, pausada.

---

### Rediseño del modal (2026-09-28, pedido del usuario "Puntos a corregir notas contables")

- **M.O. por servicio**: `/adjustable` devuelve por servicio `labor_net`, `labor_remaining`, `mechanic_name`, `commission`. La pestaña Mano de obra es una tabla con un campo de rebaja por servicio. Cada servicio pesa según su `labor_allocation` (`serviceLaborShares`); el motor valida un **tope por servicio** además del global. Las rebajas anteriores se reconstruyen en orden (`previous.laborItems`, `distributeLabor`): una sin servicio se reparte según lo que le **queda** a cada servicio, así nunca deja uno en negativo.
- **Tipo de descuento**: "Descuento Comercial Taller" = `post_discount` (417595, no toca comisión). "Descuento sobre Servicio" = `labor_adjustment` con `as_discount: true` (417595, rebaja la M.O. y la comisión a prorrata). No hizo falta tipo ni columna nueva: el ítem queda como `labor_adjustment` con descripción "Descuento sobre servicio".
- **Bug real del buscador de comprobantes (2026-09-29, reportado desde un taller de prueba)**: al elegir NC-0001 en el buscador (Notas Contables / Observaciones) salía "No se encontró el comprobante AJV-0001 en esta sede". El buscador entrega el número GUARDADO (`AJV-1`) y `utils/documentDisplay.js:matchesParsed` solo lo comparaba contra el nombre de pantalla (`NC`). Ahora acepta los dos (espejo en el frontend); afectaba también ligar una NC a una nota contable u observación. El mensaje de "no encontrado" muestra el nombre de pantalla. **Aclaración de producto**: la sección "Notas Contables" del Módulo Contador lista solo NTC (las del contador); una NC es una devolución hecha desde Órdenes (dueño/admin) y aparece en Flujo de Caja, Finanzas, el Libro Mayor, el buscador y la ficha de la orden, no en esa lista.
- **Servicio con varios mecánicos (2026-09-29)**: cada mecánico pesa `labor_allocation / N` en la línea (la misma parte sobre la que `settleOrder` le causó comisión), así que rebajar una línea compartida le quita a cada uno su proporción. En el modal, cada fila de M.O. lista a sus mecánicos con lo que **ganó** cada uno en ese servicio y, mientras se escribe la rebaja, lo que **se le descuenta** (en rojo); el resumen de abajo dice "Total de comisión que se descuenta en Mi Equipo". Antes la fila decía solo "comisión $X" y se confundía con el descuento.
- **Órdenes viejas** con la comisión en el ledger pero sin `mechanic_id` en los servicios: esa comisión se reversa en proporción a toda la M.O. rebajada (encontrado verificando con datos reales).
- **Leyenda de impacto** arriba del modal: "Impacto: Salida de dinero de Caja/Bancos" (contado o crédito ya pagado) y/o "Impacto: Reducción de saldo en el Panel de Cobros" (crédito con saldo); si la vista previa trae las dos cosas, se muestran ambas.
- **Anular Factura Completa**: el cuerpo lleva `full_reversal: { restock }` en vez de `items`; el backend arma los ítems con `buildFullReversalItems` (todas las unidades restantes, la M.O. restante de cada servicio y un `post_discount` por la base que sobre) y pasa por el mismo motor y el mismo RPC. Un solo `AJV-`, sin migración. Verificado en solo lectura con 10 órdenes reales: 20 vistas previas cuadradas.

## 2. Bolsas de gasto con cuenta PUC (Fase 5)

Antes, `payFixedCost` tenía 3 categorías con la cuenta fija en el código (512005 / 513505 / 519595). Ahora cada sede tiene sus bolsas en `expense_categories`:

| Bolsa sembrada (`code`) | Cuenta inicial |
|:---|:---|
| Arriendo (`arriendo`) | 512005 |
| Servicios Públicos (`servicios`) | 513505 |
| Aseo y Cafetería (`aseo`) | 519525 |
| Insumos del Taller (`insumos`) | 519595 |
| Herramientas Menores (`herramientas`) | 519595 |
| Otros Gastos Generales (`otros`) | 519595 |

- **Dueño/admin/contador** crean, renombran y desactivan bolsas (nunca se borran: los gastos ya registrados las usan). Una bolsa nueva arranca en 519595 hasta que el contador la mapee.
- **Solo el contador** cambia la cuenta, y solo a clase 5, 6 o 7.
- **Registrar un gasto**: `POST /api/finance/fixed-costs/pay` (alias `POST /api/finance/expenses`), ahora con `category_id` o el `category` (código) de siempre, y `cash_channel` (`efectivo`/`banco`, default banco). Lo pueden hacer dueño y admin. DEBIT `MAN_EGR` a la cuenta de la bolsa + CREDIT `CASH_PAYMENT` a Caja o Bancos.
- **UI**: `ExpensePaymentModal.jsx` ("Registrar Gasto") reemplazó los dos modales duplicados de "Pagar Costo Fijo" (Finanzas y Ajustes). `ExpenseCategoriesManager.jsx` vive en Ajustes → Datos del Taller (dueño, sin columna de cuenta) y en el panel del contador (con columna de cuenta).
- **Orden estable**: las 6 bolsas sembradas comparten `created_at`; se ordenan por un orden fijo en `expenseCategories.controller.js:sortCategories` (bug real: la bolsa editada saltaba al final de la lista).
- **Gotcha**: `Config.jsx` y `ContadorInventario.jsx` tienen **su propia lista blanca** de campos PUC al cargar. Un campo nuevo en `ContadorPanel.jsx` sale vacío (y se guardaría vacío) si no se agrega ahí también — pasó con 4175/531095.

---

## 3. Notas Contables del contador (Fase 6)

Ajustes o reclasificaciones sobre **cualquier** transacción del periodo, sin tocar el comprobante del cliente. Comprobante `NTC-N`, encabezado en `accounting_notes`, líneas `ACC_NOTE` en `cash_flow_ledger`.

- **Solo el contador** crea y anula; el dueño lee. La nota puede ir a cualquier sede del grupo del contador, **nunca** a "todas" ni a un taller ajeno.
- `create_accounting_note` valida en Postgres: ≥ 2 líneas, cuentas numéricas, montos > 0, **cuadre exacto**, tercero (`work_order_id`/`related_purchase_id`) de la misma sede, y (trigger) mes no cerrado. Todo en una transacción: si falla, ni siquiera se consume el número `NTC`.
- **Fecha**: hoy (hora real) o un día anterior del mes abierto (queda a mediodía de ese día en Bogotá). Nunca futura.
- **UI** (`AccountingNotesSection.jsx` + `AccountingNoteModal.jsx`, en Reportes & Auditoría del panel del contador, siguiendo el selector de sede): líneas con cuenta (sugerencias de `GET /api/finance/puc-accounts` con nombre), detalle, débito, crédito; totales en vivo; no deja guardar si no cuadra. Atajo: buscar un comprobante (`GET /api/finance/ledger-entries?document_number=FV-38`) y **"Reclasificar"** una de sus líneas agrega el contrasiento de esa línea + una contrapartida con la cuenta por escribir, heredando el tercero. El modal va en un **portal** a `document.body` para que Enter nunca envíe el formulario "Guardar Configuración Legal".
- `utils/pucAccountNames.js`: el catálogo de nombres de cuentas se movió desde `finance.controller.js` para compartirlo con las sugerencias.

---

## 4. Libro Mayor exportable (Fase 8)

`GET /api/finance/ledger-book`:

- **Columnas A–O intactas** (plantillas de importación ya armadas). Nuevas al final: **P = Tipo de Asiento**, **Q = Documento Afectado** (`reference_document_number`), **R = Observaciones / Notas del Contador**. El ID interno pasó a la **S**.
- **Tipo de Asiento**: Venta (Comprobante de Venta), Devolución / Ajuste a Venta, Anulación de Ajuste a Venta, Nota Contable (tipo), Anulación de Nota Contable, Compra a Proveedor, Pago a Empleado, Saldo Inicial, Movimiento Manual / Gasto.
- **Observaciones**: motivo del ajuste, descripción de la nota y las observaciones del contador de ese comprobante **en esa sede** (la numeración es por sede), separadas por " | ".
- **Storno**: columna contraria, positivo.
- **Cuenta**: si el código no está en `PUC_ACCOUNT_NAMES`, toma el nombre de la bolsa de gasto.
- **Hoja 2 "Ajustes y Notas"**: un renglón por comprobante AJV/NTC del rango (fecha, sede si es consolidado, número, tipo, documento afectado, motivo, totales, Anulado/Vigente, quién lo registró, observaciones).
- **Observaciones del contador**: `ledger_observations` (tabla aparte, nunca se edita el ledger). `GET/POST /api/finance/ledger-observations`, solo el contador escribe, sobre comprobantes que existen en la sede. UI: `LedgerObservationsSection.jsx` en Reportes & Auditoría.
- El archivo descargado se nombra con la **fecha de Colombia** (antes UTC: después de las 7 p. m. salía el día siguiente).

---

## 5. Módulo del Contador y numeración visible (2026-09-28)

Segunda ronda de correcciones pedida por el usuario ("Puntos a corregir notas contables").

- **Numeración visible** (`utils/documentDisplay.js`, espejo en backend y frontend): máscara de 4 dígitos (`FV-38` → `FV-0038`) y prefijos del sector: **AJV → NC**; **CC según la dirección: entrada → RC, salida → CE** (un gasto nunca se muestra como Recibo de Caja, corrección del usuario 2026-09-28); NTC y los demás igual. Es presentación: la base guarda `FV-38`, `AJV-1`, `CC-4` como siempre. La dirección de un CC sale del tipo de línea (`CC_OUTFLOW_TYPES` = `MAN_EGR`/`CASH_PAYMENT`); donde solo hay el número (notas, observaciones, documento afectado) el backend devuelve `display_number` ya resuelto. `parseDocumentQuery` entiende lo que escribe el usuario ("nc 1", "RC-0012", "ce 4": un CE o un CC de salida) y `matchesParsed` filtra; `storedDocumentVariants` cubre también números guardados con ceros (`FV-006`).
- **Gastos en la serie CE**: desde el 2026-09-28 `payFixedCost` y un movimiento manual de salida reclaman `CE-n` (Comprobante de Egreso), la misma serie de los pagos a empleados; en el Libro Mayor su "Tipo de Asiento" es "Movimiento Manual / Gasto" (no "Pago a Empleado"). La migración `2026-09-28_ce_sequence_after_cc_egresos.sql` (la corre el usuario) adelanta `doc_seq_ce` de cada sede después de su CC de salida más alto, para que un CE nuevo nunca repita en pantalla un número viejo (caso real: un taller de prueba tiene el gasto `CC-1`, que se ve como CE-0001).
- **Finanzas y Flujo de Caja para el contador**, en solo lectura: rutas y menú habilitados; `financeReadWorkshopId` (finance.controller) aplica `?scope_workshop_id=` en `dashboard-summary`, `cashflow` y `monthly-books` (una sede del grupo; "todas" cae a la propia porque el panel calcula por sede). Las escrituras de esa pantalla ya exigían dueño/admin. Frontend: `useContadorScopeStore` (la sede elegida se mantiene entre Finanzas, Flujo y Libro Mensual) y los botones de registrar ocultos.
- **Columna N° Comprobante** en el Libro Auxiliar del Dashboard, en el Libro Mensual (pantalla y CSV) y en Flujo de Caja ("FV-0038 | concepto", también en el CSV). Notas, observaciones, tarjeta de ajustes, modal y Nota PDF muestran el número visible.
- **Buscador de comprobantes** (`GET /api/finance/documents`, `DocumentSearchInput.jsx`) en Nueva Nota Contable y en Observaciones del Contador. El valor de cada comprobante es su línea más grande (total facturado, cuenta por pagar o reembolso); sumar los débitos inflaría con asientos informativos. Crear nota u observación acepta el número de pantalla y guarda el del ledger.
- **Libro Mayor Excel**: hoja 1 con A–S intactas + **T = Tipo de Comprobante**. Hoja nueva **"Mayor por Cuenta"** con el orden pedido (Fecha, Tipo, Consecutivo, Documento Afectado, Tercero con NIT/cédula, PUC, Cuenta, Detalle, Débito, Crédito, **Saldo**, Observaciones): agrupada por cuenta, "Saldo anterior" al rango, acumulado por naturaleza (clases 1/5/6/7 débito, 2/3/4 crédito) y total por cuenta. Una fecha sin hora en `from`/`to` ahora cubre el día completo (antes Postgres la rechazaba). Verificado con el Excel real de un taller de prueba: saldo final de Caja y Bancos idéntico al del panel del dueño.

## Seguridad y aislamiento por sucursal

- **RLS activado sin políticas** (deny-by-default) en las 5 tablas nuevas, igual que el resto de `public`. Una política "por `workshop_id`" no protegería nada acá: el frontend nunca consulta Supabase directo, el backend usa el rol de servicio del backend (que salta RLS) y los empleados no son usuarios de Supabase Auth, así que no hay `auth.uid()` ni claim con qué compararla. Deny-by-default es más estricto: con la clave pública nadie ve nada.
- **Candados en la base** (triggers de la Fase 1): un ajuste debe ser de la misma sede que su orden; cada ítem debe referirse a un repuesto/servicio de esa orden y sede; cada línea del ledger de un ajuste o nota debe ser de la misma sede que su encabezado, y nunca dentro de un mes archivado. Ni un bug del backend puede cruzar datos entre sucursales.
- **Permisos** (validados en el backend en cada endpoint):

| Acción | Dueño | Admin | Contador | Mecánico |
|:---|:---:|:---:|:---:|:---:|
| Registrar / anular devolución o ajuste | ✅ | ✅ | Lee | ❌ |
| Registrar gasto en una bolsa | ✅ | ✅ | ❌ | ❌ |
| Crear / renombrar / desactivar bolsas | ✅ | ✅ | ✅ | ❌ |
| Asignar la cuenta PUC de una bolsa | ❌ | ❌ | ✅ | ❌ |
| Crear / anular notas contables, observaciones | Lee | ❌ | ✅ | ❌ |

- **Borrado de talleres**: `admin.controller.js` y los scripts `force-delete-workshop.mjs`/`reset-workshop-data.mjs` borran `sales_voucher_adjustments`, `accounting_notes`, `ledger_observations` (y `expense_categories`, salvo el reset) **justo después del ledger y del Kardex**, antes de `inventory`/`work_order_services`/`work_orders` — si no, las llaves de los ítems bloquean el borrado.

---

## Verificación hecha (2026-09-27)

- **Tests**: 918 en total al cierre de la Fase 8. Destacan `tests/billing/saleAdjustment.test.js` (31, incluye el invariante "devolver TODO deja Utilidad, IVA, Caja y Cartera en $0"), `salesAdjustments.controller.test.js`, `finance.accountingNotes.test.js`, `finance.ledgerBook.adjustmentsNotes.test.js`, `workshop.expenseCategories.test.js`.
- **En producción, con el un taller de prueba** (autorizado por el usuario, todo con efecto neto cero):
  - AJV-1 (rebaja de $1.000 de M.O. sobre FV-38, reembolso en Caja) + anulación AJV-2: Caja, Utilidad, Saldo Real e IVA idénticos antes y después.
  - NTC-1 (reclasifica $1.000 de 415505 a 413510 sobre FV-38) + anulación NTC-2: las cifras del dueño no se movieron nunca (decisión 4); las cuentas volvieron a cero.
  - Cuenta de la bolsa Aseo editada por el contador y devuelta a 519525.
  - Una **observación de prueba sobre FV-38 quedó guardada a propósito** (el usuario pidió dejarla).
  - Libro Mayor real: 52 comprobantes, todos cuadrados, 0 celdas negativas, hoja "Ajustes y Notas" con los 4 comprobantes.

## Verificación hecha (2026-09-28, segunda ronda)

Todo en producción, en el un taller de prueba y sus contadores (autorizado por el usuario), además de 964 tests:

- **Fase A**: vista previa de "Anular Factura Completa" sobre 10 órdenes reales (20 previews cuadradas, comisiones reversadas, casos de cartera y de saldo en contra); en el navegador, leyendas de impacto de contado, crédito y mixto, tabla de M.O. por servicio, descuento sobre servicio a 417595. Ningún ajuste se registró.
- **Fase D**: buscador y consulta por número de pantalla (`FV-0038`, `NC-0001`, `CE-0001`); el Excel real con "Mayor por Cuenta" da el mismo saldo final de Caja ($44.941) y Bancos ($262.292) que el panel del dueño; un contador de otro grupo que pide esa sede cae a la suya.
- **CE/RC**: el único gasto viejo `CC-1` se ve como `CE-0001` en buscador, libro mensual y Excel; tras la migración, el próximo `CE` de esa sede es el 2.
- **Fase C**: en 9 empleados reales a comisión, comisión del mes = desglose = Libro Mayor.
- **Fase B**: carga +1 "Filtro de Aceite" (`ASI-0005`) y merma −1 desde la pantalla (`ASI-0006`, 531095/1435, con motivo en el Kardex): Utilidad −$15.000 exacto, Caja y Bancos intactos.

## Fase 7 — Nota Crédito electrónica DIAN (PAUSADA)

**Pausada por decisión del usuario (2026-09-27)**: no se implementa hasta que haya un taller que **de verdad use la facturación electrónica** con Dataico. Construirla sin un caso real significaría escribir contra una API que no se puede probar de punta a punta, y un error ahí sí tiene consecuencias ante la DIAN.

**Mientras tanto (ya funciona así):**
- Un ajuste sobre una venta con CUFE se registra igual (contabilidad, inventario, caja/cartera), con `sales_voucher_adjustments.dian_status = 'pendiente'`.
- El modal lo avisa antes de registrar, la tarjeta del ajuste muestra "Nota Crédito DIAN pendiente" y la Nota de Ajuste en PDF lo dice.
- Las ventas sin CUFE (Comprobante de Venta interno) usan `dian_status = 'no_aplica'`: no les falta nada.
- Un ajuste con `dian_status = 'emitida'` no se puede anular desde EFISCO (ya está listo en `void_sales_adjustment`, para cuando exista la emisión).

**Qué hacer al retomarla:**
1. Conseguir un taller con Dataico configurado (credenciales + resolución + prefijo) en **ambiente de pruebas**.
2. Investigar el endpoint de notas crédito de Dataico (referencia a la factura original por CUFE/número, motivo DIAN de la nota, ítems) y cómo lo resuelve hoy `backend/services/dataico.service.js` para las facturas (sub-cuenta por taller, `resolveCredentials`).
3. Emitir de forma **no bloqueante** después de `apply_sales_adjustment` (mismo criterio que la factura en `settleOrder`): si Dataico falla, el ajuste contable ya quedó y `dian_status` pasa a `error` con opción de reintentar.
4. Guardar `dian_cude` y `dian_note_number`; `dian_status = 'emitida'`.
5. Pantalla: botón "Reintentar Nota Crédito" en la tarjeta del ajuste cuando esté en `pendiente`/`error`, y los ajustes pendientes de ventas viejas (hechos mientras la fase estuvo pausada) se emiten desde ahí.
6. Actualizar este documento, la guía de pruebas manuales y `api-reference.md`.

## Pendientes

**Pregunta pendiente (dejada por el usuario el 2026-09-28 para retomarla después):**

> ¿El **saldo en contra de un mecánico** (comisión ya pagada que una devolución o rebaja de mano de obra reversó) debe poder descontarse **también del sueldo**, con su autorización escrita?

- **Hoy**: se descuenta solo de sus **próximas comisiones** (el pendiente negativo se compensa solo con lo que gane después). Mientras esté en contra, `payMechanic` no deja pagar comisión (409); el sueldo se paga completo aparte.
- **Por qué no se hizo automático**: en Colombia, descontar del salario exige autorización escrita del trabajador (CST art. 149). Un empleado de sueldo fijo o mixto con poca comisión futura podría quedar con el saldo en contra indefinidamente.
- **Si la respuesta es sí**, la propuesta es una opción explícita en el modal "Registrar Pago" ("Descontar saldo en contra del sueldo", solo si hay saldo en contra y sueldo a pagar), que registre el cruce en el mismo comprobante `CE-`: el sueldo sale completo como gasto (`MECH_SALARY_PAY`), se paga en caja solo el neto, y la diferencia salda el `233525` en contra. Hace falta definir dónde queda constancia de la autorización del trabajador (por ejemplo, una casilla "Autorización escrita firmada" obligatoria para usar la opción).
