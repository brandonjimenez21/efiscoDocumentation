# API Reference

> Ver también: [README](../README.md) · [ARCHITECTURE](ARCHITECTURE.md) · [SECURITY](SECURITY.md) · [BUSINESS_RULES](../Reglas%20de%20Negocio%20y%20Finanzas/BUSINESS_RULES.md) · [OPERATIONS](../Reglas%20de%20Negocio%20y%20Finanzas/OPERATIONS.md) · [INVENTORY](../Reglas%20de%20Negocio%20y%20Finanzas/INVENTORY.md) · [FINANCE](../Reglas%20de%20Negocio%20y%20Finanzas/FINANCE.md) · [FINANCIAL_ENGINE](../Reglas%20de%20Negocio%20y%20Finanzas/FINANCIAL_ENGINE.md) · [BILLING](../Reglas%20de%20Negocio%20y%20Finanzas/BILLING.md) · [GROWTH_ACQUISITION](../Estrategia%20Comercial%20y%20Ventas/GROWTH_ACQUISITION.md) · [PRICING_SALES](../Estrategia%20Comercial%20y%20Ventas/PRICING_SALES.md) · [MONITORING](../MONITORING.md) · [TESTING](../TESTING.md)

Todas las rutas de taller (no-admin) están montadas bajo `/api/*` (`backend/server.js`) y requieren `requireAuth` salvo que se indique **Pública**. `requireAuth` inyecta `workshop_id`/rol desde la sesión — ver el modelo de aislamiento en [SECURITY.md](SECURITY.md).

**Errores 500 (fix 2026-07-19)**: todos los controllers responden `{ error: friendlyDbError(error) }` (`backend/utils/dbErrors.js`) en vez de `{ error: error.message }` crudo — antes un error real de Postgres (constraints, nombres de columna, "duplicate key value...") se mostraba tal cual al cajero/dueño. `friendlyDbError` distingue por `error.code` (SQLSTATE, solo lo tienen los errores reales de base de datos): si hay `code`, devuelve un mensaje mapeado en español; si no (un `Error` de aplicación que ya construyó su propio mensaje, ej. "Repuesto no encontrado"), devuelve `error.message` intacto — no cambia ningún mensaje 4xx existente, solo cierra la fuga de detalles internos en los 500 reales.

---

## Auth (`/api/auth`)

