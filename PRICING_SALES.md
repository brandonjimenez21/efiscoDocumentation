# Tarificación y Ventas — Taller → Cliente Final

> Ver también: [README](../README.md) · [ARCHITECTURE](../Arquitectura%20y%20Sistema%20Core/ARCHITECTURE.md) · [SECURITY](../Arquitectura%20y%20Sistema%20Core/SECURITY.md) · [API](../Arquitectura%20y%20Sistema%20Core/API.md) · [BUSINESS_RULES](../Reglas%20de%20Negocio%20y%20Finanzas/BUSINESS_RULES.md) · [OPERATIONS](../Reglas%20de%20Negocio%20y%20Finanzas/OPERATIONS.md) · [INVENTORY](../Reglas%20de%20Negocio%20y%20Finanzas/INVENTORY.md) · [FINANCE](../Reglas%20de%20Negocio%20y%20Finanzas/FINANCE.md) · [FINANCIAL_ENGINE](../Reglas%20de%20Negocio%20y%20Finanzas/FINANCIAL_ENGINE.md) · [BILLING](../Reglas%20de%20Negocio%20y%20Finanzas/BILLING.md) · [GROWTH_ACQUISITION](GROWTH_ACQUISITION.md) · [MONITORING](../MONITORING.md) · [TESTING](../TESTING.md)

Cómo EFISCO le da al **taller** las herramientas para tarificar y cerrar ventas con **sus propios clientes** (dueños de vehículos). Para cómo EFISCO adquiere y cobra a los talleres, ver [GROWTH_ACQUISITION.md](GROWTH_ACQUISITION.md). El detalle de fórmulas y asientos contables de todo lo aquí descrito vive en [FINANCIAL_ENGINE.md](../Reglas%20de%20Negocio%20y%20Finanzas/FINANCIAL_ENGINE.md).

---

## Tarificación dinámica por vehículo

`vehicleClassifier.js` asigna un tier de servicio (**Premium** / **Básico**) a cada vehículo, a partir de un catálogo de marcas (`backend/utils/marcas_vehiculos.json`, carros y motos por separado, cada uno con su `gama`: Alta/Normal) y tres reglas en cascada:

1. Gama de marca **Alta** → Premium
2. Gama **Normal** + (antigüedad > 10 años **o** complejidad técnica Alta) → Premium
3. Cualquier otro caso → Básico

El tier resultante decide qué margen (`base_margin_basic` / `base_margin_premium`, configurados por servicio en el Catálogo) se aplica al liquidar — así un mismo servicio (ej. cambio de aceite) puede cotizar distinto según si el vehículo es un premium de gama alta o un básico corriente, sin que el taller tenga que mantener dos catálogos de precios paralelos. `assignServiceTier` se llama desde `createWorkOrder`/`updateWorkOrder` (`workOrders.controller.js`) — es el tier **real** que termina afectando el margen cobrado, con match exacto (no parcial) de marca contra el catálogo, cacheado en memoria del proceso tras la primera lectura del JSON (`_brandCatalogCache` — un cambio al archivo no se refleja hasta reiniciar el backend).

- **Catálogo ampliado con investigación de mercado colombiano/Cali (2026-09-10, pedido explícito del usuario)**: el catálogo original solo tenía 5 marcas de carro en gama Alta (BMW, Mercedes-Benz, Audi, Porsche, Volvo) y 5 de moto — mucho más corto que el listado de marcas premium que ya usaban `Bahia.jsx`/`Cotizaciones.jsx` para el **hint en pantalla** ("Tier estimado") mientras se arma la orden/cotización, una lógica duplicada a propósito en el frontend (ver [OPERATIONS.md](../Reglas%20de%20Negocio%20y%20Finanzas/OPERATIONS.md)) que nunca estuvo sincronizada con este catálogo real del backend — **bug real encontrado en esta sesión** (no introducido por ella, ya existía; se hizo más grande al ampliar solo el frontend en el primer intento): un cajero podía ver "Premium" en la vista previa de una marca que, al crear la orden de verdad, el backend seguía clasificando como Básico por no estar en este JSON, cobrando un margen distinto del que se le mostró. Corregido sincronizando ambos lados con la misma lista, basada en las 17 marcas premium registradas por ANDEMOS (Asociación Nacional de Movilidad Sostenible) en Colombia, menos Ferrari/Lamborghini/Rivian (van a su propio servicio de marca, no a un taller multimarca) más las de moto de alta gama con presencia real en Cali (concesionario tipo Potenza): **Carros Alta** — BMW, Mercedes-Benz, Audi, Porsche, Volvo, Land Rover, Jaguar, Lexus, Mini, Cupra, Tesla, DS, Cadillac, Infiniti. **Motos Alta** — Ducati, BMW Motorrad, Harley-Davidson, Triumph, MV Agusta, Aprilia, Indian, Royal Enfield. **KTM se sacó de los dos lados** (quedó en Alta en el backend original) — pedido explícito del usuario: tiene tanto modelos de gama baja/comuter comunes en Cali como modelos premium, etiquetar la marca entera como Premium metía ruido. AKT/Yamaha/Honda/Suzuki/Bajaj quedan fuera a propósito — dominan el mercado colombiano de motos con modelos accesibles.

## Margen de mano de obra por Gama/Complejidad (rev. 49)

Al crear un servicio, el taller ya no elige el margen a ciegas: clasifica el servicio por **Gama** (Alta/Baja) y **Complejidad** (Alta/Baja) y el formulario sugiere el margen según la tabla oficial del documento fuente del producto:

