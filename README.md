# Efisco ERP — Automotive Workshop SaaS

> **Ingeniería de software de alta precisión aplicada a la rentabilidad y automatización del sector automotriz.**

Efisco es una plataforma SaaS diseñada para transformar talleres mecánicos en centros operativos inteligentes. Integra un **Motor Financiero (Fiscal/Contable)** adaptado a la normativa colombiana 2026, un **Clasificador de Vehículos** para tarificación dinámica, **OCR con IA** para el control de egresos, y un **Panel de Administración Interno** para la gestión del ecosistema de talleres.

![React 19](https://img.shields.io/badge/React-19-61DAFB?style=for-the-badge&logo=react)
![Express 5](https://img.shields.io/badge/Express-5-000000?style=for-the-badge&logo=express)
![Supabase](https://img.shields.io/badge/Supabase-PostgreSQL-3ECF8E?style=for-the-badge&logo=supabase)
![Tailwind v4](https://img.shields.io/badge/Tailwind-v4-06B6D4?style=for-the-badge&logo=tailwind-css)
![AWS Textract](https://img.shields.io/badge/OCR-AWS_Textract-FF9900?style=for-the-badge&logo=amazonaws)
![Meta API](https://img.shields.io/badge/WhatsApp-Meta_Cloud_API-25D366?style=for-the-badge&logo=whatsapp)

---

## Documentación

Este README es un resumen general. El detalle técnico vive en documentos dedicados:

| Documento | Contenido |
|:---|:---|
| **[ARCHITECTURE.md](ARCHITECTURE.md)** | Mapa de componentes, ciclo de vida operativo end-to-end, stack tecnológico, decisiones técnicas clave, Kardex, variables de entorno |
| **[SECURITY.md](SECURITY.md)** | Aislamiento multi-tenant, autenticación admin separada, gates de suspensión/contrato B2B/autorización del taller, RLS de Supabase, webhooks de pago |
| **[TESTING.md](TESTING.md)** | Suite Jest (35 suites / 175 tests), qué cubre cada una, CI en GitHub Actions |
| **[BUSINESS_RULES.md](BUSINESS_RULES.md)** | Rutas del frontend y reglas de negocio de cada módulo: Recepción, Bahías, Inventario, Proveedores, Equilibrio, Cobros, Flujo de Caja, Referidos, Configuración y Panel Admin EFISCO |
| **[FINANCIAL_ENGINE.md](FINANCIAL_ENGINE.md)** | `financialEngine.js`: liquidación de servicios, liquidación de compras, salud financiera global, tipos de movimiento del Libro Auxiliar |
| **[API.md](API.md)** | Referencia completa de endpoints REST (`/api/*` y `/api/admin/*`) |

---

## Descripción General

Efisco no es un ERP genérico adaptado al sector automotriz — fue construido desde cero para resolver los problemas reales de los talleres colombianos:

- **Tarificación dinámica** basada en clasificador de vehículos por segmento (básico / premium)
- **Liquidación fiscal precisa** según normativa colombiana 2026: IVA, ReteFuente, ReteICA, ReteIVA, GMF 4×1000
- **Control de egresos con OCR** — extracción automática de datos de facturas de proveedores vía AWS Textract
- **Facturación electrónica** integrada con Dataico/DIAN bajo sub-cuenta propia de cada taller (no bloqueante, guarda CUFE en base de datos)
- **Comunicación automatizada** con clientes vía WhatsApp Cloud API (Meta)
- **Multi-tenant** con aislamiento de datos por taller verificado a nivel de aplicación — cada endpoint valida `workshop_id` contra la sesión del usuario autenticado. RLS de Postgres es una capa adicional de defensa (ver [SECURITY.md](SECURITY.md)), no la principal
- **Ventas a crédito** con plan de cuotas: INC_GROSS se registra en $0 al liquidar y el ingreso real entra vía pagos de cuotas
- **Panel de equilibrio interactivo** con gráfica SVG hover, proyección what-if con cálculo fiel de margen
- **IVA por categoría de repuesto** — tasas configurables por las 10 categorías de inventario
- **Presets PUC colombianos** — 3 regímenes fiscales que auto-rellenan los 21 códigos contables
- **Panel Admin EFISCO** — gestión interna de talleres, comisiones, referidos y suspensión de acceso

---

## Arquitectura (resumen)

Arquitectura **multi-tenant** con aislamiento de datos verificado en cada endpoint del backend (`workshop_id` de la sesión) y un núcleo de cálculo financiero inmutable. El frontend (React 19 + Vite) habla con un backend Express 5 dividido en tres módulos — Operativo, Logístico y Financiero — sobre una base Supabase/PostgreSQL. El panel `/admin` de EFISCO corre en paralelo con autenticación y JWT completamente separados. Diagramas de componentes, el ciclo de vida operativo end-to-end (recepción → bahía → liquidación → factura) y las decisiones técnicas clave (ledger inmutable, pipeline OCR asíncrono, margen what-if fiel, gotchas de PDF/mobile) están en **[ARCHITECTURE.md](ARCHITECTURE.md)**.

**Stack**: React 19 · Express 5 · Supabase (PostgreSQL) · Tailwind v4 · AWS Textract (OCR) · Meta WhatsApp Cloud API · Puppeteer (PDF) · Dataico (DIAN) · Bold/Addi (pasarelas).

---

## Módulos del Sistema (resumen)

El dueño del taller opera desde `/dashboard`, `/recepcion`, `/bahia`, `/inventario`, `/proveedores`, `/config`, y paneles financieros (`/finanzas`, `/equilibrio`, `/cobros`, `/flujo-caja`). El cliente final interactúa vía un link público de WhatsApp (`/cliente/registro/:workshopId/:intakeId`) para completar su registro y ver su score crediticio. El contador tiene un panel propio (`/contador`) para los campos fiscales/PUC/Dataico que el dueño solo puede leer.

Flujo típico: **Recepción** (ingreso + score crediticio) → **Bahía** (ejecución, consumo de inventario con Kardex inmutable, clasificación de tier) → **Liquidación** (motor financiero + factura Dataico + comprobante por WhatsApp) → **Cobros/Flujo de Caja** (si fue a crédito).

Reglas de negocio completas de cada módulo, tabla de rutas con niveles de acceso, y el Panel de Administración EFISCO (talleres, cobros SaaS, pagos de comisión, referidos, contabilidad interna) en **[BUSINESS_RULES.md](BUSINESS_RULES.md)**.

---

## Motor Financiero (resumen)

`backend/utils/financialEngine.js` es el núcleo de cálculo inmutable: liquida servicios (mano de obra + repuestos, IVA, retenciones si el cliente es agente retenedor, comisión de pasarela Bold/Addi), liquida compras a proveedor (retenciones según régimen tributario, GMF 4×1000), y calcula la salud financiera global (`bankBalance`, `ivaLiability`, `realBankBalance`). Constantes 2026: UVT = $50.318, umbral de retenciones = 27 UVT ≈ $1.358.586. Cada movimiento queda registrado en el **Libro Auxiliar** (`cash_flow_ledger`), un ledger append-only con 15 tipos de movimiento (`INC_GROSS`, `TAX_IVA`, `RET_FUENT`, `GW_FEE`, etc.).

Fórmulas completas, matriz de decisión de liquidación y la tabla de tipos de movimiento en **[FINANCIAL_ENGINE.md](FINANCIAL_ENGINE.md)**.

---

## Seguridad (resumen)

La defensa principal contra acceso cruzado entre talleres es **a nivel de aplicación**: cada endpoint valida `workshop_id`/rol contra la sesión antes de tocar la base. RLS de Postgres (activado en las 28 tablas públicas, deny-by-default) es una capa adicional de defensa en profundidad, no el mecanismo principal — el backend siempre usa `service_role`, que ignora RLS. El panel admin EFISCO tiene JWT y credenciales completamente separados de los talleres. Existen gates en dos capas (frontend + middleware) para suspensión de talleres, firma del Contrato B2B, y autorización del taller por parte de empleados, cada uno con su propia prueba legal append-only.

Estado auditado del Security Advisor de Supabase, autenticación admin, llaves de acceso de soporte y la nota de seguridad sobre webhooks de pago en **[SECURITY.md](SECURITY.md)**.

---

## Testing (resumen)

`backend/tests/` — Jest con ESM nativo (`--experimental-vm-modules`): **35 suites / 175 tests**, todos contra lógica pura o con Supabase/auth mockeados, sin tocar la base real. Cubre el motor financiero, clasificador de vehículos, OCR, inventario, comisiones de mecánicos, gates de seguridad (contrato B2B, autorización del taller, sesión de soporte), facturación Dataico (incl. Caso 2 — EFISCO como emisor), y Mercado Pago. CI en GitHub Actions corre la suite en cada push/PR a `main`/`develop`.

Detalle de qué cubre cada suite y configuración de CI en **[TESTING.md](TESTING.md)**.

---

## Variables de Entorno

`backend/.env` requiere credenciales de Supabase, JWT (taller y admin, separados), AWS Textract, WhatsApp Cloud API, Dataico y (opcional pero recomendado) tokens de verificación de webhooks Bold/Addi.

Plantilla completa y notas de seguridad en **[ARCHITECTURE.md](ARCHITECTURE.md#variables-de-entorno)**.

---

## 📬 Contacto
efiscosas@gmail.com

Desarrollado y mantenido por un solo desarrollador.

---

**Efisco ERP** — *Impulsando la ingeniería automotriz a través de software de alto rendimiento.*
