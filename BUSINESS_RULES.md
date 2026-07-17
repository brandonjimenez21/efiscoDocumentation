# Reglas de Negocio y Módulos del Sistema

> Ver también: [README](README.md) · [FINANCIAL_ENGINE](FINANCIAL_ENGINE.md) · [API](API.md) · [SECURITY](SECURITY.md)

## Rutas del frontend

| Ruta | Módulo | Acceso |
|:---|:---|:---:|
| `/` | Landing — página de presentación pública (funciones, demo, CTA registro/login) | Público (sin sesión) |
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

---

## Recepción

Punto de ingreso rápido de vehículos. Registra cédula, nombre y apellido por separado, teléfono y síntoma reportado — la cédula se pide aquí mismo para poder activar el score crediticio de inmediato (antes requería que el cliente completara el Registro Seguro EFISCO primero). El nombre y apellido separados existen para poder facturar electrónicamente con los campos exactos que exige Dataico/DIAN (`first_name`/`family_name`); el resto de la app sigue mostrando el nombre combinado. Los tres campos validan formato en vivo: cédula (solo dígitos, 3–10), nombre/apellido (solo letras) y WhatsApp (solo dígitos, exactamente 10 — formato celular colombiano).

- Botón de WhatsApp por cada ingreso en cola: envía el link de **Registro Seguro EFISCO** (`/cliente/registro/:workshopId/:intakeId`), vinculado tanto al taller como al ingreso específico
- **Score de crédito en dos niveles**: el score básico (local + global, sin desglose) se ve gratis con un solo clic, sin código — el desglose por pilar (pago 60% / estabilidad 10% / fidelidad 30%) sí exige verificación por OTP (código de 6 dígitos al WhatsApp del cliente) y es el único punto donde se cobra el SKU "consulta puntaje" ($100 COP). El código de OTP se envía primero por una plantilla de WhatsApp de categoría Autenticación (`codigo_verificacion_score`, entrega sin ventana de 24h), con fallback a texto libre + código de demo si la plantilla aún no está aprobada por Meta. Vence a los 30 minutos (el cliente puede irse del taller antes de que le llegue el mensaje)
- El reporte en PDF con el desglose completo se **genera en el servidor (Puppeteer/Chromium headless) y se envía automáticamente por WhatsApp** al cliente apenas se verifica el OTP (`verifyScoreOTP` → `scorePdfRenderer.service.js` → `whatsappService.sendDocument`) — best-effort, no bloquea la respuesta si WhatsApp falla o no está configurado. Recepción ya no tiene botón de descarga propio; el PDF es exclusivo del cliente (ahí o desde su propio link de Registro Seguro EFISCO, `GET /api/clients/public-score-pdf/:workshopId/:intakeId`) — es exactamente el mismo documento en ambos casos, generado una sola vez en el backend

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
- **Mano de obra auto-calculada por mecánico**: tarifa hora = `monthly_salary ÷ 30 ÷ 8`, multiplicada por las horas estimadas de esa fila — editable (al editar manualmente deja de recalcular; editar una fila recalcula también el resto de filas en modo automático). **Mecánicos 100% comisión** (sin `monthly_salary` propio) no usan ese cálculo: su tarifa hora es el **promedio del sueldo mensual de los mecánicos de nómina fija asignados a esa misma orden** (`hourlyRateFor` en `Bahia.jsx`); si todos los asignados son de comisión o solo hay uno, cae al **SMLV + prestaciones ($2.860.000/mes)** como piso legal. La comisión de cada mecánico se calcula sobre SU porción de M.O.; la utilidad de inventario se reparte en partes iguales entre los asignados (`getMechanicMetrics`). Al liquidar, cada mecánico `comision`/`mixto` genera un asiento `MECH_COMMISSION` (devengo, no pago) en el Libro Auxiliar — ver [FINANCIAL_ENGINE.md](FINANCIAL_ENGINE.md#caja-real-vs-informativodevengo--por-qué-existe-la-distinción)
- **Solo rol `mecanico` es asignable a órdenes** — el dropdown de Bahía filtra por rol y el backend rechaza con 400 cualquier `mechanic_id` de un empleado con otro rol (contador/admin), en `createWorkOrder` y `updateWorkOrder`
- **Fecha/hora de entrega solo futura** — inputs con `min` (fecha local, no UTC — en Colombia UTC-5 el `toISOString()` marcaría "mañana" desde las 7pm) + guard en el submit + validación espejo en el backend (400 si `estimated_delivery_at` < ahora − 2 min; en update solo si el valor cambió, para no bloquear la edición de órdenes ya vencidas)
- Consumo de inventario con registro automático en el Kardex; cada ítem guarda `vat_percentage` (0%, 5% o 19%)
- Estado de la orden: `pending → ejecucion → ready_to_invoice → completed`
- **Modal de Liquidación** con pre-cálculo en vivo:
  - Simulación de comisiones Bold/Addi antes de confirmar
  - Selector de tipo de tarjeta (Débito/Crédito) en Bold físico/online, y tipo de movimiento (Débito/Crédito) en Addi — necesario para mapear correctamente el medio de pago que exige Dataico, ya que ninguna de las dos pasarelas distingue esto en su "tipo de transacción"
  - Retenciones si el cliente es agente retenedor
  - Modo crédito: selector de cuotas (2/3/4), fecha del primer pago
- Al liquidar se emite factura a Dataico/DIAN de forma no bloqueante bajo la sub-cuenta propia del taller; si tiene éxito guarda `cufe` e `invoice_pdf_url`. Si el cliente es agente retenedor y la base supera el umbral UVT, la factura incluye el bloque `retentions` (RET_FUENTE/RET_ICA) con la misma tasa y monto que ya calculó `financialEngine` (ver [FINANCIAL_ENGINE.md](FINANCIAL_ENGINE.md)); si no, se envía `retentions: []` (estructura básica). Con multi-servicio, la factura lleva **un ítem por servicio** (cada uno con su porción de M.O. y su margen propio) + un ítem por repuesto (`kind: 'service' | 'part'`), y en los textos de display (WhatsApp, cuotas, comprobante) el servicio aparece como "A + B"
- Al confirmar la liquidación aparece un toast "Orden liquidada con éxito" (abajo a la derecha) — antes la orden solo desaparecía de la bahía sin ningún aviso
- **Comprobante de Venta por WhatsApp**: apenas se liquida, se genera y se manda automáticamente un PDF al cliente (`saleReceiptTemplate.js`, mismo motor Puppeteer que el resto de los documentos) — best-effort, no bloquea la liquidación. Contado: pago recibido en su totalidad. Crédito: plan de cartera pactado con la tabla completa de cuotas. Incluye razón social/NIT/dirección/teléfono del taller, ítems detallados, IVA y comisión de pasarela desglosada (base + IVA de la comisión por separado), y saldo neto real en caja — mismos campos que `documentation/Plantilla de comprobante de venta.txt`, con el lenguaje visual de tarjetas del resto de la app
- **Historial (`/ordenes`)**: la vista de órdenes completadas muestra qué mecánico ejecutó cada orden (columna en la tabla + tarjeta en el detalle), vía join con `employees` en `getCompletedOrders` — visible para el dueño y para el propio mecánico en sus órdenes

---

## Inventario

Control de existencias con trazabilidad completa. El diagrama del flujo Kardex está en [ARCHITECTURE.md](ARCHITECTURE.md#5-inventario-y-kardex-inmutable).

- **Ficha de repuesto ampliada**: además de Nombre/Categoría/Costo/Margen, el formulario de alta separa dos catálogos que antes estaban mezclados en un solo selector — **Unidad de Medida** (mm, cm, in, ml, L, gal, g, kg, A, V, W: la unidad física del repuesto) y **Unidad de Inventario** (N/A / Unidad / Par / Kit: cómo se cuenta/empaqueta), más un campo libre **Número** (referencia del fabricante/proveedor). Ambos catálogos son columnas visibles en la tabla principal de Inventario
- **Kardex inmutable**: cada movimiento genera una transacción en `inventory_transactions`
- **Descuento de stock 100% en código de aplicación** (`inventory.controller.js:addItemToWorkOrder`), respetando si el ítem es de stock actual o nueva facturación — requiere haber corrido la migración 5, que elimina el trigger de base de datos que antes duplicaba el descuento
- **Consumo gradual para líquidos/químicos** (categoría "Lubricantes y Químicos", decidido con el usuario vía AskUserQuestion: "el mecánico decide CUÁNDO baja el stock"): añadir uno de estos repuestos a una orden **no descuenta stock físico automáticamente** — el cliente se cobra igual, pero `current_stock` no baja hasta que el mecánico marca explícitamente el checkbox "¿Se agotó el contenedor?" (`container_emptied`) en el modal de Bahía. Evita que una orden de cambio de aceite baje el contenedor completo del stock aunque solo se haya usado una fracción de él
- **IVA por categoría**: al seleccionar la categoría del repuesto, el porcentaje de IVA se aplica automáticamente desde `workshop_config.category_vat_rates`
- **Rotación y stock mínimo vital persistidos** (Fase 3, `utils/inventoryMetrics.js`): `rotación = stock_actual / uso_promedio_por_servicio`; `min_stock_vital = stock_actual × (1 + rotación)` (rotación capada a 20x). Se recalcula y **guarda** en `inventory.min_stock_vital` tras cada movimiento de stock (compra, consumo, ajuste manual) — antes se calculaba al vuelo solo en el tab Matriz y nunca se persistía, dejando las alertas de stock bajo del Dashboard, la liquidación y la lista principal de Inventario siempre en el valor por defecto
- **Tab "Matriz"**: rotación, uso promedio, stock mínimo vital y valor de inventario por ítem
- `getItemHistory` ordena por `requested_at`

---

## Proveedores y Egresos

Gestión de proveedores y registro de compras con liquidación fiscal (ver reglas de cálculo en [FINANCIAL_ENGINE.md — Liquidación de compras](FINANCIAL_ENGINE.md#2-liquidación-de-compras-liquidatesupplierpurchase)).

- **Perfil tributario del proveedor** (4 regímenes): Persona Natural · Régimen Simple · Régimen Ordinario · Gran Contribuyente
  - `simple`: no aplican retenciones
  - `ordinario` / `gran_contribuyente`: retenciones plenas según UVT (27 UVT ≈ $1.358.586)
- **Tasa de ReteICA por proveedor** (`reteica_rate_supplier`, en por-mil — ej. `9.66` = 0.966%, mismo formato que `workshop_config.reteica_rate`)
- **OCR de facturas**: AWS Textract extrae proveedor, ítems, valores. Tras el escaneo aparece un **banner de verificación** con los datos extraídos, para que el mecánico/dueño los confirme o corrija antes de registrar la compra — el OCR nunca escribe directo a la base sin pasar por esta revisión. El IVA/impuesto que trae la factura escaneada se toma explícito del OCR (`invoiceVatAmount`) en vez de derivarlo siempre al 19% plano en `liquidateSupplierPurchase`, corrigiendo egresos mal calculados con facturas de tasas distintas
- **Método de pago**: `banco` → GMF 4×1000 | `tarjeta` → costo de transacción | `efectivo` → sin costos
- **Código PUC por proveedor** (`puc_account_expense`): usado en asiento del ledger si está definido
- **Proveedor EFISCO de solo lectura para el taller**: nombre, NIT, ciudad, teléfono, régimen, tasa ICA y código PUC vienen únicamente de `efisco_provider_template` (admin-efisco, ver [Panel de Administración EFISCO — Cobros](#cobros-admincobros)) — `updateProvider` responde 403 si el proveedor es `is_system_provider`, y el botón de editar ni siquiera aparece en el frontend
- **Eficiencia de proveedor** (Fase 3, tab "Eficiencia"): al registrar una compra se puede indicar `order_placed_at` (fecha del pedido); se compara el tiempo de entrega de la última compra contra el promedio de las últimas 5 compras con fecha de pedido registrada (`GET /api/providers/efficiency`, `utils/providerEfficiency.js`). Solo hacia adelante — las compras registradas antes de este campo no tienen fecha de pedido

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

- Lista de cuotas pendientes con fecha de vencimiento
- Registro de pago: genera asiento `INC_GROSS` con PUC `puc_income_code || '4135'`
- **Comprobante de Pago por WhatsApp**: al marcar una cuota como pagada se genera y se manda automáticamente un PDF (`paymentReceiptTemplate.js`) con el monto recibido, "cuota X de Y" y el saldo pendiente (o "✓ crédito pagado en su totalidad" si era la última) — best-effort, no bloquea el registro del pago. El teléfono se toma de `work_orders.client_phone` (siempre disponible, sin importar si el cliente vino por el link público o se cargó manualmente en Bahía)

---

## Flujo de Caja (`/flujo-caja`)

Libro mayor de todos los movimientos financieros (tipos de movimiento documentados en [FINANCIAL_ENGINE.md — Libro Auxiliar](FINANCIAL_ENGINE.md#libro-auxiliar-cash-flow-ledger)).

- Filtro por rango de fechas y tipo de impacto (CREDIT / DEBIT / Todos)
- Agrupación por día con subtotales diarios
- Balance acumulado por movimiento (`running_balance`)
- Descarga CSV vía `/api/finance/report/ledger`
- **Deep-link con rango explícito**: `?from=YYYY-MM-DD&to=YYYY-MM-DD` inicializa los filtros (usado por el modal de libros mensuales y el botón "Ver todo el mes" del dashboard financiero); sin params, el default sigue siendo el mes en curso
- **Capital Inicial separado de Ingresos** (rev. 22): el saldo migrado del cuaderno/Excel previo (`OPENING_BALANCE`) tiene su propia tarjeta de resumen — antes se sumaba dentro de "Total Ingresos" e inflaba el ingreso real del período con dinero que ya existía antes de usar Efisco
- **Tarjeta "Costos"** (rev. 22): agrupa `INV_COGS` + `MECH_COMMISSION` + `MECH_SALARY_PAY` + `MAN_EGR` + `TAX_GMF` + `CARD_FEE` + `GW_FEE` + `GW_VAT` + `FOLIO_ADV` — el costo de un repuesto/mano de obra se reconoce al **liquidar** la orden (consumo real), no al comprar el repuesto ni al pagarle la comisión al mecánico; `MECH_COMMISSION_PAY` (pago en efectivo de una comisión ya devengada) se excluye para no contar el costo dos veces. Ver el porqué completo en [FINANCIAL_ENGINE.md](FINANCIAL_ENGINE.md#caja-real-vs-informativodevengo--por-qué-existe-la-distinción)
- **Pago a mecánicos reflejado aquí**: el botón "Pagar a Mecánico" de `/config` (ver [Configuración del Taller](#configuración-del-taller-config)) genera `MECH_SALARY_PAY`/`MECH_COMMISSION_PAY` con `mechanic_id`, visibles como cualquier otro movimiento del Libro Auxiliar

### Libros Mensuales archivados

Cada mes terminado se archiva como un **libro cerrado con snapshot** (tabla `monthly_ledger_books`, UNIQUE `(workshop_id, period)`): totales de ingresos/egresos/neto, cadena de saldos (`opening_balance` = `closing_balance` del mes anterior) y las filas del ledger congeladas en `snapshot` (jsonb). El **cierre es perezoso** — no hay cron (Render free): la primera consulta a `GET /api/finance/monthly-books` después del cambio de mes genera los libros faltantes, con upsert idempotente para que dos requests simultáneos no dupliquen. El mes en curso nunca se cierra. Se excluyen los `INC_GROSS` con `net_amount = 0` (marcadores de reconocimiento de crédito, mismo filtro del dashboard).

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

**1. Datos del Taller** — Nombre, dirección, teléfono, horarios, costos fijos (arriendo + servicios). El teléfono se usa en el encabezado del Comprobante de Venta

**2. Mi Equipo & Roles** — Alta de empleados, esquemas de compensación (fijo / comisión / híbrido). Crear, editar, desactivar empleados y habilitar sus credenciales de acceso está restringido al rol `owner`. **El dueño también puede registrarse a sí mismo como mecánico** (`POST /api/mechanics/self`, upsert sobre `employees.user_id = owner.id`, sin necesitar email/password propios) para llevar su propio sueldo/comisión separado de los costos fijos genéricos del taller y aparecer como cualquier otro mecánico en el dropdown de asignación de Bahía, con sus propias métricas. La tarjeta del dueño en el grid ya no muestra el botón "ver detalles" (redundante — el banner "Tu Perfil de Mecánico" tiene su propio "Ver mis datos"), y tanto el banner como el header del modal de detalles usan avatar redondo consistente con las demás tarjetas. Nota: el rol `contador` NO es asignable como mecánico en órdenes (gate en frontend y backend, ver Bahías)

- **"Pagar a Mecánico"** (rev. 22, dentro del detalle de cada empleado): los mecánicos `comision`/`mixto` ya generan su comisión automáticamente por orden (asiento `MECH_COMMISSION`, devengo — ver [FINANCIAL_ENGINE.md](FINANCIAL_ENGINE.md#caja-real-vs-informativodevengo--por-qué-existe-la-distinción)); este botón es para registrar el **pago real en efectivo**, sea el sueldo fijo o la comisión ya devengada. Muestra primero lo pendiente por pagar (`GET /api/mechanics/:id/pending-payment` = `Σ MECH_COMMISSION − Σ MECH_COMMISSION_PAY`), y un modal para capturar los montos de sueldo/comisión a liquidar (`POST /api/mechanics/:id/pay`) — el pago queda reflejado de inmediato en Flujo de Caja

**3. Catálogo de Servicios** — CRUD con márgenes básico/premium por tipo de vehículo

**4. Pasarelas**

Editable directamente por el dueño (no requiere al contador):

*Tasas de pasarelas*: Bold físico (2.99%), Bold online (3.49%), Addi (10.5%)

*Anticipo de Folios*: calculadora de recarga de folios Dataico (cantidad × costo unitario + margen + comisión de pasarela), con registro automático del egreso en el flujo de caja

**5. Módulo del Contador**

*Parámetros Fiscales y de Recaudo* — visible aquí en modo solo lectura para el dueño (la edición real vive en el panel del contador, `/contador`). Régimen fiscal, tasas, los 23 códigos PUC, credenciales Dataico e identidad legal son campos **exclusivos del rol `contador`** — el backend (`updateWorkshop`) los descarta en silencio si el llamador no es contador, ni siquiera el dueño puede modificarlos vía API, solo leerlos (ver [SECURITY.md](SECURITY.md#decisiones-técnicas-de-seguridad)):

Régimen Fiscal (4 opciones):
| Opción | IVA | Reg. Simple | Agente Retenedor |
|:---|:---:|:---:|:---:|
| No Responsable de IVA | ✗ | ✗ | ✗ |
| Régimen Simple (SIMPLE) | ✓ | ✓ | ✗ |
| Régimen Ordinario | ✓ | ✗ | ✓ |
| Gran Contribuyente | ✓ | ✗ | ✓ |

Tasas: IVA (19%), ReteICA (‰), ReteFuente declarantes/no declarantes, ReteIVA (15%)

*Identidad Legal*: NIT, Razón Social, Prefijo — edición exclusiva del rol `contador` (el dueño solo lee). (No existe un campo de "Clave Técnica DIAN" propia del taller: bajo el modelo de sub-cuentas de Dataico, el CUFE lo calcula Dataico internamente por cada sub-cuenta — account_id + token + resolución —, el taller nunca necesita ni envía una clave técnica propia.)

*Presets PUC* — 3 botones que auto-rellenan los 21 códigos según régimen:
- **Régimen Ordinario** (estándar DIAN)
- **Régimen Simple** (subcuentas simplificadas)
- **Gran Contribuyente** (subcuentas retención en la fuente)

*Plan Único de Cuentas — 21 códigos en 5 bloques*:

| Bloque | Códigos | Defaults |
|:---|:---|:---|
| Ingresos & Ventas | `puc_income_code`, `puc_parts_income_code`, `puc_gateway_fee_code`, `puc_gateway_vat_code` | `4135`, `4135`, `5290`, `2408` |
| IVA | `puc_iva_generated_code`, `puc_iva_generated_5_code`, `puc_iva_deductible_code`, `puc_devolucion_iva_code` | `240805`, `240810`, `240820`, `135520` |
| Retenciones por Pagar | `puc_retefuente_code`, `puc_retefuente_compras_decl_code`, `puc_retefuente_compras_nodecl_code`, `puc_retefuente_servicios_code`, `puc_reteiva_code`, `puc_reteica_code` | `2365`, `236540`, `236540`, `236525`, `2367`, `2368` |
| Retenciones a Favor | `puc_anticipo_retefuente_code`, `puc_anticipo_reteica_code`, `puc_pasarela_retencion_code` | `135515`, `135518`, `135595` |
| Control Financiero | `puc_cxc_clientes_code`, `puc_cxp_proveedores_code`, `puc_otros_ingresos_code`, `puc_gastos_financieros_code` | `130505`, `220505`, `4210`, `5305` |

*IVA por Categoría de Repuesto* — 10 categorías con tasa individual configurada que se aplica automáticamente al seleccionar categoría en Inventario.

*Exportación contable*: CSV de facturas, compras a proveedores, CxC, CxP, libro fiscal e inventario valorizado.

*Integración Dataico*: cada taller tiene su propia sub-cuenta (creada manualmente por Efisco dentro de la cuenta maestra) — se configura aquí el ID de cuenta, el token, número de resolución DIAN + prefijo, y departamento/ciudad (DIVIPOLA). El entorno (`PRUEBAS`/`PRODUCCION`) ya no es un toggle manual: se detecta solo según si corre en Vercel o en local. Botón de prueba de conexión contra la API real de Dataico.

---

## Panel de Administración EFISCO

Panel interno en `/admin` con autenticación completamente separada de los talleres — ver [SECURITY.md — Autenticación Admin](SECURITY.md#autenticación-admin-panel-efisco).

### Dashboard

4 KPIs: talleres activos/totales, órdenes del mes, ingresos brutos del mes, pagos de comisiones pendientes.

### Talleres

Lista paginada y buscable de todos los talleres:
- Ver configuración completa (grid de 11 campos, incluye conteo y total de "Consultas de puntaje" — derivado de `pending_score_query_fees / 100`, sin contador dedicado)
- Columna "Consultas Score" en la tabla principal: conteo + total en pesos de consultas detalladas de score cobradas a ese taller
- Crear taller nuevo (crea usuario Supabase Auth + `workshop_config` en transacción)
- **Suspender / Reactivar** — `PATCH /api/admin/workshops/:id/toggle` actualiza `is_active` en `workshop_config`
  - La suspensión bloquea el login (`auth.controller.js`) Y los requests en curso (`auth.middleware.js`)
  - El taller suspendido ve una pantalla de "Cuenta suspendida" con canales de contacto
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
- **Facturación propia de EFISCO**: resolución/prefijo DIAN y consecutivo propios (`efisco_billing_config`, fila única — mismo patrón UPSERT que la plantilla de proveedor en Cobros, mismo bug corregido: antes quedaba inutilizable si la tabla estaba vacía).
- **Captura manual** de lo que ya se cobró/pagó por fuera del sistema (sin pasarela de cobro real todavía): Infraestructura No Residente (AWS/Vercel/etc., convierte a COP con la TRM del día — Documento Soporte a No Residente, SKU "costos infraestructura"), MP/WhatsApp/Dataico, Salarios/Honorarios (Documento Soporte Básico), Suscripción de Taller.
- **"Suscripción Taller" fusiona las dos contabilidades del mismo hecho económico en un solo paso** (`subscriptionBilling.service.js:emitSubscriptionInvoiceForWorkshop`): calcula el precio con descuento por referidos, emite la factura DIAN real de EFISCO al taller, registra el ingreso en el libro de EFISCO, **y además** dispara automáticamente el gasto espejo en la contabilidad del propio taller (mismo motor `financialEngine.liquidateSupplierPurchase` que usa "Talleres → Gasto de Software", mismo monto ya descontado, sin pedir PDF — la factura Dataico recién emitida es el respaldo legal). Checkbox opcional "Aplicar 4x1000 (GMF)" para cuando el pago real fue por transferencia bancaria (apagado por defecto). El botón manual de "Talleres → Gasto de Software" sigue existiendo intacto como vía alterna para casos puntuales (backdating, ajustes). Misma función la usará el futuro webhook de Mercado Pago.
- Comisiones de referidos pagadas disparan un Documento Soporte Básico (sku "pago referido") de forma no bloqueante — el pago ya quedó marcado como completado aunque Dataico falle.
- Ledger contable propio de EFISCO (`efisco_accounting_ledger`, sin `workshop_id`) con totales de ingreso/costo/neto por SKU.
- El bloqueo para emitir esto en producción real no es de código: EFISCO necesita completar su propia habilitación de facturación electrónica ante la DIAN (numeración en MUISCA + asociarla a Dataico como proveedor tecnológico).

Las rutas API completas del panel admin están en [API.md — Admin](API.md#admin).