| Método | Ruta | Auth | Descripción |
|:---|:---|:---:|:---|
| POST | `/api/auth/register` | Pública | Registro de un nuevo taller/dueño |
| POST | `/api/auth/login` | Pública | Login (dueño, empleado o contador) — rate-limit de 10 intentos fallidos por `(IP, email)` cada 15 min (429), ver [SECURITY.md](SECURITY.md#endurecimiento-de-seguridad-2026-07-19). Setea la cookie httpOnly `efisco_token` (ver [Migración a cookies httpOnly](SECURITY.md#migración-de-autenticación-a-cookies-httponly-2026-07-21)); el body ya no incluye el token, solo `user` y `expiresAt`. Para un empleado, `user` incluye `can_manage_catalog`/`available_as_mechanic` (2026-08-24 — antes solo `GET /me` los traía, así que un admin recién logueado no veía "Ajustes" hasta refrescar) |
| POST | `/api/auth/logout` | Pública | Limpia la cookie `efisco_token` |
| GET | `/api/auth/me` | Taller | Restaura la sesión al recargar la página (la cookie httpOnly ya no se puede leer/decodificar en el cliente) — devuelve el mismo `user` rico que `login()`, exento de los gates de contrato/autorización del taller |
| GET | `/api/auth/support-session/exchange` | Pública (token de un solo uso en la URL) | Puente de la llave de acceso a cuenta del cliente — valida el `support_token` emitido por `POST /api/admin/workshops/:id/access-keys/redeem` y setea la cookie de sesión |
| POST | `/api/auth/request-password-reset` | Pública | Self-service "olvidé mi contraseña" — bifurca por tipo de cuenta (2026-08-24): si el correo es de un **empleado** (mecánico/admin/contador, activo y con `password_hash`), manda un JWT propio de 30 min por `emailService.sendEmployeePasswordResetEmail` (no toca Supabase, ver [SECURITY.md](SECURITY.md#empleados-no-viven-en-supabase-auth--recuperación-de-contraseña-propia-2026-08-24)); si es el **dueño**, sigue enviando el correo real de Supabase solo si el taller ya fue activado por un admin (`workshop_config.admin_activated_at`). Respuesta genérica siempre igual en ambos casos, ver [SECURITY.md](SECURITY.md) |
| POST | `/api/auth/reset-password` | Taller (token de recovery) | Confirma la nueva contraseña del **dueño** — recibe el `access_token` del link de recovery de Supabase como Bearer y el password nuevo (excepción: este token viaja por header, no por cookie — ver [SECURITY.md](SECURITY.md#migración-de-autenticación-a-cookies-httponly-2026-07-21)) |
| POST | `/api/auth/reset-employee-password` | Pública (JWT de un solo uso en el body) | (2026-08-24) Contraparte de `reset-password` para **empleados** — recibe `{token, password}`; el JWT lleva `pwv` = sha256 del `password_hash` vigente al generarlo, así que en cuanto la contraseña cambia (por este mismo link o por otro medio) el link queda invalidado sin necesitar una tabla de tokens usados. Ver [SECURITY.md](SECURITY.md#empleados-no-viven-en-supabase-auth--recuperación-de-contraseña-propia-2026-08-24) |
| POST | `/api/auth/sign-contract` | Taller | Firma electrónica del Contrato de Afiliación B2B (reingreso de contraseña) |
| POST | `/api/auth/accept-staff-terms` | Taller | Aceptación de la autorización del taller (staff no-dueño) |
| POST | `/api/auth/acknowledge-legal-notice` | Taller | Registra el reconocimiento (dueño o empleado, IP + timestamp) de un aviso corto y no bloqueante de cambio en Términos/Privacidad — `key` se valida contra el espejo del servidor, no se confía la fecha del cliente. Ver [SECURITY.md](SECURITY.md#aviso-no-bloqueante-de-cambios-legales-con-evidencia-server-side-2026-09-02) |
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
| PUT | `/api/workshop/:id` | Taller | Actualizar configuración (campos fiscales/PUC/Dataico exclusivos de `contador`, ver [BUSINESS_RULES.md](../Reglas%20de%20Negocio%20y%20Finanzas/BUSINESS_RULES.md#configuración-del-taller-config)). Incluye `city` (campo básico, no fiscal — editable por cualquier rol permitido igual que `address`/`phone`; usado como Tercero por defecto del taller en el Libro Mayor, 2026-07-29) |
| POST | `/api/workshop/finance-settings` | Taller (contador) | Actualizar parámetros financieros — exclusivo del rol `contador` (403 para cualquier otro rol), mismo criterio que los campos fiscales de `PUT /:id`. Sin caller activo en el frontend (la UI usa `PUT /:id`); si se envía `fiscal_regime`, se sincroniza con los 3 booleans de régimen igual que en `PUT /:id` (rev. 35 — antes desincronizaba `fiscal_regime` de Gran Contribuyente/Ordinario) |

## Mechanics (`/api/mechanics`)

| Método | Ruta | Auth | Descripción |
|:---|:---|:---:|:---|
| POST | `/api/mechanics/` | Taller (owner, o admin con `can_manage_catalog=true`) | Crear mecánico/empleado — `role`/`payment_type` validados contra los valores reales del sistema (`mecanico/admin/contador`, `salario/comision/mixto`), 400 si no coinciden (fix 2026-07-19). Acepta además `document_type`/`nit`/`address`/`city`/`phone`, opcionales — Datos de Tercero para el Libro Mayor (2026-07-29), ver [FINANCIAL_ENGINE.md](../Reglas%20de%20Negocio%20y%20Finanzas/FINANCIAL_ENGINE.md#tercero-completo--tipo-de-documentocontribuyente-dirección-ciudad-correo-teléfono-2026-07-29). `can_manage_catalog` en el body (2026-08-23) solo se respeta si el actor es `owner` — un admin no puede otorgárselo a otro admin que él cree, ver [SECURITY.md](SECURITY.md#permisos-delegables-de-admin-catálogo-desde-la-orden-y-modo-mecánico-2026-08-23) |
| POST | `/api/mechanics/self` | Taller (owner) | El dueño se registra a sí mismo como mecánico |
| PATCH | `/api/mechanics/self/mechanic-mode` | Taller (admin) | (2026-08-23) Auto-servicio: el propio admin activa/desactiva `available_as_mechanic` — con esto en `true` puede quedar asignado como mecánico en una orden sin perder sus permisos de admin. 403 si `role !== 'admin'`. Ver [SECURITY.md](SECURITY.md#permisos-delegables-de-admin-catálogo-desde-la-orden-y-modo-mecánico-2026-08-23) |
| GET | `/api/mechanics/:id/salary-history` | Taller | Historial salarial |
| GET | `/api/mechanics/:id/metrics` | Taller | Métricas del mecánico |
| GET | `/api/mechanics/:id/pending-payment` | Taller | Comisión devengada pendiente de pago (`Σ MECH_COMMISSION − Σ MECH_COMMISSION_PAY`) |
| POST | `/api/mechanics/:id/pay` | Taller (owner) | Registrar pago real a un mecánico (sueldo y/o comisión) — genera `MECH_SALARY_PAY`/`MECH_COMMISSION_PAY` en el Libro Auxiliar, ver [FINANCIAL_ENGINE.md](../Reglas%20de%20Negocio%20y%20Finanzas/FINANCIAL_ENGINE.md#caja-real-vs-informativodevengo--por-qué-existe-la-distinción) |
| PATCH | `/api/mechanics/:id/deactivate` | Taller (owner) | Desactivar/reactivar empleado. Al desactivar (2026-08-23) también desconecta el Google Calendar de ese empleado si tenía uno conectado (`calendar_disconnected` en la respuesta) — ver [SECURITY.md](SECURITY.md#decisiones-técnicas-de-seguridad) |
| PATCH | `/api/mechanics/:id` | Taller | Editar empleado — `payment_type` (si se envía) validado contra `salario/comision/mixto`, 400 si no coincide (fix 2026-07-19). Único endpoint para completar/corregir `document_type`/`nit`/`address`/`city`/`phone` de un empleado ya creado (2026-07-29) — sin historial de auditoría, a diferencia de sueldo/comisión/esquema de pago. **2026-09-08**: también acepta `hourly_rate` (mano de obra por hora fijada a mano) — no enviarlo deja el valor actual intacto, enviarlo vacío lo limpia a `NULL` (vuelve a automático) |
| POST | `/api/mechanics/:id/account` | Taller (owner, o admin con `can_manage_catalog=true`) | Habilitar credenciales de acceso |
| GET | `/api/mechanics/:workshop_id` | Taller | Listar mecánicos del taller |
| PATCH | `/api/mechanics/:id/calendar-sync` | Taller (owner) | Habilita/deshabilita que la conexión de Google Calendar de ese empleado reciba eventos de órdenes — 403 si quien llama no es `owner`. No conecta a nadie por sí solo, ver [Google Calendar](#google-calendar-apigoogle-calendar) |
| PATCH | `/api/mechanics/:id/permissions` | Taller (owner) | (2026-08-23) Otorga/revoca `can_manage_catalog` de un empleado — 400 si el target no tiene `role === 'admin'`. **2026-09-07**: también togglea `restricted_view` — 400 si el target no tiene `role === 'mecanico'`. Ver [SECURITY.md](SECURITY.md#vista-restringida-de-mecánico--bug-real-de-aislamiento-de-órdenes-2026-090708) |

## Google Calendar (`/api/google-calendar`)

| Método | Ruta | Auth | Descripción |
|:---|:---|:---:|:---|
| GET | `/api/google-calendar/connect` | Taller | Redirige al consentimiento OAuth de Google (`state` = JWT corto de 10 min firmado con `JWT_SECRET`). 400 si el usuario aún no tiene `employee_id` (debe auto-registrarse como mecánico primero, ver [BUSINESS_RULES.md](../Reglas%20de%20Negocio%20y%20Finanzas/BUSINESS_RULES.md#configuración-del-taller-config)) |
| GET | `/api/google-calendar/callback` | Pública (la redirección la hace Google) | Intercambia el `code`, valida que el scope realmente otorgado incluya `calendar.events` (ver [SECURITY.md](SECURITY.md#google-calendar-y-tokens-oauth-por-persona-2026-07-27)) y guarda/actualiza la conexión. Redirige a `${FRONTEND_URL}${return_path}?gcal=success` o `?gcal=error&reason=missing_calendar_scope` |
| DELETE | `/api/google-calendar/connect` | Taller | Desconecta — revoca el token en Google (best-effort) y borra la fila de conexión |
| GET | `/api/google-calendar/status` | Taller | `{ connected, google_email, calendar_id, connected_at }` — nunca devuelve tokens |

## Work Orders (`/api/work-orders`)

| Método | Ruta | Auth | Descripción |
|:---|:---|:---:|:---|
| POST | `/api/work-orders/` | Taller | Crear orden de trabajo — acepta `services[]`, `mechanics[]` (multi-servicio/multi-mecánico) y `tools[]` (array de `tool_id`, opcional, solo informativo — agregado 2026-08-01, ver [Tools](#tools--herramientas-apitools)); el payload legacy de escalares sigue soportado. También acepta `client_name` (agregado 2026-08-01 — `work_orders` no tenía columna de nombre, solo cédula/teléfono; ver [OPERATIONS.md](../Reglas%20de%20Negocio%20y%20Finanzas/OPERATIONS.md#bahías-órdenes-de-trabajo)). **`quote_id`** (2026-09-10, opcional): si la orden viene de una cotización aprobada, actualiza `quotes.work_order_id` con el id de la orden recién creada (best-effort, nunca hace fallar la creación) — ver [OPERATIONS.md — Cotizaciones](../Reglas%20de%20Negocio%20y%20Finanzas/OPERATIONS.md#cotizaciones-cotizaciones). Valida fecha de entrega futura, dentro del horario del taller (`workshop_config.open_time`/`close_time`, agregado 2026-07-25, 400 si no), que todos los asignados tengan rol `mecanico` (400 si no) y **409 si alguna herramienta pedida ya no tiene unidades disponibles** (`validateToolsAvailable`, agregado 2026-08-01). **La orden nace pausada** (`is_paused:true`, 2026-08-09) — el cronómetro de tiempo real no arranca hasta que el mecánico asignado le da "Reanudar", ver [OPERATIONS.md — Bahías](../Reglas%20de%20Negocio%20y%20Finanzas/OPERATIONS.md#bahías-órdenes-de-trabajo) |
| GET | `/api/work-orders/` | Taller | Listar órdenes activas (incluye `services[]` con márgenes, `mechanics_detail[]` y `tools_used[]` — `{tool_id, name}`, agregado 2026-08-01) |
| GET | `/api/work-orders/history` | Taller | Historial de órdenes completadas (incl. `mechanics_names[]`/`services_names[]`/`parts_used[]` — repuesto, cantidad, lado/posición y proveedor si se eligió uno, agregado 2026-07-23; `tools_names[]` agregado 2026-08-01; `client_email` agregado 2026-08-18, join a `clients` por `client_cedula`, `null` si el cliente nunca completó el registro público; para rol mecánico filtra por sus filas en `work_order_mechanics` + el escalar legacy). Acepta `from`/`to` (rango sobre `end_time`) y `limit`/`offset` (default 50, tope 100) — respuesta `{ data, total }`, no un array plano (fix 2026-07-19, antes traía todo el historial sin límite, ver [OPERATIONS.md](../Reglas%20de%20Negocio%20y%20Finanzas/OPERATIONS.md#bahías-órdenes-de-trabajo)) |
| GET | `/api/work-orders/history/calendar` | Taller | Conteo de órdenes completadas por día para un mes (`?month=YYYY-MM`, default mes en curso) — agregado 2026-07-25 para el calendario de `Historial.jsx`, mismo filtro de rol/mecánico que `/history` pero sin traer filas completas (solo `id, end_time`) |
| GET | `/api/work-orders/pending-by-cedula/:cedula` | Taller | Orden `pending` precargada desde Registro Seguro EFISCO (query `intake_id`) |
| GET | `/api/work-orders/last-vehicle-by-cedula/:cedula` | Taller | Último vehículo real de esa cédula (cualquier estado salvo `pending`) — sugerencia "¿Mismo vehículo de la última vez?" en Bahía, ver [OPERATIONS.md — Bahías](../Reglas%20de%20Negocio%20y%20Finanzas/OPERATIONS.md#bahías-órdenes-de-trabajo). **Bug real corregido 2026-08-10** (reportado por el usuario, 500 en este endpoint): seleccionaba una columna `vehicle_type` que nunca existió en `work_orders` (la real es `vehicle_age_category`) — corregido con alias de Postgrest (`vehicle_type:vehicle_age_category`) para no tener que tocar el frontend |
| PUT | `/api/work-orders/:id/pause` | Taller | Pausar/reanudar orden |
| PUT | `/api/work-orders/:id/finish` | Taller | Marcar como lista para facturar |
| PUT | `/api/work-orders/:id/status` | Taller | Cambiar estado — validado contra los 4 valores reales del flujo (`pending/ejecucion/ready_to_invoice/completed`), 400 si no coincide (fix 2026-07-19; antes aceptaba cualquier string, ver [OPERATIONS.md — Bahías](../Reglas%20de%20Negocio%20y%20Finanzas/OPERATIONS.md#bahías-órdenes-de-trabajo)) |
| PUT | `/api/work-orders/:id` | Taller | Editar orden — mismos arrays y validaciones que el POST (incl. horario del taller, "solo si el valor cambió" igual que el guard de fecha en el pasado, y disponibilidad de herramientas excluyendo los enlaces de la propia orden); reemplaza las filas hijas (replace-children, incl. `work_order_tools`). **409 si la orden ya está `ready_to_invoice`** (agregado 2026-07-25 — una orden finalizada no se puede editar, ver [OPERATIONS.md — Bahías](../Reglas%20de%20Negocio%20y%20Finanzas/OPERATIONS.md#bahías-órdenes-de-trabajo)) |
| DELETE | `/api/work-orders/:id` | Taller | Eliminar orden — **409 si la orden ya está `ready_to_invoice`** (agregado 2026-07-25, mismo criterio que el PUT) |
| POST | `/api/work-orders/:id/photos` | Taller | (2026-08-26) Subir foto del vehículo — `multipart/form-data` (`photo`, `type: 'before' \| 'after'`). Sube al bucket público `vehicle-photos` (creado on-demand, mismo criterio que `tool-photos`) — ver [OPERATIONS.md — Fotos del vehículo](../Reglas%20de%20Negocio%20y%20Finanzas/OPERATIONS.md#bahías-órdenes-de-trabajo) |
| GET | `/api/work-orders/:id/photos` | Taller | Listar fotos de una orden (ambos `type`, el frontend filtra) |
| DELETE | `/api/work-orders/photos/:photoId` | Taller | Eliminar una foto |

## Tools / Herramientas (`/api/tools`)

Ver [INVENTORY.md — Herramientas](../Reglas%20de%20Negocio%20y%20Finanzas/INVENTORY.md#herramientas). Tablero visual de herramientas, separado de `/api/inventory` (una herramienta no se consume ni se vende). Agregado 2026-08-01, pedido explícito del usuario.

| Método | Ruta | Auth | Descripción |
|:---|:---|:---:|:---|
| GET | `/api/tools/` | Taller | Listar herramientas del taller, con `status`/`in_use_count`/`available_count`/`in_use_by[]` (`{work_order_id, client_plate, mechanics[]}`) calculados en vivo según qué unidades están enlazadas ahora mismo (`work_order_tools`) a una orden en `status='ejecucion'` — nunca se leen de una columna guardada |
| POST | `/api/tools/` | Taller | Crear herramienta — `multipart/form-data` (`name`, `quantity` ≥1 default 1, `damaged_quantity` ≥0 tope `quantity`, `photo` opcional). Sube la foto a Supabase Storage (bucket público `tool-photos`, creado on-demand) |
| PUT | `/api/tools/:id` | Taller | Editar herramienta — mismos campos que el POST, todos opcionales (parche parcial); reemplaza la foto si viene una nueva. **409 si se intenta bajar `quantity` por debajo de las unidades en uso ahora mismo** |
| DELETE | `/api/tools/:id` | Taller | Eliminar herramienta — **409 si ya tiene historial de uso** en `work_order_tools` (activo o pasado), mismo criterio que `DELETE /api/inventory/:id` |

## Inventory (`/api/inventory`)

| Método | Ruta | Auth | Descripción |
|:---|:---|:---:|:---|
| GET | `/api/inventory/` | Taller | Listar inventario |
| POST | `/api/inventory/standalone` | Taller | Alta de ítem sin compra asociada — cantidad inicial validada (≥0) y `measure` contra `validateMeasure` si viene con `category`, mismo criterio que añadir un repuesto a una orden (fix 2026-07-19). Acepta `default_part_side`/`default_part_position` (agregado 2026-07-23) — ubicación habitual del ítem, autocompleta (editable) el Lado/Posición al agregarlo a una orden en Bahía. Si `cantidad × costo_unitario > 0`, inserta el par `INV_OPENING_BALANCE`/`INV_OPENING_BALANCE_EQUITY` en `cash_flow_ledger` (bug real corregido 2026-07-28, antes no tocaba el ledger — ver [FINANCIAL_ENGINE.md](../Reglas%20de%20Negocio%20y%20Finanzas/FINANCIAL_ENGINE.md#inventario-inicial--partida-doble-2026-07-28)) |
| GET | `/api/inventory/total-investment` | Taller | Valor total de inventario |
| GET | `/api/inventory/matrix` | Taller | Tab "Matriz": rotación, uso promedio, stock mínimo vital |
| GET | `/api/inventory/pending-invoices/today` | Taller | Facturas OCR pendientes de revisión del día |
| GET | `/api/inventory/history/:id` | Taller | Kardex de un ítem, ordenado por `requested_at` (bug real corregido 2026-07-23: antes ordenaba por `created_at`, columna inexistente en `inventory_transactions` — Postgres rechazaba la consulta completa, 500 sin importar el ítem; expuesto como `created_at` en la respuesta por alias, no por columna, porque el frontend ya esperaba ese nombre) |
| PUT | `/api/inventory/add-stock/:id` | Taller | Registrar entrada de stock — cantidad validada como número positivo, 400 si no (fix 2026-07-19; antes un valor no numérico o negativo corrompía `current_stock`). Mismo asiento `INV_OPENING_BALANCE`/`INV_OPENING_BALANCE_EQUITY` que `/standalone` (bug real corregido 2026-07-28, mismo helper compartido `postInventoryOpeningBalance`) |
| GET | `/api/inventory/work-order/:work_order_id` | Taller | Ítems consumidos por una orden, incl. `providers.name` (join, agregado 2026-07-23 — `null` si no se eligió proveedor al agregarlo) |
| POST | `/api/inventory/work-order/:work_order_id` | Taller | Añadir repuesto a una orden (descuenta stock; excepción: categoría "Lubricantes y Químicos" con `container_emptied=false` no descuenta, ver [INVENTORY.md — Inventario](../Reglas%20de%20Negocio%20y%20Finanzas/INVENTORY.md#inventario)) — cantidad validada como entero positivo y topada contra el stock disponible (409 si no alcanza), fix 2026-07-19. Acepta `provider_id`/`part_side`/`part_position` (bug real corregido 2026-07-23 — el frontend ya los mandaba desde antes, pero este endpoint nunca los leía del body ni los guardaba; ver [OPERATIONS.md — Bahías](../Reglas%20de%20Negocio%20y%20Finanzas/OPERATIONS.md#bahías-órdenes-de-trabajo)) |
| PUT | `/api/inventory/work-order-item/:item_id` | Taller | Editar ítem dentro de una orden |
| DELETE | `/api/inventory/work-order-item/:item_id` | Taller | Quitar ítem de una orden |
| PUT | `/api/inventory/:id` | Taller | Editar ítem de inventario, incl. `default_part_side`/`default_part_position` (agregado 2026-07-23). **Ignora `quantity`** (bug real corregido 2026-07-25) — este endpoint no puede tocar el stock aunque se lo manden; solo `PUT /api/inventory/add-stock/:id` ("Añadir Stock") lo mueve, porque es el único que genera `inventory_transactions` |
| DELETE | `/api/inventory/:id` | Taller | Eliminar ítem de inventario |

## Providers (`/api/providers`)

| Método | Ruta | Auth | Descripción |
|:---|:---|:---:|:---|
| GET | `/api/providers/` | Taller | Listar proveedores |
| POST | `/api/providers/` | Taller | Crear proveedor |
| PUT | `/api/providers/:id` | Taller | Editar proveedor — scoped al taller de la sesión (404 si no es suyo), lista blanca de campos editables (`name`/`nit`/`email`/`phone`/`address`/`city`/`document_type`/`taxpayer_type`/`supplier_regime`/`is_responsible_vat`/`is_declarante`/`reteica_rate_supplier`/`puc_account_expense`; no acepta `workshop_id`/`is_system_provider`), 403 si `is_system_provider`. `document_type`/`taxpayer_type` agregados 2026-07-28 — capturados en el modal "Nuevo/Editar Proveedor" de `Proveedores.jsx` (rol dueño), no desde `/contador`; `taxpayer_type` no tiene input propio, se deriva de `supplier_regime` (ver [BUSINESS_RULES.md](../Reglas%20de%20Negocio%20y%20Finanzas/BUSINESS_RULES.md#configuración-del-taller-config)) |
| DELETE | `/api/providers/:id` | Taller | Eliminar proveedor — scoped al taller de la sesión (404 si no es suyo), 403 si `is_system_provider` |
| POST | `/api/providers/invoice-ocr` | Taller | OCR de factura (AWS Textract, imagen ≤5MB) |
| POST | `/api/providers/purchase` | Taller | Registrar compra y liquidar (`financialEngine.liquidateSupplierPurchase`) — acepta `payment_mode:'credito'`+`num_installments`+`first_payment_date` para pago a plazos (genera `supplier_installments`, cabecera en `status:'pendiente'`); `due_dates` opcional (array de "YYYY-MM-DD", una por cuota) manda sobre el cálculo automático por intervalo si el frontend edita las fechas a mano (rev. 48). **`payment_method` obligatorio** (`banco`/`tarjeta`/`efectivo`, 400 si falta o es otro valor — bug real corregido 2026-07-25, antes tenía default silencioso `'banco'`) |
| GET | `/api/providers/efficiency` | Taller | Eficiencia de entrega (Fase 3) |
| GET | `/api/providers/installments` | Taller | Listar cuotas por pagar a proveedores |
| PATCH | `/api/providers/installments/:id/pay` | Taller | Pagar una cuota — genera la salida de caja real (`SUP_PAY` con `net_amount = amount`) |
| GET | `/api/providers/:id/purchases` | Taller | Historial de compras a un proveedor |

## Services (`/api/services`)

| Método | Ruta | Auth | Descripción |
|:---|:---|:---:|:---|
| POST | `/api/services/` | Taller (owner, o admin con `can_manage_catalog=true`) | Crear servicio maestro — `gama`/`complejidad` obligatorios (`'Alta'`\|`'Baja'`, 400 si no), usados solo para sugerir `base_margin_basic`/`base_margin_premium` en el formulario (rev. 49). El gate de rol se agregó 2026-08-23 (antes no tenía ninguno en el backend — bug real corregido, ver [SECURITY.md](SECURITY.md#permisos-delegables-de-admin-catálogo-desde-la-orden-y-modo-mecánico-2026-08-23)) |
| GET | `/api/services/:workshop_id/:vehicleType` | Taller | Servicios por tipo de vehículo |
| GET | `/api/services/` | Taller | Listar todos los servicios |
| PUT | `/api/services/:id` | Taller (owner, o admin con `can_manage_catalog=true`) | Editar servicio (márgenes básico/premium, `gama`/`complejidad` obligatorios igual que en creación). Gate de rol agregado 2026-08-24 (antes no tenía ninguno — mismo bug de `createServiceMaster` pre-2026-08-23, ver [SECURITY.md](SECURITY.md#permisos-delegables-de-admin-catálogo-desde-la-orden-y-modo-mecánico-2026-08-23)) |
| DELETE | `/api/services/:id` | Taller (owner, o admin con `can_manage_catalog=true`) | Eliminar servicio. Mismo gate de rol agregado 2026-08-24 |
| GET | `/api/services/:serviceId/kit` | Taller | Kit del servicio — `{found:false}` si no hay kit guardado, o `{found:true, mechanic, estimated_minutes, items[], tools[]}`. **Sin ningún llamador en el frontend desde 2026-09-10** (feature de kits quitada de Bahía por redundante con Cotizaciones, ver [OPERATIONS.md — Bahías](../Reglas%20de%20Negocio%20y%20Finanzas/OPERATIONS.md#bahías-órdenes-de-trabajo)); se deja intacto a propósito, no se borró nada del backend |
| POST | `/api/services/kit/save-from-order` | Taller | `{work_order_id}` → guarda/reemplaza el kit del servicio de esa orden con lo que de verdad pasó (mecánico primario, tiempo real, repuestos/herramientas usados). **Sin ningún llamador en el frontend desde 2026-09-10**, mismo motivo que el endpoint de arriba |

## Quick Intakes / Recepción (`/api/quick-intakes`)

| Método | Ruta | Auth | Descripción |
|:---|:---|:---:|:---|
| POST | `/api/quick-intakes/` | Taller | Registrar ingreso rápido — **solo `client_cedula`/`reported_issue` obligatorios** (2026-08-09, antes también nombre/apellido/teléfono; ver [OPERATIONS.md — Recepción](../Reglas%20de%20Negocio%20y%20Finanzas/OPERATIONS.md#recepción)). Requiere la migración `2026-08-09_quick_intakes_optional_name_phone.sql` |
| GET | `/api/quick-intakes/` | Taller | Listar cola de ingresos ("Por Hacer") — cada fila trae `client_registered` (2026-08-09): `true` si esa cédula ya tiene ficha en `clients`, usado para no ofrecer el QR de registro a un cliente ya registrado. **2026-09-08**: también `client_plate` (propio o heredado por cédula de la última `work_orders` real, ver [OPERATIONS.md](../Reglas%20de%20Negocio%20y%20Finanzas/OPERATIONS.md#bahías-órdenes-de-trabajo)) y `claimed_mechanics` (quiénes ya se unieron, ver `/claim` abajo) |
| GET | `/api/quick-intakes/ready` | Taller | Ingresos listos para pasar a Bahía — mismo enriquecimiento (`client_plate`/`claimed_mechanics`) que arriba |
| POST | `/api/quick-intakes/:id/claim` | Taller (cualquier rol excepto `contador`) | (2026-09-08) "Unirme" — auto-asignación rápida del actor a un ingreso de "Por Hacer" (`upsert` idempotente en `quick_intake_claims`). 400 si el actor no tiene `employee_id`. Ver [SECURITY.md](SECURITY.md#unirme--auto-asignación-en-por-hacer) |
| PATCH | `/api/quick-intakes/:id/plate` | Taller | (2026-09-08) Actualiza `client_plate` del ingreso — usado para reflejar en "Por Hacer" la placa que se está tecleando en el formulario de Bahía, antes de crear la orden |
| GET | `/api/quick-intakes/:id/registration-status` | Taller | `{registered: boolean}` (2026-08-09) — polling del modal de QR en Recepción cada 3s (mismo patrón que `/telegram-link-status`); la señal es una fila en `client_legal_acceptances` con ese `intake_id`, lo último que escribe `publicRegister` |
| PUT | `/api/quick-intakes/:id/bahia` | Taller | Convertir ingreso en orden de Bahía. **Sin ningún llamador en el frontend desde 2026-09-10** — el camino de revisión que lo usaba pasó a crear una cotización en su lugar (ver `/quote` justo abajo); se deja la ruta y el controlador intactos a propósito (pedido explícito del usuario: "solo quitar del frontend"), no se borró nada del backend |
| POST | `/api/quick-intakes/:id/revision-order` | Taller | (2026-08-26) "¿Es solo revisión?" — crea una orden mínima (`work_orders.status='revision'`, invisible para el Kanban de Bahía) directo desde placa/marca/modelo/monto, sin pasar por `createWorkOrder`, para poder facturar solo el diagnóstico vía `ModalLiquidacion`. `client_cedula` nulo (ingreso anónimo) fluye tal cual — ver [OPERATIONS.md — Consumidor Final genérico DIAN](../Reglas%20de%20Negocio%20y%20Finanzas/OPERATIONS.md#consumidor-final-genérico-dian-2026-08-22) |
| PUT | `/api/quick-intakes/:id/revision-liquidada` | Taller | (2026-08-26) Marca el ingreso como `revision_liquidada` tras liquidar con éxito la orden de `revision-order` — lo saca de la cola de Recepción |
| POST | `/api/quick-intakes/:id/quote` | Taller | (2026-09-09) Botón "Cotizar" — crea una `quotes` vacía precargada con cliente/motivo/placa del intake, pasa `quick_intakes.status` a `'cotizacion'`. Usado tanto por un ingreso normal como, desde 2026-09-10, por uno de revisión tras cobrar (o no) el diagnóstico — el body acepta opcionalmente `client_plate`/`vehicle_brand`/`vehicle_model`/`vehicle_year`/`vehicle_mileage` (lo que ya se capturó en el mini-formulario de revisión), con fallback a lo que ya tenía el intake si no vienen. Ver [OPERATIONS.md — Cotizaciones](../Reglas%20de%20Negocio%20y%20Finanzas/OPERATIONS.md#cotizaciones-cotizaciones) |
| DELETE | `/api/quick-intakes/:id` | Taller | Eliminar ingreso |
| POST | `/api/quick-intakes/:id/whatsapp` | Taller | Enviar link de Registro Seguro EFISCO (WhatsApp, o Telegram si `client_cedula` del ingreso ya está vinculado y WhatsApp no está configurado). Sin cédula capturada (campo opcional del ingreso rápido), Telegram no puede actuar como respaldo |

## Quotes / Cotizaciones (`/api/quotes`)

Estimación no vinculante entre Recepción y Bahía — ver [OPERATIONS.md — Cotizaciones](../Reglas%20de%20Negocio%20y%20Finanzas/OPERATIONS.md#cotizaciones-cotizaciones) para el flujo completo. Todas las rutas devuelven/reciben la cotización con `items[]` embebido (tabla `quote_items`).

| Método | Ruta | Auth | Descripción |
|:---|:---|:---:|:---|
| GET | `/api/quotes/` | Taller | Listar todas las cotizaciones del taller (las 3 columnas del tablero se filtran en el frontend por `status`). Cada cotización con `work_order_id` trae además `bahia_parts[]` (2026-09-10) — todo lo que el mecánico agregó en "Repuestos" de esa orden después de cotizar (nombre/cantidad/precio/`created_at`), para el bloque "Agregado en Bahía" de la tarjeta. El frontend hace polling de este endpoint cada ~18s mientras `/cotizaciones` está abierta |
| GET | `/api/quotes/by-intake/:intakeId` | Taller | Cotización vinculada a un ingreso de Recepción — `null` (no 404) si el intake nunca pasó por Cotizaciones. Bahía la consulta al abrir "Iniciar Orden" para precargar el formulario |
| GET | `/api/quotes/by-work-order/:workOrderId` | Taller | (2026-09-10) Cotización vinculada a una orden real ya creada, vía `quotes.work_order_id` (no `intake_id`, que queda `null` apenas se crea la orden — ver el bug real corregido en [OPERATIONS.md](../Reglas%20de%20Negocio%20y%20Finanzas/OPERATIONS.md#cotizaciones-cotizaciones)). Bahía la consulta al abrir "Añadir Piezas" para sugerir los repuestos cotizados |
| PUT | `/api/quotes/:id` | Taller | Reemplaza cabecera (cliente/vehículo/complejidad/horas) + la lista completa de ítems. Editable en `pendiente` o `aprobada`, 400 si `rechazada` (estado terminal) |
| PATCH | `/api/quotes/:id/approve` | Taller | Solo desde `pendiente`. 400 si no tiene al menos un ítem |
| PATCH | `/api/quotes/:id/reject` | Taller | Desde `pendiente` **o** `aprobada` (2026-09-10 — el cliente puede retirarse después de haber aprobado). 400 si ya estaba `rechazada`. Si tiene `intake_id`, pasa el intake a `quick_intakes.status = 'cotizacion_rechazada'` |
| POST | `/api/quotes/:id/send-to-bahia` | Taller | Solo desde `aprobada`. Pasa `quick_intakes.status = 'in_bahia'` (mismo estado que antes disparaba directo desde Recepción) — reaparece en "Por Hacer" de Bahía |

## Clients (`/api/clients`)

| Método | Ruta | Auth | Descripción |
|:---|:---|:---:|:---|
| GET | `/api/clients/public-intake/:workshopId/:intakeId` | Pública | Info del ingreso vinculado (Registro Seguro EFISCO) |
| GET | `/api/clients/public-score-pdf/:workshopId/:intakeId` | Pública | Descarga del PDF de score del propio cliente |
| POST | `/api/clients/public-register/:workshopId` | Pública | Envío del formulario de Registro Seguro EFISCO — incluye `tipo_documento` (2026-07-29, mismo dato de tercero para el Libro Mayor que ya se pide en el registro de proveedores, guardado en `clients.document_type`; `ciudad` se pedía también desde esa fecha pero se quitó del formulario 2026-08-18, ver [FINANCIAL_ENGINE.md](../Reglas%20de%20Negocio%20y%20Finanzas/FINANCIAL_ENGINE.md#tercero-completo--tipo-de-documentocontribuyente-dirección-ciudad-correo-teléfono-2026-07-29)). **Acepta `nombre`/`apellido`/`celular` del body como respaldo (2026-08-09)** cuando el intake vinculado no los trae (Recepción ahora solo exige cédula+motivo) — si el intake ya los tenía, se ignora cualquier valor del body para esos campos, igual que siempre se ignoró un intento de mandar otra cédula; 400 si ni el intake ni el body los traen. Ver [OPERATIONS.md — Registro Seguro EFISCO](../Reglas%20de%20Negocio%20y%20Finanzas/OPERATIONS.md#registro-seguro-efisco-clienteregistroworkshopidintakeid) |
| POST | `/api/clients/sweep-freezes` | Taller | Barrido de congelamientos de score vencidos |
| GET | `/api/clients/by-phone/:phone` | Taller | Buscar cliente por teléfono (match parcial, últimos 10 dígitos) |
| GET | `/api/clients/by-cedula/:cedula` | Taller | Buscar cliente por cédula (match exacto por `nit`, scoped al taller) — usado por Recepción para autocompletar nombre/apellido/WhatsApp de clientes ya registrados. **También trae `lastVehicle` (2026-08-09)**, buscado en paralelo en `work_orders` (mismo criterio que `last-vehicle-by-cedula`) — viaja aparte de `found` y puede venir presente aunque `found` sea `false` (cliente con historial de órdenes que nunca llenó el formulario público) |
| POST | `/api/clients/:cedula/request-otp` | Taller | Enviar OTP de verificación (WhatsApp, o Telegram si el cliente ya vinculó y WhatsApp no está configurado — ver `messagingChannel.service.js`). `reason: 'no_channel_linked'` si Telegram está disponible pero este cliente no vinculó (el frontend muestra el QR). El `demo_code` (código devuelto directo al recepcionista) solo existe fuera de producción o con `OTP_DEMO_MODE=true` y si NINGÚN canal está configurado; en producción responde 503/502 |
| POST | `/api/clients/:cedula/verify-otp` | Taller | Verificar OTP → única forma de ver el score (local + global, con desglose) — decisión de producto 2026-07-19: se eliminó `GET /:cedula/risk-score` (score sin código, gratis); ver [OPERATIONS.md — Recepción](../Reglas%20de%20Negocio%20y%20Finanzas/OPERATIONS.md#recepción). Envía el PDF por el canal disponible. Anti fuerza-bruta: 5 intentos fallidos por (taller, cédula) → 429 por 15 min; comparación constant-time |
| POST | `/api/clients/:cedula/telegram-link-request` | Taller | Genera el QR de vinculación de Telegram (deep-link `t.me/<bot>?start=<token>`, expira en 10 min) — `alreadyLinked: true` si la cédula ya está vinculada (global, cualquier taller de la red), sin generar uno nuevo. 503 si Telegram no está configurado |
| GET | `/api/clients/:cedula/telegram-link-status` | Taller | Estado de vinculación — usado por Recepción para hacer polling corto (~3s) mientras el QR está en pantalla |

## Billing / Liquidación (`/api/billing`)

| Método | Ruta | Auth | Descripción |
|:---|:---|:---:|:---|
| POST | `/api/billing/settle` | Taller | Liquidar orden (`financialEngine.liquidateClientInvoice`, emite Dataico + comprobante WhatsApp) — acepta `payment_mode:'credito'`+`num_installments`+`first_payment_date` para venta a crédito; `due_dates` opcional (array de "YYYY-MM-DD", una por cuota) manda sobre el cálculo automático por intervalo si el frontend edita las fechas a mano (rev. 48) |
| GET | `/api/billing/installments` | Taller | Listar cuotas de crédito |
| PATCH | `/api/billing/installments/:id/pay` | Taller | Pagar cuota (genera `INC_GROSS`, envía comprobante WhatsApp) — `amount` opcional (2026-09-08): sin él paga la cuota completa; menor al saldo = abono parcial (`paid_amount`); mayor = se reparte hacia las siguientes cuotas pendientes de la orden, ver [FINANCE.md](../Reglas%20de%20Negocio%20y%20Finanzas/FINANCE.md#panel-de-cobros-cobros) |
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
| GET | `/api/messaging/status` | Taller | `{whatsapp, telegram, frontendUrl}` — qué canales están configurados a nivel servidor. El frontend (Recepción) lo usa para mostrar el botón de WhatsApp activo/deshabilitado y ofrecer Telegram al lado sin tener que intentar un envío primero. `frontendUrl` (2026-08-09) es `process.env.FRONTEND_URL` del servidor — el QR de registro de Recepción lo usa en vez de `window.location.origin` del navegador, para no depender de dónde esté cargado el navegador del recepcionista |

## Finance / Análisis Financiero (`/api/finance`)

| Método | Ruta | Auth | Descripción |
|:---|:---|:---:|:---|
| GET | `/api/finance/dashboard-summary` | Taller | Resumen agregado para el Dashboard — incluye `today` (ingresos/egresos/vehículos del día en curso) y `financialMetrics.cashAccounts` (desglose Efectivo/Bancos, 2026-09-08, ver [FINANCIAL_ENGINE.md](../Reglas%20de%20Negocio%20y%20Finanzas/FINANCIAL_ENGINE.md#4-desglose-efectivo-vs-bancos-calculatecashaccountbalances)) |
| POST | `/api/finance/referral` | Taller (owner) | Registrar ingreso por comisión de referido — 403 para cualquier otro rol (fix 2026-07-19, ver [SECURITY.md](SECURITY.md#endurecimiento-de-seguridad-2026-07-19)) |
| GET | `/api/finance/ledger-book` | Taller | Libro Mayor único en Excel real (`.xlsx`, vía `exceljs`) — reemplaza los 6 reportes CSV anteriores (`invoices`/`purchases`/`receivable`/`payable`/`ledger`/`inventory-valuation`, hoy eliminados). Orden: Fecha, Número de Documento (comprobante FV-/CP-/CE-/ASI-/CC- por evento, consecutivo sin ceros a la izquierda — `ASI-1`, `ASI-2`, reclamo atómico vía RPC de Postgres desde 2026-07-29 — ver [FINANCIAL_ENGINE.md](../Reglas%20de%20Negocio%20y%20Finanzas/FINANCIAL_ENGINE.md#número-de-documento-2026-07-28)), Código PUC, Cuenta, Detalle, Identificación, Razón Social — resuelta en cadena de 7 prioridades, ver [FINANCIAL_ENGINE.md](../Reglas%20de%20Negocio%20y%20Finanzas/FINANCIAL_ENGINE.md#resolución-de-tercero-en-el-libro-mayor-2026-07-28) —, Tipo de Documento, Tipo de Contribuyente, Dirección, Ciudad, Correo, Teléfono (resueltos para clientes, proveedores, empleados y — desde 2026-07-29 — cualquier asiento sin tercero externo explícito, que cae al fallback de la identidad del propio taller; ver [FINANCIAL_ENGINE.md](../Reglas%20de%20Negocio%20y%20Finanzas/FINANCIAL_ENGINE.md#tercero-completo--tipo-de-documentocontribuyente-dirección-ciudad-correo-teléfono-2026-07-29)), Débito, Crédito, y el UUID interno al final como "ID Interno EFISCO". Acepta `from`/`to` opcionales |
| POST | `/api/finance/manual-movement` | Taller (owner) | Movimiento manual (`type`: `NON_OP_INC` ingreso no operacional, `VAT_REFUND` devolución de IVA) — 403 para cualquier otro rol (fix 2026-07-19, ver [SECURITY.md](SECURITY.md#endurecimiento-de-seguridad-2026-07-19)). Sin `work_order_id`/`related_purchase_id`/`mechanic_id`: cae al fallback de tercero del propio taller en el Libro Mayor (2026-07-29). Inserta 2 líneas de partida doble desde 2026-07-30 (antes solo una, sin contrapartida — ver [FINANCIAL_ENGINE.md](../Reglas%20de%20Negocio%20y%20Finanzas/FINANCIAL_ENGINE.md#validación-de-partida-doble-en-el-insert-2026-07-30-pedido-explícito-del-usuario)): la línea del `type` pedido + `CASH_RECEIPT`/`CASH_PAYMENT` según el `impact` |
| POST | `/api/finance/fixed-costs/pay` | Taller (owner) | Pago real de un costo operacional fijo (2026-07-30, pedido explícito) — `{ category: 'arriendo'\|'servicios'\|'otros', amount, concept? }`. DEBIT al gasto (512005/513505/519595 según categoría) + CREDIT a Bancos (`CASH_PAYMENT`) — ver [FINANCIAL_ENGINE.md](../Reglas%20de%20Negocio%20y%20Finanzas/FINANCIAL_ENGINE.md#costos-fijos-operacionales--arriendoservicios-públicosotros-gastos-2026-07-30-pedido-explícito-del-usuario) |
| GET | `/api/finance/opening-balance` | Taller (owner) | Saldo inicial migrado (cuaderno/Excel previo) — `cashAmount`/`bankAmount` por separado desde 2026-09-08, además del `currentAmount` combinado |
| POST | `/api/finance/opening-balance` | Taller (owner) | Fijar/corregir saldo inicial — inserta 2 líneas de partida doble (`OPENING_BALANCE` + contrapartida `OPENING_BALANCE_EQUITY`, ver [FINANCIAL_ENGINE.md](../Reglas%20de%20Negocio%20y%20Finanzas/FINANCIAL_ENGINE.md#saldo-inicial--partida-doble-2026-07-28)). `channel:'efectivo'\|'banco'` opcional (2026-09-08, default `'banco'`) — cada canal lleva su propia corrección independiente, ver [FINANCIAL_ENGINE.md — Saldo Inicial por canal](../Reglas%20de%20Negocio%20y%20Finanzas/FINANCIAL_ENGINE.md#saldo-inicial-por-canal-2026-09-08-mismo-pedido) |
| GET | `/api/finance/folios-by-service` | Taller | Folios de facturación electrónica consumidos por servicio |
| POST | `/api/finance/subscription/checkout` | Taller | Checkout Pro de Mercado Pago para pagar la suscripción a EFISCO. Rechaza con 400 si el taller sigue dentro de su mes gratis de bienvenida (`free_month_until` en el futuro, 2026-08-31) — defensa en profundidad, el frontend ya oculta esta tarjeta por completo mientras dure, ver [BILLING.md](../Reglas%20de%20Negocio%20y%20Finanzas/BILLING.md#mes-gratis-de-bienvenida-2026-08-31-pedido-explícito-del-usuario) |
| GET | `/api/finance/breakeven-panel` | Taller | Datos del Panel de Equilibrio |
| GET | `/api/finance/cashflow` | Taller | Flujo de caja detallado por fecha — `summary.cash_accounts` (Efectivo/Bancos, 2026-09-08) calculado sobre todo el histórico previo a `from` + el rango elegido, mismo criterio que `bank_balance`/`net_balance` |
| GET | `/api/finance/referral-discounts` | Taller | Descuentos ganados por referidos |
| GET | `/api/finance/referral-payout/summary` | Taller | Estado de comisiones de referidos pendientes |
| GET | `/api/finance/monthly-books` | Taller | Libros mensuales archivados del Libro Auxiliar — hace el **cierre perezoso** de todo mes terminado sin libro (upsert idempotente; el mes en curso nunca se cierra) |
| GET | `/api/finance/monthly-books/:period` | Taller | Libro completo de un mes (`YYYY-MM`) con su snapshot congelado; 404 si ese mes no está cerrado |
| GET | `/api/finance/monthly-operating-summary/:period` | Taller | Ingresos/Costos/Utilidad Neta de un mes cualquiera (cerrado o en curso) en base de DEVENGO, calculado al vuelo desde `cash_flow_ledger` — nunca desde `monthly_ledger_books` (base de caja, no comparable). Usado por el gráfico de comparación mensual del Dashboard, ver [FINANCIAL_ENGINE.md](../Reglas%20de%20Negocio%20y%20Finanzas/FINANCIAL_ENGINE.md#caja-real-vs-informativodevengo--por-qué-existe-la-distinción). 400 si el período no tiene formato `YYYY-MM` |

---

## Financial Reports (`/api/financial-reports`)

Reportes financieros (PDF trimestral/anual) para la página pública de Relación con Inversionistas — ver [BUSINESS_RULES.md](../Reglas%20de%20Negocio%20y%20Finanzas/BUSINESS_RULES.md#rutas-del-frontend) (`/reportes-financieros`) y [BILLING.md](../Reglas%20de%20Negocio%20y%20Finanzas/BILLING.md#reportes-financieros-adminreportes-financieros) (panel de carga).

| Método | Ruta | Auth | Descripción |
|:---|:---|:---:|:---|
| GET | `/api/financial-reports` | Pública | Lista reportes publicados, más recientes primero. `?period_type=trimestral\|anual` filtra por tipo |
| POST | `/api/financial-reports` | Admin | Sube el PDF (multipart, campo `file`, ≤15MB, solo `application/pdf`) a un bucket público (`efisco-financial-reports`) e inserta la fila (`period_type`, `period_label`, `title`) en un solo paso |
| DELETE | `/api/financial-reports/:id` | Admin | Borra el archivo del bucket y la fila |

---

## Admin (`/api/admin`)

Autenticación separada (`requireAdmin`, `ADMIN_JWT_SECRET`) — ver [SECURITY.md](SECURITY.md#autenticación-admin-panel-efisco).

| Método | Ruta | Auth | Descripción |
|:---|:---|:---:|:---|
| POST | `/api/admin/bootstrap` | Pública | Crea primer admin (solo si tabla vacía). Setea la cookie httpOnly `efisco_admin_token` — el body ya no incluye el token |
| POST | `/api/admin/login` | Pública | Login admin con email/password — rate-limit de 10 intentos fallidos por `(IP, email)` cada 15 min (429) + alerta de Telegram al equipo justo en el intento que cruza el umbral (agregado 2026-07-26, ver [SECURITY.md](SECURITY.md#endurecimiento-de-seguridad-2026-07-26) y [MONITORING.md](../MONITORING.md)). Setea la cookie httpOnly `efisco_admin_token` — el body ya no incluye el token |
| POST | `/api/admin/logout` | Pública | Limpia la cookie `efisco_admin_token` |
| GET | `/api/admin/me` | Admin | Restaura la sesión admin al recargar la página (mismo motivo que `GET /api/auth/me`) |
| GET | `/api/admin/stats` | Admin | KPIs del dashboard |
| GET | `/api/admin/workshops` | Admin | Lista talleres (con emails de Supabase Auth). `?search=` filtra por nombre/email/NIT/código referido. `?view=` selecciona una de 4 vistas mutuamente excluyentes: sin el param o `?view=active` (default) excluye suspendidos y de prueba (`is_active !== false` y `is_test_workshop !== true`, vista normal: En proceso + Activos); `?view=suspended` (2026-07-29, pestaña "Suspendidos") devuelve SOLO los suspendidos (`is_active === false`); `?view=test` (2026-07-30, pestaña "Prueba") devuelve SOLO los talleres marcados como de prueba (`is_test_workshop === true`); `?view=free_month` (2026-08-31, pestaña "Mes Gratis") devuelve SOLO los talleres con `free_month_until` vigente, ordenados por el que termina más pronto primero |
| GET | `/api/admin/workshops/:id` | Admin | Detalle de taller — incluye `contract_acceptance` (constancia de firma del Contrato B2B: versión, hash, IP, fecha; `null` si no ha firmado), ver [SECURITY.md](SECURITY.md) |
| GET | `/api/admin/workshops/:id/contract-pdf` | Admin | Descarga del PDF firmado del Contrato B2B del taller — signed URL de 1h, self-heal si `pdf_storage_path` no existe aún (regenera con `generateAndStoreContractPdf`), 404 si el taller no ha firmado. Mismo servicio que usa el dueño desde `/config` (`GET /api/auth/contract-pdf`), pero sin requerir sesión del taller (rev. 46) |
| POST | `/api/admin/workshops` | Admin | Crear taller nuevo |
| PATCH | `/api/admin/workshops/:id/toggle` | Admin | Suspender / reactivar — bloquea/desbloquea el acceso real del taller (`is_active`). Distinto de "Eliminar" (DELETE, abajo), que sí borra datos pero nunca bloquea acceso por sí solo |
| PATCH | `/api/admin/workshops/:id/toggle-test` | Admin | Marcar / desmarcar taller de prueba (`is_test_workshop`, 2026-07-30, pedido explícito) — excluye al taller de los KPIs de plataforma (`GET /api/admin/stats`) sin borrarlo ni afectar su operación. Exige `{ password }` en el body — mismo patrón que `DELETE /api/admin/workshops/:id` (400 si falta, 401 si no coincide) |
| PATCH | `/api/admin/workshops/:id/grant-free-month` | Admin | Otorga/extiende 30 días de mes gratis de bienvenida (`free_month_until`, 2026-08-31, pedido explícito) — sin password, no destructivo. Rechaza con 400 si el taller YA tiene un mes gratis vigente (no se acumula, pedido explícito del usuario): solo se puede volver a otorgar una vez que el período anterior termina. Ver [BILLING.md — Mes Gratis de Bienvenida](../Reglas%20de%20Negocio%20y%20Finanzas/BILLING.md#mes-gratis-de-bienvenida-2026-08-31-pedido-explícito-del-usuario) |
| DELETE | `/api/admin/workshops/:id` | Admin | Elimina permanentemente un taller **no activado** (400 si `admin_activated_at` ya tiene valor) — borra en cascada + notifica por correo al dueño (Resend). Exige `{ password }` en el body — la contraseña del propio admin de EFISCO que hace la request (`req.admin.adminId` contra `efisco_admins`, no la del dueño del taller), 400 si falta, 401 si no coincide (agregado 2026-07-29; se probó primero convertir esto en un soft-delete disponible para cualquier taller y se revirtió — las solicitudes de acceso sin activar son spam candidato, mejor borrarlas de verdad que dejarlas acumularse). Ver [SECURITY.md](SECURITY.md) |
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
| GET | `/api/admin/telegram-alerts/link-qr` | Admin | Genera el QR de vinculación al bot interno de alertas (`t.me/<bot>?start=<token>`, expira en 10 min) — ver [MONITORING.md](../MONITORING.md) |
| GET | `/api/admin/telegram-alerts/subscribers` | Admin | Lista quién recibe las alertas (`telegram_username`, `telegram_first_name`, `linked_at`) |
| DELETE | `/api/admin/telegram-alerts/subscribers/:id` | Admin | Quitar un suscriptor |
| POST | `/api/admin/telegram-alerts/test` | Admin | Manda una alerta real (no simulada) de prueba a todos los suscriptores actuales — sin `dedupeKey`, un envío pedido a mano siempre debe salir |

Detalle funcional de cada módulo admin en [BILLING.md — Panel de Administración EFISCO](../Reglas%20de%20Negocio%20y%20Finanzas/BILLING.md#panel-de-administración-efisco).

## Alertas de Telegram — interno (`/api/telegram-alerts`)

Bot separado del bot de clientes de arriba — ver [MONITORING.md](../MONITORING.md) para el catálogo completo de las 20 alertas y la política de privacidad.

| Método | Ruta | Auth | Descripción |
|:---|:---|:---:|:---|
| POST | `/api/telegram-alerts/webhook` | Pública (webhook) | Recibe los updates del bot interno — solo procesa `/start <token>` (vinculación de un admin/dueño). **Fail-closed** vía `X-Telegram-Bot-Api-Secret-Token` (`TELEGRAM_ALERTS_WEBHOOK_SECRET`, distinto del secret del bot de clientes). Siempre responde 200 salvo secret inválido |