| Tier | Gama | Complejidad | Rango | Sugerido |
|:---|:---:|:---:|:---:|:---:|
| Premium | Alta | Alta | 60%–45% | 50% |
| Premium | Baja | Alta | 60%–45% | 50% |
| Básico  | Alta | Baja | 60%–45% | 50% |
| Básico  | Baja | Baja/Media | 55%–40% | 45% |

El número sugerido es editable — el taller conserva el precio final bajo su propio criterio comercial, la tabla solo evita tener que memorizarla cada vez. No confundir esta "gama/complejidad" (metadata estática por servicio del catálogo) con la gama de marca del vehículo ni con la complejidad de la orden (`service_complexity`, elegida en Bahía) — son tres conceptos distintos que reusan las mismas palabras. Detalle completo en [FINANCIAL_ENGINE.md — Margen de mano de obra](../Reglas%20de%20Negocio%20y%20Finanzas/FINANCIAL_ENGINE.md#margen-de-mano-de-obra-por-gamacomplejidad-rev-49) y en [BUSINESS_RULES.md — Catálogo de Servicios](../Reglas%20de%20Negocio%20y%20Finanzas/BUSINESS_RULES.md#configuración-del-taller-config).

---

## Descuento por regateo

Herramienta de cierre de venta para el caso típico de negociación de último momento en un taller mecánico: un descuento comercial NO condicionado, en pesos COP, editable directamente al liquidar (`ModalLiquidacion.jsx`, default $0). Reduce la base gravable de toda la factura (IVA y retenciones incluidos, según normativa DIAN para descuentos no condicionados) y se prorratea entre mano de obra y cada repuesto antes de calcular su IVA individual, para que un repuesto a tasa distinta (0%/5%/19%) siga cobrando el impuesto real sobre su porción ya descontada. Queda registrado como un asiento contable propio (`SALES_DISCOUNT`, contra-ingreso), no como un simple recorte silencioso del precio — el taller conserva la trazabilidad de cuánto descontó y en qué orden. Detalle completo en [FINANCIAL_ENGINE.md — Descuento por regateo](../Reglas%20de%20Negocio%20y%20Finanzas/FINANCIAL_ENGINE.md#descuento-por-regateo-feature-2026-08-16-pedido-explícito-del-usuario).

---

## Ventas a crédito (plan de cuotas)

El taller puede cerrar una venta sin cobrar de contado, dividiéndola en cuotas (`/cobros`, Panel de Cobros):

- El ingreso se reconoce **completo y sin IVA** al liquidar la orden (`INC_GROSS` en $0 en caja — el crédito queda como cuenta por cobrar); el ingreso de caja real entra después, cuota por cuota, vía `payInstallment`.
- Cada pago de cuota genera un comprobante (PDF con monto, "cuota X de Y" y saldo pendiente) enviado por la misma cascada de mensajería que el resto del sistema (WhatsApp → Telegram → Email, ver [ARCHITECTURE.md](../Arquitectura%20y%20Sistema%20Core/ARCHITECTURE.md#variables-de-entorno)) — el cajero ve por cuál canal llegó.
- "Cuota vencida" se evalúa en hora de Bogotá, no UTC, y se muestra tanto en el Panel de Cobros como en un badge de alerta en el Dashboard/Sidebar.
- El motor evita duplicar el ingreso operativo entre la liquidación original y el cobro de cada cuota — ver [FINANCE.md — Panel de Cobros](../Reglas%20de%20Negocio%20y%20Finanzas/FINANCE.md#panel-de-cobros-cobros).

Vender a crédito es, en la práctica, la palanca comercial del taller para no perder una venta por falta de liquidez del cliente final, sin que EFISCO le exija ningún tipo de evaluación de riesgo previa más allá del [Registro Seguro EFISCO / score crediticio](../Reglas%20de%20Negocio%20y%20Finanzas/OPERATIONS.md#recepción) opcional en Recepción.

---

## Pasarelas de pago

Tasas visibles y editables por el propio dueño desde `/config` (sin necesitar al contador) — parte de la propuesta de valor comercial: el taller sabe exactamente cuánto le descuenta cada medio de pago antes de ofrecérselo al cliente.

| Pasarela | Modalidad | Tarifa |
|:---|:---|:---:|
| Bold | Presencial, tarjeta nacional | 2.99% + $300 |
| Bold | Presencial, tarjeta internacional | 3.99% + $300 |
| Bold | Presencial, QR/billetera | 1.50% |
| Bold | Online, tarjeta nacional | 3.49% + $900 |
| Bold | Online, tarjeta internacional | 4.49% + $900 |
| Bold | Online, QR/billetera | 1.50% |
| Addi | — | 10.5% (configurable) |

Un taller en Régimen Ordinario sufre además una retención propia de la pasarela sobre el giro (ReteRenta 1.5% + ReteIVA 15% + ReteICA), independiente de si el cliente final es agente retenedor — ver la matriz completa en [FINANCIAL_ENGINE.md — Retención de la Pasarela](../Reglas%20de%20Negocio%20y%20Finanzas/FINANCIAL_ENGINE.md#retención-de-la-pasarela-sobre-el-giro-al-taller-rev-38).

---

## Venta Directa (mostrador)

Canal de venta de repuestos sin orden de servicio asociada (`inventory.controller.js:sellDirect`) — el taller también vende como mostrador de repuestos, no solo como taller de servicio. Cada unidad vendida así genera $150 COP de tarifa para EFISCO (a propósito, no aplica a un repuesto ya facturado dentro de una orden — ver [GROWTH_ACQUISITION.md — Modelo de precio](GROWTH_ACQUISITION.md#modelo-de-precio-pay-per-uso)), pero es un canal de ingreso adicional real para el taller sin fricción de crear una orden completa.
