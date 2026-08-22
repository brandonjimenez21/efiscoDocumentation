# Monitoreo y Alertas Internas (Bot de Telegram EFISCO)

> Ver también: [README](README.md) · [ARCHITECTURE](Arquitectura%20y%20Sistema%20Core/ARCHITECTURE.md) · [SECURITY](Arquitectura%20y%20Sistema%20Core/SECURITY.md) · [API](Arquitectura%20y%20Sistema%20Core/API.md) · [BUSINESS_RULES](Reglas%20de%20Negocio%20y%20Finanzas/BUSINESS_RULES.md) · [OPERATIONS](Reglas%20de%20Negocio%20y%20Finanzas/OPERATIONS.md) · [INVENTORY](Reglas%20de%20Negocio%20y%20Finanzas/INVENTORY.md) · [FINANCE](Reglas%20de%20Negocio%20y%20Finanzas/FINANCE.md) · [FINANCIAL_ENGINE](Reglas%20de%20Negocio%20y%20Finanzas/FINANCIAL_ENGINE.md) · [BILLING](Reglas%20de%20Negocio%20y%20Finanzas/BILLING.md) · [GROWTH_ACQUISITION](Estrategia%20Comercial%20y%20Ventas/GROWTH_ACQUISITION.md) · [PRICING_SALES](Estrategia%20Comercial%20y%20Ventas/PRICING_SALES.md) · [TESTING](TESTING.md)

