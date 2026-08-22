# Reglas de Negocio y Módulos del Sistema

> Ver también: [README](../README.md) · [ARCHITECTURE](../Arquitectura%20y%20Sistema%20Core/ARCHITECTURE.md) · [SECURITY](../Arquitectura%20y%20Sistema%20Core/SECURITY.md) · [API](../Arquitectura%20y%20Sistema%20Core/API.md) · [OPERATIONS](OPERATIONS.md) · [INVENTORY](INVENTORY.md) · [FINANCE](FINANCE.md) · [FINANCIAL_ENGINE](FINANCIAL_ENGINE.md) · [BILLING](BILLING.md) · [GROWTH_ACQUISITION](../Estrategia%20Comercial%20y%20Ventas/GROWTH_ACQUISITION.md) · [PRICING_SALES](../Estrategia%20Comercial%20y%20Ventas/PRICING_SALES.md) · [MONITORING](../MONITORING.md) · [TESTING](../TESTING.md)

> **Nota (2026-07-31)**: este archivo se dividió en 4 — las reglas de negocio de cada módulo operativo ahora viven en [OPERATIONS.md](OPERATIONS.md) (Recepción, Registro Seguro EFISCO, Bahías/Órdenes de Trabajo, Mi Perfil, Google Calendar), [INVENTORY.md](INVENTORY.md) (Inventario, Proveedores y Egresos), [FINANCE.md](FINANCE.md) (Dashboard, Panel de Equilibrio, Panel de Cobros, Flujo de Caja) y [BILLING.md](BILLING.md) (Referidos, Panel de Administración EFISCO). Este archivo se queda como índice y con lo que no encajaba limpio en un solo módulo: Rutas del frontend, onboarding de un taller nuevo, el Gate de Configuración Inicial, la Demo pública, y la sección completa de Configuración del Taller (`/config`, mezcla Mi Equipo/Catálogo de Servicios operativos con Pasarelas/PUC/Dataico financieros).

---

## Rutas del frontend

