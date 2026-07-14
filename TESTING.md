# Testing

> Ver también: [README](README.md) · [FINANCIAL_ENGINE](FINANCIAL_ENGINE.md) · [SECURITY](SECURITY.md)

`backend/tests/` — Jest con `--experimental-vm-modules` (ESM nativo, sin transpilar). 35 suites / 175 tests, todos contra lógica pura o con Supabase/auth mockeados (`jest.unstable_mockModule`), sin tocar la base real:

| Suite | Cubre |
|:---|:---|
| `financialEngine.test.js` | Liquidación de servicios, compras, retenciones (incl. gates por régimen fiscal del taller), comisiones de pasarela, salud financiera global |
| `pricing.test.js` | Márgenes básico/premium por tier |
| `vehicleClassifier.test.js` | Clasificación de vehículos por catálogo de marcas |
| `validators.test.js` | Validaciones de entrada de finanzas/configuración |
| `ocr.test.js` | Parsing de resultados de AWS Textract |
| `inventory.integration.test.js` | Flujo de consumo de inventario con Supabase mockeado |
| `inventoryMetrics.test.js` | Fórmula pura de rotación/`min_stock_vital` (Fase 3), incl. el cap de rotación en 20x |
| `dataico.service.test.js` | Bloque `retentions` (RET_FUENTE/RET_ICA) de la factura electrónica Dataico |
| `mechanics.integration.test.js` | Comisión de mecánicos sobre utilidad combinada inventario+servicio |
| `admin.accessKeyRedeem.test.js` | Canje de la llave de acceso a cuenta del cliente — emite un token de soporte real verificado con `jwt.verify` |
| `supportSession.middleware.test.js` | El token de sesión de soporte contra `auth.middleware.js` REAL (sin mockear) — incl. que no se salta los gates de taller suspendido/contrato B2B |
| `admin.softwareInvoice.test.js` | Factura de EFISCO como proveedor real: auto-creación del proveedor, liquidación real vía `financialEngine` (retenciones/IVA/GMF), reuso del proveedor en facturas siguientes (no duplica), gate de Régimen Simple, y protección 403 de `deleteProvider` sobre `is_system_provider` |
| `auth.signContract.test.js` | Firma electrónica del Contrato de Afiliación B2B |
| `auth.contractGate.test.js` | Gate de "contrato no firmado" en `auth.middleware.js` (middleware real, sin mockear) |
| `auth.staffTermsGate.test.js` | Gate de "autorización del taller no aceptada" para empleados en `auth.middleware.js` (middleware real, sin mockear) |
| `auth.acceptStaffTerms.test.js` | Aceptación de la autorización del taller: registra prueba legal (hora+IP) y marca `terms_accepted_at` |
| `providerEfficiency.test.js` | Fórmula pura de eficiencia de proveedor (Fase 3): promedio de últimas 5 entregas, descarte de datos inválidos |
| `providers.systemProvider.test.js` | El proveedor EFISCO (`is_system_provider`) responde 403 al intentar editarlo; un proveedor normal sí se puede editar |
| `admin.cobros.test.js` | Panel Cobros: plantilla editable del proveedor EFISCO (lectura/escritura), sincronización de cada cambio hacia la fila EFISCO de todos los talleres, `uploadSoftwareInvoice` usando los valores de la plantilla (no literales hardcodeados) al crear un proveedor nuevo, listado de facturas enriquecido con taller+`pdf_url` firmado, y marcar-pagada actualizando estado y reactivando el taller |
| `confidenceMatrix.test.js` | Fase 4: matriz de confianza estadística por celda (vehículo × tier), con meta de n≥10 registros por celda para rigurosidad estadística |
| `mercadopago.service.test.js` | Checkout Pro de Mercado Pago: armado del payload de preferencia (`notification_url`/`back_urls`), y `fetchPayment` consultando el estado real del pago en la API de MP (nunca confía en el body del webhook) |
| `billing.mercadopagoWebhook.test.js` | Confirmación real de pago de suscripción vía webhook de Mercado Pago |
| `billing.settleOrder.pendingServiceFees.test.js` | Modelo de cobro por orden liquidada: `settleOrder` acumula `software_valuation_unit` en `workshop_config.pending_service_fees` en vez de depender de un ciclo mensual fijo |
| `billing.settleOrder.pucAccounts.test.js` | Cuentas PUC dedicadas al liquidar: mano de obra/repuestos van a `puc_income_code`/`puc_parts_income_code` por separado; retenciones a cuenta de anticipo vs. cuenta de pasarela según el canal de pago |
| `providers.registerPurchase.pucAccounts.test.js` | ReteFuente practicada a un proveedor con su propia línea en el ledger (cuenta PUC declarante vs. no declarante), en vez de quedar neteada dentro del monto total de la compra |
| `subscriptionBilling.referralCommission.test.js` | Comisión recurrente del 15% al referidor: se calcula sobre lo que el taller referido realmente pagó ese ciclo, no sobre el valor de la propia suscripción del referidor |
| `subscriptionBilling.referralDiscount.test.js` | Descuento a la factura propia del referidor calculado en vivo según el conteo actual de referidos activos, reemplazando el acumulado histórico por orden de bahía |
| `clients.publicRegister.legalConsent.test.js` | Doble consentimiento legal en el registro público del cliente: acepta, rechaza y llamada sin consentimiento — los tres casos quedan registrados en `client_legal_acceptances` |
| `dataico.efiscoAccounting.test.js` | Dataico Caso 2: `buildSupportDocNoResidente`/`buildSupportDocBasico` comparados campo a campo contra los payloads reales de Postman; `emitSupportDocument`/`emitEfiscoInvoice` con numbering/credenciales propios de EFISCO |
| `admin.payoutSupportDoc.test.js` | Al marcar un pago de comisión como pagado, se dispara un Documento Soporte Básico a Dataico (sku "pago referido") de forma no bloqueante |
| `subscriptionBilling.workshopExpenseMirror.test.js` | Al emitir "Suscripción taller", se espeja automáticamente el gasto en la contabilidad del propio taller con el mismo monto ya descontado (mismo proveedor EFISCO lazy-creado, mismo motor de liquidación), con y sin GMF, y sin bloquear la facturación a EFISCO si el espejo falla |
| `workshop.updateWorkshop.roleGate.test.js` | Los campos fiscales/contables/PUC/Dataico/identidad legal de `updateWorkshop` son exclusivos del rol `contador` — ni el dueño ni un empleado con otro rol pueden modificarlos vía API, solo leerlos |

`npm test` (dentro de `backend/`) corre toda la suite. `backend/test.mjs` (si existe) es un script de verificación manual, no una suite de Jest — genera PDFs de ejemplo con datos falsos para revisión visual; Jest lo detecta y lo reporta como "suite fallida" al no encontrar tests dentro, pero no afecta a las 175 pruebas reales.

## CI

**GitHub Actions (`.github/workflows/ci-backend.yml`)**: corre esta misma suite en cada push/PR a `main`/`develop`, con Node 22 y pnpm 10.34.5 (mismas versiones que usa el `Dockerfile` de producción — mantenerlas alineadas evita mismatches de lockfile/runtime entre CI y el build real). Requiere los secrets `SUPABASE_URL`, `SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_ROLE_KEY`, `JWT_SECRET` y `ADMIN_JWT_SECRET` configurados en el repo — el resto de integraciones (AWS, WhatsApp, Dataico, Mercado Pago, Bold/Addi) están mockeadas en los tests y no hacen falta en CI.
