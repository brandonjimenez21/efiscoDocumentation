# API Reference

> Ver también: [README](README.md) · [ARCHITECTURE](ARCHITECTURE.md) · [SECURITY](SECURITY.md) · [BUSINESS_RULES](BUSINESS_RULES.md)

Todas las rutas de taller (no-admin) están montadas bajo `/api/*` (`backend/server.js`) y requieren `requireAuth` salvo que se indique **Pública**. `requireAuth` inyecta `workshop_id`/rol desde la sesión ver el modelo de aislamiento en [SECURITY.md](SECURITY.md).

**Errores 500 (fix 2026-07-19)**: todos los controllers responden `{ error: friendlyDbError(error) }` (`backend/utils/dbErrors.js`) en vez de `{ error: error.message }` crudo antes un error real de Postgres (constraints, nombres de columna, "duplicate key value...") se mostraba tal cual al cajero/dueño. `friendlyDbError` distingue por `error.code` (SQLSTATE, solo lo tienen los errores reales de base de datos): si hay `code`, devuelve un mensaje mapeado en español; si no (un `Error` de aplicación que ya construyó su propio mensaje, ej. "Repuesto no encontrado"), devuelve `error.message` intacto no cambia ningún mensaje 4xx existente, solo cierra la fuga de detalles internos en los 500 reales.

---

## Auth (`/api/auth`)