| Ruta | Módulo | Acceso |
|:---|:---|:---:|
| `/` | Landing — página de presentación pública (funciones, resumen de demo, CTA registro/login) | Público (sin sesión) |
| `/demo` | Recorrido de 5 videos reales (Recepción → Bahía → Facturación → Flujo de Caja → Equilibrio), no datos de ejemplo — ver [Demo pública](#demo-pública-demo-rev-50) (rev. 50) | Público (sin sesión) |
| `/tarifas` | Tarifas — modelo real de cobro (pay-per-servicio, sin plan mensual fijo) | Público (sin sesión) |
| `/reportes-financieros` | Relación con Inversionistas — reportes trimestrales/anuales de EFISCO SAS (PDF), publicados desde `/admin/reportes-financieros` — ver [BILLING.md](BILLING.md#reportes-financieros-adminreportes-financieros) | Público (sin sesión) |
| `/privacidad` `/terminos` `/politica-cookies` | Textos legales completos (`Legal.jsx`, un solo componente parametrizado por `type`) | Público (sin sesión) |
| `/login` | Autenticación — mismo componente alterna login / registro / recuperar contraseña | Público (sin sesión) |
| `/dashboard` | Dashboard | Todos |
| `/recepcion` | Recepción | Todos |
| `/bahia` | Bahías | Todos |
| `/inventario` | Inventario | Todos |
| `/proveedores` | Proveedores | Todos |
| `/ordenes` | Historial de órdenes completadas | Todos |
| `/referidos` | Referidos | Todos |
| `/soporte` | Soporte | Todos |
| `/config` | Configuración | Owner |
| `/finanzas` | Dashboard Financiero | Owner |
| `/equilibrio` | Panel de Equilibrio | Owner |
| `/cobros` | Panel de Cobros | Owner |
| `/flujo-caja` | Flujo de Caja | Owner |
| `/mi-perfil` | Perfil propio — métricas del mes, compensación, conexión de Google Calendar, cambio de contraseña. Excluye `owner` (su equivalente vive en `/config` → Mi Equipo & Roles) | Admin, Mecánico, Contador |
| `/contador` | Panel del Contador (edición fiscal real) | Contador |
| `/cliente/registro/:workshopId/:intakeId` | Registro Seguro EFISCO (link de WhatsApp o QR desde Recepción, ver [OPERATIONS.md](OPERATIONS.md#recepción)) | Público |
| `/support-session` | Bootstrap de sesión de soporte (llave de acceso canjeada) | Público (requiere token) |
| `/admin` | Panel Admin EFISCO | Admin interno |
| `/admin/talleres` | Gestión de talleres | Admin interno |
| `/admin/cobros` | Facturas de EFISCO **a** los talleres (SaaS fee) | Admin interno |
| `/admin/pagos` | Comisiones de referidos que EFISCO paga **a** los talleres | Admin interno |
| `/admin/referidos` | Árbol de referidos | Admin interno |
| `/admin/contabilidad` | Contabilidad Interna — EFISCO como emisor (Dataico Caso 2) | Admin interno |
| `/admin/reportes-financieros` | Publicar/eliminar reportes financieros (PDF) que alimentan `/reportes-financieros` | Admin interno |
| `/admin/alertas` | Catálogo y estado de las alertas internas del bot de Telegram — ver [MONITORING.md](../MONITORING.md) | Admin interno |

> **Nota — "Cobros" aparece dos veces con significados distintos:** `/cobros` (arriba, vista del dueño) es la cartera de cuotas que el **taller** le cobra a **sus propios clientes**. `/admin/cobros` es un panel completamente distinto: las facturas que **EFISCO** le cobra **al taller** por el uso del software. No confundir tampoco `/admin/cobros` con `/admin/pagos`: el dinero fluye en direcciones opuestas — Cobros es taller→EFISCO, Pagos es EFISCO→taller (comisión por referidos).

---

## Tarifas públicas (`/tarifas`)

Modelo pay-per-uso, sin plan mensual fijo (`frontend/src/pages/Tarifas.jsx`):

- **Tarifa por orden liquidada** (IVA incluido) — no por vehículo, no por mes, no por cantidad de servicios dentro de la orden: $535 COP hasta 119 órdenes/mes, baja a $416 COP desde 120 órdenes/mes en adelante. El cambio de tarifa es automático a mitad de mes si el taller cruza el umbral.
- **Tarifa por repuesto de Venta Directa** (mostrador, sin orden asociada) — $150 COP por unidad; si el repuesto va dentro de una orden, ya está cubierto por la tarifa de arriba, no se cobra dos veces. Detalle técnico del asiento contable en [BILLING.md — Venta Directa](BILLING.md#venta-directa--tarifa-por-repuesto-2026-08-05).
- **Descuento por referidos**: 33% con 1 referido activo, 66% con 2, 100% (gratis) con 3 o más — ver [BILLING.md](BILLING.md) para el árbol de referidos y cómo se cuenta un referido "activo".

---

## Paleta de comandos (Cmd/Ctrl+K)

`CommandPalette.jsx`, montado globalmente en la app autenticada: salta a cualquier sección del menú (misma fuente que `Sidebar.jsx`, `config/navigation.js`, filtrada por rol) o dispara una acción rápida ("Nueva orden" → `/recepcion` para owner/admin/mecánico, "Ver cobros vencidos" → `/cobros` solo owner). Búsqueda por texto sobre el nombre de cada ítem.

---

## Registro de un taller nuevo (`/login`)

Un taller nuevo se **solicita**, no se activa solo — EFISCO verifica manualmente cada taller antes de que opere de verdad (no hay ningún gate técnico nuevo en el backend: `register` sigue creando el `workshop_config` con `is_active` en su default `true`; la verificación es un proceso manual del equipo de EFISCO, no una condición que el código imponga).

- **Copy corregido 2026-07-25**: antes el formulario decía "Registrar Taller" y, al enviarlo, solo aparecía un toast pequeño ("Solicitud enviada correctamente") que desaparecía solo y cambiaba el formulario a login sin más explicación — el dueño no tenía forma de saber que hacía falta una verificación nuestra, y varios interpretaron el silencio como que el registro había fallado. El título/subtítulo del formulario ahora dicen **"Solicitar Acceso"** / "Te contactaremos para verificar tu taller", y los CTAs de la Landing ("Regístrate"/"Regístrate gratis") pasaron a "Solicitar Acceso"/"Solicita tu acceso gratis" para que el mensaje sea consistente desde antes de llegar al formulario.
- **Modal de confirmación** (reemplaza el toast): al enviar la solicitud aparece un modal centrado que hay que cerrar a propósito ("¡Gracias por confiar en EFISCO! ... nos pondremos en contacto contigo muy pronto para verificar tu taller y darte acceso") — mucho más difícil de pasar por alto que un toast que se autodesvanece.

---

## Gate de Configuración Inicial (SetupGate)

Un taller recién activado no puede operar (ni siquiera navegar a Recepción/Bahía/Inventario) hasta completar 3 requisitos mínimos — `frontend/src/components/SetupGate.jsx` bloquea toda la app (salvo `/config`) mientras falte alguno, leyendo el estado de `useSetupStore` (`GET /api/workshop/setup-status`):

1. **Datos del taller** configurados
2. **Al menos 1 empleado** (`employees` con `workshop_id` del taller y `is_active=true`)
3. **Al menos 1 servicio** en el catálogo

- **El check de "al menos 1 empleado" es genérico, no distingue quién ni cómo se creó** (`workshop.controller.js:getSetupStatus`) — solo cuenta filas de `employees` por `workshop_id` + `is_active=true`, sin filtrar por `role` ni excluir el propio `user_id` del dueño. Confirmado 2026-07-26: si el dueño usa **"Mi Perfil de Mecánico"** (`POST /api/mechanics/self`, ver [Configuración del Taller — Mi Equipo & Roles](#configuración-del-taller-config)) en vez de dar de alta un empleado nuevo por "Alta de Talento", esa fila **sí** cuenta y desbloquea el gate igual que cualquier otro empleado — ambos flujos insertan en la misma tabla con la misma forma (`workshop_id`, `role: 'mecanico'`, `is_active: true`), y el check no tiene ninguna condición que los distinga. Sin test dedicado que fije este comportamiento explícitamente (nadie lo verificó hasta ahora, solo coincide porque el filtro es genérico) — candidato a agregar si algún día el check gana una condición de rol y rompe esto en silencio.

---

## Demo pública (`/demo`, rev. 50)

Antes era una demo estática de solo Flujo de Caja (8 tarjetas + resumen diario con cifras inventadas, hardcodeadas en `Demo.jsx`). Ahora `Demo.jsx` renderiza 5 secciones (Recepción → Bahía → Facturación → Flujo de Caja → Equilibrio), cada una con un `<video>` real (`autoPlay muted loop playsInline`, `poster` de primer frame) apuntando a `frontend/public/videos/demo/{etapa}.mp4` — no hay ningún número inventado en la página.

- **Origen de los videos**: grabados sobre una cuenta de taller de prueba real ("EFISCO Demo", registrada en el Supabase de producción — no hay entorno de staging separado) operando el ciclo completo de una orden: Recepción (intake) → Bahía (crear orden, cronómetro corriendo/pausado, repuesto instalado) → Facturación (`ModalLiquidacion` real, liquidación de $45.958) → Flujo de Caja/Equilibrio mostrando los datos reales que esa liquidación generó.
- **Pipeline de render, fuera del repo de la app** (`media/hf-project/`, proyecto [HyperFrames](https://github.com/heygen-com/hyperframes) independiente, no es dependencia de `frontend/`): `media/hf-project/assets/<etapa>/*.jpg` son las capturas reales; `generate.mjs` genera `compositions/<etapa>.html` (HTML+GSAP con `data-start`/`data-duration`/`data-track-index` por clip) a partir de esos assets; `npx hyperframes render . -c compositions/<etapa>.html -o renders/<etapa>.mp4 -q high --crf 13` renderiza cada MP4 (requiere `ffmpeg` en el `PATH`, instalado vía `winget install Gyan.FFmpeg`). Los `.mp4`/`-poster.jpg` resultantes se copian a mano a `frontend/public/videos/demo/`.
- **Para regenerar o mejorar los videos** (más nitidez, tipografía, animación) **no hace falta repetir la cuenta de prueba ni las capturas** — están en disco (`media/hf-project/assets/`). Basta con editar `media/hf-project/generate.mjs` (estilos, fuente, Ken Burns, `--crf`/`-q` del render) y volver a correr `node generate.mjs` + `npx hyperframes render`. Solo si las capturas mismas se ven borrosas (fuente a 1559×789, upscaleada al frame de 1920×1080) hace falta volver a loguearse en la cuenta de prueba y recapturar — la cuenta y sus datos siguen intactos en Supabase.
- **Botón de pantalla completa** por video (`handleFullscreen` en `Demo.jsx`): llama `video.requestFullscreen()` con fallback a `video.webkitEnterFullscreen()` para iOS Safari, que no soporta la Fullscreen API estándar en `<video>`. Los controles nativos (`controls={isFullscreen}`) solo se muestran mientras el video está en pantalla completa — en el flujo normal de la página el preview queda limpio, sin controles.

---

## Configuración del Taller (`/config`)

Panel de administración fiscal y operativa. Cinco pestañas:

**1. Datos del Taller** — Nombre, dirección, **ciudad** (`workshop_config.city`, campo nuevo 2026-07-29 — no existía ningún campo de ciudad legible para el taller, `dataico_city_code` es un código DIAN numérico interno, no un nombre; usada como Ciudad del Tercero por defecto en el Libro Mayor para cualquier asiento sin proveedor/empleado explícito — incluidos los de un cliente desde 2026-08-18 (ver [FINANCIAL_ENGINE.md](FINANCIAL_ENGINE.md#tercero-completo--tipo-de-documentocontribuyente-dirección-ciudad-correo-teléfono-2026-07-29))), teléfono, horarios, costos fijos (arriendo + servicios). El teléfono se usa en el encabezado del Comprobante de Venta. Incluye la card "Contrato de Afiliación" (`ContractViewerModal.jsx`) para volver a leer el contrato B2B firmado al registrar el taller — desde rev. 45 muestra además un banner "Firmado digitalmente" con fecha, razón social y NIT (mismo componente reutilizado en el detalle de taller del [Panel Admin](BILLING.md#talleres))

- **Card "Tu suscripción"** (solo visible si hay algo pendiente): desglosa los 3 acumulados que EFISCO le cobra al taller — tarifa por orden liquidada (escalonada por volumen mensual, ver [BILLING.md](BILLING.md#dashboard)), consultas de puntaje y venta directa de repuestos (ver [BILLING.md — Venta Directa](BILLING.md#venta-directa--tarifa-por-repuesto-2026-08-05)) — con un botón "Pagar con Mercado Pago" por el total. Antes solo mostraba la tarifa por orden (fija, `software_valuation_unit`); pasó a lista con desglose el 2026-08-05 al agregarse el tercer acumulado.

- **Botón "Guardar Cambios Globales" ya no tapa el Footer** (bug corregido en rev. 46): el botón usaba `position: fixed` anclado al viewport (`bottom-6 right-6`), así que se quedaba flotando sobre el `Footer` de la app (enlaces "Privacidad", "Términos", etc.) al hacer scroll hasta el final de la página — el `Footer` vive dentro del mismo contenedor con scroll (`<main>` en `App.jsx`), no fuera de él. Cambiado a `position: sticky` dentro de un contenedor `flex justify-end` como último hijo del `<form>`: se mantiene flotando bottom-right mientras se hace scroll por el contenido del formulario, pero deja de "pegarse" en cuanto el `<form>` termina — justo antes de llegar al `Footer` — así que nunca puede solaparlo

**2. Mi Equipo & Roles** — Alta de empleados, esquemas de compensación (fijo / comisión / híbrido). Crear, editar, desactivar empleados y habilitar sus credenciales de acceso está restringido al rol `owner`. **El dueño también puede registrarse a sí mismo como mecánico** (`POST /api/mechanics/self`, upsert sobre `employees.user_id = owner.id`, sin necesitar email/password propios) para llevar su propio sueldo/comisión separado de los costos fijos genéricos del taller y aparecer como cualquier otro mecánico en el dropdown de asignación de Bahía, con sus propias métricas — esta misma fila también satisface el requisito de "al menos 1 empleado" del [Gate de Configuración Inicial](#gate-de-configuración-inicial-setupgate), confirmado 2026-07-26. La tarjeta del dueño en el grid ya no muestra el botón "ver detalles" (redundante — el banner "Tu Perfil de Mecánico" tiene su propio "Ver mis datos"), y tanto el banner como el header del modal de detalles usan avatar redondo consistente con las demás tarjetas. Nota: el rol `contador` NO es asignable como mecánico en órdenes (gate en frontend y backend, ver Bahías)

- **Datos de Tercero para el Libro Mayor (2026-07-29, opcional)**: el modal "Alta de Talento" y el panel de detalle de cada empleado (sección nueva junto a Compensación, mismo botón "Editar"/"Guardar") agregan Tipo de Documento, Cédula, Dirección, Ciudad y Teléfono (`employees.document_type`/`nit`/`address`/`city`/`phone`, columnas nuevas — `email` ya existía). Pedido explícito del usuario: que el pago de sueldo (CE-) en el Libro Mayor traiga el Tercero completo del empleado, no solo el nombre. Todos opcionales — un empleado creado antes de esta migración, o sin estos campos llenados al crearlo, simplemente sale con esas columnas vacías en el Libro Mayor hasta que se completen. Detalle técnico completo en [FINANCIAL_ENGINE.md — Tercero completo](FINANCIAL_ENGINE.md#tercero-completo--tipo-de-documentocontribuyente-dirección-ciudad-correo-teléfono-2026-07-29)

- **"Pagar a Mecánico"** (rev. 22, dentro del detalle de cada empleado): los mecánicos `comision`/`mixto` ya generan su comisión automáticamente por orden (asiento `MECH_COMMISSION`, devengo — ver [FINANCIAL_ENGINE.md](FINANCIAL_ENGINE.md#caja-real-vs-informativodevengo--por-qué-existe-la-distinción)); este botón es para registrar el **pago real en efectivo**, sea el sueldo fijo o la comisión ya devengada. Muestra primero lo pendiente por pagar (`GET /api/mechanics/:id/pending-payment` = `Σ MECH_COMMISSION − Σ MECH_COMMISSION_PAY`), y un modal para capturar los montos de sueldo/comisión a liquidar (`POST /api/mechanics/:id/pay`) — el pago queda reflejado de inmediato en Flujo de Caja
- **Panel de detalle del empleado sin condición de carrera** (bug corregido en rev. 42): el `useEffect` que carga historial salarial/métricas/pago pendiente (keyed por `showDetailsModal.employee?.id`) no cancelaba las 3 peticiones del empleado anterior — abrir el panel de un empleado y luego rápido el de otro, antes de que resolviera el primero, podía sobrescribir el panel del segundo con datos del primero si esa respuesta tardía llegaba después. Corregido con el patrón estándar de React para este caso (flag `cancelled` chequeado antes de cada `setState`, seteado en el cleanup del efecto)
- **Botón de conexión propia de Google Calendar** (dueño) + **switch "Calendario: Sí/No" por empleado** (2026-07-27): el dueño conecta su propio Google Calendar desde la card de este panel (mismo componente que en [Mi Perfil](OPERATIONS.md#mi-perfil-mi-perfil)) y, por cada mecánico, un switch (`PATCH /api/mechanics/:id/calendar-sync`) habilita que la conexión de Google Calendar **de ese mecánico** (si ya la hizo desde su propio `/mi-perfil`) empiece a recibir eventos de las órdenes que se le asignen — el switch nunca conecta a nadie, solo autoriza. Detalle del motor de sincronización en [Sincronización con Google Calendar](OPERATIONS.md#sincronización-con-google-calendar). Etiqueta corta a propósito ("Calendario: Sí/No", no "Sync Calendario: ON/OFF") tras feedback directo de que el texto técnico no se entendía a simple vista

**3. Catálogo de Servicios** — CRUD con márgenes básico/premium por tipo de vehículo

- **Sugerencia de margen por Gama/Complejidad** (rev. 49): al crear/editar un servicio, ahora hay que clasificarlo por **Gama** (Alta/Baja) y **Complejidad** (Alta/Baja) — esto no cambia cómo se factura (`base_margin_basic`/`base_margin_premium` siguen siendo los valores reales que usan Cobros/Bahía/Finanzas), solo sirve para autocompletar esos dos campos con el margen sugerido de la tabla oficial (ver [FINANCIAL_ENGINE.md](FINANCIAL_ENGINE.md#margen-de-mano-de-obra-por-gamacomplejidad-rev-49)) y mostrar el rango típico ("Rango sugerido: Básico 55%–40% (sugerido 45%) · Premium 60%–45% (sugerido 50%)"). El usuario puede sobrescribir el número sugerido a mano. `gama`/`complejidad` son obligatorios en el backend (400 si faltan o no son `'Alta'`/`'Baja'`) y se muestran como badges en la tarjeta de cada servicio del catálogo
- El modal "Crear Servicio" (y el de "Añadir Empleado" en Mi Equipo) tienen scroll interno defensivo (`max-h-[90vh]` + `overflow-y-auto`) para que un formulario que crezca (como pasó al agregar Gama/Complejidad) nunca quede cortado por el alto de la pantalla — antes desbordaban el modal `fixed inset-0` sin forma de hacer scroll

**4. Pasarelas**

Editable directamente por el dueño (no requiere al contador):

*Tasas de pasarelas*: Bold físico (2.99%), Bold online (3.49%), Addi (10.5%)

*Anticipo de Folios*: calculadora de recarga de folios Dataico (cantidad × costo unitario + margen + comisión de pasarela), con registro automático del egreso en el flujo de caja

**5. Módulo del Contador**

*Parámetros Fiscales y de Recaudo* — visible aquí en modo solo lectura para el dueño (la edición real vive en el panel del contador, `/contador`). Régimen fiscal, tasas, los 27 códigos PUC, credenciales Dataico e identidad legal son campos **exclusivos del rol `contador`** — el backend (`updateWorkshop`) los descarta en silencio si el llamador no es contador, ni siquiera el dueño puede modificarlos vía API, solo leerlos (ver [SECURITY.md](../Arquitectura%20y%20Sistema%20Core/SECURITY.md#decisiones-técnicas-de-seguridad)):

Régimen Fiscal (4 opciones):
| Opción | IVA | Reg. Simple | Agente Retenedor |
|:---|:---:|:---:|:---:|
| No Responsable de IVA | ✗ | ✗ | ✗ |
| Régimen Simple (SIMPLE) | ✓ | ✓ | ✗ |
| Régimen Ordinario | ✓ | ✗ | ✓ |
| Gran Contribuyente | ✓ | ✗ | ✓ |

Régimen Simple SÍ cobra IVA (solo no practica retenciones) — verificado contra la DIAN: el RST unifica renta+ICA en un solo pago pero el IVA se declara aparte para la generalidad de actividades (la "tarifa con IVA incluido" solo aplica a tiendas/minimercados pequeños del numeral 1, no a un taller de servicios). Bug corregido esta revisión: `ContadorInventario.jsx` guardaba `is_responsable_iva:false` para esta opción, contradiciendo su propia descripción y al backend.

**Bug real corregido en rev. 48 — el panel podía "mentir" sobre qué régimen ya estaba guardado**: tanto `ContadorInventario.jsx` como `Config.jsx` inicializaban su estado local (`DEFAULT_WORKSHOP`/`workshopData`) con `fiscal_regime: 'ordinario'` ANTES de que el `fetch` a `GET /api/workshop/:id` resolviera con los datos reales — si el fetch tardaba, fallaba, o el contador solo alcanzaba a ver el panel un instante, "Régimen Ordinario" aparecía como ya seleccionado sin que fuera cierto. Diagnosticado tras confirmar con una consulta directa a la base de datos real que los 4 talleres existentes seguían en `fiscal_regime='no_iva'` pese a que el dueño creía haber configurado "Ordinario" — nadie había guardado el cambio de verdad. Corregido el default local a `'no_iva'`, el mismo que usa `auth.controller.js:register` al crear el taller. Bug relacionado, mismo origen (migración a cookies incompleta, ver [SECURITY.md](../Arquitectura%20y%20Sistema%20Core/SECURITY.md#bugs-de-sesión-encontrados-en-producción-tras-el-deploy-2026-07-21-rev-48)): `ContadorInventario.jsx` leía `WORKSHOP_ID` de `localStorage['user']` (ya no se escribe desde la migración) — sin él, el fetch de guardado ni siquiera podía dispararse, lo cual probablemente explica por qué el cambio de régimen nunca llegó a persistirse en los 4 talleres.

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

**Proveedores** (tab nueva dentro de `/contador`, `frontend/src/components/ContadorProveedores.jsx`): lista los proveedores del taller con badge de perfil tributario y un selector para corregirlo directamente (reusa `GET/PUT /api/providers`), más acceso al historial de compras de cada uno — antes el contador no tenía ninguna forma de ver ni editar la clasificación fiscal de los proveedores del taller. Solo eso: no edita nombre, NIT, ni ningún dato de tercero (ver el punto siguiente).

*Datos de tercero completos para el Libro Mayor (2026-07-28, reubicado 2026-07-29)* — Siigo/Alegra piden Tipo de Documento, Tipo de Contribuyente, Dirección, Ciudad, Correo y Teléfono al importar un tercero, además de Identificación/Razón Social. Un primer intento agregó un modal de edición completo ("Editar datos", ícono de lápiz) dentro de `ContadorProveedores.jsx` — se revirtió el mismo día por pedido del usuario: esos datos no son responsabilidad del contador, son datos operativos del proveedor que el dueño ya conoce al darlo de alta. Quedaron así:

- **Proveedores** — el modal "Nuevo/Editar Proveedor" de `Proveedores.jsx` (rol dueño) agrega **Tipo de Documento** (`document_type`: CC/NIT/CIE, select explícito, columna con `CHECK`, default `NIT`, migración `2026-07-28_provider_document_taxpayer_type.sql`) y **Dirección**/**Correo Electrónico** (Ciudad y Teléfono ya existían en el formulario). **Tipo de Contribuyente** (`taxpayer_type`: `natural`/`juridica`) NO tiene selector propio — se deriva automáticamente del selector de "Perfil Tributario" que ya existía ahí: `Persona Natural` ⇒ `natural`, cualquiera de los otros 3 regímenes (Simple/Ordinario/Gran Contribuyente) ⇒ `juridica`, porque son la misma decisión vista dos veces y un segundo selector solo agregaría la posibilidad de que queden inconsistentes entre sí. Los 4 campos entraron a la lista blanca `EDITABLE_FIELDS` de `providers.controller.js:updateProvider`/`createProvider` (ya estaban ahí desde el intento revertido, no cambiaron).
- **Clientes** — `ClienteRegistro.jsx` (registro público, ver [Recepción](OPERATIONS.md#recepción)) agrega **Tipo de Documento** (`clients.document_type`: CC/NIT/CIE, select explícito, junto a `email`/`phone` que ya vivían en `clients`). **Tipo de Contribuyente** para clientes se deriva igual que para proveedores, pero de `work_orders.client_type` (que ya existía): `'Natural'` ⇒ Persona Natural, los otros 3 ⇒ Persona Jurídica. Correo/Dirección ya se pedían ahí desde antes. **Ciudad ya no se pide aquí (quitada 2026-08-18)**: el campo existió brevemente (`work_orders.client_city`, migración `2026-07-29_client_document_type_city.sql`) pero casi siempre quedaba vacío en la práctica — se eliminó del formulario y la columna Ciudad del cliente en el Libro Mayor ahora siempre usa la ciudad del propio taller (`workshop_config.city`), ver [FINANCIAL_ENGINE.md](FINANCIAL_ENGINE.md#tercero-completo--tipo-de-documentocontribuyente-dirección-ciudad-correo-teléfono-2026-07-29).
- **Empleados** (misma pasada, 2026-07-29, pedido explícito del usuario) — "Alta de Talento" y el panel de detalle de cada empleado agregan Tipo de Documento, Cédula, Dirección, Ciudad y Teléfono (`employees.document_type`/`nit`/`address`/`city`/`phone`, columnas nuevas). Un empleado siempre es Persona Natural — a diferencia de clientes/proveedores, no hay ningún selector ni inferencia de Tipo de Contribuyente. Ver [Configuración del Taller — Mi Equipo](#configuración-del-taller-config) arriba.
- **El taller mismo, como fallback** (misma pasada) — cualquier asiento sin proveedor/cliente/empleado explícito (Costos Fijos, movimientos manuales) usa la identidad del propio taller: Dirección/Teléfono ya existían (`workshop_config.address`/`phone`), **Ciudad** es campo nuevo (`workshop_config.city`, ver "1. Datos del Taller" arriba), y el Correo se resuelve del email de login del dueño (Auth), no de una columna nueva — ningún taller activado se queda sin correo de contacto. Tipo de Documento/Contribuyente se infieren del formato de `legal_nit` (cédula vs. NIT), ver [FINANCIAL_ENGINE.md — Tipo de Contribuyente del propio taller](FINANCIAL_ENGINE.md#tipo-de-contribuyente-del-propio-taller-2026-07-29).

Detalle completo de qué tabla vive cada campo y por qué, en [FINANCIAL_ENGINE.md — Tercero completo](FINANCIAL_ENGINE.md#tercero-completo--tipo-de-documentocontribuyente-dirección-ciudad-correo-teléfono-2026-07-29).

*Identidad Legal*: NIT, Razón Social, Prefijo — edición exclusiva del rol `contador` (el dueño solo lee). (No existe un campo de "Clave Técnica DIAN" propia del taller: bajo el modelo de sub-cuentas de Dataico, el CUFE lo calcula Dataico internamente por cada sub-cuenta — account_id + token + resolución —, el taller nunca necesita ni envía una clave técnica propia.)

*Presets PUC* — 3 botones que auto-rellenan los 27 códigos según régimen:
- **Régimen Ordinario** (estándar DIAN)
- **Régimen Simple** (subcuentas simplificadas)
- **Gran Contribuyente** (subcuentas retención en la fuente)

*Plan Único de Cuentas — 27 códigos en 5 bloques*:

| Bloque | Códigos | Defaults |
|:---|:---|:---|
| Ingresos & Ventas | `puc_income_code`, `puc_parts_income_code`, `puc_descuento_ventas_code`, `puc_inventory_purchase_code`, `puc_costo_ventas_code`, `puc_comisiones_mecanicos_code`, `puc_gateway_fee_code`, `puc_gateway_vat_code` | `4135`, `4135`, `417595`, `1435`, `6135`, `510506`, `5290`, `2408` |
| IVA | `puc_iva_generated_code`, `puc_iva_generated_5_code`, `puc_iva_deductible_code`, `puc_devolucion_iva_code` | `240805`, `240810`, `240820`, `135520` |
| Retenciones por Pagar | `puc_retefuente_code`, `puc_retefuente_compras_decl_code`, `puc_retefuente_compras_nodecl_code`, `puc_retefuente_servicios_code`, `puc_reteiva_code`, `puc_reteica_code` | `2365`, `236540`, `236540`, `236525`, `2367`, `2368` |
| Retenciones a Favor | `puc_anticipo_retefuente_code`, `puc_anticipo_reteica_code`, `puc_pasarela_retencion_code` | `135515`, `135518`, `135595` |
| Control Financiero | `puc_cxc_clientes_code`, `puc_cxp_proveedores_code`, `puc_otros_ingresos_code`, `puc_gastos_financieros_code`, `puc_bancos_code`, `puc_capital_inicial_code` | `130505`, `220505`, `4210`, `5305`, `111005`, `3705` |

`puc_inventory_purchase_code`, `puc_costo_ventas_code` y `puc_comisiones_mecanicos_code` se agregaron en rev. 25: el backend (`billing.controller.js`) ya los leía para el asiento de costo de repuestos/comisiones, pero no estaban en la lista blanca de campos editables por el contador (`updateWorkshop`) — el primero se descartaba en silencio al guardar (siempre caía a su default `1435`), los otros dos ni tenían input en ninguna pantalla.

`puc_descuento_ventas_code` se agregó el 2026-08-16 (feature de Descuento por Regateo, pedido explícito del usuario): cuenta contra-ingreso (grupo PUC 4175 "Descuentos en Ventas", default `417595`) para el DÉBITO que registra `billing.controller.js:settleOrder` cuando un cajero aplica `descuento_monto` al liquidar — ver [FINANCIAL_ENGINE.md](FINANCIAL_ENGINE.md#descuento-por-regateo-feature-2026-08-16-pedido-explícito-del-usuario). Migración `2026-08-16_puc_descuento_ventas.sql`.

`puc_bancos_code` se agregó en rev. 39: bug real corregido en `finance.controller.js:setOpeningBalance` — el asiento del **Saldo Inicial** (Panel Financiero → "Configurar Saldo Inicial", el efectivo migrado del cuaderno/Excel previo del taller) se registraba contra `puc_otros_ingresos_code` (una cuenta de INGRESO), como si el capital que el taller ya tenía en el banco antes de usar EFISCO fuera una venta del período. Ahora usa esta cuenta dedicada (PUC clase 11 "Disponible"), configurable por el contador en el bloque "Control Financiero" de `ContadorPanel.jsx`.

**Bug real corregido (2026-07-28) — Saldo Inicial sin partida doble**: el asiento de arriba se quedó, desde rev. 39, como una única línea `CREDIT` a `puc_bancos_code` para representar un aumento — contablemente al revés (Bancos es un activo, aumenta por DÉBITO, no por crédito) y sin ninguna contrapartida que lo balanceara. Puertas adentro de EFISCO el signo no se notaba (el resto del motor también leía "CREDIT = aumenta" para este tipo, así que los totales de Finanzas/Flujo de Caja igual cuadraban), pero al exportar el Libro Mayor e importarlo a un sistema contable estándar (Siigo/Alegra, que sí aplican la regla real de partida doble) esa línea CREDIT-solo se leía como una **reducción** de Bancos, dejando el saldo en negativo — y la fila ya se veía roja/negativa dentro de la propia Lista de Movimientos de EFISCO, porque esa sí aplicaba la regla estándar de Activo (`ledgerLabels.js:getMovementFormatting`) y detectaba la inconsistencia. Corregido insertando **dos líneas** por evento: `OPENING_BALANCE` (DEBIT a `puc_bancos_code`, aumenta el activo) + su contrapartida `OPENING_BALANCE_EQUITY` (CREDIT a `puc_capital_inicial_code`, la cuenta de capital nueva) — mismo patrón de partida doble que ya usa el resto del ledger (`AR_RECOGNITION`/`CASH_RECEIPT`, `AP_RECOGNITION`/`CASH_PAYMENT`). El signo de lectura (`financialEngine.js`/`ledgerLabels.js:openingBalanceValue`) se invirtió para que DEBIT sea el que suma — las filas históricas que ya existían con la convención vieja se corrigieron con un script de un solo uso (`backend/scripts/fix-opening-balance-double-entry.mjs`), que invierte su `impact` y les agrega la contrapartida que les faltaba. Ver [FINANCIAL_ENGINE.md](FINANCIAL_ENGINE.md#saldo-inicial--partida-doble-2026-07-28).

**Bug real corregido (2026-07-28) — Inventario Inicial, mismo defecto pero en dos endpoints de Inventario**: registrar un repuesto "al costo" sin pasar por una Compra a Proveedor (alta directa en `/inventario`, o el botón "Añadir Stock" para sumarle unidades a uno existente) nunca insertaba nada en el Libro Auxiliar — solo el Kardex físico se enteraba. La cuenta `puc_inventory_purchase_code` (1435) quedaba cada vez más negativa cuanto más se vendía de ese inventario, porque `INV_COGS` sí descarga esa cuenta al vender pero nunca hubo un DEBIT que la compensara al entrar. Corregido con el mismo par DEBIT/CREDIT que el Saldo Inicial, ahora también para Inventario — detalle técnico y script de backfill (`backend/scripts/backfill-inventory-opening-balance.mjs`) en [FINANCIAL_ENGINE.md](FINANCIAL_ENGINE.md#inventario-inicial--partida-doble-2026-07-28).

*IVA por Categoría de Repuesto* — 10 categorías con tasa individual configurada aquí; se aplica automáticamente al seleccionar categoría en Inventario — ver [INVENTORY.md](INVENTORY.md#inventario) para el detalle del campo (`workshop_config.category_vat_rates`).

*Exportación contable — Libro Mayor único* (reemplaza los 6 reportes CSV anteriores — facturas/compras/CxC/CxP/ledger/inventario valorizado, cada uno con columnas distintas y sin protección contra inyección de fórmulas): un solo Excel real (`.xlsx`, vía `exceljs`) con la estructura `Fecha | Número de Documento | Código PUC | Cuenta | Detalle | Identificación | Razón Social/Nombre | Tipo de Documento | Tipo de Contribuyente | Dirección | Ciudad | Correo Electrónico | Número de Contacto | Débito | Crédito | ID Interno EFISCO`, todo el histórico del taller (o un rango `from`/`to`) en orden cronológico. El UUID interno de Supabase se movió al final, con nombre explícito ("ID Interno EFISCO") — antes vivía al frente como columna "ID", entre el tercero y los montos, arriesgando que un sistema contable la importara como si fuera un dato contable real.

"Tercero" (Identificación/Razón Social) se resuelve en cadena según el tipo de evento — ventas con orden (`work_order_id` → `work_orders` → `clients`), compras (`related_purchase_id` → `supplier_purchases → providers`), pagos a mecánicos (`mechanic_id` → `employees`), ingresos por referido (nombre extraído del `concept`), ventas de mostrador sin cliente registrado ("Cliente Ocasional"), las líneas de Capital (Saldo Inicial/Inventario Inicial, desde 2026-07-28) y — desde 2026-07-29 — **cualquier otro asiento sin tercero externo explícito** (Costos Fijos, movimientos manuales), usan el NIT/Razón Social **legal del propio taller** (`legal_nit`/`legal_razon_social` de la Identidad Legal de arriba), porque esa cuenta se lleva "por socio" y el socio, en un taller sin estructura societaria formal, es el taller mismo. Detalle completo de las 7 prioridades de esta cadena en [FINANCIAL_ENGINE.md — Resolución de Tercero](FINANCIAL_ENGINE.md#resolución-de-tercero-en-el-libro-mayor-2026-07-28). Las 6 columnas adicionales (Tipo de Documento/Contribuyente, Dirección, Ciudad, Correo, Teléfono) se resuelven para clientes, proveedores, empleados y el fallback al taller — solo ventas de mostrador e ingresos por referido quedan en blanco ahí, a propósito (el tercero real es un cliente anónimo o solo un nombre en texto libre, no el taller mismo), ver [Configuración del Taller — datos de tercero completos](#configuración-del-taller-config) arriba y [FINANCIAL_ENGINE.md](FINANCIAL_ENGINE.md#tercero-completo--tipo-de-documentocontribuyente-dirección-ciudad-correo-teléfono-2026-07-29).

**Número de Documento (2026-07-28)** — pedido explícito del contador: sin un número de comprobante que agrupe las líneas de un mismo evento, Siigo/Alegra no podían importar el libro sin descuadres. Cada evento contable (una venta liquidada, una compra registrada, un pago a mecánico, un saldo inicial, un ajuste manual) recibe un consecutivo con prefijo por categoría — `FV-` ventas, `CP-` compras a proveedores, `CE-` pagos a mecánicos, `ASI-` saldos iniciales, `CC-` ajustes/movimientos manuales —, y **todas** las líneas débito/crédito de ese mismo evento comparten el mismo número. El consecutivo va **sin ceros a la izquierda** (`ASI-1`, `ASI-2`, ... `ASI-10` — pedido explícito del contador, corregido 2026-07-29; antes rellenaba a 3 dígitos, `ASI-001`). Se asigna una sola vez, en el momento del insert (`backend/utils/documentNumbering.js`), nunca recalculado al exportar — así el número de un evento no cambia entre una exportación y otra según el rango de fechas elegido.

**Reclamo + insert atómicos, en una sola transacción de Postgres (2026-07-29)** — pedido explícito del usuario ("que la generación sea atómica, determinista e inmutable"): el patrón anterior (reclamar el consecutivo en dos pasos desde el cliente, luego insertar las filas en una llamada aparte) dejaba una ventana real donde un evento con número MAYOR podía llegar a `cash_flow_ledger` antes que uno con número MENOR, si el insert del primero tardaba más que el reclamo completo del segundo. Cerrado con dos funciones de Postgres que reclaman e insertan en una sola transacción — primera vez que este proyecto despliega una función/RPC (ver nota de convención de migraciones en [ARCHITECTURE.md](../Arquitectura%20y%20Sistema%20Core/ARCHITECTURE.md#4-decisiones-técnicas-clave)). Detalle técnico completo (contador atómico, reuso por FK, script de backfill histórico) en [FINANCIAL_ENGINE.md](FINANCIAL_ENGINE.md#número-de-documento-2026-07-28).

*Integración Dataico*: cada taller tiene su propia sub-cuenta (creada manualmente por Efisco dentro de la cuenta maestra) — se configura aquí el ID de cuenta, el token, número de resolución DIAN + prefijo, y departamento/ciudad (DIVIPOLA). El entorno (`PRUEBAS`/`PRODUCCION`) ya no es un toggle manual: se detecta solo según si corre en Vercel o en local. Botón de prueba de conexión contra la API real de Dataico.

- **Bug real corregido (2026-07-27)**: los inputs de "ID de Cuenta Dataico"/"Token de Autenticación Dataico" (`ContadorPanel.jsx`) tenían `type="password"` sin `autocomplete="off"` — el gestor de contraseñas del navegador, viendo un campo de tipo password dentro de un formulario, ocasionalmente los rellenaba con el correo/contraseña guardados de OTRO sitio en vez de dejarlos vacíos, sin que el contador lo notara antes de guardar. Ninguno de los dos valores es realmente secreto en el sentido de "hay que ocultarlo mientras se escribe" (el token no se vuelve a mostrar en pantalla una vez guardado, pero tampoco hay riesgo de shoulder-surfing que justifique ocultarlo mientras se tipea) — cambiados a `type="text"` con `autocomplete="off"` explícito en ambos inputs del formulario.
