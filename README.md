# EFISCO: documentación técnica y de negocio

**EFISCO** es una plataforma SaaS (ERP) para talleres mecánicos y de bicicletas en Colombia: órdenes de trabajo en bahía, inventario con costo promedio, facturación y retenciones según la normativa colombiana, flujo de caja con partida doble y un puntaje de comportamiento comercial (EFISCO Score) que conecta a talleres, clientes y distribuidoras.

Este repositorio contiene la **documentación** del proyecto, pensada para quien quiera entender cómo está construido y qué decisiones hay detrás. El código vive en repositorios privados.

> Es una versión pública derivada de la documentación interna completa. Se omiten a propósito la configuración de infraestructura, las rutas de administración y el detalle operativo de seguridad.

## Qué hay aquí

| Carpeta | Contenido |
|:---|:---|
| [`arquitectura/`](./arquitectura/architecture-overview.md) | Mapa de componentes, ciclo de vida de una orden de punta a punta, stack, decisiones técnicas y lecciones aprendidas |
| [`negocio/`](./negocio/) | Recepción y cotizaciones, bahías y órdenes de trabajo, inventario y proveedores, configuración y precios del taller, disputas de pago |
| [`puntaje/`](./puntaje/) | EFISCO Score (escala 50 a 250), Score B2B para distribuidoras y el portal de distribuidoras |
| [`finanzas/`](./finanzas/) | Motor financiero (IVA, retenciones, GMF), Libro Mayor, notas contables y devoluciones, reservas de impuestos, tableros |
| [`comercial/`](./comercial/growth-and-subscription-billing.md) | Modelo de negocio: captación, cobro por uso, comisiones y panel de administración |
| [`ingenieria/`](./ingenieria/testing-and-monitoring.md) | Suite de pruebas, integración continua y monitoreo |
| [`SEGURIDAD.md`](./SEGURIDAD.md) | Resumen de los controles de seguridad del sistema |

## Lo más interesante de revisar

- **Motor financiero** ([`finanzas/financial-engine-settlement.md`](./finanzas/financial-engine-settlement.md)): liquidación de servicios y compras con IVA, Retefuente, ReteIVA, ReteICA y GMF 4x1000, con una matriz de retenciones taller por proveedor y partida doble verificada.
- **EFISCO Score** ([`puntaje/efisco-score-matriz-requisitos.md`](./puntaje/efisco-score-matriz-requisitos.md)): puntaje de 50 a 250 sobre una ventana móvil de 90 días, con pilares de pago, estabilidad y fidelidad.
- **Score B2B y portal de distribuidoras** ([`puntaje/b2b-score.md`](./puntaje/b2b-score.md), [`puntaje/b2b-distributor-portal.md`](./puntaje/b2b-distributor-portal.md)): riesgo comercial del taller para almacenes de repuestos, compras de la red con costo promedio ponderado y retenciones por perfil tributario.
- **Notas contables y devoluciones** ([`finanzas/adjustments-and-accounting-notes.md`](./finanzas/adjustments-and-accounting-notes.md)): ajustes a comprobantes de venta, bolsas de gasto operativo y libro mayor por cuenta.
- **Calidad** ([`ingenieria/testing-and-monitoring.md`](./ingenieria/testing-and-monitoring.md)): más de 1.700 pruebas automáticas en más de 220 suites.

## Stack

| Capa | Tecnología |
|:---|:---|
| Frontend | React 19, Vite, Tailwind CSS 4, Zustand |
| Backend | Node.js (ESM), Express 5 |
| Datos | Supabase (PostgreSQL) con seguridad a nivel de fila |
| Documentos | Puppeteer (PDF), AWS Textract (OCR de facturas) |
| Integraciones | WhatsApp (Meta Cloud API), Telegram, Resend (correo), Dataico (facturación electrónica DIAN), Google Calendar |
| Calidad | Jest, GitHub Actions |

## Autor

Brandon Jiménez ([@brandonjimenez21](https://github.com/brandonjimenez21)).