| Método | Ruta | Auth | Descripción |
|:---|:---|:---:|:---|
| POST | `/api/auth/register` | Pública | Registro de un nuevo taller/dueño |
| POST | `/api/auth/login` | Pública | Login (dueño, empleado o contador) — rate-limit de 10 intentos fallidos por `(IP, email)` cada 15 min (429), ver [SECURITY.md](SECURITY.md#endurecimiento-de-seguridad-2026-07-19). Setea la cookie httpOnly `efisco_token` (ver [Migración a cookies httpOnly](SECURITY.md#migración-de-autenticación-a-cookies-httponly-2026-07-21)); el body ya no incluye el token, solo `user` y `expiresAt` |
| POST | `/api/auth/logout` | Pública | Limpia la cookie `efisco_token` |
| GET | `/api/auth/me` | Taller | Restaura la sesión al recargar la página (la cookie httpOnly ya no se puede leer/decodificar en el cliente) — devuelve el mismo `user` rico que `login()`, exento de los gates de contrato/autorización del taller |
| GET | `/api/auth/support-session/exchange` | Pública (token de un solo uso en la URL) | Puente de la llave de acceso a cuenta del cliente — valida el `support_token` emitido por `POST /api/admin/workshops/:id/access-keys/redeem` y setea la cookie de sesión |
| POST | `/api/auth/request-password-reset` | Pública | Self-service "olvidé mi contraseña" — solo envía el correo real de Supabase si el taller ya fue activado por un admin (`workshop_config.admin_activated_at`); respuesta genérica siempre igual, ver [SECURITY.md](SECURITY.md) |
| POST | `/api/auth/reset-password` | Taller (token de recovery) | Confirma la nueva contraseña — recibe el `access_token` del link de recovery como Bearer y el password nuevo (excepción: este token viaja por header, no por cookie — ver [SECURITY.md](SECURITY.md#migración-de-autenticación-a-cookies-httponly-2026-07-21)) |
| POST | `/api/auth/sign-contract` | Taller | Firma electrónica del Contrato de Afiliación B2B (reingreso de contraseña) |
| POST | `/api/auth/accept-staff-terms` | Taller | Aceptación de la autorización del taller (staff no-dueño) |
| POST | `/api/auth/verify-password` | Taller | Verifica la contraseña del usuario autenticado |

## Workshop (`/api/workshop`)

| Método | Ruta | Auth | Descripción |
|:---|:---|:---:|:---|
| POST | `/api/workshop/` | Taller | Crear configuración de taller |
| GET | `/api/workshop/referrals/active` | Taller | Referidos activos |
| POST | `/api/workshop/referrals/link` | Taller | Vincular "quién te refirió" (sin ciclos) |
| POST | `/api/workshop/dataico/test` | Taller | Probar conexión con Dataico |
| POST | `/api/workshop/access-key` | Taller (owner) | Generar código de acceso de soporte (30 min) |
| GET | `/api/workshop/software-invoices` | Taller (owner) | Listar facturas de EFISCO con link de descarga firmado |
| PUT | `/api/workshop/lock-pin` | Taller | Configurar PIN de bloqueo por inactividad |
| POST | `/api/workshop/lock-pin/verify` | Taller | Verificar PIN de bloqueo — rate-limit de 5 intentos fallidos por taller cada 15 min (429), ver [SECURITY.md](SECURITY.md#endurecimiento-de-seguridad-2026-07-19) |
| GET | `/api/workshop/:id` | Taller | Detalle de configuración del taller |
| PUT | `/api/workshop/:id` | Taller | Actualizar configuración (campos fiscales/PUC/Dataico exclusivos de `contador`, ver [BUSINESS_RULES.md](BUSINESS_RULES.md#configuración-del-taller-config)) |
| POST | `/api/workshop/finance-settings` | Taller (contador) | Actualizar parámetros financieros — exclusivo del rol `contador` (403 para cualquier otro rol), mismo criterio que los campos fiscales de `PUT /:id`. Sin caller activo en el frontend (la UI usa `PUT /:id`); si se envía `fiscal_regime`, se sincroniza con los 3 booleans de régimen igual que en `PUT /:id` (rev. 35 — antes desincronizaba `fiscal_regime` de Gran Contribuyente/Ordinario) |

## Mechanics (`/api/mechanics`)

| Método | Ruta | Auth | Descripción |
|:---|:---|:---:|:---|
| POST | `/api/mechanics/` | Taller | Crear mecánico/empleado — `role`/`payment_type` validados contra los valores reales del sistema (`mecanico/admin/contador`, `salario/comision/mixto`), 400 si no coinciden (fix 2026-07-19) |
| POST | `/api/mechanics/self` | Taller (owner) | El dueño se registra a sí mismo como mecánico |
| GET | `/api/mechanics/:id/salary-history` | Taller | Historial salarial |
| GET | `/api/mechanics/:id/metrics` | Taller | Métricas del mecánico |
| GET | `/api/mechanics/:id/pending-payment` | Taller | Comisión devengada pendiente de pago (`Σ MECH_COMMISSION − Σ MECH_COMMISSION_PAY`) |
| POST | `/api/mechanics/:id/pay` | Taller (owner) | Registrar pago real a un mecánico (sueldo y/o comisión) — genera `MECH_SALARY_PAY`/`MECH_COMMISSION_PAY` en el Libro Auxiliar, ver [FINANCIAL_ENGINE.md](FINANCIAL_ENGINE.md#caja-real-vs-informativodevengo--por-qué-existe-la-distinción) |
| PATCH | `/api/mechanics/:id/deactivate` | Taller | Desactivar empleado |
| PATCH | `/api/mechanics/:id` | Taller | Editar empleado — `payment_type` (si se envía) validado contra `salario/comision/mixto`, 400 si no coincide (fix 2026-07-19) |
| POST | `/api/mechanics/:id/account` | Taller | Habilitar credenciales de acceso |
| GET | `/api/mechanics/:workshop_id` | Taller | Listar mecánicos del taller |

## Work Orders (`/api/work-orders`)

| Método | Ruta | Auth | Descripción |
|:---|:---|:---:|:---|
| POST | `/api/work-orders/` | Taller | Crear orden de trabajo — acepta `services[]` y `mechanics[]` (multi-servicio/multi-mecánico); el payload legacy de escalares sigue soportado. Valida fecha de entrega futura y que todos los asignados tengan rol `mecanico` (400 si no) |
| GET | `/api/work-orders/` | Taller | Listar órdenes activas (incluye `services[]` con márgenes y `mechanics_detail[]`) |
| GET | `/api/work-orders/history` | Taller | Historial de órdenes completadas (incl. `mechanics_names[]`/`services_names[]`; para rol mecánico filtra por sus filas en `work_order_mechanics` + el escalar legacy). Acepta `from`/`to` (rango sobre `end_time`) y `limit`/`offset` (default 50, tope 100) — respuesta `{ data, total }`, no un array plano (fix 2026-07-19, antes traía todo el historial sin límite, ver [BUSINESS_RULES.md](BUSINESS_RULES.md#bahías-órdenes-de-trabajo)) |
| GET | `/api/work-orders/pending-by-cedula/:cedula` | Taller | Orden `pending` precargada desde Registro Seguro EFISCO (query `intake_id`) |
| PUT | `/api/work-orders/:id/pause` | Taller | Pausar/reanudar orden |
| PUT | `/api/work-orders/:id/finish` | Taller | Marcar como lista para facturar |
| PUT | `/api/work-orders/:id/status` | Taller | Cambiar estado — validado contra los 4 valores reales del flujo (`pending/ejecucion/ready_to_invoice/completed`), 400 si no coincide (fix 2026-07-19; antes aceptaba cualquier string, ver [BUSINESS_RULES.md — Bahías](BUSINESS_RULES.md#bahías-órdenes-de-trabajo)) |
| PUT | `/api/work-orders/:id` | Taller | Editar orden — mismos arrays y validaciones que el POST; reemplaza las filas hijas (replace-children) |
| DELETE | `/api/work-orders/:id` | Taller | Eliminar orden |

## Inventory (`/api/inventory`)

| Método | Ruta | Auth | Descripción |
|:---|:---|:---:|:---|
| GET | `/api/inventory/` | Taller | Listar inventario |
| POST | `/api/inventory/standalone` | Taller | Alta de ítem sin compra asociada — cantidad inicial validada (≥0) y `measure` contra `validateMeasure` si viene con `category`, mismo criterio que añadir un repuesto a una orden (fix 2026-07-19) |
| GET | `/api/inventory/total-investment` | Taller | Valor total de inventario |
| GET | `/api/inventory/matrix` | Taller | Tab "Matriz": rotación, uso promedio, stock mínimo vital |
| GET | `/api/inventory/pending-invoices/today` | Taller | Facturas OCR pendientes de revisión del día |
| GET | `/api/inventory/history/:id` | Taller | Kardex de un ítem (`requested_at`) |
| PUT | `/api/inventory/add-stock/:id` | Taller | Registrar entrada de stock — cantidad validada como número positivo, 400 si no (fix 2026-07-19; antes un valor no numérico o negativo corrompía `current_stock`) |
| GET | `/api/inventory/work-order/:work_order_id` | Taller | Ítems consumidos por una orden |
| POST | `/api/inventory/work-order/:work_order_id` | Taller | Añadir repuesto a una orden (descuenta stock; excepción: categoría "Lubricantes y Químicos" con `container_emptied=false` no descuenta, ver [BUSINESS_RULES.md — Inventario](BUSINESS_RULES.md#inventario)) — cantidad validada como entero positivo y topada contra el stock disponible (409 si no alcanza), fix 2026-07-19 |
| PUT | `/api/inventory/work-order-item/:item_id` | Taller | Editar ítem dentro de una orden |
| DELETE | `/api/inventory/work-order-item/:item_id` | Taller | Quitar ítem de una orden |
| PUT | `/api/inventory/:id` | Taller | Editar ítem de inventario |
| DELETE | `/api/inventory/:id` | Taller | Eliminar ítem de inventario |

## Providers (`/api/providers`)

| Método | Ruta | Auth | Descripción |
|:---|:---|:---:|:---|
| GET | `/api/providers/` | Taller | Listar proveedores |
| POST | `/api/providers/` | Taller | Crear proveedor |
| PUT | `/api/providers/:id` | Taller | Editar proveedor — scoped al taller de la sesión (404 si no es suyo), lista blanca de campos editables (no acepta `workshop_id`/`is_system_provider`), 403 si `is_system_provider` |
| DELETE | `/api/providers/:id` | Taller | Eliminar proveedor — scoped al taller de la sesión (404 si no es suyo), 403 si `is_system_provider` |
| POST | `/api/providers/invoice-ocr` | Taller | OCR de factura (AWS Textract, imagen ≤5MB) |
| POST | `/api/providers/purchase` | Taller | Registrar compra y liquidar (`financialEngine.liquidateSupplierPurchase`) — acepta `payment_mode:'credito'`+`num_installments`+`first_payment_date` para pago a plazos (genera `supplier_installments`, cabecera en `status:'pendiente'`); `due_dates` opcional (array de "YYYY-MM-DD", una por cuota) manda sobre el cálculo automático por intervalo si el frontend edita las fechas a mano (rev. 48) |
| GET | `/api/providers/efficiency` | Taller | Eficiencia de entrega (Fase 3) |
| GET | `/api/providers/installments` | Taller | Listar cuotas por pagar a proveedores |
| PATCH | `/api/providers/installments/:id/pay` | Taller | Pagar una cuota — genera la salida de caja real (`SUP_PAY` con `net_amount = amount`) |
| GET | `/api/providers/:id/purchases` | Taller | Historial de compras a un proveedor |

## Services (`/api/services`)

| Método | Ruta | Auth | Descripción |
|:---|:---|:---:|:---|
| POST | `/api/services/` | Taller | Crear servicio maestro — `gama`/`complejidad` obligatorios (`'Alta'`\|`'Baja'`, 400 si no), usados solo para sugerir `base_margin_basic`/`base_margin_premium` en el formulario (rev. 49) |
| GET | `/api/services/:workshop_id/:vehicleType` | Taller | Servicios por tipo de vehículo |
| GET | `/api/services/` | Taller | Listar todos los servicios |
| PUT | `/api/services/:id` | Taller | Editar servicio (márgenes básico/premium, `gama`/`complejidad` obligatorios igual que en creación) |
| DELETE | `/api/services/:id` | Taller | Eliminar servicio |

## Quick Intakes / Recepción (`/api/quick-intakes`)

| Método | Ruta | Auth | Descripción |
|:---|:---|:---:|:---|
| POST | `/api/quick-intakes/` | Taller | Registrar ingreso rápido |
| GET | `/api/quick-intakes/` | Taller | Listar cola de ingresos |
| GET | `/api/quick-intakes/ready` | Taller | Ingresos listos para pasar a Bahía |
| PUT | `/api/quick-intakes/:id/bahia` | Taller | Convertir ingreso en orden de Bahía |
| DELETE | `/api/quick-intakes/:id` | Taller | Eliminar ingreso |
| POST | `/api/quick-intakes/:id/whatsapp` | Taller | Enviar link de Registro Seguro EFISCO (WhatsApp, o Telegram si `client_cedula` del ingreso ya está vinculado y WhatsApp no está configurado). Sin cédula capturada (campo opcional del ingreso rápido), Telegram no puede actuar como respaldo |

## Clients (`/api/clients`)

| Método | Ruta | Auth | Descripción |
|:---|:---|:---:|:---|
| GET | `/api/clients/public-intake/:workshopId/:intakeId` | Pública | Info del ingreso vinculado (Registro Seguro EFISCO) |
| GET | `/api/clients/public-score-pdf/:workshopId/:intakeId` | Pública | Descarga del PDF de score del propio cliente |
| POST | `/api/clients/public-register/:workshopId` | Pública | Envío del formulario de Registro Seguro EFISCO |
| POST | `/api/clients/sweep-freezes` | Taller | Barrido de congelamientos de score vencidos |
| GET | `/api/clients/by-phone/:phone` | Taller | Buscar cliente por teléfono (match parcial, últimos 10 dígitos) |
| GET | `/api/clients/by-cedula/:cedula` | Taller | Buscar cliente por cédula (match exacto por `nit`, scoped al taller) — usado por Recepción para autocompletar nombre/apellido/WhatsApp de clientes ya registrados |
| POST | `/api/clients/:cedula/request-otp` | Taller | Enviar OTP de verificación (WhatsApp, o Telegram si el cliente ya vinculó y WhatsApp no está configurado — ver `messagingChannel.service.js`). `reason: 'no_channel_linked'` si Telegram está disponible pero este cliente no vinculó (el frontend muestra el QR). El `demo_code` (código devuelto directo al recepcionista) solo existe fuera de producción o con `OTP_DEMO_MODE=true` y si NINGÚN canal está configurado; en producción responde 503/502 |
| POST | `/api/clients/:cedula/verify-otp` | Taller | Verificar OTP → única forma de ver el score (local + global, con desglose) — decisión de producto 2026-07-19: se eliminó `GET /:cedula/risk-score` (score sin código, gratis); ver [BUSINESS_RULES.md — Recepción](BUSINESS_RULES.md#recepción). Envía el PDF por el canal disponible. Anti fuerza-bruta: 5 intentos fallidos por (taller, cédula) → 429 por 15 min; comparación constant-time |
| POST | `/api/clients/:cedula/telegram-link-request` | Taller | Genera el QR de vinculación de Telegram (deep-link `t.me/<bot>?start=<token>`, expira en 10 min) — `alreadyLinked: true` si la cédula ya está vinculada (global, cualquier taller de la red), sin generar uno nuevo. 503 si Telegram no está configurado |
| GET | `/api/clients/:cedula/telegram-link-status` | Taller | Estado de vinculación — usado por Recepción para hacer polling corto (~3s) mientras el QR está en pantalla |

## Billing / Liquidación (`/api/billing`)

| Método | Ruta | Auth | Descripción |
|:---|:---|:---:|:---|
| POST | `/api/billing/settle` | Taller | Liquidar orden (`financialEngine.liquidateClientInvoice`, emite Dataico + comprobante WhatsApp) — acepta `payment_mode:'credito'`+`num_installments`+`first_payment_date` para venta a crédito; `due_dates` opcional (array de "YYYY-MM-DD", una por cuota) manda sobre el cálculo automático por intervalo si el frontend edita las fechas a mano (rev. 48) |
| GET | `/api/billing/installments` | Taller | Listar cuotas de crédito |
| PATCH | `/api/billing/installments/:id/pay` | Taller | Pagar cuota (genera `INC_GROSS`, envía comprobante WhatsApp) |
| POST | `/api/billing/webhook/bold` | Pública (webhook) | Confirmación de pago Bold — **fail-closed**: rechaza si `BOLD_WEBHOOK_TOKEN` no está configurado; comparación de token constant-time + log de auditoría en cada rechazo, ver [SECURITY.md](SECURITY.md#webhooks-de-pago) |
| POST | `/api/billing/webhook/addi` | Pública (webhook) | Confirmación de pago Addi — **fail-closed**: rechaza si `ADDI_WEBHOOK_TOKEN` no está configurado; comparación de token constant-time + log de auditoría en cada rechazo, ver [SECURITY.md](SECURITY.md#webhooks-de-pago) |
| POST | `/api/billing/webhook/mercadopago` | Pública (webhook) | Confirmación de pago Mercado Pago — **fail-closed** (`MERCADOPAGO_WEBHOOK_TOKEN` por query string) y además verifica contra la API real de MP, nunca confía en el body |

## Telegram (`/api/telegram`)

| Método | Ruta | Auth | Descripción |
|:---|:---|:---:|:---|
| POST | `/api/telegram/webhook` | Pública (webhook) | Recibe los updates de Telegram — solo procesa `/start <token>` (vinculación, ver `telegram-link-request` arriba). **Fail-closed** vía el header nativo `X-Telegram-Bot-Api-Secret-Token` (`TELEGRAM_WEBHOOK_SECRET`). Siempre responde 200 salvo secret inválido — Telegram reintenta muy agresivo ante cualquier otro código |

## Messaging (`/api/messaging`)

| Método | Ruta | Auth | Descripción |
|:---|:---|:---:|:---|
| GET | `/api/messaging/status` | Taller | `{whatsapp, telegram}` — qué canales están configurados a nivel servidor. El frontend (Recepción) lo usa para mostrar el botón de WhatsApp activo/deshabilitado y ofrecer Telegram al lado sin tener que intentar un envío primero |

## Finance / Análisis Financiero (`/api/finance`)

| Método | Ruta | Auth | Descripción |
|:---|:---|:---:|:---|
| GET | `/api/finance/dashboard-summary` | Taller | Resumen agregado para el Dashboard |
| POST | `/api/finance/referral` | Taller (owner) | Registrar ingreso por comisión de referido — 403 para cualquier otro rol (fix 2026-07-19, ver [SECURITY.md](SECURITY.md#endurecimiento-de-seguridad-2026-07-19)) |
| GET | `/api/finance/ledger-book` | Taller | Libro Mayor único en Excel real (`.xlsx`, vía `exceljs`) — reemplaza los 6 reportes CSV anteriores (`invoices`/`purchases`/`receivable`/`payable`/`ledger`/`inventory-valuation`, hoy eliminados). Acepta `from`/`to` opcionales |
| POST | `/api/finance/manual-movement` | Taller (owner) | Movimiento manual (`MAN_INC`/`MAN_EGR`, ingreso no operacional, devolución IVA) — 403 para cualquier otro rol (fix 2026-07-19, ver [SECURITY.md](SECURITY.md#endurecimiento-de-seguridad-2026-07-19)) |
| GET | `/api/finance/opening-balance` | Taller (owner) | Saldo inicial migrado (cuaderno/Excel previo) |
| POST | `/api/finance/opening-balance` | Taller (owner) | Fijar saldo inicial |
| GET | `/api/finance/folios-by-service` | Taller | Folios de facturación electrónica consumidos por servicio |
| POST | `/api/finance/subscription/checkout` | Taller | Checkout Pro de Mercado Pago para pagar la suscripción a EFISCO |
| GET | `/api/finance/breakeven-panel` | Taller | Datos del Panel de Equilibrio |
| GET | `/api/finance/cashflow` | Taller | Flujo de caja detallado por fecha |
| GET | `/api/finance/referral-discounts` | Taller | Descuentos ganados por referidos |
| GET | `/api/finance/referral-payout/summary` | Taller | Estado de comisiones de referidos pendientes |
| GET | `/api/finance/monthly-books` | Taller | Libros mensuales archivados del Libro Auxiliar — hace el **cierre perezoso** de todo mes terminado sin libro (upsert idempotente; el mes en curso nunca se cierra) |
| GET | `/api/finance/monthly-books/:period` | Taller | Libro completo de un mes (`YYYY-MM`) con su snapshot congelado; 404 si ese mes no está cerrado |

---

## Admin (`/api/admin`)

Autenticación separada (`requireAdmin`, `ADMIN_JWT_SECRET`) — ver [SECURITY.md](SECURITY.md#autenticación-admin-panel-efisco).

| Método | Ruta | Auth | Descripción |
|:---|:---|:---:|:---|
| POST | `/api/admin/bootstrap` | Pública | Crea primer admin (solo si tabla vacía). Setea la cookie httpOnly `efisco_admin_token` — el body ya no incluye el token |
| POST | `/api/admin/login` | Pública | Login admin con email/password. Setea la cookie httpOnly `efisco_admin_token` — el body ya no incluye el token |
| POST | `/api/admin/logout` | Pública | Limpia la cookie `efisco_admin_token` |
| GET | `/api/admin/me` | Admin | Restaura la sesión admin al recargar la página (mismo motivo que `GET /api/auth/me`) |
| GET | `/api/admin/stats` | Admin | KPIs del dashboard |
| GET | `/api/admin/workshops` | Admin | Lista talleres (con emails de Supabase Auth) |
| GET | `/api/admin/workshops/:id` | Admin | Detalle de taller — incluye `contract_acceptance` (constancia de firma del Contrato B2B: versión, hash, IP, fecha; `null` si no ha firmado), ver [SECURITY.md](SECURITY.md) |
| GET | `/api/admin/workshops/:id/contract-pdf` | Admin | Descarga del PDF firmado del Contrato B2B del taller — signed URL de 1h, self-heal si `pdf_storage_path` no existe aún (regenera con `generateAndStoreContractPdf`), 404 si el taller no ha firmado. Mismo servicio que usa el dueño desde `/config` (`GET /api/auth/contract-pdf`), pero sin requerir sesión del taller (rev. 46) |
| POST | `/api/admin/workshops` | Admin | Crear taller nuevo |
| PATCH | `/api/admin/workshops/:id/toggle` | Admin | Suspender / reactivar |
| DELETE | `/api/admin/workshops/:id` | Admin | Elimina permanentemente un taller **no activado** (400 si `admin_activated_at` ya tiene valor) — borra en cascada + notifica por correo al dueño (Resend), ver [SECURITY.md](SECURITY.md) |
| POST | `/api/admin/workshops/:id/send-password-reset` | Admin | Enviar email de reseteo de contraseña al dueño |
| POST | `/api/admin/workshops/:id/access-keys/redeem` | Admin | Canjear llave de acceso → token de sesión de soporte |
| POST | `/api/admin/workshops/:id/software-invoice` | Admin | Subir PDF de factura propia de EFISCO al taller (multipart, ≤10MB, solo PDF) |
| GET | `/api/admin/payouts` | Admin | Lista de solicitudes de pago (comisión de referidos) |
| PATCH | `/api/admin/payouts/:id/mark-paid` | Admin | Marcar pago como completado |
| GET | `/api/admin/referrals` | Admin | Árbol de referidos |
| GET | `/api/admin/efisco-provider-template` | Admin | Leer la plantilla del proveedor EFISCO |
| PUT | `/api/admin/efisco-provider-template` | Admin | Editar la plantilla (UPSERT, sincroniza hacia todos los talleres) |
| GET | `/api/admin/software-invoices` | Admin | Lista de todas las facturas de EFISCO a talleres, enriquecida |
| PATCH | `/api/admin/software-invoices/:id/mark-paid` | Admin | Marcar factura como pagada + reactivar el taller |
| GET | `/api/admin/billing-config` | Admin | Configuración de facturación propia de EFISCO (Dataico Caso 2) |
| PUT | `/api/admin/billing-config` | Admin | Editar configuración de facturación propia (UPSERT) |
| GET | `/api/admin/accounting-ledger` | Admin | Ledger contable propio de EFISCO (`efisco_accounting_ledger`) |
| POST | `/api/admin/accounting/infra-cost` | Admin | Registrar costo de infraestructura no residente (Documento Soporte a No Residente) |
| POST | `/api/admin/accounting/internal-cost` | Admin | Registrar costo interno (MP/WhatsApp/Dataico) |
| POST | `/api/admin/accounting/salary-cost` | Admin | Registrar costo de salarios/honorarios (Documento Soporte Básico) |
| POST | `/api/admin/accounting/subscription-invoice` | Admin | Emitir "Suscripción Taller" (factura DIAN real + espejo de gasto en el taller) |

Detalle funcional de cada módulo admin en [BUSINESS_RULES.md — Panel de Administración EFISCO](BUSINESS_RULES.md#panel-de-administración-efisco).
