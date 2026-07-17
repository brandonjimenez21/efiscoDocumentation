# API Reference

> Ver también: [README](README.md) · [ARCHITECTURE](ARCHITECTURE.md) · [SECURITY](SECURITY.md) · [BUSINESS_RULES](BUSINESS_RULES.md)

Todas las rutas de taller (no-admin) están montadas bajo `/api/*` (`backend/server.js`) y requieren `requireAuth` salvo que se indique **Pública**. `requireAuth` inyecta `workshop_id`/rol desde la sesión — ver el modelo de aislamiento en [SECURITY.md](SECURITY.md).

---

## Auth (`/api/auth`)

| Método | Ruta | Auth | Descripción |
|:---|:---|:---:|:---|
| POST | `/api/auth/register` | Pública | Registro de un nuevo taller/dueño |
| POST | `/api/auth/login` | Pública | Login (dueño, empleado o contador) |
| POST | `/api/auth/request-password-reset` | Pública | Self-service "olvidé mi contraseña" — solo envía el correo real de Supabase si el taller ya fue activado por un admin (`workshop_config.admin_activated_at`); respuesta genérica siempre igual, ver [SECURITY.md](SECURITY.md) |
| POST | `/api/auth/reset-password` | Taller (token de recovery) | Confirma la nueva contraseña — recibe el `access_token` del link de recovery como Bearer y el password nuevo |
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
| POST | `/api/workshop/lock-pin/verify` | Taller | Verificar PIN de bloqueo |
| GET | `/api/workshop/:id` | Taller | Detalle de configuración del taller |
| PUT | `/api/workshop/:id` | Taller | Actualizar configuración (campos fiscales/PUC/Dataico exclusivos de `contador`, ver [BUSINESS_RULES.md](BUSINESS_RULES.md#configuración-del-taller-config)) |
| POST | `/api/workshop/finance-settings` | Taller | Actualizar parámetros financieros |

## Mechanics (`/api/mechanics`)

| Método | Ruta | Auth | Descripción |
|:---|:---|:---:|:---|
| POST | `/api/mechanics/` | Taller | Crear mecánico/empleado |
| POST | `/api/mechanics/self` | Taller (owner) | El dueño se registra a sí mismo como mecánico |
| GET | `/api/mechanics/:id/salary-history` | Taller | Historial salarial |
| GET | `/api/mechanics/:id/metrics` | Taller | Métricas del mecánico |
| GET | `/api/mechanics/:id/pending-payment` | Taller | Comisión devengada pendiente de pago (`Σ MECH_COMMISSION − Σ MECH_COMMISSION_PAY`) |
| POST | `/api/mechanics/:id/pay` | Taller (owner) | Registrar pago real a un mecánico (sueldo y/o comisión) — genera `MECH_SALARY_PAY`/`MECH_COMMISSION_PAY` en el Libro Auxiliar, ver [FINANCIAL_ENGINE.md](FINANCIAL_ENGINE.md#caja-real-vs-informativodevengo--por-qué-existe-la-distinción) |
| PATCH | `/api/mechanics/:id/deactivate` | Taller | Desactivar empleado |
| PATCH | `/api/mechanics/:id` | Taller | Editar empleado |
| POST | `/api/mechanics/:id/account` | Taller | Habilitar credenciales de acceso |
| GET | `/api/mechanics/:workshop_id` | Taller | Listar mecánicos del taller |

## Work Orders (`/api/work-orders`)

| Método | Ruta | Auth | Descripción |
|:---|:---|:---:|:---|
| POST | `/api/work-orders/` | Taller | Crear orden de trabajo — acepta `services[]` y `mechanics[]` (multi-servicio/multi-mecánico); el payload legacy de escalares sigue soportado. Valida fecha de entrega futura y que todos los asignados tengan rol `mecanico` (400 si no) |
| GET | `/api/work-orders/` | Taller | Listar órdenes activas (incluye `services[]` con márgenes y `mechanics_detail[]`) |
| GET | `/api/work-orders/history` | Taller | Historial de órdenes completadas (incl. `mechanics_names[]`/`services_names[]`; para rol mecánico filtra por sus filas en `work_order_mechanics` + el escalar legacy) |
| GET | `/api/work-orders/pending-by-cedula/:cedula` | Taller | Orden `pending` precargada desde Registro Seguro EFISCO (query `intake_id`) |
| PUT | `/api/work-orders/:id/pause` | Taller | Pausar/reanudar orden |
| PUT | `/api/work-orders/:id/finish` | Taller | Marcar como lista para facturar |
| PUT | `/api/work-orders/:id/status` | Taller | Cambiar estado |
| PUT | `/api/work-orders/:id` | Taller | Editar orden — mismos arrays y validaciones que el POST; reemplaza las filas hijas (replace-children) |
| DELETE | `/api/work-orders/:id` | Taller | Eliminar orden |

## Inventory (`/api/inventory`)

| Método | Ruta | Auth | Descripción |
|:---|:---|:---:|:---|
| GET | `/api/inventory/` | Taller | Listar inventario |
| POST | `/api/inventory/standalone` | Taller | Alta de ítem sin compra asociada |
| GET | `/api/inventory/total-investment` | Taller | Valor total de inventario |
| GET | `/api/inventory/matrix` | Taller | Tab "Matriz": rotación, uso promedio, stock mínimo vital |
| GET | `/api/inventory/pending-invoices/today` | Taller | Facturas OCR pendientes de revisión del día |
| GET | `/api/inventory/history/:id` | Taller | Kardex de un ítem (`requested_at`) |
| PUT | `/api/inventory/add-stock/:id` | Taller | Registrar entrada de stock |
| GET | `/api/inventory/work-order/:work_order_id` | Taller | Ítems consumidos por una orden |
| POST | `/api/inventory/work-order/:work_order_id` | Taller | Añadir repuesto a una orden (descuenta stock; excepción: categoría "Lubricantes y Químicos" con `container_emptied=false` no descuenta, ver [BUSINESS_RULES.md — Inventario](BUSINESS_RULES.md#inventario)) |
| PUT | `/api/inventory/work-order-item/:item_id` | Taller | Editar ítem dentro de una orden |
| DELETE | `/api/inventory/work-order-item/:item_id` | Taller | Quitar ítem de una orden |
| PUT | `/api/inventory/:id` | Taller | Editar ítem de inventario |
| DELETE | `/api/inventory/:id` | Taller | Eliminar ítem de inventario |

## Providers (`/api/providers`)

| Método | Ruta | Auth | Descripción |
|:---|:---|:---:|:---|
| GET | `/api/providers/` | Taller | Listar proveedores |
| POST | `/api/providers/` | Taller | Crear proveedor |
| PUT | `/api/providers/:id` | Taller | Editar proveedor (403 si `is_system_provider`) |
| DELETE | `/api/providers/:id` | Taller | Eliminar proveedor (403 si `is_system_provider`) |
| POST | `/api/providers/invoice-ocr` | Taller | OCR de factura (AWS Textract, imagen ≤5MB) |
| POST | `/api/providers/purchase` | Taller | Registrar compra y liquidar (`financialEngine.liquidateSupplierPurchase`) |
| GET | `/api/providers/efficiency` | Taller | Eficiencia de entrega (Fase 3) |
| GET | `/api/providers/:id/purchases` | Taller | Historial de compras a un proveedor |

## Services (`/api/services`)

| Método | Ruta | Auth | Descripción |
|:---|:---|:---:|:---|
| POST | `/api/services/` | Taller | Crear servicio maestro |
| GET | `/api/services/:workshop_id/:vehicleType` | Taller | Servicios por tipo de vehículo |
| GET | `/api/services/` | Taller | Listar todos los servicios |
| PUT | `/api/services/:id` | Taller | Editar servicio (márgenes básico/premium) |
| DELETE | `/api/services/:id` | Taller | Eliminar servicio |

## Quick Intakes / Recepción (`/api/quick-intakes`)

| Método | Ruta | Auth | Descripción |
|:---|:---|:---:|:---|
| POST | `/api/quick-intakes/` | Taller | Registrar ingreso rápido |
| GET | `/api/quick-intakes/` | Taller | Listar cola de ingresos |
| GET | `/api/quick-intakes/ready` | Taller | Ingresos listos para pasar a Bahía |
| PUT | `/api/quick-intakes/:id/bahia` | Taller | Convertir ingreso en orden de Bahía |
| DELETE | `/api/quick-intakes/:id` | Taller | Eliminar ingreso |
| POST | `/api/quick-intakes/:id/whatsapp` | Taller | Enviar link de Registro Seguro EFISCO por WhatsApp |

## Clients (`/api/clients`)

| Método | Ruta | Auth | Descripción |
|:---|:---|:---:|:---|
| GET | `/api/clients/public-intake/:workshopId/:intakeId` | Pública | Info del ingreso vinculado (Registro Seguro EFISCO) |
| GET | `/api/clients/public-score-pdf/:workshopId/:intakeId` | Pública | Descarga del PDF de score del propio cliente |
| POST | `/api/clients/public-register/:workshopId` | Pública | Envío del formulario de Registro Seguro EFISCO |
| POST | `/api/clients/sweep-freezes` | Taller | Barrido de congelamientos de score vencidos |
| GET | `/api/clients/by-phone/:phone` | Taller | Buscar cliente por teléfono |
| GET | `/api/clients/:cedula/risk-score` | Taller | Score básico (local + global, sin desglose) |
| POST | `/api/clients/:cedula/request-otp` | Taller | Enviar OTP de verificación por WhatsApp |
| POST | `/api/clients/:cedula/verify-otp` | Taller | Verificar OTP → desbloquea desglose + envía PDF por WhatsApp |

## Billing / Liquidación (`/api/billing`)

| Método | Ruta | Auth | Descripción |
|:---|:---|:---:|:---|
| POST | `/api/billing/settle` | Taller | Liquidar orden (`financialEngine.liquidateClientInvoice`, emite Dataico + comprobante WhatsApp) |
| GET | `/api/billing/installments` | Taller | Listar cuotas de crédito |
| PATCH | `/api/billing/installments/:id/pay` | Taller | Pagar cuota (genera `INC_GROSS`, envía comprobante WhatsApp) |
| POST | `/api/billing/webhook/bold` | Pública (webhook) | Confirmación de pago Bold — ver [SECURITY.md](SECURITY.md#webhooks-de-pago) |
| POST | `/api/billing/webhook/addi` | Pública (webhook) | Confirmación de pago Addi — ver [SECURITY.md](SECURITY.md#webhooks-de-pago) |
| POST | `/api/billing/webhook/mercadopago` | Pública (webhook) | Confirmación de pago Mercado Pago (verifica contra la API real, no confía en el body) |

## Finance / Análisis Financiero (`/api/finance`)

| Método | Ruta | Auth | Descripción |
|:---|:---|:---:|:---|
| GET | `/api/finance/dashboard-summary` | Taller | Resumen agregado para el Dashboard |
| POST | `/api/finance/referral` | Taller | Registrar ingreso por comisión de referido |
| GET | `/api/finance/report/:type` | Taller | Generación de reportes CSV (incl. `ledger`) |
| POST | `/api/finance/manual-movement` | Taller | Movimiento manual (`MAN_INC`/`MAN_EGR`, ingreso no operacional, devolución IVA) |
| GET | `/api/finance/opening-balance` | Taller | Saldo inicial migrado (cuaderno/Excel previo) |
| POST | `/api/finance/opening-balance` | Taller | Fijar saldo inicial |
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
| POST | `/api/admin/bootstrap` | Pública | Crea primer admin (solo si tabla vacía) |
| POST | `/api/admin/login` | Pública | Login admin con email/password |
| GET | `/api/admin/stats` | Admin | KPIs del dashboard |
| GET | `/api/admin/workshops` | Admin | Lista talleres (con emails de Supabase Auth) |
| GET | `/api/admin/workshops/:id` | Admin | Detalle de taller |
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
