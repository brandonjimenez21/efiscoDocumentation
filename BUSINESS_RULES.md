# Reglas de Negocio y Módulos del Sistema

> Ver también: [README](README.md) · [FINANCIAL_ENGINE](FINANCIAL_ENGINE.md) · [API](API.md) · [SECURITY](SECURITY.md)

## Rutas del frontend

| Ruta | Módulo | Acceso |
|:---|:---|:---:|
| `/` | Landing — página de presentación pública (funciones, resumen de demo, CTA registro/login) | Público (sin sesión) |
| `/demo` | Demo del Flujo de Caja — detalle completo (8 tarjetas + resumen diario expandible) con cifras de ejemplo; el botón "Demo" de la Landing navega aquí en vez de hacer scroll a un ancla | Público (sin sesión) |
| `/login` | Autenticación — mismo componente alterna login / registro / recuperar contraseña | Público (sin sesión) |
| `/dashboard` | Dashboard | Todos |
| `/recepcion` | Recepción | Todos |
| `/bahia` | Bahías | Todos |
| `/inventario` | Inventario | Todos |
| `/proveedores` | Proveedores | Todos |
| `/ordenes` | Órdenes | Todos |
| `/referidos` | Referidos | Todos |
| `/soporte` | Soporte | Todos |
| `/config` | Configuración | Owner |
| `/finanzas` | Dashboard Financiero | Owner |
| `/equilibrio` | Panel de Equilibrio | Owner |
| `/cobros` | Panel de Cobros | Owner |
| `/flujo-caja` | Flujo de Caja | Owner |
| `/ordenes` | Historial de órdenes completadas | Todos |
| `/contador` | Panel del Contador (edición fiscal real) | Contador |
| `/cliente/registro/:workshopId/:intakeId` | Registro Seguro EFISCO (link de WhatsApp) | Público |
| `/support-session` | Bootstrap de sesión de soporte (llave de acceso canjeada) | Público (requiere token) |
| `/admin` | Panel Admin EFISCO | Admin interno |
| `/admin/talleres` | Gestión de talleres | Admin interno |
| `/admin/cobros` | Facturas de EFISCO **a** los talleres (SaaS fee) | Admin interno |
| `/admin/pagos` | Comisiones de referidos que EFISCO paga **a** los talleres | Admin interno |
| `/admin/referidos` | Árbol de referidos | Admin interno |
| `/admin/contabilidad` | Contabilidad Interna — EFISCO como emisor (Dataico Caso 2) | Admin interno |

> **Nota — "Cobros" aparece dos veces con significados distintos:** `/cobros` (arriba, vista del dueño) es la cartera de cuotas que el **taller** le cobra a **sus propios clientes**. `/admin/cobros` es un panel completamente distinto: las facturas que **EFISCO** le cobra **al taller** por el uso del software. No confundir tampoco `/admin/cobros` con `/admin/pagos`: el dinero fluye en direcciones opuestas — Cobros es taller→EFISCO, Pagos es EFISCO→taller (comisión por referidos).

---

## Dashboard (`/dashboard`)

Landing del dueño tras iniciar sesión: alertas que requieren atención (folios, cuotas vencidas, stock bajo), estado de la bahía en tiempo real, saldo real bancario y cartera pendiente.

- **Bienvenida de primer login** (`WelcomeModal.jsx`): si `workshop_config.welcome_seen` es falso, invita al dueño a configurar primero `/config` antes de operar, y a coordinarse con su contador si tiene uno. "Ir a Ajustes" marca el aviso como visto de forma permanente; "Lo haré después" solo lo oculta por la sesión actual — vuelve a aparecer en el próximo login (`useAuthStore.login()` limpia esa marca de sesión).
- **Refresco periódico** (mejora de UX, rev. 43): antes solo cargaba datos al montar — dejar la pestaña abierta mientras se opera el taller mostraba cifras cada vez más viejas sin ningún indicio. Ahora refresca al recuperar el foco de la pestaña/ventana (`visibilitychange`/`focus`) y cada 5 min de respaldo si queda enfocada mucho tiempo seguido; botón "Actualizado HH:MM" en el header (spinner mientras refresca) para que la frescura de los datos sea explícita.

---

## Recepción

Punto de ingreso rápido de vehículos. Registra cédula, nombre y apellido por separado, teléfono y síntoma reportado — la cédula se pide aquí mismo para poder activar el score crediticio de inmediato (antes requería que el cliente completara el Registro Seguro EFISCO primero). El nombre y apellido separados existen para poder facturar electrónicamente con los campos exactos que exige Dataico/DIAN (`first_name`/`family_name`); el resto de la app sigue mostrando el nombre combinado. Los tres campos validan formato en vivo: cédula (solo dígitos, 3–10), nombre/apellido (solo letras) y WhatsApp (solo dígitos, exactamente 10 — formato celular colombiano).

