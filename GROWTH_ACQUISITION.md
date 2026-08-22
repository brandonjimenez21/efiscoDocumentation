# Adquisición y Crecimiento — EFISCO → Taller

> Ver también: [README](../README.md) · [ARCHITECTURE](../Arquitectura%20y%20Sistema%20Core/ARCHITECTURE.md) · [SECURITY](../Arquitectura%20y%20Sistema%20Core/SECURITY.md) · [API](../Arquitectura%20y%20Sistema%20Core/API.md) · [BUSINESS_RULES](../Reglas%20de%20Negocio%20y%20Finanzas/BUSINESS_RULES.md) · [OPERATIONS](../Reglas%20de%20Negocio%20y%20Finanzas/OPERATIONS.md) · [INVENTORY](../Reglas%20de%20Negocio%20y%20Finanzas/INVENTORY.md) · [FINANCE](../Reglas%20de%20Negocio%20y%20Finanzas/FINANCE.md) · [FINANCIAL_ENGINE](../Reglas%20de%20Negocio%20y%20Finanzas/FINANCIAL_ENGINE.md) · [BILLING](../Reglas%20de%20Negocio%20y%20Finanzas/BILLING.md) · [PRICING_SALES](PRICING_SALES.md) · [MONITORING](../MONITORING.md) · [TESTING](../TESTING.md)

Cómo EFISCO capta, convierte y hace crecer su base de talleres clientes. Para cómo el **taller** vende y tarifica a **sus propios clientes finales**, ver [PRICING_SALES.md](PRICING_SALES.md). Para el detalle técnico del cobro de EFISCO al taller y del árbol de referidos, ver [BILLING.md](../Reglas%20de%20Negocio%20y%20Finanzas/BILLING.md).

---

## Embudo público

`Landing.jsx` (`/`) es la puerta de entrada sin sesión — presenta las funciones del producto (Bahía en tiempo real, Proveedores con IA, Inventario que entiende el taller, Facturación electrónica DIAN, Flujo de caja real, Nómina flexible, Punto de equilibrio, Red de confianza, Referidos entre talleres) y dos CTA que reaparecen en toda la página: **"Solicitar Acceso"**.

Tres páginas públicas alimentan la decisión antes de pedir acceso:

