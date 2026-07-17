# Seguridad

> Ver también: [README](README.md) · [ARCHITECTURE](ARCHITECTURE.md) · [API](API.md)

La defensa principal contra acceso cruzado entre talleres es **a nivel de aplicación**: cada endpoint valida `workshop_id`/rol contra la sesión del usuario autenticado antes de tocar la base. Row Level Security (RLS) de Postgres es una **capa adicional de defensa en profundidad**, no un reemplazo de esa validación — su propósito es que, aunque una clave de Supabase se filtrara, la API REST automática de PostgREST (siempre activa en paralelo al backend Express, sin que la app la use directamente) no exponga datos por sí sola.

---

## Decisiones Técnicas de Seguridad

- **Multi-tenant a nivel de aplicación** — cada controlador valida `workshop_id`/rol del usuario autenticado en cada operación, sin bases de datos independientes por taller. RLS de Postgres es defensa en profundidad (ver [RLS y Supabase](#rls-y-supabase-security-advisor)), no reemplaza esta validación.
- **Admin completamente aislado** — JWT separado (`ADMIN_JWT_SECRET`), Zustand store separado (`useAdminStore`), rutas `/admin/*` renderizadas antes del catch-all, localStorage keys distintas (`admin_token`/`admin_user` vs `token`/`user`).
- **Suspensión en dos capas** — `auth.controller.js` bloquea el login y `auth.middleware.js` invalida tokens ya emitidos, evitando que un token pre-suspensión siga funcionando.
- **El dueño autentica con JWT propio, no con el `access_token` nativo de Supabase** — ese token expira en 1h por defecto sin refresh, lo que forzaba un cierre de sesión cada hora sin importar actividad (los empleados ya usaban JWT propio de 12h). El dueño ahora recibe el mismo tipo de token (`type:'owner'`, 12h, `JWT_SECRET`), verificado en la misma rama de `auth.middleware.js` que los empleados. Con esto, el bloqueo por PIN de inactividad (30 min) vuelve a ser lo único que interrumpe una sesión activa en el uso normal.
- **Contrato B2B en dos capas** — igual que la suspensión: `ContractGate.jsx` bloquea visualmente el frontend y `auth.middleware.js` rechaza con 403 (`contract_not_signed`) cualquier request de un dueño sin firmar, salvo el propio endpoint de firma (`POST /api/auth/sign-contract`). Firmar = reingresar la contraseña (Ley 527/1999), verificada contra Supabase Auth vía `signInWithPassword` — no se guarda ni deriva nada de la contraseña real. El texto del contrato (`legalTexts.js`/`ContractText.jsx`) incluye secciones dedicadas de Términos y Condiciones y Política de Privacidad, con mejoras de legibilidad (tipografía, scroll suave) para que el scroll-lock obligatorio sea una lectura real y no un trámite.
- **Autorización del taller en dos capas** (staff no-dueño) — mismo patrón que el Contrato B2B pero sin firma con contraseña: `StaffTermsGate.jsx` bloquea visualmente y `auth.middleware.js` rechaza con 403 (`staff_terms_not_accepted`) cualquier request de un empleado (mecánico, recepcionista, contador...) que no haya aceptado, salvo `/api/auth/accept-staff-terms`. Prueba legal append-only en `staff_legal_acceptances` (hora + IP). El dueño está exento — ya tiene el Contrato B2B.
- **Gotcha real de los gates "scroll para habilitar"** — si el texto cabe completo sin necesitar scroll (pantallas grandes), el evento `onScroll` nunca se dispara y el botón queda deshabilitado para siempre. `ContractGate.jsx` y `StaffTermsGate.jsx` verifican `scrollHeight <= clientHeight` en un `useEffect` al montar (y en resize) para habilitar el botón automáticamente en ese caso.
- **"Olvidé mi contraseña" gateado por activación previa del admin** — `register()` permite autorregistrarse como "taller" sin ninguna verificación (genera password aleatoria si no se manda una), así que un self-service de recuperación sin restricciones dejaría entrar a cualquiera sin que EFISCO vetee el negocio. `workshop_config.admin_activated_at` se marca una sola vez — al crear el taller directamente desde `/admin/talleres` (`createWorkshop`), o la primera vez que un admin usa "Enviar email de acceso" (`sendPasswordResetEmail`) — y nunca se toca al suspender/reactivar (`is_active` es un concepto aparte, deliberadamente no reusado: un taller suspendido pero ya vetted antes sigue pudiendo recuperar su contraseña, aunque no podrá iniciar sesión hasta ser reactivado). `POST /api/auth/request-password-reset` (público) solo dispara `resetPasswordForEmail` de Supabase si esa columna ya tiene valor, y devuelve **siempre el mismo mensaje genérico** exista o no el correo, para no filtrar por enumeración qué talleres están registrados o activados.
- **Por qué el borrado de talleres es seguro (`DELETE /api/admin/workshops/:id`)** — el gate anterior tiene un corolario útil: como `register()` nunca deja que el propio usuario elija su contraseña, **ningún taller con `admin_activated_at` vacío pudo haber iniciado sesión jamás** (su dueño no conoce ninguna contraseña válida hasta que un admin se la manda). Eso garantiza que ese subconjunto de talleres está siempre vacío — cero órdenes, cero facturas DIAN, cero movimientos de caja — así que borrarlos por completo no tiene ninguna implicación de retención fiscal/legal. Por eso el endpoint rechaza con 400 cualquier intento de borrar un taller que ya tenga `admin_activated_at`, sin excepción ni flag para forzarlo: para un taller ya activado (con posible actividad real), el borrado permanente es una decisión distinta que este botón deliberadamente no cubre — ahí sigue aplicando "Suspender".

---

## Autenticación Admin (Panel EFISCO)

Panel interno en `/admin` con autenticación completamente separada de los talleres:

- JWT firmado con `ADMIN_JWT_SECRET` (distinto al JWT de Supabase de los talleres)
- Hash de contraseñas con `crypto.scrypt` + sal de 16 bytes (sin dependencias externas)
- Bootstrap: `POST /api/admin/bootstrap` — solo funciona si `efisco_admins` está vacío
- Token almacenado en `admin_token` (localStorage), distinto de `token` de talleres

---

## Llave de Acceso a Cuenta del Cliente (Soporte)

El dueño genera un código de un solo uso (`POST /api/workshop/access-key`, válido 30 min) desde `/soporte` y se lo pasa a soporte; el admin lo canjea desde el detalle del taller (`POST /api/admin/workshops/:id/access-keys/redeem`). El canje emite un **token de sesión de soporte** firmado con `JWT_SECRET` (nunca `ADMIN_JWT_SECRET` — los dos sistemas de auth no se mezclan), válido 45 min, con los mismos privilegios que el dueño. Se abre en una pestaña nueva vía `/support-session?token=...`, que hace bootstrap de la sesión en `useAuthStore`. Todo intento de canje (exitoso o no) queda en `workshop_access_audit_log` (append-only).

---

## Pruebas Legales Append-Only

- **`client_legal_acceptances`** — aceptación de Términos y Condiciones / Política de Datos (Ley 1581 de 2012) en el Registro Seguro EFISCO público, con scroll-lock obligatorio antes de habilitar el botón. Se registra versión, IP y momento exacto (`created_at`) para una eventual auditoría de la SIC.
- **`staff_legal_acceptances`** — aceptación de la autorización del taller por parte de empleados no-dueño (hora + IP).
- **`workshop_access_audit_log`** — todo intento de canje de llave de acceso a cuenta del cliente (exitoso o no).

---

## RLS y Supabase (Security Advisor)

**Estado actual (auditado con el Security Advisor de Supabase, 2026-07-10):**

- **RLS activado (deny-by-default, sin políticas) en las 28 tablas de `public` que tenía el linter marcadas** — el backend accede a todas ellas exclusivamente con la clave `service_role`, que ignora RLS siempre, así que esto no cambió ningún comportamiento de la app; solo cierra el acceso directo vía PostgREST si la clave `anon` llegara a filtrarse.
- Los flujos que antes consultaban tablas con la clave `anon` (login, alta/gestión de mecánicos, Bahía, recepción rápida) fueron migrados a `service_role` — mismo patrón que ya usaba el resto del backend. La clave `anon` ahora solo se usa donde corresponde: las llamadas reales de Supabase Auth (`signUp`/`signInWithPassword`/`getUser`), que no dependen de RLS sobre `public.*` en absoluto. Verificado extremo a extremo contra la base real antes y después del cambio (login + endpoints de mecánicos/bahía/recepción con datos reales).
- Corregido: una función de base de datos (`update_inventory_stock`) sin `search_path` fijo, lo que la hacía depender del `search_path` de sesión de quien la invocara (`function_search_path_mutable` del linter).
- De paso quedó resuelto el hallazgo de columna sensible expuesta (`workshop_config`) — al activarle RLS junto con el resto.
- Pendiente por limitación de plan: "Leaked Password Protection" de Supabase Auth (verificación contra HaveIBeenPwned al crear contraseña) requiere plan Pro; el proyecto está en plan Free. Los dueños de taller inician sesión con Supabase Auth real (`signInWithPassword`); los empleados usan un sistema JWT + `scrypt` propio, separado.
- Las migraciones SQL de esta fase se aplicaron directo contra la base real vía la API de administración de Supabase y luego se borraron del repo — `backend/backup/efiscodb.sql`, mantenido manualmente por el usuario, es la referencia de qué tiene la base real (confirmado: las 28 tablas con RLS activo sí aparecen ahí).

---

## Webhooks de Pago

`boldWebhook`/`addiWebhook`/`mercadopagoWebhook` (`backend/controllers/billing.controller.js`) marcan una orden/suscripción como pagada al recibir la notificación. Sin `BOLD_WEBHOOK_TOKEN`/`ADDI_WEBHOOK_TOKEN` configurados (ver [ARCHITECTURE.md](ARCHITECTURE.md#variables-de-entorno)), no verifican que la notificación venga realmente de Bold/Addi — configúralos antes de manejar pagos reales en producción. `mercadopago.service.js` sí es seguro por diseño: el webhook nunca confía en el body de la notificación, siempre confirma el estado del pago consultando la API real de Mercado Pago (`fetchPayment`).