- **Autocompletado por cédula**: al escribir 5+ dígitos de cédula, se consulta `GET /api/clients/by-cedula/:cedula` (match exacto por `nit`, scoped al taller) con debounce — si el cliente ya se registró antes (por Recepción o por el Registro Seguro EFISCO), se rellenan nombre/apellido/WhatsApp automáticamente, solo en los campos que sigan vacíos (nunca pisa lo que el operador ya tecleó). Badge visual bajo el campo: "Buscando…" / "Cliente ya registrado — datos autocompletados" / "Cliente nuevo"
- Botón de WhatsApp por cada ingreso en cola: envía el link de **Registro Seguro EFISCO** (`/cliente/registro/:workshopId/:intakeId`), vinculado tanto al taller como al ingreso específico. Junto a él, un botón de **Telegram** (canal alternativo mientras Meta no verifica la cuenta de WhatsApp Business, ver `messagingChannel.service.js`): `GET /api/messaging/status` le dice al frontend qué canales están configurados a nivel servidor — si WhatsApp no lo está (o está apagado a propósito con `WHATSAPP_ENABLED=false`, ver ARCHITECTURE.md), su botón queda visible pero **deshabilitado** (gris, con tooltip) y el de Telegram aparece activo al lado, dándole al recepcionista los dos canales a la vista sin tener que adivinar cuál sirve. Clic en Telegram abre un modal con `TelegramLinkPanel.jsx` (mismo QR de vinculación global por cédula que usa el flujo de OTP) — si el cliente ya está vinculado, manda el mensaje de una vez sin mostrar nada; si no, espera a que escanee y lo manda automáticamente apenas se vincula, sin un segundo clic
- **Score de crédito, siempre detrás del código del cliente** (decisión de producto 2026-07-19: antes había un score básico "gratis" — local + global, sin desglose — visible con un solo clic sin código; se eliminó por completo, incluido el endpoint `GET /:cedula/risk-score` que lo exponía. Hoy no existe ninguna forma de ver el score, básico o con desglose por pilar (pago 60% / estabilidad 10% / fidelidad 30%), sin que el cliente autorice la consulta dictándole al recepcionista el código de 6 dígitos que le llega por WhatsApp/Telegram — `verifyScoreOTP` es el único endpoint que devuelve el score, y de ahí sale ya completo). El cobro del SKU "consulta puntaje" ($100 COP, solo si el cliente ya lleva ≥30 días con historial real en la red) sigue disparándose en cada verificación exitosa del código, igual que antes — no cambió, solo dejó de haber una vía para ver el score sin pasar por ahí. El código se manda por el canal disponible (`messagingChannelService.sendOTP` — WhatsApp si está configurado y habilitado, si no Telegram si el cliente ya vinculó su cédula, ver `client_telegram_links`): por WhatsApp, primero una plantilla de categoría Autenticación (`codigo_verificacion_score`, entrega sin ventana de 24h) con fallback a texto libre; por Telegram, un solo mensaje (no aplica ni plantillas ni ventana de 24h). Si WhatsApp no está disponible y este cliente puntual **todavía no vinculó Telegram**, el backend responde `reason: 'no_channel_linked'` y Recepción muestra ahí mismo el QR de vinculación (`TelegramLinkPanel.jsx`, fase `needs_telegram_link` de la máquina de estados) — en cuanto se vincula, vuelve a pedir el código automáticamente, sin que el recepcionista tenga que hacer clic dos veces. El **código de demo** (devuelto directo al recepcionista si NINGÚN canal está configurado) solo existe fuera de producción o con `OTP_DEMO_MODE=true` — en producción el endpoint responde con error claro en vez de filtrar el código, porque entregárselo al recepcionista anularía la autorización del cliente que el OTP protege. Vence a los 30 minutos (el cliente puede irse del taller antes de que le llegue el mensaje); la verificación tiene anti fuerza-bruta (5 intentos fallidos por taller+cédula → bloqueo de 15 min) y comparación constant-time — ver [SECURITY.md](SECURITY.md#endurecimiento-de-seguridad-2026-07-17)
- El reporte en PDF con el desglose completo se **genera en el servidor (Puppeteer/Chromium headless) y se envía automáticamente por el canal disponible** al cliente apenas se verifica el OTP (`verifyScoreOTP` → `scorePdfRenderer.service.js` → `messagingChannelService.sendDocument`) — best-effort, no bloquea la respuesta si el envío falla o no hay canal. Recepción ya no tiene botón de descarga propio; el PDF es exclusivo del cliente (ahí o desde su propio link de Registro Seguro EFISCO, `GET /api/clients/public-score-pdf/:workshopId/:intakeId`) — es exactamente el mismo documento en ambos casos, generado una sola vez en el backend

---

## Registro Seguro EFISCO (`/cliente/registro/:workshopId/:intakeId`)

Formulario público que el cliente llena desde su propio WhatsApp tras recibir el link tras registrar el ingreso en Recepción. La identidad (cédula, nombre, celular) se trae automáticamente del ingreso vinculado — el cliente no la vuelve a digitar.

- Muestra su propio score EFISCO (local y global) de forma transparente, o un mensaje de "aún no tienes historial" si es cliente nuevo
- **Clasificación fiscal**: Persona Natural | Empresa con sub-régimen (Simple / Ordinario / Gran Contribuyente) — se captura aquí, no en Recepción, y queda persistida en la orden de trabajo que se crea al enviar el formulario
- Pide correo y dirección de residencia, datos del vehículo (placa con el mismo formato autoordenado de Bahía: 3 letras + 3 números), y consentimiento de tratamiento de datos (Ley 1581 de 2012) antes de enviar
- El link solo funciona si viene vinculado a un ingreso de Recepción existente
- **Prueba legal persistida**: la aceptación de Términos y Condiciones / Política de Datos (scroll-lock obligatorio antes de habilitar el botón) se registra en `client_legal_acceptances` — versión, IP y momento exacto (`created_at`), append-only para trazabilidad ante una eventual auditoría de la SIC (ver [SECURITY.md](SECURITY.md#pruebas-legales-append-only)). El texto oficial completo es consultable públicamente desde el footer ("Términos y Condiciones" / "Privacidad")

---

## Bahías (Órdenes de Trabajo)

Gestión del trabajo en taller: asignación de técnicos, registro de mano de obra y repuestos.

- **Indicador de progreso Recepción → Bahía → Liquidación** (`EstadoProgreso.jsx`): stepper visual de 3 pasos, visible en la tarjeta de cada orden en Bahía, en la ficha del Historial y en la cola de Recepción, para que siempre sea claro en qué etapa del flujo está un vehículo
- **Adopción de orden pendiente**: si el cliente ya llenó su propio formulario de Registro Seguro EFISCO (placa/marca/modelo/año/dirección) antes de que el recepcionista mueva su ingreso a Bahía, esos datos quedan en una orden `pending` esperando — al abrir "Completar" en Bahía se buscan por cédula **y por la recepción exacta que la originó** (`GET /api/work-orders/pending-by-cedula/:cedula?intake_id=...`) y precargan el formulario en vez de arrancar en blanco; el filtro por `intake_id` evita que una recepción nueva sin formulario lleno herede por error los datos de una recepción vieja no relacionada. Los duplicados huérfanos se borran automáticamente al crear la orden real
- **Dirección y fecha/hora de entrega** persisten como tal (`client_address`, `estimated_delivery_at`) — se pueden editar en el formulario de Bahía y se restauran correctamente al reabrir "editar orden" (antes solo se guardaba una duración en minutos calculada una vez, sin forma de reconstruir la fecha/hora original)
- Clasificador de vehículos determina el tier de servicio (Básico / Premium)
- **Multi-servicio y multi-mecánico**: una orden lleva N servicios (tabla `work_order_services` — el total facturado es la SUMA de todos, cada uno con su margen básico/premium, e ítems separados en la factura/comprobante) y N mecánicos (tabla `work_order_mechanics` — cada uno con su propia mano de obra). Los escalares `service_name`/`mechanic_id`/`labor_cost` de `work_orders` se mantienen sincronizados (primer servicio / mecánico primario / suma de M.O.) para los consumidores legacy; órdenes pre-migración sin filas hijas siguen funcionando por la rama fallback
- **Mano de obra auto-calculada por mecánico**: tarifa hora = `monthly_salary ÷ 30 ÷ 8`, multiplicada por las horas estimadas de esa fila — editable (al editar manualmente deja de recalcular; editar una fila recalcula también el resto de filas en modo automático). **Mecánicos 100% comisión** (sin `monthly_salary` propio) no usan ese cálculo: su tarifa hora es el **promedio del sueldo mensual de los mecánicos de nómina fija asignados a esa misma orden** (`hourlyRateFor` en `Bahia.jsx`); si todos los asignados son de comisión o solo hay uno, cae al **SMLV + prestaciones ($2.860.000/mes)** como piso legal. La comisión de cada mecánico se calcula sobre SU porción de M.O.; la utilidad de inventario se reparte en partes iguales entre los asignados (`getMechanicMetrics`). En órdenes multi-servicio, `getMechanicMetrics` pondera el margen de CADA servicio de la orden con su propio margen del catálogo (consultando `work_order_services`) para derivar un margen efectivo sobre el `labor_cost` total — el mismo cálculo que hace `settleOrder` al liquidar (bug corregido en rev. 25: antes usaba solo el margen del servicio primario, y divergía de lo realmente liquidado cuando los servicios de una orden tenían márgenes distintos entre sí); cae al margen del servicio único solo en órdenes legacy sin filas hijas. Al liquidar, cada mecánico `comision`/`mixto` genera un asiento `MECH_COMMISSION` (devengo, no pago) en el Libro Auxiliar — ver [FINANCIAL_ENGINE.md](FINANCIAL_ENGINE.md#caja-real-vs-informativodevengo--por-qué-existe-la-distinción)
- **Fecha/hora de entrega solo futura** — inputs con `min` (fecha local, no UTC — en Colombia UTC-5 el `toISOString()` marcaría "mañana" desde las 7pm) + guard en el submit + validación espejo en el backend (400 si `estimated_delivery_at` < ahora − 2 min; en update solo si el valor cambió, para no bloquear la edición de órdenes ya vencidas)
- Consumo de inventario con registro automático en el Kardex; cada ítem guarda `vat_percentage` (0%, 5% o 19%)
- **Selector de repuestos con buscador** (mejora de UX, rev. 43): el `<select>` nativo listando todo el inventario (el flujo más repetido de la app, varias veces por orden) se reemplazó por `frontend/src/components/SearchableSelect.jsx` — combobox con filtro de texto insensible a mayúsculas/acentos, mismo componente reusado en Proveedores (ver abajo)
- Estado de la orden: `pending → ejecucion → ready_to_invoice → completed` — `updateStatus` valida que el `status` recibido sea uno de estos 4 (400 si no), bug corregido 2026-07-19: antes aceptaba cualquier string, y un valor inesperado podía dejar la orden en un estado "fantasma" que ningún filtro `.in('status', [...])` reconoce (ej. `getDashboardSummary`/Bahía), desapareciéndola sin explicación
- **Solo rol `mecanico` es asignable a órdenes** — el dropdown de Bahía filtra por rol y el backend rechaza con 400 cualquier `mechanic_id` de un empleado con otro rol (contador/admin), en `createWorkOrder` y `updateWorkOrder`. El propio alta/edición de empleado (`createMechanic`/`updateMechanic`) valida `role`/`payment_type` contra los mismos valores reales que usa el resto del sistema (`mecanico/admin/contador`, `salario/comision/mixto`) — bug corregido 2026-07-19: antes no validaba nada, y un valor mal tipeado (o mandado directo a la API sin pasar por el formulario de Config.jsx) rompía silenciosamente comparaciones por igualdad exacta en `getMechanicMetrics`/`settleOrder`
- **Modal de Liquidación** con pre-cálculo en vivo:
  - Simulación de comisiones Bold/Addi antes de confirmar
  - Selector de tipo de tarjeta (Débito/Crédito) en Bold físico/online, y tipo de movimiento (Débito/Crédito) en Addi — necesario para mapear correctamente el medio de pago que exige Dataico, ya que ninguna de las dos pasarelas distingue esto en su "tipo de transacción"
  - Retenciones si el cliente es agente retenedor — el preview replica exactamente `financialEngine.liquidateClientInvoice` (tasa de IVA configurable + IVA real por repuesto consumido, umbral de 4 UVT sobre la base completa, gate por `is_agente_retenedor_renta` del taller, tasas ReteFuente/ReteICA configurables): bug corregido, antes usaba IVA fijo al 19% y tasas de retención fijas (4%/0.966%) sin evaluar umbral ni régimen del taller, así que el cajero podía ver un "Total Neto" en pantalla distinto del que de verdad calculaba el backend al confirmar
  - **Bug real corregido en rev. 40**: el preview tampoco calculaba el IVA (19%) que el backend cobra sobre la propia comisión de Bold/Addi (`commissionVat`, restada también de `net_cash_inflow`) — el cajero veía un "Total Neto a Recibir" más alto del que de verdad liquidaba `settleOrder` en cuanto se cobraba por pasarela. La línea "Comisión Plataforma" ahora muestra el costo total real (comisión + su IVA), ver [FINANCIAL_ENGINE.md](FINANCIAL_ENGINE.md#1-liquidación-de-servicios-liquidateclientinvoice)
  - Modo crédito: selector de cuotas (2/3/4), fecha del primer pago — el monto de cada cuota se reparte con `utils/installments.js:splitIntoInstallments` (redondeo hacia abajo por cuota, la última absorbe el residuo) para que la suma cuadre exacto con el total facturado (bug corregido en rev. 25: antes cada cuota se redondeaba por separado y podía sobrar/faltar 1 peso)
- Al liquidar se emite factura a Dataico/DIAN de forma no bloqueante bajo la sub-cuenta propia del taller; si tiene éxito guarda `cufe` e `invoice_pdf_url`. Si el cliente es agente retenedor y la base supera el umbral UVT, la factura incluye el bloque `retentions` (RET_FUENTE/RET_ICA) con la misma tasa y monto que ya calculó `financialEngine` (ver [FINANCIAL_ENGINE.md](FINANCIAL_ENGINE.md)); si no, se envía `retentions: []` (estructura básica). El ítem de mano de obra de la factura usa `financial_summary.general_vat_rate_pct` (la tasa real del taller) en vez de un 19% fijo (bug corregido en rev. 25, ver [FINANCIAL_ENGINE.md](FINANCIAL_ENGINE.md)). Con multi-servicio, la factura lleva **un ítem por servicio** (cada uno con su porción de M.O. y su margen propio) + un ítem por repuesto (`kind: 'service' | 'part'`), y en los textos de display (WhatsApp, cuotas, comprobante) el servicio aparece como "A + B". **Probado end-to-end en producción real por primera vez en rev. 28** (`DIAN_ACEPTADO`, CUFE real) — hasta entonces este camino nunca se había ejercitado contra la API real de Dataico y tenía un bug real: `buildInvoiceItem` (`dataico.service.js`) espera un campo `sku` por ítem, pero quien arma los ítems en `billing.controller.js` nunca lo incluía — al quedar `undefined`, `JSON.stringify` borraba la clave completa del body (no la mandaba vacía) y Dataico respondía con un error interno críptico ("Unable to find data source...") en vez de una validación clara. Ahora `buildInvoiceItem` genera un `sku` secuencial (`"01"`, `"02"`...) cuando no se provee uno explícito. También crítico para que esto funcione: `DATAICO_ENV=PRODUCCION` debe estar configurado en el servidor — sin él, cualquier resolución real de la DIAN falla con "No se encuentra numeración..." porque el payload manda `env:"PRUEBAS"` (ver [ARCHITECTURE.md](ARCHITECTURE.md#variables-de-entorno)).
- Al confirmar la liquidación aparece un toast "Orden liquidada con éxito" (abajo a la derecha) — antes la orden solo desaparecía de la bahía sin ningún aviso
- **Comprobante de Venta** (WhatsApp, o Telegram si el cliente ya vinculó y WhatsApp no está disponible — ver `messagingChannelService.sendDocument`): apenas se liquida, se genera y se manda automáticamente un PDF al cliente (`saleReceiptTemplate.js`, mismo motor Puppeteer que el resto de los documentos) — best-effort, no bloquea la liquidación. Contado: pago recibido en su totalidad. Crédito: plan de cartera pactado con la tabla completa de cuotas. Incluye razón social/NIT/dirección/teléfono del taller, ítems detallados, IVA (con la tasa real del taller en la etiqueta, no un "19%" fijo — mismo fix que la factura Dataico, aplica también al comprobante descargable de `ModalLiquidacion.jsx`) y comisión de pasarela desglosada (base + IVA de la comisión por separado), y saldo neto real en caja — mismos campos que `documentation/Plantilla de comprobante de venta.txt`, con el lenguaje visual de tarjetas del resto de la app. El footer (aviso legal + habeas data) pagina en bloques independientes para que un comprobante simple no se parta en 2 páginas por un `page-break-inside` demasiado agresivo (bug visual corregido en rev. 25)
- **Historial (`/ordenes`)**: la vista de órdenes completadas muestra qué mecánico ejecutó cada orden (columna en la tabla + tarjeta en el detalle), vía join con `employees` en `getCompletedOrders` — visible para el dueño y para el propio mecánico en sus órdenes. **Filtro de fecha y paginación real** (mejora de UX, rev. 43): antes traía TODO el historial en una sola respuesta sin límite; `GET /api/work-orders/history` ahora acepta `from`/`to` (sobre `end_time`) y `limit`/`offset` (default 50, tope 100), respuesta `{ data, total }` — el frontend agrega inputs de rango de fechas y un botón "Cargar más" con contador "Mostrando N de Total"
- **6 bugs de frontend corregidos en rev. 42** (`Bahia.jsx`): (1) `handleRepuestoSelection` trataba un margen personalizado de repuesto de **0%** como inexistente (`!!item.default_margin`), aplicando el margen del tier en su lugar — corregido con chequeo explícito contra `null`/`undefined`; (2) el `setInterval` del cronómetro en vivo de las tarjetas de orden corría cada 100ms todo el tiempo que Bahía está montada, para un display que solo tiene resolución de minutos — bajado a 15s; (3) `handleSaveAllParts` no revisaba `res.ok` de cada repuesto del `Promise.all` — un fallo parcial (ej. stock insuficiente) se reportaba como "éxito total" y perdía el repuesto sin dejar rastro; reescrito para evaluar cada resultado por separado, dejar en la lista solo los que fallaron (con motivo) y no cerrar el modal hasta que todo se guarde; (4) `handleDeleteOrder`/`handleTogglePause`/`handleConfirmEditQty`/`handleConfirmDeletePart` no tenían rama de error si el backend rechazaba la operación — el botón "no hacía nada" sin ningún toast, a diferencia de `handleDeleteIntake`/`handleFinishOrder` en el mismo archivo; (5) el botón "Guardar Todos" del modal de repuestos tenía la clase visual de deshabilitado pero no el atributo `disabled={loading}` — doble clic podía duplicar el POST de cada repuesto; (6) el campo Cédula/NIT del ingreso directo en Bahía era texto libre, ahora usa `onlyDigits(value, 10)` igual que `Recepcion.jsx`, evitando que un cliente cargado por esta vía no coincida con el mismo cliente registrado por otra

---

## Inventario

Control de existencias con trazabilidad completa. El diagrama del flujo Kardex está en [ARCHITECTURE.md](ARCHITECTURE.md#5-inventario-y-kardex-inmutable).

- **Ficha de repuesto ampliada**: además de Nombre/Categoría/Costo/Margen, el formulario de alta separa dos catálogos que antes estaban mezclados en un solo selector — **Unidad de Medida** (mm, cm, in, ml, L, gal, g, kg, A, V, W: la unidad física del repuesto) y **Unidad de Inventario** (N/A / Unidad / Par / Kit: cómo se cuenta/empaqueta), más un campo libre **Número** (referencia del fabricante/proveedor). Ambos catálogos son columnas visibles en la tabla principal de Inventario (bug corregido 2026-07-20: en `Inventario.jsx` los dos selects estaban cruzados con el campo equivocado del formulario — elegir "ml" en Unidad de Medida en realidad se guardaba como `cm`, y el valor real enviado al backend como `measure` era el de Unidad de Inventario, N/A/Unidad/Par/Kit, por lo que `validateMeasure` lo rechazaba)
- **Kardex inmutable**: cada movimiento genera una transacción en `inventory_transactions`
- **Descuento de stock 100% en código de aplicación** (`inventory.controller.js:addItemToWorkOrder`), respetando si el ítem es de stock actual o nueva facturación — requiere haber corrido la migración 5, que elimina el trigger de base de datos que antes duplicaba el descuento
- **Validación de cantidades contra el stock real** — bug corregido 2026-07-19: ni `addItemToWorkOrder` (añadir repuesto a una orden) ni `addStandaloneInventory` (alta directa) ni `updateStock`/`PUT /api/inventory/add-stock/:id` (entrada de stock) validaban que la cantidad fuera un número válido — un valor no numérico dejaba `current_stock` en `NaN`, y no había tope contra el stock disponible al consumir (podía quedar negativo). Ahora las 3 rutas rechazan con 400 una cantidad no numérica/negativa, y `addItemToWorkOrder` además responde 409 si la cantidad pedida supera el stock actual del repuesto. `addStandaloneInventory` de paso empezó a llamar `validateMeasure` (ya lo hacía `addItemToWorkOrder`, pero el alta directa no)
- **Consumo gradual para líquidos/químicos** (categoría "Lubricantes y Químicos", decidido con el usuario vía AskUserQuestion: "el mecánico decide CUÁNDO baja el stock"): añadir uno de estos repuestos a una orden **no descuenta stock físico automáticamente** — el cliente se cobra igual, pero `current_stock` no baja hasta que el mecánico marca explícitamente el checkbox "¿Se agotó el contenedor?" (`container_emptied`) en el modal de Bahía. Evita que una orden de cambio de aceite baje el contenedor completo del stock aunque solo se haya usado una fracción de él. Esa decisión (si el alta movió stock o no) no se persiste en una columna aparte — `removeItemFromWorkOrder`/`updateItemInWorkOrder` la infieren de si existe la `inventory_transactions` asociada, la única señal confiable de que el stock sí bajó (bug corregido en rev. 25: antes restauraban/ajustaban stock siempre, así que quitar o editar un ítem gradual pendiente SUBÍA el inventario sin que nunca hubiera bajado)
- **IVA por categoría**: al seleccionar la categoría del repuesto, el porcentaje de IVA se aplica automáticamente desde `workshop_config.category_vat_rates`
- **Rotación y stock mínimo vital persistidos** (Fase 3, `utils/inventoryMetrics.js`): `rotación = stock_actual / uso_promedio_por_servicio` (informativo). `min_stock_vital = ceil(uso_promedio × 2)` — umbral de reorden basado en CONSUMO, independiente del stock actual; sin historial de consumo cae a un piso de 1 unidad. (Bug corregido en rev. 25: la fórmula original, `stock_actual × (1 + rotación)`, era por construcción siempre ≥ stock actual — como las alertas comparan `current_stock <= min_stock_vital`, todo ítem con la métrica calculada quedaba perpetuamente "en stock bajo".) Se recalcula y **guarda** en `inventory.min_stock_vital` tras cada movimiento de stock (compra, consumo, ajuste manual)
- **Tab "Matriz"**: rotación, uso promedio, stock mínimo vital y valor de inventario por ítem
- `getItemHistory` ordena por `requested_at`
- **Un repuesto con historial de uso no se puede eliminar** (`deleteInventoryItem`): si ya aparece en `service_inventory_items` de alguna orden (completada o no), el borrado responde 409 en vez de cascadear — esa tabla no guarda el nombre del repuesto (solo `inventory_id`), así que borrar el maestro dejaría el detalle histórico de esas órdenes sin descripción, y el `inventory_cost`/`INV_COGS` ya liquidado quedaría sin soporte. Si el repuesto ya no se va a comprar más, la recomendación es dejar su stock en 0 en vez de eliminarlo. Sin historial de uso, se borra normalmente (incluidas sus `inventory_transactions` propias)

---

## Proveedores y Egresos

Gestión de proveedores y registro de compras con liquidación fiscal (ver reglas de cálculo en [FINANCIAL_ENGINE.md — Liquidación de compras](FINANCIAL_ENGINE.md#2-liquidación-de-compras-liquidatesupplierpurchase)).

- **Perfil tributario del proveedor** (4 regímenes): Persona Natural · Régimen Simple · Régimen Ordinario · Gran Contribuyente — matriz completa de qué retención aplica según el régimen del taller comprador Y el del proveedor en [FINANCIAL_ENGINE.md — Matriz de retenciones](FINANCIAL_ENGINE.md#matriz-de-retenciones-taller-comprador--proveedor) (proveedor Gran Contribuyente nunca se retiene, sin importar el régimen del comprador; proveedor Simple solo recibe ReteICA+ReteIVA si el comprador es Gran Contribuyente). Tope UVT de compras: 10 UVT ≈ $503.180 (Decreto 572/2025 vigente — ver nota de litigio en FINANCIAL_ENGINE.md)
- **Tasa de ReteICA por proveedor** (`reteica_rate_supplier`, en por-mil — ej. `9.66` = 0.966%, mismo formato que `workshop_config.reteica_rate`)
- **Pago a plazos / Cuentas por Pagar**: en "Registrar Egreso", toggle "¿Va a pagar a plazos?" con número de cuotas y fecha del primer vencimiento — genera una cuenta por pagar (`supplier_installments`) en vez de un pago inmediato, espejo exacto del crédito a clientes de [Panel de Cobros](#panel-de-cobros-cobros). Tab nueva **"Cuotas por Pagar"** en Proveedores (agrupadas por compra, botón "Pagar" por cuota, mismo componente visual que Panel de Cobros) para ir liquidándolas — cada pago genera la salida de caja real y "mata" la cuenta por pagar, igual que `payInstallment` hace con clientes
- **OCR de facturas**: AWS Textract extrae proveedor, ítems, valores. Tras el escaneo aparece un **banner de verificación** con los datos extraídos, para que el mecánico/dueño los confirme o corrija antes de registrar la compra — el OCR nunca escribe directo a la base sin pasar por esta revisión. El IVA/impuesto que trae la factura escaneada se toma explícito del OCR (`invoiceVatAmount`) en vez de derivarlo siempre al 19% plano en `liquidateSupplierPurchase`, corrigiendo egresos mal calculados con facturas de tasas distintas
- **Método de pago**: `banco` → GMF 4×1000 | `tarjeta` → costo de transacción | `efectivo` → sin costos
- **Código PUC por proveedor** (`puc_account_expense`): usado en asiento del ledger si está definido
- **Proveedor EFISCO de solo lectura para el taller**: nombre, NIT, ciudad, teléfono, régimen, tasa ICA y código PUC vienen únicamente de `efisco_provider_template` (admin-efisco, ver [Panel de Administración EFISCO — Cobros](#cobros-admincobros)) — `updateProvider` responde 403 si el proveedor es `is_system_provider`, y el botón de editar ni siquiera aparece en el frontend
- **Eficiencia de proveedor** (Fase 3, tab "Eficiencia"): al registrar una compra se puede indicar `order_placed_at` (fecha del pedido); se compara el tiempo de entrega de la última compra contra el promedio de las últimas 5 compras con fecha de pedido registrada (`GET /api/providers/efficiency`, `utils/providerEfficiency.js`). Solo hacia adelante — las compras registradas antes de este campo no tienen fecha de pedido
- **Selector de producto con buscador** en "Vincular con Inventario" al registrar una compra (mejora de UX, rev. 43) — mismo `SearchableSelect.jsx` que Bahía, ver [Bahías](#bahías-órdenes-de-trabajo)
- **Confirmación de borrado con estado de carga** (mejora de UX, rev. 43): eliminar un proveedor ya tenía confirmación (`ElegantConfirmModal`), pero el modal solo se cerraba en el `finally` de la petición — el botón "Eliminar" seguía clickeable mientras el DELETE estaba en curso, permitiendo doble clic. Corregido con el nuevo prop `loading` del modal (ver [ARCHITECTURE.md](ARCHITECTURE.md))

---

## Panel de Equilibrio (`/equilibrio`)

Análisis de punto de equilibrio, capacidad operativa y proyección de escenarios.

```
Costos Fijos = arriendo + servicios públicos + nómina (fixed_costs_salaries)
Margen de Contribución = ingresos netos / ingresos brutos
Punto de Equilibrio = costos fijos / margen de contribución
```

**Gráfica SVG interactiva:**
- Hover con crosshair vertical que sigue el cursor
- Tooltip en tiempo real mostrando ingresos, costo total y ganancia/pérdida en ese punto
- Eje X con cantidad de órdenes equivalente a cada nivel de ingreso
- Zonas de pérdida (roja) y ganancia (verde) con relleno semitransparente
- Puntos "PE" (ámbar) y "Hoy" (azul) siempre visibles
- **Soporte touch** (mejora de UX, rev. 43): el tooltip solo escuchaba `onMouseMove`/`onMouseLeave` — en una tablet (dispositivo típico del piso de un taller) el gráfico era decorativo, sin forma de tocar para ver el detalle. Agregado `onTouchStart`/`onTouchMove`/`onTouchEnd` sobre la misma lógica de posición (`BreakevenChart.updateHoverFromClientX`)

**Proyección What-if (cálculo fiel):**
- Los costos variables son **absolutos por orden** (precio de partes), no porcentuales
- Al subir precios, el margen % mejora automáticamente: `wiMarginPct = (wiRevenue − varCosts) / wiRevenue`
- El nuevo PE se recalcula con el margen proyectado: `wiPE = wiFixedCosts / wiMarginPct`
- Panel de resultado muestra comparativa lado a lado: órdenes actuales vs proyectadas, costos fijos actuales vs proyectados
- El hint de órdenes necesarias usa contribución neta por orden: `ceil(wiGap / (wiTicket − varCostPerOrder))`
- Tarjeta con fondo blanco en modo claro (antes siempre oscura sin importar el tema) — el modo oscuro se mantiene exactamente igual

**Matriz de confianza estadística** (Fase 4, `utils/confidenceMatrix.js`): confianza por celda (vehículo × servicio) con meta de n≥10 registros. El histórico se lee de `work_orders.vehicle_age_category` (mapeada a `vehicle_type` antes de llamar el útil) — la columna `vehicle_type` no existe en `work_orders`; seleccionar ese nombre inexistente era el bug que dejaba toda la matriz en 0%. **Bug real corregido en rev. 22**: en una orden con varios servicios (ver multi-servicio arriba), `getBreakevenPanel` solo leía el `service_name` escalar (el primer servicio) — un servicio como "Cambio filtro de aire" nunca aparecía en la matriz si no era el primero de la orden. Corregido uniendo `work_order_services` y expandiendo cada orden en una entrada por servicio antes de construir la matriz.

---

## Panel de Cobros (`/cobros`)

Gestión de cuentas por cobrar — ventas a crédito e installments.

- Lista de cuotas pendientes con fecha de vencimiento, agrupadas por `work_order_id` (cada crédito de una orden en su propia tarjeta expandible, aunque dos créditos compartan vehículo/servicio — bug corregido: antes agrupaba por un campo inexistente, `billing_id`, con fallback placa+servicio que mezclaba créditos distintos en un solo grupo)
- "Vencida" se evalúa contra la fecha de hoy en hora de Bogotá (`Intl.DateTimeFormat('en-CA', { timeZone: 'America/Bogota' })`), no UTC — bug corregido: con `toISOString()` una cuota que vencía hoy ya aparecía "Vencida" desde las 7pm hora colombiana, 5 horas antes de tiempo
- **Confirmación antes de marcar una cuota como pagada** (mejora de UX, rev. 43): antes el botón "Pagar" registraba el cobro al primer toque, sin confirmación — en una lista táctil (el dispositivo típico de un cajero) un toque accidental registraba un pago que no ocurrió. Ahora abre un `ElegantConfirmModal` (monto + número de cuota + placa) antes de llamar al endpoint
- Registro de pago: genera asiento `INC_GROSS` con PUC `puc_income_code || '4135'`. **Este asiento es de caja pura** (`net_amount = gross_amount` = valor de la cuota, con IVA incluido) — refleja el cobro real, no una venta nueva; el ingreso operativo de la venta a crédito ya se reconoció una sola vez, sin IVA, al liquidar la orden (ver [FINANCIAL_ENGINE.md — venta a crédito](FINANCIAL_ENGINE.md#1-liquidación-de-servicios-liquidateclientinvoice)). El Flujo de Caja identifica estos asientos por su concepto fijo (`"Cuota N/M — ..."`) para excluirlos del ingreso operativo y no contar la venta dos veces (bug corregido — ver [Flujo de Caja](#flujo-de-caja-flujo-caja) abajo)
- **Comprobante de Pago** (WhatsApp, o Telegram si el cliente ya vinculó y WhatsApp no está disponible): al marcar una cuota como pagada se genera y se manda automáticamente un PDF (`paymentReceiptTemplate.js`) con el monto recibido, "cuota X de Y" y el saldo pendiente (o "✓ crédito pagado en su totalidad" si era la última) — best-effort, no bloquea el registro del pago. El teléfono se toma de `work_orders.client_phone` (siempre disponible, sin importar si el cliente vino por el link público o se cargó manualmente en Bahía)

---

## Flujo de Caja (`/flujo-caja`)

Libro mayor de todos los movimientos financieros (tipos de movimiento documentados en [FINANCIAL_ENGINE.md — Libro Auxiliar](FINANCIAL_ENGINE.md#libro-auxiliar-cash-flow-ledger)).

- Filtro por rango de fechas y tipo de impacto (CREDIT / DEBIT / Todos)
- Agrupación por día con subtotales diarios
- Balance acumulado por movimiento (`running_balance`)
- **Descarga CSV armada en el frontend** a partir de los movimientos ya cargados en pantalla (`entries`, ya filtrados por `from`/`to`) — con escape RFC 4180 (`""`) y neutralización de prefijos `=`,`+`,`-`,`@` (inyección de fórmulas en Excel/Sheets). Bug corregido: antes descargaba `/api/finance/report/ledger`, que trae el libro **completo** del taller sin filtrar por fecha, pero el archivo se nombraba `flujo_caja_{from}_{to}.csv` como si sí lo estuviera
- **Deep-link con rango explícito**: `?from=YYYY-MM-DD&to=YYYY-MM-DD` inicializa los filtros (usado por el modal de libros mensuales y el botón "Ver todo el mes" del dashboard financiero); sin params, el default sigue siendo el mes en curso
- **Utilidad Neta no duplica el ingreso de ventas a crédito**: los asientos `INC_GROSS` que genera el cobro de una cuota (`payInstallment`, identificados por su concepto fijo `"Cuota N/M — ..."`) se excluyen del ingreso operativo — la venta ya se reconoció completa (base sin IVA) al liquidar la orden; contar también el cobro de cada cuota (con IVA incluido en su `gross_amount`) inflaba y duplicaba el ingreso operativo de talleres que venden a crédito (bug corregido)
- **Capital Inicial separado de Ingresos** (rev. 22): el saldo migrado del cuaderno/Excel previo (`OPENING_BALANCE`) tiene su propia tarjeta de resumen — antes se sumaba dentro de "Total Ingresos" e inflaba el ingreso real del período con dinero que ya existía antes de usar Efisco
- **"Saldo Real"/"IVA Pendiente DIAN"/"Balance del Período" son una posición ACUMULADA real, no solo del rango elegido** (bug corregido): antes se calculaban solo con los asientos dentro de `[from,to]` — con el rango por defecto (mes en curso), cualquier taller con más de un mes de uso veía un "Capital Inicial" de $0 y un saldo/IVA que ignoraban todo lo acumulado en meses anteriores, divergiendo de lo que muestra el Dashboard Financiero (histórico completo). `getCashFlow` ahora calcula, con el mismo motor que usa el Dashboard (`financialEngine.calculateGlobalHealth`), el saldo/IVA reales que ya existían antes de `from` y la posición acumulada hasta `to` — "Saldo Real" de esta pantalla coincide con "Saldo Real Bancario" del Dashboard cuando el rango llega hasta hoy. Tampoco se clampea el IVA a 0: un taller con más IVA descontable/retenido que generado queda con un crédito real a favor, igual que ya reconocía `financialEngine`
- **Tarjeta "Costos"** (rev. 22): agrupa `INV_COGS` + `MECH_COMMISSION` + `MECH_SALARY_PAY` + `MAN_EGR` + `TAX_GMF` + `CARD_FEE` + `GW_FEE` + `GW_VAT` + `FOLIO_ADV` — el costo de un repuesto/mano de obra se reconoce al **liquidar** la orden (consumo real), no al comprar el repuesto ni al pagarle la comisión al mecánico; `MECH_COMMISSION_PAY` (pago en efectivo de una comisión ya devengada) se excluye para no contar el costo dos veces. Ver el porqué completo en [FINANCIAL_ENGINE.md](FINANCIAL_ENGINE.md#caja-real-vs-informativodevengo--por-qué-existe-la-distinción)
- **Pago a mecánicos reflejado aquí**: el botón "Pagar a Mecánico" de `/config` (ver [Configuración del Taller](#configuración-del-taller-config)) genera `MECH_SALARY_PAY`/`MECH_COMMISSION_PAY` con `mechanic_id`, visibles como cualquier otro movimiento del Libro Auxiliar
- **"Costos"/"Utilidad Neta" ahora comparten la misma definición con el Dashboard Financiero** (`/finanzas`): `COST_TYPES`/`INCOME_TYPES`/`calculateOperatingProfit` viven en `financialEngine.js` (espejo en `ledgerLabels.js` del frontend) en vez de constantes locales de esta página — antes Finanzas usaba un cálculo más angosto (solo gasto a proveedores + costos fijos) y ni siquiera mostraba una tarjeta de Utilidad Neta, divergiendo de lo que veía el taller aquí. La tarjeta nueva "Utilidad Neta" del Dashboard Financiero usa exactamente este mismo cálculo (bug corregido)

### Libros Mensuales archivados

Cada mes terminado se archiva como un **libro cerrado con snapshot** (tabla `monthly_ledger_books`, UNIQUE `(workshop_id, period)`): totales de ingresos/egresos/neto, cadena de saldos (`opening_balance` = `closing_balance` del mes anterior) y las filas del ledger congeladas en `snapshot` (jsonb). El **cierre es perezoso** — no hay cron (Render free): la primera consulta a `GET /api/finance/monthly-books` después del cambio de mes genera los libros faltantes, con upsert idempotente para que dos requests simultáneos no dupliquen. El mes en curso nunca se cierra. Se excluyen los `INC_GROSS` con `net_amount = 0` (marcadores de reconocimiento de crédito, mismo filtro del dashboard). El array de libros se reordena por `period` en cada libro que se genera dentro de la misma corrida — si hay un hueco entre libros existentes (ej. existe 2025-03 pero falta 2025-02), el opening de un mes posterior encadena con el `closing_balance` del libro cronológicamente más reciente, no con el último insertado en el array (bug corregido en rev. 25). Las fronteras de cada mes (y `currentPeriod`, "cuál es el mes en curso") se calculan en hora de Bogotá (UTC-5 fijo, sin horario de verano) en vez de medianoche UTC — un movimiento entre las 7pm y la medianoche colombiana ya tenía un `created_at` (UTC) del día siguiente, así que antes se archivaba en el libro del mes equivocado. El saldo migrado del cuaderno (`OPENING_BALANCE`) que caiga dentro de un mes se excluye de `total_income`/`total_expenses` y se suma directo a `opening_balance`/`closing_balance` de ESE libro — bug corregido: antes se contaba como venta del mes (inflando "Ingresos" del mes de la migración) y el libro nunca reflejaba su propio saldo de apertura real, quedando en $0.

En el Dashboard Financiero aparecen como tarjetas ("Libros Mensuales": mes, neto, # movimientos); el click abre `ModalLibroMensual.jsx`, que muestra el **snapshot congelado** (no el ledger vivo — aunque el flujo de caja siga creciendo, lo archivado no cambia) con un botón secundario "Abrir en Flujo de Caja" que navega con el rango del mes.

---

## Referidos

Sistema de referidos entre talleres con descuentos acumulados.

| Suscripciones referidas | Descuento aplicado |
|:---:|:---|
| 1 | 33% sobre cuota mensual |
| 2 | 66% sobre cuota mensual |
| 3+ | 100% (mes gratis) |
| Platino (>5) | 15% comisión directa (aplicado por EFISCO) |

- **"Quién te refirió" es de una sola vez y sin ciclos**: `linkReferral` rechaza el intento si el taller ya tiene un referidor vinculado, y recorre la cadena de referidos hacia arriba antes de guardar — un taller no puede vincularse con otro que, directa o indirectamente, ya fue referido por él mismo (rompería el árbol jerárquico de Admin → Red de Referidos, que necesita una raíz). No afecta el nivel propio: el nivel sube por cuántos OTROS talleres usan tu código, sin límite, sin importar si tú ya fuiste referido
- "Talleres que Referiste" (dashboard del dueño) lista los talleres que usaron tu propio código, con fecha de unión y ubicación — separado de "¿Alguien te recomendó?", que es la relación inversa (quién te trajo a ti)
- Banco/entidad para el pago de comisiones: selector con los 18 bancos más comunes de Colombia + "Otro" con texto libre

---

## Configuración del Taller (`/config`)

Panel de administración fiscal y operativa. Cinco pestañas:

**1. Datos del Taller** — Nombre, dirección, teléfono, horarios, costos fijos (arriendo + servicios). El teléfono se usa en el encabezado del Comprobante de Venta. Incluye la card "Contrato de Afiliación" (`ContractViewerModal.jsx`) para volver a leer el contrato B2B firmado al registrar el taller — desde rev. 45 muestra además un banner "Firmado digitalmente" con fecha, razón social y NIT (mismo componente reutilizado en el detalle de taller del [Panel Admin](#talleres))

- **Botón "Guardar Cambios Globales" ya no tapa el Footer** (bug corregido en rev. 46): el botón usaba `position: fixed` anclado al viewport (`bottom-6 right-6`), así que se quedaba flotando sobre el `Footer` de la app (enlaces "Privacidad", "Términos", etc.) al hacer scroll hasta el final de la página — el `Footer` vive dentro del mismo contenedor con scroll (`<main>` en `App.jsx`), no fuera de él. Cambiado a `position: sticky` dentro de un contenedor `flex justify-end` como último hijo del `<form>`: se mantiene flotando bottom-right mientras se hace scroll por el contenido del formulario, pero deja de "pegarse" en cuanto el `<form>` termina — justo antes de llegar al `Footer` — así que nunca puede solaparlo

**2. Mi Equipo & Roles** — Alta de empleados, esquemas de compensación (fijo / comisión / híbrido). Crear, editar, desactivar empleados y habilitar sus credenciales de acceso está restringido al rol `owner`. **El dueño también puede registrarse a sí mismo como mecánico** (`POST /api/mechanics/self`, upsert sobre `employees.user_id = owner.id`, sin necesitar email/password propios) para llevar su propio sueldo/comisión separado de los costos fijos genéricos del taller y aparecer como cualquier otro mecánico en el dropdown de asignación de Bahía, con sus propias métricas. La tarjeta del dueño en el grid ya no muestra el botón "ver detalles" (redundante — el banner "Tu Perfil de Mecánico" tiene su propio "Ver mis datos"), y tanto el banner como el header del modal de detalles usan avatar redondo consistente con las demás tarjetas. Nota: el rol `contador` NO es asignable como mecánico en órdenes (gate en frontend y backend, ver Bahías)

- **"Pagar a Mecánico"** (rev. 22, dentro del detalle de cada empleado): los mecánicos `comision`/`mixto` ya generan su comisión automáticamente por orden (asiento `MECH_COMMISSION`, devengo — ver [FINANCIAL_ENGINE.md](FINANCIAL_ENGINE.md#caja-real-vs-informativodevengo--por-qué-existe-la-distinción)); este botón es para registrar el **pago real en efectivo**, sea el sueldo fijo o la comisión ya devengada. Muestra primero lo pendiente por pagar (`GET /api/mechanics/:id/pending-payment` = `Σ MECH_COMMISSION − Σ MECH_COMMISSION_PAY`), y un modal para capturar los montos de sueldo/comisión a liquidar (`POST /api/mechanics/:id/pay`) — el pago queda reflejado de inmediato en Flujo de Caja
- **Panel de detalle del empleado sin condición de carrera** (bug corregido en rev. 42): el `useEffect` que carga historial salarial/métricas/pago pendiente (keyed por `showDetailsModal.employee?.id`) no cancelaba las 3 peticiones del empleado anterior — abrir el panel de un empleado y luego rápido el de otro, antes de que resolviera el primero, podía sobrescribir el panel del segundo con datos del primero si esa respuesta tardía llegaba después. Corregido con el patrón estándar de React para este caso (flag `cancelled` chequeado antes de cada `setState`, seteado en el cleanup del efecto)

**3. Catálogo de Servicios** — CRUD con márgenes básico/premium por tipo de vehículo

**4. Pasarelas**

Editable directamente por el dueño (no requiere al contador):

*Tasas de pasarelas*: Bold físico (2.99%), Bold online (3.49%), Addi (10.5%)

*Anticipo de Folios*: calculadora de recarga de folios Dataico (cantidad × costo unitario + margen + comisión de pasarela), con registro automático del egreso en el flujo de caja

**5. Módulo del Contador**

*Parámetros Fiscales y de Recaudo* — visible aquí en modo solo lectura para el dueño (la edición real vive en el panel del contador, `/contador`). Régimen fiscal, tasas, los 26 códigos PUC, credenciales Dataico e identidad legal son campos **exclusivos del rol `contador`** — el backend (`updateWorkshop`) los descarta en silencio si el llamador no es contador, ni siquiera el dueño puede modificarlos vía API, solo leerlos (ver [SECURITY.md](SECURITY.md#decisiones-técnicas-de-seguridad)):

Régimen Fiscal (4 opciones):
| Opción | IVA | Reg. Simple | Agente Retenedor |
|:---|:---:|:---:|:---:|
| No Responsable de IVA | ✗ | ✗ | ✗ |
| Régimen Simple (SIMPLE) | ✓ | ✓ | ✗ |
| Régimen Ordinario | ✓ | ✗ | ✓ |
| Gran Contribuyente | ✓ | ✗ | ✓ |

Régimen Simple SÍ cobra IVA (solo no practica retenciones) — verificado contra la DIAN: el RST unifica renta+ICA en un solo pago pero el IVA se declara aparte para la generalidad de actividades (la "tarifa con IVA incluido" solo aplica a tiendas/minimercados pequeños del numeral 1, no a un taller de servicios). Bug corregido esta revisión: `ContadorInventario.jsx` guardaba `is_responsable_iva:false` para esta opción, contradiciendo su propia descripción y al backend.

**Ojo — la columna "Agente Retenedor" de arriba solo describe el campo `is_agente_retenedor_renta`, relevante para COMPRAS (¿puede este taller retenerle a sus proveedores?), NO para ventas.** Quién sufre retención al VENDER depende de otra regla, corregida dos veces en esta sesión (rev. 34 y rev. 38, investigada contra Decreto 1091/2020 y Art. 911/437-2 ET):

| Régimen del taller (vendedor) | ReteFuente | ReteICA | ReteIVA |
|:---|:---:|:---:|:---:|
| No Responsable de IVA | SÍ (si el cliente es agente retenedor + supera umbral) | SÍ | $0 natural (no genera IVA) |
| Régimen Simple | NO (Art. 911) | NO (Art. 911 — protege renta E ICA, no solo renta) | SÍ |
| Régimen Ordinario | SÍ | SÍ | SÍ |
| Gran Contribuyente | NO (autorretenedor) | NO (autorretenedor) | NO (autorretenedor) |

El régimen del CLIENTE (Ordinario vs. Gran Contribuyente) no cambia el resultado en ninguna fila — solo importa si el cliente está marcado como agente retenedor. Ver la matriz completa, con la investigación detrás de cada exención, en [FINANCIAL_ENGINE.md](FINANCIAL_ENGINE.md#matriz-de-retenciones-taller--cliente-en-ventas-rev-38).

Además, si el cobro es por **pasarela** (Bold/Addi), un taller Régimen Ordinario sufre una retención adicional que practica la propia pasarela (ReteRenta 1.5% + ReteIVA 15% + ReteICA propio) — independiente de si el cliente es agente retenedor. Ver [FINANCIAL_ENGINE.md — Retención de la Pasarela](FINANCIAL_ENGINE.md#retención-de-la-pasarela-sobre-el-giro-al-taller-rev-38).

**Régimen por defecto de un taller recién registrado**: `auth.controller.js: register` fija explícitamente **No Responsable de IVA** (`fiscal_regime:'no_iva'`, los 3 booleans en `false`) al crear el `workshop_config` — el régimen más conservador (no cobra IVA, no retiene nada) hasta que el contador configure el real desde `/contador`. Bug corregido en rev. 35: antes `register` no seteaba estos campos y dependía de los defaults de columna en la BD (`fiscal_regime` default `'ordinario'` pero `is_agente_retenedor_renta` default `false`), una combinación que no correspondía a ningún régimen real — un taller podía facturar antes de que el contador lo configurara y no se le practicaba ninguna retención pese a figurar nominalmente como "Ordinario".

Tasas: IVA (configurable, no fija en 19%), ReteICA (‰), ReteFuente declarantes/no declarantes, ReteIVA (editable, default 15% — antes hardcodeado a 15% en el motor sin importar lo configurado, bug corregido en rev. 34), las tasas de retención a PROVEEDORES (`supplier_retefuente_rate`, `supplier_reteica_rate` — separadas de las tasas de retención a clientes de arriba; agregadas en rev. 25, antes existían como columnas pero no eran configurables desde ninguna pantalla), y (rev. 38) las tasas de retención de la PASARELA (`gateway_reterenta_rate` default 1.5%, `gateway_reteica_rate` en ‰ default 4.0 = 0.4% — sección "Tasas de Pasarelas Externas")

**Proveedores** (tab nueva dentro de `/contador`): lista los proveedores del taller con badge de perfil tributario y un selector para corregirlo directamente (reusa `GET/PUT /api/providers`), más acceso al historial de compras de cada uno — antes el contador no tenía ninguna forma de ver ni editar la clasificación fiscal de los proveedores del taller.

*Identidad Legal*: NIT, Razón Social, Prefijo — edición exclusiva del rol `contador` (el dueño solo lee). (No existe un campo de "Clave Técnica DIAN" propia del taller: bajo el modelo de sub-cuentas de Dataico, el CUFE lo calcula Dataico internamente por cada sub-cuenta — account_id + token + resolución —, el taller nunca necesita ni envía una clave técnica propia.)

*Presets PUC* — 3 botones que auto-rellenan los 21 códigos según régimen:
- **Régimen Ordinario** (estándar DIAN)
- **Régimen Simple** (subcuentas simplificadas)
- **Gran Contribuyente** (subcuentas retención en la fuente)

*Plan Único de Cuentas — 25 códigos en 5 bloques*:

| Bloque | Códigos | Defaults |
|:---|:---|:---|
| Ingresos & Ventas | `puc_income_code`, `puc_parts_income_code`, `puc_inventory_purchase_code`, `puc_costo_ventas_code`, `puc_comisiones_mecanicos_code`, `puc_gateway_fee_code`, `puc_gateway_vat_code` | `4135`, `4135`, `1435`, `6135`, `510506`, `5290`, `2408` |
| IVA | `puc_iva_generated_code`, `puc_iva_generated_5_code`, `puc_iva_deductible_code`, `puc_devolucion_iva_code` | `240805`, `240810`, `240820`, `135520` |
| Retenciones por Pagar | `puc_retefuente_code`, `puc_retefuente_compras_decl_code`, `puc_retefuente_compras_nodecl_code`, `puc_retefuente_servicios_code`, `puc_reteiva_code`, `puc_reteica_code` | `2365`, `236540`, `236540`, `236525`, `2367`, `2368` |
| Retenciones a Favor | `puc_anticipo_retefuente_code`, `puc_anticipo_reteica_code`, `puc_pasarela_retencion_code` | `135515`, `135518`, `135595` |
| Control Financiero | `puc_cxc_clientes_code`, `puc_cxp_proveedores_code`, `puc_otros_ingresos_code`, `puc_gastos_financieros_code`, `puc_bancos_code` | `130505`, `220505`, `4210`, `5305`, `111005` |

`puc_inventory_purchase_code`, `puc_costo_ventas_code` y `puc_comisiones_mecanicos_code` se agregaron en rev. 25: el backend (`billing.controller.js`) ya los leía para el asiento de costo de repuestos/comisiones, pero no estaban en la lista blanca de campos editables por el contador (`updateWorkshop`) — el primero se descartaba en silencio al guardar (siempre caía a su default `1435`), los otros dos ni tenían input en ninguna pantalla.

`puc_bancos_code` se agregó en rev. 39: bug real corregido en `finance.controller.js:setOpeningBalance` — el asiento del **Saldo Inicial** (Panel Financiero → "Configurar Saldo Inicial", el efectivo migrado del cuaderno/Excel previo del taller) se registraba contra `puc_otros_ingresos_code` (una cuenta de INGRESO), como si el capital que el taller ya tenía en el banco antes de usar EFISCO fuera una venta del período. Ahora usa esta cuenta dedicada (PUC clase 11 "Disponible"), configurable por el contador en el bloque "Control Financiero" de `ContadorPanel.jsx`.

*IVA por Categoría de Repuesto* — 10 categorías con tasa individual configurada que se aplica automáticamente al seleccionar categoría en Inventario.

*Exportación contable — Libro Mayor único* (reemplaza los 6 reportes CSV anteriores — facturas/compras/CxC/CxP/ledger/inventario valorizado, cada uno con columnas distintas y sin protección contra inyección de fórmulas): un solo Excel real (`.xlsx`, vía `exceljs`) con la estructura `Fecha | Código PUC | Cuenta | Detalle | Tercero | ID | Débito | Crédito | Saldo Acumulado`, todo el histórico del taller (o un rango `from`/`to`) en orden cronológico. "Tercero" se resuelve por `work_order_id` (ventas, join a `work_orders`) o `related_purchase_id` (compras, join a `supplier_purchases → providers` — columna nueva en `cash_flow_ledger`, poblada desde `registerPurchase`/pago de cuotas hacia adelante; los asientos anteriores a esta migración quedan sin tercero resuelto, no hay forma confiable de reconstruirlo desde el texto libre de `concept`).

*Integración Dataico*: cada taller tiene su propia sub-cuenta (creada manualmente por Efisco dentro de la cuenta maestra) — se configura aquí el ID de cuenta, el token, número de resolución DIAN + prefijo, y departamento/ciudad (DIVIPOLA). El entorno (`PRUEBAS`/`PRODUCCION`) ya no es un toggle manual: se detecta solo según si corre en Vercel o en local. Botón de prueba de conexión contra la API real de Dataico.

---

## Panel de Administración EFISCO

Panel interno en `/admin` con autenticación completamente separada de los talleres — ver [SECURITY.md — Autenticación Admin](SECURITY.md#autenticación-admin-panel-efisco).

### Dashboard

4 KPIs: talleres activos/totales, **servicios del mes**, ingreso real de EFISCO en el mes, pagos de comisiones pendientes. El KPI de ingreso (`getStats.revenue_month`) suma `efisco_accounting_ledger` con `direction='income'` — mismo criterio que usa la pestaña [Contabilidad](#contabilidad-admincontabilidad) — no la venta bruta de los talleres a sus propios clientes (bug corregido: antes sumaba `cash_flow_ledger`/`INC_GROSS` de todos los talleres, un número sin relación con lo que el taller le paga a EFISCO — podía superar los $40M con solo un puñado de talleres activos). El cobro real de EFISCO es por **servicio liquidado** (`software_valuation_unit` × servicios de la orden en `settleOrder`, acumulado en `pending_service_fees`), nunca por el valor de la orden.

**"Servicios del mes" cuenta servicios, no órdenes** (bug corregido en rev. 44): `getStats.services_month` (y las mismas cifras en `getWorkshops`/`getWorkshopDetail`, ver [Talleres](#talleres) abajo) usaban `orders_month` — contaban ÓRDENES completadas del mes, no servicios. Una orden puede agrupar varios servicios (`work_order_services`) y cada uno se cobra por separado, así que el KPI podía mostrar un número menor al que realmente se factura (ej. una orden con 3 servicios sumaba 1, no 3). El helper nuevo `countCompletedServices` (`admin.controller.js`) replica el mismo criterio que ya usa `settleOrder` para acumular `pending_service_fees`: cuenta filas de `work_order_services` por orden completada del mes en curso, o 1 si la orden es legacy sin filas hijas.

### Talleres

Lista paginada y buscable de todos los talleres:
- **Búsqueda con debounce real** — bug corregido en rev. 42: el `setTimeout` de 400ms vivía dentro del `onChange` (`handleSearch`); el `return () => clearTimeout(t)` de un event handler nunca lo ejecuta nadie, solo funciona dentro de un `useEffect` — cada tecla disparaba un fetch nuevo sin cancelar los anteriores, con el riesgo real de que una respuesta vieja pisara resultados más nuevos. Movido a un `useEffect` con `clearTimeout` real en el cleanup
- Ver configuración completa (grid de 11 campos, incluye "Servicios mes" — servicios liquidados del mes en curso, no órdenes, ver arriba — y conteo/total de "Consultas de puntaje" — derivado de `pending_score_query_fees / 100`, sin contador dedicado)
- Columna "Serv./Mes" en la tabla principal (antes "Órd./Mes" — mismo fix de servicios-no-órdenes) y columna "Consultas Score": conteo + total en pesos de consultas detalladas de score cobradas a ese taller
- **Contrato de Afiliación** (nueva sección en el detalle del taller, rev. 45): antes la prueba legal de firma (`workshop_contract_acceptances`) se guardaba pero nadie podía verla — ni el propio admin. Muestra estado firmado/sin firmar, fecha, versión del contrato, IP y el hash de firma (`GET /api/admin/workshops/:id` ahora hace `SELECT` a `workshop_contract_acceptances`, ver [API.md](API.md#admin-apiadmin) y [SECURITY.md](SECURITY.md#decisiones-técnicas-de-seguridad)); botón "Ver contrato" reutiliza `ContractViewerModal.jsx` (el mismo visor de solo lectura de `Config.jsx`) para leer el texto completo firmado con el mismo banner de constancia
  - **Descargar PDF** (rev. 46): junto a "Ver contrato", nuevo botón que descarga el PDF firmado del contrato del taller — `GET /api/admin/workshops/:id/contract-pdf` (`admin.controller.js#getContractPdf`), gateado por `requireAdmin`. Reutiliza el mismo servicio `contractPdfStorage.service.js` (`generateAndStoreContractPdf` / `getContractPdfSignedUrl`) que ya usaba el dueño desde `/config` (`GET /api/auth/contract-pdf`), incluyendo el self-heal: si la fila de auditoría no tiene `pdf_storage_path` (la generación falló al momento de firmar), lo regenera bajo demanda resolviendo el email del dueño vía `supabaseAdmin.auth.admin.getUserById`. Antes el admin solo podía leer el contrato en pantalla, sin forma de obtener el PDF para archivo/soporte
- Crear taller nuevo (crea usuario Supabase Auth + `workshop_config` en transacción)
- **Suspender / Reactivar** — `PATCH /api/admin/workshops/:id/toggle` actualiza `is_active` en `workshop_config`
  - La suspensión bloquea el login (`auth.controller.js`) Y los requests en curso (`auth.middleware.js`)
  - El taller suspendido ve una pantalla de "Cuenta suspendida" con canales de contacto
  - **Con confirmación** (mejora de UX, rev. 43): antes se ejecutaba con un solo clic, sin ningún paso intermedio, a diferencia de "Eliminar" que sí lo tenía — un clic accidental bloqueaba de inmediato el acceso de TODO el personal de un taller que paga por el servicio. Ahora pasa por `ElegantConfirmModal` (mensaje distinto según se vaya a suspender o activar)
- **Eliminar (registro no activado)** — `DELETE /api/admin/workshops/:id`, botón visible solo para talleres con `admin_activated_at` vacío (autorregistros que un admin nunca activó). Borra en cascada todos los datos del taller y notifica por correo al dueño vía Resend (best-effort — si el correo falla, el borrado ya se considera exitoso). Un taller ya activado no muestra este botón — para ese caso sigue siendo "Suspender". Ver el razonamiento de por qué esto es seguro en [SECURITY.md](SECURITY.md).
- **Llave de acceso a cuenta del cliente** — ver flujo completo en [SECURITY.md](SECURITY.md#llave-de-acceso-a-cuenta-del-cliente-soporte)
- **Factura propia de EFISCO al taller — EFISCO como proveedor real** (`documentation/Ampliación software.txt`, líneas 160 y 185): el admin sube el PDF y el monto/método de pago (`POST /api/admin/workshops/:id/software-invoice`) y el backend, en vez de guardar un registro aislado, provisiona (la primera vez, por taller) un proveedor real `EFISCO S.A.S.` en la tabla `providers` del propio taller — con los valores de la **plantilla editable** `efisco_provider_template` (ver Cobros abajo), y liquida la factura con **el mismo motor** que cualquier otra compra a proveedor (`financialEngine.liquidateSupplierPurchase`), aplicando ReteFuente/ReteICA/IVA/GMF **según la configuración fiscal que el contador le haya dado al taller** (Régimen Simple no retiene, dos grandes contribuyentes no se retienen entre sí, etc. — el gate ya existente se aplica también a EFISCO, tal como pide la documentación). La compra queda registrada en `supplier_purchases` y en `cash_flow_ledger` (igual que `registerPurchase`), y la fila de seguimiento en `efisco_software_invoices` se liga a esa compra vía `supplier_purchase_id`. El dueño ve la factura simple (monto/vencimiento/PDF) como "Gasto de Software" en `Config.jsx` (`GET /api/workshop/software-invoices`, signed URL de 1h) y ve el detalle completo de la liquidación (retenciones, IVA, historial) como cualquier otro proveedor en **Proveedores** (con una insignia "Proveedor EFISCO", sin botón de editar ni de eliminar — `updateProvider`/`deleteProvider` responden 403 si `is_system_provider` es true — y sin botón de "Registrar Compra", porque el taller no le compra nada a EFISCO: es EFISCO quien le factura al taller)

### Cobros (`/admin/cobros`)

Seguimiento de las facturas que EFISCO le emite a los talleres (`documentation/Ampliación software.txt`, línea 162: *"activar o desactivarle el software al cliente en caso de que pague o no"*):
- Card superior **"Plantilla proveedor EFISCO"**: NIT/nombre/ciudad/teléfono/régimen/IVA/declarante/tasa ICA/código PUC — única fuente de verdad de la identidad de EFISCO como proveedor (`GET/PUT /api/admin/efisco-provider-template`, tabla `efisco_provider_template`, fila única). Cada actualización se **sincroniza automáticamente** hacia la fila "EFISCO S.A.S." que cada taller ya tenga creada — el taller no puede editar su copia (`updateProvider` responde 403 sobre `is_system_provider`), evitando que un contador local desalinee los datos fiscales reales de EFISCO. `updateProviderTemplate` hace UPSERT (crea la fila si no existe, actualiza si ya existe) — antes solo sabía actualizar, así que si la tabla estaba recién migrada y vacía, quedaba permanentemente inutilizable ("corre la migración" para siempre aunque ya hubiera corrido); el frontend ya no bloquea la pantalla en ese caso, muestra el formulario vacío para llenar.
- Tabla de todas las facturas de todos los talleres (`GET /api/admin/software-invoices?status=`), enriquecida con nombre/NIT/estado del taller y link de descarga (signed URL). Marca "Vencida" en el frontend si `due_date` ya pasó y sigue `pendiente`.
- **Marcar como pagada** (`PATCH /api/admin/software-invoices/:id/mark-paid`): cambia `status` a `pagado` + `paid_at`, y **reactiva el taller** (`workshop_config.is_active = true`) en la misma operación — decisión explícita: el pago del software es lo que condiciona el acceso.

### Pagos

Gestión de solicitudes de pago de comisiones por referidos:
- Filtros: pendiente / en proceso / pagado
- KPIs por estado
- `PATCH /api/admin/payouts/:id/mark-paid`

### Referidos

Árbol jerárquico recursivo de referidos con colapso/expansión:
- Colores por nivel: gris (0), azul (1-2), púrpura (Platino: 3+)
- KPIs: total nodos, referidores activos, Platinos, comisiones totales
- Datos bancarios del taller (banco, tipo de cuenta, número) visibles bajo cada nodo con al menos un referido, para que EFISCO pueda procesar el pago de la comisión sin saltar a otra pantalla — aviso ámbar si el taller aún no los registró

### Contabilidad (`/admin/contabilidad`)

Caso de uso 2 de Dataico: EFISCO como emisor de sus propios ingresos/costos (`dataico.service.js`: `buildSupportDocNoResidente`, `buildSupportDocBasico`, `emitSupportDocument`, `emitEfiscoInvoice`), ya implementado en código:
- **Facturación propia de EFISCO**: resolución/prefijo DIAN y consecutivo propios (`efisco_billing_config`, fila única — mismo patrón UPSERT que la plantilla de proveedor en Cobros, mismo bug corregido: antes quedaba inutilizable si la tabla estaba vacía). **Probado end-to-end en producción real (rev. 28)**: la tabla nunca había tenido una fila hasta esta revisión — se pobló con la resolución/prefijo de Factura Electrónica de Venta de la cuenta real de EFISCO y se emitió un documento real (`POST /admin/accounting/subscription-invoice`, `DIAN_ACEPTADO`, CUFE real). De paso se corrigieron 2 bugs reales que nunca se habían ejercitado contra la API real de Dataico: `emitEfiscoInvoice` leía `workshop.legal_address_line` para la dirección del cliente facturado, pero esa columna no existe (`workshop_config.address` es la real) — la dirección siempre salía "N/A"; y `emitSubscriptionInvoiceForWorkshop`/`recordWorkshopSideExpense` guardaban `dataico_document_id` leyendo `dataicoResult?.id || dataicoResult?.document_id`, campos que la respuesta real de Dataico nunca trae (trae `cufe`/`uuid`/`pdf_url`) — el CUFE real quedaba sin registrar en el ledger propio de EFISCO ni en el número de factura del espejo de gasto del taller.
- **Captura manual** de lo que ya se cobró/pagó por fuera del sistema (sin pasarela de cobro real todavía): Infraestructura No Residente (AWS/Vercel/etc., convierte a COP con la TRM del día — Documento Soporte a No Residente, SKU "costos infraestructura"), MP/WhatsApp/Dataico, Salarios/Honorarios (Documento Soporte Básico), Suscripción de Taller.
- **"Suscripción Taller" fusiona las dos contabilidades del mismo hecho económico en un solo paso** (`subscriptionBilling.service.js:emitSubscriptionInvoiceForWorkshop`): calcula el precio con descuento por referidos, emite la factura DIAN real de EFISCO al taller, registra el ingreso en el libro de EFISCO, **y además** dispara automáticamente el gasto espejo en la contabilidad del propio taller (mismo motor `financialEngine.liquidateSupplierPurchase` que usa "Talleres → Gasto de Software", mismo monto ya descontado, sin pedir PDF — la factura Dataico recién emitida es el respaldo legal). Checkbox opcional "Aplicar 4x1000 (GMF)" para cuando el pago real fue por transferencia bancaria (apagado por defecto). El botón manual de "Talleres → Gasto de Software" sigue existiendo intacto como vía alterna para casos puntuales (backdating, ajustes). El webhook real de Mercado Pago (`POST /api/billing/webhook/mercadopago`) ya usa esta misma función: el checkout (`POST /api/finance/subscription/checkout`) cobra por la pasarela el monto YA con descuento por referidos aplicado (mismo tier que calcula esta función), y codifica el monto sin descontar como `workshopId:baseAmount` en el `external_reference` de Mercado Pago para que el webhook sepa cuánto saldar de `pending_service_fees` al facturar — antes el checkout cobraba el total sin descuento y esta función facturaba después por menos, así que un taller con descuento pagaba de más por la pasarela frente a su propia factura DIAN (bug corregido en rev. 25).
- **Las consultas de score facturables ahora se cobran en el mismo ciclo que la tarifa por servicio** (bug corregido en rev. 44): `pending_score_query_fees` (ver [Recepción](#recepción) — $100 COP por consulta con ≥30 días de historial, `verifyScoreOTP`) se acumulaba desde que existe la funcionalidad, pero ningún flujo de cobro lo tocaba ni lo reseteaba — ni el checkout de Mercado Pago del taller ni la factura manual del admin lo incluían; el único sitio que lo tocaba era un script interno de borrado de datos de prueba, así que solo crecía para siempre sin reflejarse nunca como cobrado. Decisión del usuario (vía AskUserQuestion): sumarlo a la misma factura de suscripción, manteniendo la diferencia visible para que el taller sepa qué se le está cobrando. Ahora `emitSubscriptionInvoiceForWorkshop` reparte el monto del ciclo primero contra `pending_service_fees` y el resto contra `pending_score_query_fees` (saldando ambos), dejando el desglose ("tarifa por servicio: $X + consultas de puntaje: $Y") en la descripción de la factura Dataico, el ledger propio de EFISCO y el concepto del libro mayor del taller cuando aplica; el checkout de Mercado Pago y la factura manual del admin ahora suman ambos campos por defecto, y `AdminAccounting.jsx` precarga el monto ya sumado con un hint del desglose y un toast de confirmación que distingue ambas partes.
- Comisiones de referidos pagadas disparan un Documento Soporte Básico (sku "pago referido") de forma no bloqueante — el pago ya quedó marcado como completado aunque Dataico falle.
- Ledger contable propio de EFISCO (`efisco_accounting_ledger`, sin `workshop_id`) con totales de ingreso/costo/neto por SKU.
- ~~El bloqueo para emitir esto en producción real no es de código: EFISCO necesita completar su propia habilitación de facturación electrónica ante la DIAN~~ — **resuelto en rev. 28**: EFISCO ya completó la habilitación DIAN (numeración asociada a Dataico como proveedor tecnológico) y Caso 2 quedó probado en real (ver arriba). El único requisito operativo que sigue siendo 100% manual es asociar la resolución/prefijo dentro del propio dashboard de Dataico (Ventas → Facturas → Numeraciones, botón "seleccionado") además de tramitarla ante la DIAN — tener la autorización de la DIAN no basta, Dataico necesita su propio paso de activación interno.

Las rutas API completas del panel admin están en [API.md — Admin](API.md#admin).