Bot de Telegram **separado por completo** del bot de clientes (`telegram.service.js`, ver [OPERATIONS.md — Recepción](Reglas%20de%20Negocio%20y%20Finanzas/OPERATIONS.md#recepción)): este es de uso **interno del equipo EFISCO** (dueño + admins), para enterarse de fallos de la plataforma y eventos operativos de alto valor sin depender de revisar logs de Render a mano. Se mantiene aparte a propósito — un incidente en el bot de clientes (rate limit, token revocado) no debe afectar la capacidad de recibir alertas, y viceversa.

---

## Regla de privacidad — no negociable

Instrucción explícita del usuario (2026-07-26): **ninguna alerta puede contener datos reales/sensibles de un taller o de sus clientes** (nombres, cédulas, teléfonos embebidos en texto de error de terceros, montos financieros del cliente, etc.). Como mucho, una alerta puede **identificar de qué taller viene el fallo** vía su nombre, correo y/o teléfono — nunca datos de sus clientes.

Dos mecanismos separados según el origen del error:

- **Errores internos** (Postgres, lógica propia): se pasan por `sanitizeErrorMessage()` antes de entrar al texto del mensaje.
- **Errores de terceros** (Dataico, Meta/WhatsApp): su texto **ni siquiera se sanea** — se **omite por completo**, porque sus mensajes de validación citan el dato exacto que rechazaron (nombre del cliente, NIT, teléfono) en formatos impredecibles que un regex no puede cubrir con confianza. En su lugar, la alerta solo dice qué taller/orden se vio afectado y remite a revisar los logs de Render.

```js
// backend/services/telegramAlerts.service.js
export const sanitizeErrorMessage = (message, maxLen = 300) => {
  let msg = String(message ?? 'Error desconocido');
  msg = msg.split(/\bDETAIL:/i)[0].trim();               // Postgres cita el valor real en el DETAIL de un error de restricción
  msg = msg.replace(/\([\w."]+\)\s*=\s*\([^)]*\)/g, '(...)'); // patrón residual "(columna)=(valor)" fuera del DETAIL
  return msg.slice(0, maxLen) || 'Error desconocido';
};
```

Ejemplo real de por qué hace falta: `duplicate key value violates unique constraint "clients_cedula_key"\nDETAIL: Key (cedula)=(1020304050) already exists.` → sin `sanitizeErrorMessage`, la cédula real de un cliente llegaría al chat de Telegram del equipo. Cubierto por `tests/telegramAlerts.sanitizeErrorMessage.test.js`.

---

## Infraestructura

| Pieza | Detalle |
|:---|:---|
| `backend/services/telegramAlerts.service.js` | `broadcast(text, {dedupeKey})` — manda a TODOS los `chat_id` suscritos en paralelo, nunca lanza (un fallo de Telegram no debe convertirse en un segundo error dentro del propio manejo de errores que disparó la alerta). `sendMessage(chatId, text)`, `registerWebhook()`, `sanitizeErrorMessage()` |
| Antirebote (`DEDUPE_WINDOW_MS = 5 min`) | Mapa en memoria `dedupeKey → timestamp` — el mismo error puede repetirse cientos de veces por minuto (ej. un endpoint roto reintentado agresivo); sin esto, el bot de Telegram terminaría limitado por su propio rate-limit y perdiendo justo las alertas que importan. Suficiente para el deploy de proceso único actual (Render); con varias instancias habría que moverlo a la base |
| `admin_alert_subscribers` | `id, chat_id (UNIQUE), telegram_username, telegram_first_name, linked_at` — quién recibe las alertas |
| `admin_alert_link_requests` | `id, token, used, used_at, expires_at, created_at` — códigos de un solo uso para el QR de vinculación (10 min de validez) |
| `backend/controllers/telegramAlerts.controller.js` | `generateLinkQr` (admin), `listSubscribers`/`removeSubscriber` (admin), `sendTest` (admin, sin `dedupeKey` a propósito — una prueba pedida a mano siempre debe salir), `webhook` (público) |
| `frontend/src/pages/admin/AdminAlerts.jsx` (`/admin/alertas`) | Panel del admin: QR de vinculación, lista de suscriptores con auto-refresco (polling), botón "Enviar alerta de prueba" |
| `backend/scripts/test-telegram-alerts.mjs` | Script standalone de verificación end-to-end (fuera de Jest) — confirma que el bot, el token y la vinculación funcionan de punta a punta contra Telegram real, sin esperar a que ocurra un error de verdad. Corrido y confirmado funcionando en vivo (2 suscriptores reales) |

### Flujo de vinculación (QR)

1. El admin abre `/admin/alertas` → `GET /api/admin/telegram-alerts/link-qr` genera un token de un solo uso (10 min) y el QR (`t.me/<bot>?start=<token>`).
2. Al escanear y darle "Iniciar" en Telegram, `POST /api/telegram-alerts/webhook` (público, fail-closed vía `X-Telegram-Bot-Api-Secret-Token` == `TELEGRAM_ALERTS_WEBHOOK_SECRET`) procesa `/start <token>`, valida que no esté usado/expirado, hace `upsert` del `chat_id` en `admin_alert_subscribers` (`onConflict: chat_id`) y marca el token usado.
3. No hay límite de cuántas personas pueden vincularse — cada escaneo genera su propio token y se suma a la lista de suscriptores. Siempre responde 200 (salvo secret inválido), mismo criterio que el bot de clientes ante el reintento agresivo de Telegram.

### Variables de entorno

Ver plantilla completa en [ARCHITECTURE.md — Variables de Entorno](Arquitectura%20y%20Sistema%20Core/ARCHITECTURE.md#variables-de-entorno). Resumen: `TELEGRAM_ALERTS_BOT_TOKEN` (bot de @BotFather, distinto del bot de clientes), `TELEGRAM_ALERTS_BOT_USERNAME` (para armar el deep-link del QR), `TELEGRAM_ALERTS_WEBHOOK_SECRET` (fail-closed del webhook). Sin estas 3 configuradas, `isConfigured()` da `false` y `broadcast`/`registerWebhook` no hacen nada (no truena, solo no avisa).

---

## Catálogo de alertas

20 puntos de alerta: 5 de infraestructura base (`server.js`, siempre activos si el bot está configurado) + 15 de eventos de negocio (agregados a partir de 2026-07-26 e incrementalmente después, cada uno con su propia justificación de por qué merece un aviso en vivo en vez de esperar a revisar logs).

### Infraestructura (`backend/server.js`)

| Alerta | Disparador | `dedupeKey` | Notas |
|:---|:---|:---|:---|
| 🔥 Error en `{método} {ruta}` | Middleware global de errores — la mayoría de controllers ya atrapan los suyos, así que esto rara vez dispara en flujo normal | `err:{path}:{message}` | `sanitizeErrorMessage(err.message)` |
| ⚠️ Unhandled Rejection | `process.on('unhandledRejection')` | `rejection:{message}` | `sanitizeErrorMessage(msg)` |
| 🆘 Uncaught Exception — el servidor casi se cae | `process.on('uncaughtException')` | **Sin dedupe** (la señal más grave de las tres, siempre avisa) | `sanitizeErrorMessage(err.message)` |
| 🗄️ La base de datos no está respondiendo | `setInterval` cada 5 min — `SELECT id FROM workshop_config LIMIT 1` | `db-health-check` | Detecta que Supabase dejó de responder aunque ningún usuario real haya tocado esa tabla; no reemplaza un monitor externo de uptime (si el proceso entero se cae, esto tampoco puede avisar) |
| 🚀 Backend reiniciado/desplegado en producción | `app.listen` callback, gateado a `NODE_ENV=production` | Sin dedupe | Confirma que un deploy llegó a producción sin revisar el dashboard de Render; también avisa de un reinicio por crash (Render reinicia solo el proceso) |

### Eventos de negocio (desde 2026-07-26)

| Alerta | Archivo:línea | `dedupeKey` | Dato de terceros omitido / saneado |
|:---|:---|:---|:---|
| 💳 Webhook de Bold falló al procesarse | `billing.controller.js:boldWebhook` catch | `bold-webhook:{orderId}` | `sanitizeErrorMessage` (error interno propio, ej. Postgres) |
| 💳 Webhook de Addi falló al procesarse | `billing.controller.js:addiWebhook` catch | `addi-webhook:{orderId}` | Ídem |
| 💳 Webhook de Mercado Pago falló al procesarse | `billing.controller.js:mercadopagoWebhook` catch externo | `mercadopago-webhook:{message}` | Ídem |
| 🧾 Factura de suscripción no se pudo emitir a Dataico | `billing.controller.js:mercadopagoWebhook` catch interno (falla `emitSubscriptionInvoiceForWorkshop`) | `subscription-invoice:{workshopId}` | Texto de Dataico **omitido por completo** — solo taller + "revisa los logs de Render" |
| 🧾 Factura electrónica no se pudo emitir a Dataico | `billing.controller.js:settleOrder` (orden ya liquidada, cliente ya pagó) | `dataico-invoice:{work_order_id}` | Texto de Dataico **omitido por completo** — sus mensajes de validación citan el dato real que falló (ej. nombre del cliente vs. NIT registrado) |
| 📵 Fallo enviando código de verificación a un cliente | `clients.controller.js:requestScoreOTP` (canal vinculado, la API falla) | `otp-send:{channel}:{code}` | Texto de la API de WhatsApp/Telegram **omitido** (puede citar el teléfono del cliente) — solo `otpResult.code` (código técnico, ej. 131047) |
| 📵 Fallo enviando el PDF de puntaje a un cliente | `clients.controller.js:verifyScoreOTP` (`sendScorePdf(...).catch`) | `score-pdf-send:{cedula}` | Ídem, sin texto crudo |
| 📵 Fallo enviando solicitud de datos de facturación a un cliente | `quick_intakes.controller.js:sendWhatsAppRequest` (canal disponible, la API falla) | `intake-request:{id}` | Texto de la API **omitido** — solo taller + código de error si vino uno |
| 📵 Error inesperado enviando solicitud de datos de facturación | `quick_intakes.controller.js:sendWhatsAppRequest` catch externo | `intake-request-crash:{message}` | `sanitizeErrorMessage` (error interno) |
| 🔒 Se bloqueó el PIN de un taller por intentos fallidos | `workshop.controller.js:verifyLockPin` — `recordPinFailure` devuelve `true` solo en el intento EXACTO que cruza el umbral (5) | Sin dedupe (ya construido para disparar una sola vez por episodio) | Sin texto de error — solo taller + "posible intento de fuerza bruta" |
| 💰 No se pudo registrar el gasto de suscripción en la contabilidad del taller | `subscriptionBilling.service.js:emitSubscriptionInvoiceForWorkshop` (falla el espejo del gasto) | `subscription-mirror:{workshopId}` | `sanitizeErrorMessage` (error interno) |
| 💰 No se pudo encolar una comisión de referido | `subscriptionBilling.service.js:queueReferralRecurringCommission` (falla el insert) | `referral-commission:{workshopId}` | `sanitizeErrorMessage`; consulta de identificación del taller hecha aparte (variables del `try` pueden no estar resueltas en el punto de falla) |
| ✨ Nuevo taller registrado | `auth.controller.js:register` (autorregistro público, éxito) | Sin dedupe | **Incluye a propósito** nombre, NIT y correo del taller — es el propio taller identificándose, no un dato de un cliente suyo; el usuario confirmó que esto entra dentro de lo permitido |
| ✨ Nuevo taller registrado (creado desde el panel admin) | `admin.controller.js:createWorkshop` (éxito) | Sin dedupe | Ídem — nombre + correo del taller |
| 🔐 Se bloqueó el login del panel admin EFISCO por intentos fallidos | `admin.controller.js:login` — mismo patrón que el PIN, `recordLoginFailure` devuelve `true` solo en el intento que cruza el umbral (10) | Sin dedupe (ya construido para disparar una sola vez por episodio) | Incluye el correo **intentado** (puede ser real o inventado por el atacante) + IP — nunca contraseñas. Ver [SECURITY.md — Endurecimiento 2026-07-26](Arquitectura%20y%20Sistema%20Core/SECURITY.md#endurecimiento-de-seguridad-2026-07-26) |

**Patrón "fire-and-forget contra un `workshop_id`/`order_id`"**: varios call sites (Bold/Addi, referral-commission, subscription-invoice) resuelven el nombre/teléfono del taller con una consulta `.then(...)` aparte, sin `await`, para no demorar la respuesta HTTP al usuario final por un problema en el envío de la alerta — el broadcast puede resolver después de que la respuesta ya salió.

---

## Cobertura de tests

Todos verificados con datos de error **realistas** (formato real de Postgres con `DETAIL`, mensajes de validación de Dataico/Meta citando nombres/teléfonos ficticios) para probar tanto que la alerta dispara como que el saneo funciona de verdad, no una versión de juguete ya limpia. Detalle de cada suite en [TESTING.md](TESTING.md):

- `telegramAlerts.sanitizeErrorMessage.test.js`
- `billing.paymentWebhookAlerts.test.js` (Bold, Addi, Mercado Pago)
- `billing.settleOrderDataicoAlert.test.js`
- `clients.messagingAlerts.test.js` (OTP + PDF de puntaje)
- `quickIntakes.messagingAlert.test.js`
- `subscriptionBilling.internalFailureAlerts.test.js` (espejo de gasto + comisión de referido)
- `workshop.pinLockoutAlert.test.js`
- `workshopRegistration.alerts.test.js` (self-service + panel admin)
- `admin.loginBruteForce.test.js`
