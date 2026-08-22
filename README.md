# Efisco ERP — Automotive Workshop SaaS

> **Ingeniería de software de alta precisión aplicada a la rentabilidad y automatización del sector automotriz.**

Efisco es una plataforma SaaS diseñada para transformar talleres mecánicos en centros operativos inteligentes. Integra un **Motor Financiero (Fiscal/Contable)** adaptado a la normativa colombiana 2026, un **Clasificador de Vehículos** para tarificación dinámica, **OCR con IA** para el control de egresos, y un **Panel de Administración Interno** para la gestión del ecosistema de talleres.

![React 19](https://img.shields.io/badge/React-19-61DAFB?style=for-the-badge&logo=react)
![Express 5](https://img.shields.io/badge/Express-5-000000?style=for-the-badge&logo=express)
![Supabase](https://img.shields.io/badge/Supabase-PostgreSQL-3ECF8E?style=for-the-badge&logo=supabase)
![Tailwind v4](https://img.shields.io/badge/Tailwind-v4-06B6D4?style=for-the-badge&logo=tailwind-css)
![AWS Textract](https://img.shields.io/badge/OCR-AWS_Textract-FF9900?style=for-the-badge&logo=amazonaws)
![Meta API](https://img.shields.io/badge/WhatsApp-Meta_Cloud_API-25D366?style=for-the-badge&logo=whatsapp)
![Google Calendar](https://img.shields.io/badge/Google-Calendar_API-4285F4?style=for-the-badge&logo=googlecalendar)

---

## Documentación

Este README es un resumen general. El detalle técnico vive en documentos dedicados, organizados en 3 carpetas por categoría (más 3 documentos transversales en la raíz de `efiscomd/`):

**[`Arquitectura y Sistema Core/`](Arquitectura%20y%20Sistema%20Core/)**

| Documento | Contenido |
|:---|:---|
| **[ARCHITECTURE.md](Arquitectura%20y%20Sistema%20Core/ARCHITECTURE.md)** | Mapa de componentes, ciclo de vida operativo end-to-end, stack tecnológico, decisiones técnicas clave, Kardex, variables de entorno |
| **[SECURITY.md](Arquitectura%20y%20Sistema%20Core/SECURITY.md)** | Aislamiento multi-tenant, autenticación admin separada, gates de suspensión/contrato B2B/autorización del taller, RLS de Supabase, webhooks de pago |
| **[API.md](Arquitectura%20y%20Sistema%20Core/API.md)** | Referencia completa de endpoints REST (`/api/*` y `/api/admin/*`) |

**[`Reglas de Negocio y Finanzas/`](Reglas%20de%20Negocio%20y%20Finanzas/)**

| Documento | Contenido |
|:---|:---|
| **[BUSINESS_RULES.md](Reglas%20de%20Negocio%20y%20Finanzas/BUSINESS_RULES.md)** | Rutas del frontend, onboarding de un taller nuevo, Gate de Configuración Inicial, Demo pública, y Configuración del Taller (`/config`: Mi Equipo, Catálogo de Servicios, Pasarelas, Módulo del Contador/PUC/Dataico) |
| **[OPERATIONS.md](Reglas%20de%20Negocio%20y%20Finanzas/OPERATIONS.md)** | Recepción, Registro Seguro EFISCO (incl. score crediticio), Bahías/Órdenes de Trabajo, Mi Perfil, Sincronización con Google Calendar |
| **[INVENTORY.md](Reglas%20de%20Negocio%20y%20Finanzas/INVENTORY.md)** | Inventario (Kardex, stock mínimo vital, rotación, importación masiva desde Excel/CSV), Herramientas (tablero visual, cantidad, disponibilidad calculada) y Proveedores y Egresos (compras, pago a plazos, OCR de facturas) |
| **[FINANCE.md](Reglas%20de%20Negocio%20y%20Finanzas/FINANCE.md)** | Dashboard de inicio, Panel de Equilibrio, Panel de Cobros, Flujo de Caja y Libros Mensuales archivados |
| **[BILLING.md](Reglas%20de%20Negocio%20y%20Finanzas/BILLING.md)** | Referidos del taller, y el Panel de Administración EFISCO completo: Talleres, Cobros/Pagos de la suscripción SaaS, árbol de Referidos, Contabilidad interna, Reportes Financieros (Relación con Inversionistas) |
| **[FINANCIAL_ENGINE.md](Reglas%20de%20Negocio%20y%20Finanzas/FINANCIAL_ENGINE.md)** | `financialEngine.js`: liquidación de servicios, liquidación de compras, salud financiera global, tipos de movimiento del Libro Auxiliar |

**[`Estrategia Comercial y Ventas/`](Estrategia%20Comercial%20y%20Ventas/)**

| Documento | Contenido |
|:---|:---|
| **[GROWTH_ACQUISITION.md](Estrategia%20Comercial%20y%20Ventas/GROWTH_ACQUISITION.md)** | Cómo EFISCO capta y hace crecer su base de talleres: embudo público (Landing/Demo/Tarifas), modelo de precio pay-per-uso, onboarding hasta taller operando, programa de Referidos entre talleres, retención vía Panel Admin |
| **[PRICING_SALES.md](Estrategia%20Comercial%20y%20Ventas/PRICING_SALES.md)** | Cómo el taller tarifica y vende a sus propios clientes: clasificador de vehículos (tiers Premium/Básico), márgenes por Gama/Complejidad, descuento por regateo, ventas a crédito, pasarelas de pago, Venta Directa |

**Raíz de `efiscomd/`** (transversales, sin carpeta de categoría)

| Documento | Contenido |
|:---|:---|
| **[TESTING.md](TESTING.md)** | Suite Jest (108 suites / 527 tests), qué cubre cada una, CI en GitHub Actions |
| **[MONITORING.md](MONITORING.md)** | Bot de Telegram interno de alertas de plataforma (separado del bot de clientes): catálogo completo de las 20 alertas, política de privacidad (qué dato de un taller/cliente puede o no aparecer), flujo de vinculación QR, panel `/admin/alertas` |

---

## Descripción General

Efisco no es un ERP genérico adaptado al sector automotriz — fue construido desde cero para resolver los problemas reales de los talleres colombianos:

- **Tarificación dinámica** basada en clasificador de vehículos por segmento (básico / premium)
- **Liquidación fiscal precisa** según normativa colombiana 2026: IVA, ReteFuente, ReteICA, ReteIVA, GMF 4×1000
- **Control de egresos con OCR** — extracción automática de datos de facturas de proveedores vía AWS Textract
- **Facturación electrónica** integrada con Dataico/DIAN bajo sub-cuenta propia de cada taller (no bloqueante, guarda CUFE en base de datos)
- **Comunicación automatizada** con clientes vía WhatsApp Cloud API (Meta)
- **Sincronización con Google Calendar** — las órdenes de trabajo aparecen automáticamente en el calendario personal del dueño y de cada mecánico, cada quien con su propia conexión OAuth (ver [OPERATIONS.md](Reglas%20de%20Negocio%20y%20Finanzas/OPERATIONS.md#sincronización-con-google-calendar))
- **Multi-tenant** con aislamiento de datos por taller verificado a nivel de aplicación — cada endpoint valida `workshop_id` contra la sesión del usuario autenticado. RLS de Postgres es una capa adicional de defensa (ver [SECURITY.md](Arquitectura%20y%20Sistema%20Core/SECURITY.md)), no la principal
- **Ventas a crédito** con plan de cuotas: INC_GROSS se registra en $0 al liquidar y el ingreso real entra vía pagos de cuotas
- **Panel de equilibrio interactivo** con gráfica SVG hover, proyección what-if con cálculo fiel de margen
- **IVA por categoría de repuesto** — tasas configurables por las 10 categorías de inventario
- **Presets PUC colombianos** — 3 regímenes fiscales que auto-rellenan los 27 códigos contables
- **Panel Admin EFISCO** — gestión interna de talleres, comisiones, referidos y suspensión de acceso

---

## Arquitectura (resumen)

Arquitectura **multi-tenant** con aislamiento de datos verificado en cada endpoint del backend (`workshop_id` de la sesión) y un núcleo de cálculo financiero inmutable. El frontend (React 19 + Vite) habla con un backend Express 5 dividido en tres módulos — Operativo, Logístico y Financiero — sobre una base Supabase/PostgreSQL. El panel `/admin` de EFISCO corre en paralelo con autenticación y JWT completamente separados. Diagramas de componentes, el ciclo de vida operativo end-to-end (recepción → bahía → liquidación → factura) y las decisiones técnicas clave (ledger inmutable, pipeline OCR asíncrono, margen what-if fiel, gotchas de PDF/mobile) están en **[ARCHITECTURE.md](Arquitectura%20y%20Sistema%20Core/ARCHITECTURE.md)**.

**Stack**: React 19 · Express 5 · Supabase (PostgreSQL) · Tailwind v4 · AWS Textract (OCR) · Meta WhatsApp Cloud API · Puppeteer (PDF) · Dataico (DIAN) · Bold/Addi (pasarelas) · Google Calendar API.

---

## Módulos del Sistema (resumen)

Para un visitante sin sesión, `/` es la **página de presentación pública** (`Landing.jsx` — funciones, resumen del Flujo de Caja con link a `/demo` para el recorrido completo, CTA de registro/login); `/demo` (rev. 50) ya no es una demo estática — son 5 videos reales (Recepción → Bahía → Facturación → Flujo de Caja → Equilibrio) grabados sobre una cuenta de taller de prueba real, con botón de pantalla completa por video (ver [BUSINESS_RULES.md](Reglas%20de%20Negocio%20y%20Finanzas/BUSINESS_RULES.md#demo-pública-demo-rev-50)); `/reportes-financieros` (2026-08-03) es la página de Relación con Inversionistas — reportes trimestrales/anuales (PDF) que el admin interno publica desde `/admin/reportes-financieros` (ver [BILLING.md](Reglas%20de%20Negocio%20y%20Finanzas/BILLING.md#reportes-financieros-adminreportes-financieros)); `/login` es el formulario de autenticación (mismo componente alterna login/registro/recuperar-contraseña). El dueño del taller, ya autenticado, opera desde `/dashboard`, `/recepcion`, `/bahia`, `/inventario`, `/proveedores`, `/config`, y paneles financieros (`/finanzas`, `/equilibrio`, `/cobros`, `/flujo-caja`). El cliente final interactúa vía un link público de WhatsApp (`/cliente/registro/:workshopId/:intakeId`) para completar su registro y ver su score crediticio. El contador tiene un panel propio (`/contador`) para los campos fiscales/PUC/Dataico que el dueño solo puede leer.

Flujo típico: **Recepción** (ingreso + score crediticio) → **Bahía** (ejecución, consumo de inventario con Kardex inmutable, clasificación de tier) → **Liquidación** (motor financiero + factura Dataico + comprobante por WhatsApp) → **Cobros/Flujo de Caja** (si fue a crédito).

Tabla de rutas con niveles de acceso y Configuración del Taller en **[BUSINESS_RULES.md](Reglas%20de%20Negocio%20y%20Finanzas/BUSINESS_RULES.md)**. Reglas de negocio completas de cada módulo operativo en **[OPERATIONS.md](Reglas%20de%20Negocio%20y%20Finanzas/OPERATIONS.md)** (Recepción, Bahías), **[INVENTORY.md](Reglas%20de%20Negocio%20y%20Finanzas/INVENTORY.md)** (Inventario, Proveedores), **[FINANCE.md](Reglas%20de%20Negocio%20y%20Finanzas/FINANCE.md)** (Dashboard, Flujo de Caja, Equilibrio) y **[BILLING.md](Reglas%20de%20Negocio%20y%20Finanzas/BILLING.md)** (Referidos, Panel de Administración EFISCO: talleres, cobros SaaS, pagos de comisión, contabilidad interna).

---

## Motor Financiero (resumen)

`backend/utils/financialEngine.js` es el núcleo de cálculo inmutable: liquida servicios (mano de obra + repuestos, IVA, retenciones si el cliente es agente retenedor, comisión de pasarela Bold/Addi), liquida compras a proveedor (matriz de retenciones según régimen tributario del taller Y del proveedor, GMF 4×1000, pago a plazos), y calcula la salud financiera global (`bankBalance`, `ivaLiability`, `realBankBalance`). Constantes 2026: UVT = $50.318, umbral de retenciones servicios = 2 UVT, compras = 10 UVT (Decreto 572/2025 vigente — en litigio activo, revisar periódicamente). Cada movimiento queda registrado en el **Libro Auxiliar** (`cash_flow_ledger`), un ledger append-only con 15+ tipos de movimiento (`INC_GROSS`, `TAX_IVA`, `RET_FUENT`, `GW_FEE`, etc.), exportable como un **Libro Mayor único en Excel** desde el panel del contador.

Fórmulas completas, matriz de decisión de liquidación y la tabla de tipos de movimiento en **[FINANCIAL_ENGINE.md](Reglas%20de%20Negocio%20y%20Finanzas/FINANCIAL_ENGINE.md)**.

---

## Estrategia Comercial y Ventas (resumen)

Sin plan mensual fijo: EFISCO cobra pay-per-uso a cada taller ($535/$416 COP por orden liquidada según volumen, $150 COP por repuesto de Venta Directa, $100 COP por consulta de puntaje), con un embudo público (Landing → `/demo` con videos reales → `/tarifas` → "Solicitar Acceso" → verificación manual → Gate de Configuración Inicial) y un programa de Referidos entre talleres como principal motor de crecimiento (33%/66%/100% de descuento por 1/2/3+ referidos, 15% de comisión directa a partir de Platino). Hacia el cliente final, el taller tarifica dinámicamente por vehículo (clasificador Premium/Básico), ajusta márgenes por Gama/Complejidad de cada servicio, y cierra ventas con descuento por regateo, plan de cuotas a crédito o pasarelas Bold/Addi.

Detalle completo en **[GROWTH_ACQUISITION.md](Estrategia%20Comercial%20y%20Ventas/GROWTH_ACQUISITION.md)** (EFISCO → taller) y **[PRICING_SALES.md](Estrategia%20Comercial%20y%20Ventas/PRICING_SALES.md)** (taller → cliente final).

---

## Seguridad (resumen)

La defensa principal contra acceso cruzado entre talleres es **a nivel de aplicación**: cada endpoint valida `workshop_id`/rol contra la sesión antes de tocar la base. RLS de Postgres (activado en las 28 tablas públicas, deny-by-default) es una capa adicional de defensa en profundidad, no el mecanismo principal — el backend siempre usa `service_role`, que ignora RLS. El panel admin EFISCO tiene JWT y credenciales completamente separados de los talleres. Existen gates en dos capas (frontend + middleware) para suspensión de talleres, firma del Contrato B2B, y autorización del taller por parte de empleados, cada uno con su propia prueba legal append-only.

Estado auditado del Security Advisor de Supabase, autenticación admin, llaves de acceso de soporte y la nota de seguridad sobre webhooks de pago en **[SECURITY.md](Arquitectura%20y%20Sistema%20Core/SECURITY.md)**.

---

## Monitoreo y Alertas (resumen)

Bot de Telegram interno (separado del bot de clientes) que avisa al equipo EFISCO en vivo ante errores de plataforma (500 no manejados, caídas, base de datos sin responder) y 15 eventos de negocio (webhooks de pago fallidos, facturación Dataico fallida, mensajería a clientes fallida, bloqueo de PIN/login por fuerza bruta, alta de taller nuevo, entre otros). Regla no negociable: ninguna alerta puede citar datos reales de un taller o de sus clientes — como mucho, identifica de qué taller viene el fallo (nombre/correo/teléfono); los errores internos se sanean (`sanitizeErrorMessage`) y los de proveedores externos (Dataico, Meta) se omiten por completo.

Catálogo completo de las 20 alertas, infraestructura, flujo de vinculación QR (`/admin/alertas`) y cobertura de tests en **[MONITORING.md](MONITORING.md)**.

---

## Testing (resumen)

`backend/tests/` — Jest con ESM nativo (`--experimental-vm-modules`): **108 suites / 527 tests**, todos contra lógica pura o con Supabase/auth mockeados, sin tocar la base real. Cubre el motor financiero (incl. la distinción caja real vs. devengo), el margen de contribución/punto de equilibrio, el Índice de Productividad del Panel de Equilibrio y la comparación mensual de Ingresos/Costos/Utilidad del Dashboard, clasificador de vehículos, OCR, inventario (incl. consumo gradual de líquidos/químicos), comisiones y pagos de mecánicos, gates de seguridad (contrato B2B, autorización del taller, sesión de soporte), facturación Dataico (incl. Caso 2 — EFISCO como emisor), Mercado Pago, y el catálogo completo de alertas de Telegram (ver [MONITORING.md](MONITORING.md)) con datos de error realistas para probar tanto que disparan como que sanean PII de verdad. CI en GitHub Actions corre la suite en cada push/PR a `main`/`develop`.

Aparte, `backend/loadtest/` (rev. 28, manual, fuera de CI) tiene una prueba de carga con Artillery contra un taller desechable sin credenciales Dataico (local y producción, ambas en verde con 0 fallos) y dos smoke-tests de un solo disparo que sí probaron la emisión real ante la DIAN (Caso 1 y Caso 2) — encontraron y confirmaron el fix de los primeros bugs reales de Dataico nunca antes ejercitados contra su API real.

Detalle de qué cubre cada suite, configuración de CI, y la prueba de carga en **[TESTING.md](TESTING.md)**.

---

## Variables de Entorno

`backend/.env` requiere credenciales de Supabase, JWT (taller y admin, separados), AWS Textract, WhatsApp Cloud API, Dataico y los tokens de verificación de webhooks (Bold/Addi/Mercado Pago) — **requeridos para confirmar pagos**: los webhooks son fail-closed y rechazan notificaciones si el token no está configurado (ver [SECURITY.md](Arquitectura%20y%20Sistema%20Core/SECURITY.md#webhooks-de-pago)).

Plantilla completa y notas de seguridad en **[ARCHITECTURE.md](Arquitectura%20y%20Sistema%20Core/ARCHITECTURE.md#variables-de-entorno)**.

---

## 📬 Contacto
efiscosas@gmail.com

Desarrollado y mantenido por un solo desarrollador.

---

**Efisco ERP** — *Impulsando la ingeniería automotriz a través de software de alto rendimiento.*