1. **`/demo`** (rev. 50) — ya no es una demo estática con cifras inventadas: son 5 videos reales (Recepción → Bahía → Facturación → Flujo de Caja → Equilibrio) grabados sobre una cuenta de taller de prueba real operando el ciclo completo de una orden. Enseñar el producto funcionando de verdad, no una maqueta, es la pieza central de la conversión — ver el pipeline de grabación en [BUSINESS_RULES.md — Demo pública](../Reglas%20de%20Negocio%20y%20Finanzas/BUSINESS_RULES.md#demo-pública-demo-rev-50).
2. **`/tarifas`** — el modelo de precio se muestra completo y sin letra pequeña antes de registrarse: pay-per-uso, sin plan mensual fijo, sin costo de entrada. Ver [Modelo de precio](#modelo-de-precio-pay-per-uso) abajo.
3. **`/reportes-financieros`** — Relación con Inversionistas: reportes trimestrales/anuales de EFISCO S.A.S. en PDF, públicos y sin sesión. No es un canal de adquisición de talleres, pero es una señal de transparencia/solidez que refuerza la confianza de quien está evaluando registrarse (mismo criterio de apertura que `/tarifas` y `/demo`) — ver [BILLING.md — Reportes Financieros](../Reglas%20de%20Negocio%20y%20Finanzas/BILLING.md#reportes-financieros-adminreportes-financieros).

---

## Modelo de precio (pay-per-uso)

Sin plan mensual fijo ni tarjeta de crédito para empezar — el taller solo paga por lo que efectivamente usa, con dos tarifas escalonadas por volumen:

| Concepto | Tarifa | Detalle |
|:---|:---:|:---|
| Orden liquidada | $535 COP (IVA incluido) | Hasta 119 órdenes/mes |
| Orden liquidada (alto volumen) | $416 COP (IVA incluido) | Desde la orden 120/mes; el cambio de tarifa es automático a mitad de mes |
| Repuesto de Venta Directa (mostrador) | $150 COP/unidad | Solo si el repuesto NO va dentro de una orden (ya cubierta por la tarifa de orden) |
| Consulta de puntaje crediticio | $100 COP | Solo si el cliente tiene ≥30 días de historial (`verifyScoreOTP`) |

Es un modelo deliberadamente sin fricción de entrada: el costo variable escala con el uso real del taller (más órdenes = más ingreso para EFISCO, no un plan que penaliza a un taller pequeño ni deja dinero sobre la mesa con uno grande). Detalle técnico completo (cómo se acumulan y se facturan estos tres cargos) en [BILLING.md — Dashboard](../Reglas%20de%20Negocio%20y%20Finanzas/BILLING.md#dashboard) y [BILLING.md — Venta Directa](../Reglas%20de%20Negocio%20y%20Finanzas/BILLING.md#venta-directa--tarifa-por-repuesto-2026-08-05).

---

## Del clic a taller operando

1. **Solicitar Acceso** (`/login`, formulario de registro): copy corregido 2026-07-25 para dejar explícito desde la Landing que se trata de una *solicitud*, no de una activación inmediata — "Solicita tu acceso gratis" en vez de "Regístrate gratis". Al enviar, un modal centrado (reemplaza un toast que desaparecía solo) confirma que EFISCO se pondrá en contacto para verificar el taller.
2. **Verificación manual** — no hay ningún gate técnico que bloquee el registro (`register` crea el `workshop_config` con `is_active: true` por default); es un proceso humano del equipo EFISCO antes de considerar al taller operativo de verdad. Ver [BUSINESS_RULES.md — Registro de un taller nuevo](../Reglas%20de%20Negocio%20y%20Finanzas/BUSINESS_RULES.md#registro-de-un-taller-nuevo-login).
3. **Gate de Configuración Inicial** (`SetupGate.jsx`) — antes de operar (ni Recepción, ni Bahía, ni Inventario), el taller debe completar 3 mínimos: datos del taller, al menos 1 empleado, al menos 1 servicio en el catálogo. Funciona como un onboarding forzoso: nadie llega a facturar sin haber configurado lo mínimo para que la liquidación cuadre. Ver [BUSINESS_RULES.md — SetupGate](../Reglas%20de%20Negocio%20y%20Finanzas/BUSINESS_RULES.md#gate-de-configuración-inicial-setupgate).
4. **Contrato de Afiliación B2B** — firma digital con prueba legal append-only (hash, IP, versión), visible después desde `/config` (dueño) y desde el detalle del taller en `/admin` (EFISCO) — ver [SECURITY.md](../Arquitectura%20y%20Sistema%20Core/SECURITY.md).

---

## Referidos: el motor de crecimiento boca a boca

El canal de adquisición más importante después del embudo público es el propio taller ya activo — un sistema de referidos entre talleres con descuento creciente sobre su propia cuota, y un vínculo jerárquico de una sola vía por diseño:

| Suscripciones referidas | Descuento aplicado |
|:---:|:---|
| 1 | 33% sobre cuota mensual |
| 2 | 66% sobre cuota mensual |
| 3+ | 100% (mes gratis) |
| Platino (>5) | 15% comisión directa (pagada por EFISCO) |

- **"Quién te refirió" es de una sola vez y sin ciclos** — un taller no puede vincularse con otro que, directa o indirectamente, ya fue referido por él mismo (raíz única para el árbol jerárquico de `/admin/referidos`).
- El nivel propio del taller sube por cuántos OTROS talleres usan su código, sin límite — el árbol crece hacia abajo sin tope, la vinculación hacia arriba es la que está bloqueada.
- El dashboard del dueño distingue "Talleres que Referiste" (los que usaron tu código) de "¿Alguien te recomendó?" (la relación inversa) — dos direcciones del mismo grafo, nunca mezcladas en la UI.
- El pago de la comisión Platino se gestiona desde `/admin/pagos` (EFISCO → taller) con estado pendiente/en proceso/pagado — ver [BILLING.md — Pagos](../Reglas%20de%20Negocio%20y%20Finanzas/BILLING.md#pagos) y [BILLING.md — Referidos (árbol admin)](../Reglas%20de%20Negocio%20y%20Finanzas/BILLING.md#referidos-árbol-admin).

Detalle completo del sistema (banco de comisión, badge "Tu Estatus Actual", historial de bugs corregidos) en [BILLING.md — Referidos](../Reglas%20de%20Negocio%20y%20Finanzas/BILLING.md#referidos).

---

## Retención vía Panel Admin EFISCO

Una vez el taller está activo, la relación comercial se sostiene desde `/admin`: seguimiento de facturas (`/admin/cobros`), suspensión/reactivación ligada al pago (marcar una factura como pagada reactiva el taller en el mismo paso), y detección temprana de riesgo vía el bot de Telegram interno (alta de taller nuevo, pagos fallidos — ver [MONITORING.md](../MONITORING.md)). No hay downgrade ni cancelación de plan porque no hay plan: dejar de operar simplemente deja de generar cargos por orden.
