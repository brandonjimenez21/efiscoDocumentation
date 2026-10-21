# Portal de Distribuidoras EFISCO B2B y compras de la red

Estado: backend, frontend y pruebas listos (2026-10-08). **Falta correr la migración** `backend/scripts/migrations/2026-10-08_b2b_distributor_portal.sql` (la corre el usuario) y crear la primera distribuidora desde Admin > Distribuidoras. Sin la migración las rutas del portal responden error y el Score B2B del taller calcula con deuda 0.

Fuentes: documentos `Estructura legal EFISCO B2B`, `Estructura del Puntaje EFISCO 1 DT B2B` y `Estructura de la plataforma a la distribuidora B2B`. Motor del puntaje: [b2b-score.md](./b2b-score.md). Contratos: legal-consent-changes.md.

## Quién es quién

| Actor | Cuenta | Sesión |
|---|---|---|
| **Taller** | El de siempre. Aparece en el directorio solo si firmó el contrato **>= 1.1.0** (cláusulas B2B) | Sesión normal |

Una distribuidora **de prueba** (`is_test`) solo ve talleres de prueba y una real solo talleres reales. El directorio excluye sucursales, talleres inactivos y los que no tienen NIT.

## Flujo de una consulta de crédito (`POST /api/b2b/distribuidor/evaluar-taller`)

1. Sesión de distribuidora activa y contrato `v2.0_B2B` firmado (`requireDistribuidor` + `requireContratoVigente`; sin contrato, 403 `CONTRATO_PENDIENTE`).
2. **Límite de consultas** (`validarAccesoYRateLimit`, cuenta TODOS los intentos del registro, también los de confirmación equivocada): 10 por minuto (429) y, al llegar a 15 en 5 minutos, **bloqueo de 30 minutos** + alerta `ALERTA_PATRON_SOSPECHOSO` (security_events, Telegram a los administradores y auditoría inmutable).
3. El directorio no muestra puntajes. Al pulsar "Consultar crédito B2B" el modal exige digitar el **NIT o el teléfono** del taller (se comparan por dígitos; el NIT con o sin dígito de verificación). Si no coincide: 422 y el intento queda registrado.
4. Se corre el motor (`evaluateWorkshopB2B`) y se escribe el **registro inmutable** `log_consultas_b2b` ANTES de devolver nada (si no se puede escribir, no se entrega el dato). Un trigger rechaza UPDATE, DELETE y TRUNCATE. Código visible `LOG-AAAAMMDD-<seq de 5 dígitos>`.
5. Aviso asíncrono al taller (WhatsApp, Telegram o correo por la cascada de mensajería): "🛡️ EFISCO Radar: El almacén X acaba de consultar tu Score B2B (N pts - Nivel)... toca aquí para reportar" con enlace a `/score-b2b?reportar=<id>`.
6. **Sesión temporal de 20 minutos** (`sesiones_consulta_b2b`, `GET /distribuidor/sesion/:token`): se puede reabrir sin evaluar de nuevo; vencida, 410 y el frontend bloquea la pantalla y borra el puntaje de su estado.

La respuesta (`vista_distribuidor`) trae solo indicadores agregados: evaluación de riesgo, pilares 0 a 100, vectores y la cartera con otras distribuidoras de la red (alias, saldo, mora, estado). Nunca clientes finales, placas, márgenes ni tarifas (Cláusula Adicional B). El taller ve lo mismo en `/score-b2b`, con la lista de quién lo consultó y el botón **Reportar** (`reportes_consulta_b2b`, una vez por consulta).

## Compras de la red (Kardex y cuentas por pagar)

1. La distribuidora despacha (`POST /distribuidor/despachos`): valida ítems y recalcula los totales (si manda totales deben coincidir), es **idempotente por factura** (409 `FACTURA_DUPLICADA`) y queda `PENDIENTE_CONFIRMACION` en `compras_red_b2b`. No se contabiliza nada todavía.
2. El taller la ve en **Proveedores > Compras de la red** (insignia con los pendientes). "Confirmar y recibir repuestos" (solo dueño o administrador) hace, en este orden: proveedor (se crea o se reutiliza, enlazado por `providers.distribuidor_id`, con el perfil tributario VERIFICADO de la distribuidora y cuenta de inventario 1435), productos (se busca por `item_number` = SKU; si no existe se crea con stock 0 y el costo recibido), y la compra por el mismo camino de la compra manual (`purchaseRegistration.service.js`).
3. La confirmación es atómica en el reclamo (solo una puede pasar de PENDIENTE a PROCESANDO). Si falla ANTES de crear la compra vuelve a pendiente; si falla DESPUÉS queda PROCESANDO (código `COMPRA_A_MEDIAS`) para revisión: reintentar duplicaría la compra.
4. Contado: origen de fondos Banco/Nequi/Daviplata (con casilla GMF 4x1000) o Caja menor. Crédito: una cuenta por pagar (`supplier_installments`) al plazo del despacho, que alimenta el pilar 3 del Score B2B.
5. **Objeción de cartera** (Cláusula Adicional C): el taller objeta una compra confirmada; durante máximo 3 días hábiles (`utils/businessDays.js`, sin festivos) no cuenta como mora en su Score (sí en la deuda). La distribuidora ve el estado `EN_DISPUTA`.

## Verificación de la partida doble (pedida en el documento)

La compra de la red usa **exactamente** la contabilidad de `registerPurchaseCore` (antes dentro de `providers.controller.js:registerPurchase`, extraída sin cambiar su comportamiento): el libro valida que cada comprobante cuadre (`documentNumbering.assertBalanced`) y el Libro Mayor usa `gross_amount` (no `amount`, que incluye el IVA) como valor contable.

| Documento (cuenta) | Línea del libro | Cuenta PUC en la plataforma | Lado |
|---|---|---|---|
| 1435 Inventario / repuestos | `SUP_PAY` | `provider.puc_account_expense` (para proveedores de la red se fija a `puc_inventory_purchase_code`, 1435) | Débito, base |
| 2408 IVA descontable | `TAX_IVA_DEDUC` | `puc_iva_deductible_code` (240820) | Débito |
| 2365 Retención en la fuente | `RET_FUENT_COMPRA` | 236540 (declarante / no declarante, configurable) | Crédito |
| 2368 ReteICA | `RET_ICA_COMPRA` | `puc_reteica_compras_code` (236802) | Crédito |
| (ReteIVA) | `RET_IVA_COMPRA` | `puc_reteiva_compras_code` (236702) | Crédito |
| 2205 Proveedores / CxP | `AP_RECOGNITION` (crédito) o `CASH_PAYMENT` (contado) | `puc_cxp_proveedores_code` (220505) / Bancos 111005 / Caja 110505 | Crédito |
| GMF 4x1000 | `TAX_GMF` | `puc_gastos_financieros_code` (5305) | Débito |

Con el ejemplo del documento (5 + 3 unidades a 50.000 = 400.000 + IVA 76.000 = 476.000) el asiento es: **débito** 1435 por 400.000 y IVA descontable por 76.000; **crédito** cuenta por pagar por 476.000. Débitos = créditos = 476.000 (prueba `tests/b2b/b2bPurchases.test.js`).

Diferencias con el ejemplo del documento (se siguió lo que ya tiene la plataforma):

- El ejemplo aplica ReteFuente 2.5% (10.000) y ReteICA a una distribuidora **GRAN_CONTRIBUYENTE_AUTORRETENEDOR**. En la matriz de retenciones de EFISCO (`utils/retentionMatrix.js`) a un Gran Contribuyente **nunca se le retiene**, y además la base de 400.000 queda por debajo del umbral de 10 UVT para compras. Resultado real: sin retenciones, cuenta por pagar de 476.000 (no 462.136). Con una distribuidora de régimen ordinario y una compra de 1.700.000 sí salen ReteFuente 42.500 y ReteICA (prueba incluida).
- Las compras **manuales** nuevas de un proveedor traen por defecto la cuenta `5145` (gasto), no 1435. Solo los proveedores creados desde la red usan 1435. Conviene revisar si el contador quiere unificarlo.
- El costo promedio ponderado solo lo usa la compra de la red (`weighted_cost`); la compra manual sigue dejando el costo de la última compra.
- La compra manual a crédito exige 2 o más cuotas; la de la red permite UNA cuota a N días (`allowSingleInstallment`). La compra manual no cambió.

## Archivos

Backend: `services/b2bDistributor.service.js`, `services/b2bPurchases.service.js`, `services/b2bNetworkDebt.service.js`, `services/purchaseRegistration.service.js`, `services/distributorContractClauses.js`, `controllers/b2bDistribuidor.controller.js`, `controllers/b2b.controller.js`, `controllers/adminDistribuidores.controller.js`, `routes/b2b.routes.js`, `middlewares/requireDistribuidor.js`, `utils/distribuidorAuth.js`, `utils/b2bIdentifiers.js`. Pruebas en `backend/tests/b2b/`.

Frontend: portal `/distribuidor` (`pages/distribuidor/DistribuidorPortal.jsx`), `components/b2b/*` (tarjeta del score, directorio, modal de confirmación, resultado con cuenta regresiva, despachos, compras de la red, asiento), `pages/ScoreB2B.jsx` (`/score-b2b`), pestaña "Compras de la red" en Proveedores, badge "Red EFISCO" + "Ver asiento" en el módulo del contador, `pages/admin/AdminDistribuidores.jsx`.

## Pendiente

- Correr la migración, crear la primera distribuidora (Admin > Distribuidoras) y probarla de punta a punta contra la base real.
- Limpieza de `sesiones_consulta_b2b` (se acumulan; el dato ya vencido no se sirve).
- Segundo factor (2FA) para las distribuidoras y recuperación de contraseña: hoy la contraseña temporal la cambia un administrador.

## Solicitud de acceso (2026-10-09)

- El login de `/distribuidor` (`components/b2b/DistribuidorAuth.jsx`) tiene dos pestañas: **Iniciar sesión** y **Solicitar acceso**, con el mismo estilo de dos paneles y animación que el login de taller.
- `POST /api/b2b/distribuidor/solicitar-acceso` es público: valida, guarda en `distribuidor_solicitudes` (estado `pendiente`) y no crea ninguna cuenta. Límite de 5 por hora por IP, campo trampa `sitio_web` para bots, y exige marcar la autorización de tratamiento de datos de contacto.
- Si el NIT o el correo ya son una distribuidora, responde igual de exitoso y no guarda nada (no revela quién está en la red).
- El administrador las ve en `/admin/distribuidores` (sección "Solicitudes de acceso"): **aprobar** crea la cuenta con `crearDistribuidora` (alias, régimen y contraseña temporal los pone el admin) y **rechazar** guarda el motivo. No se envía correo automático.
- Migración: `backend/scripts/migrations/2026-10-09_b2b_distributor_requests.sql` (la corre el usuario; requiere la del portal).

## Tipo de distribuidora: mayorista y minorista (2026-10-09)

- Columna `tipo` en `distribuidores` y `distribuidor_solicitudes` (migración `2026-10-09_b2b_distributor_types.sql`; lo existente queda `mayorista`). La solicitud pública exige elegirlo; el administrador lo confirma o lo cambia al aprobar.
- Portal: el encabezado muestra el tipo. Mayorista abre en "Despachos y cartera" con control macro (despachado, por cobrar, en mora, talleres atendidos); minorista abre en el Directorio y su segunda pestaña se llama "Ventas y cartera".
- Hasta 2026-10-18 no había módulo propio de ventas ni facturación para el minorista: usaba el mismo panel de despachos. Desde esa fecha tiene inventario propio, número de documento automático y comprobante en PDF (ver [Distribuidor minorista](#distribuidor-minorista-inventario-propio-número-automático-y-comprobante-en-pdf-2026-10-18)).

## Contraseña de las distribuidoras: igual que el taller (2026-10-09)

- Al **aprobar una solicitud** (o crear una distribuidora sin contraseña temporal) la cuenta nace sin acceso (hash aleatorio) y le llega un correo (`sendDistribuidorPasswordEmail`, modo `setup`) con un enlace `FRONTEND_URL/distribuidor#token=...&type=setup` para crear su contraseña. El enlace vale 72 horas y es de un solo uso.
- El login tiene **"¿Olvidaste tu contraseña?"** (`POST /api/b2b/distribuidor/olvide-contrasena`): respuesta siempre genérica y enlace de 30 minutos (modo `reset`). La pantalla de crear contraseña es la misma para los dos modos (`POST /api/b2b/distribuidor/definir-contrasena`).
- El enlace es un JWT propio (secreto de la distribuidora) con `pwv`, la huella del password_hash vigente: al usarlo el hash cambia y el mismo enlace deja de servir. No sirve como sesión ni la sesión sirve como enlace. Definir la contraseña también quita un bloqueo vigente.
- Administración: la contraseña temporal pasó a ser opcional en el alta manual; cada fila tiene "Enviar acceso" (un endpoint interno de administración) para reenviar el enlace si el correo no salió.
- Rate limit: comparte el de restablecimientos por IP (8 por 15 minutos).

## Perfil tributario y retenciones de los despachos (2026-10-09)

- Distribuidora: `supplier_regime`, `responsable_iva`, `es_autorretenedor`, `es_declarante` (NULL = según régimen) y `tarifa_ica_por_mil`. Se pide en la solicitud, viaja a la cuenta al aprobar y se cambia en el portal (`PATCH /api/b2b/distribuidor/perfil-tributario`). Validación y traducción al motor en `utils/b2bTaxProfile.js` (un autorretenedor se trata como gran contribuyente).
- Taller: `GET/PUT /api/workshop/tax-profile` (dueño o contador) cambia solo el régimen y el carácter de agente retenedor; el resto de la configuración fiscal sigue siendo del Contador. La tarjeta de Ajustes > Datos del Taller se quitó el 2026-10-11 (el taller no debe editarlo ahí); el endpoint sigue vigente.
- Nuevo despacho: `POST /api/b2b/distribuidor/despachos/simular` y `calcularRetencionesDespacho` (mismo motor que la compra del taller) devuelven subtotal, IVA, Retefuente, ReteIVA, ReteICA y neto por cobrar, con el motivo si no aplican. El despacho guarda `payload.retenciones` y la lista expone `retenciones` y `neto_por_cobrar_cop`.
- Paneles por tipo: mayorista = una pantalla de control de cartera (sin pestañas); minorista = pestañas Directorio, Ventas y facturación y Perfil tributario.

## Mes gratis y cobro del Puntaje B2B (2026-10-10)

- Las distribuidoras entran con un **mes gratis de 30 días desde el alta** (`crearDistribuidora`, sea alta manual o aprobación de solicitud), igual que los talleres. Pasado el mes, cada consulta **evaluada** (`resultado = 'EVALUADO'`) suma **100 COP** a `distribuidores.pending_b2b_query_fees` (lo que la distribuidora le debe a EFISCO). El precio sale de `B2B_SCORE_QUERY_FEE_COP` en `utils/b2bBilling.js`, el mismo de la página de tarifas (`Tarifas.jsx`, `SCORE_QUERY_FEE_COP`); si cambia uno, cambiar el otro.
- `acumularCobroConsultaB2B` (`utils/b2bBilling.js`) lo llama `evaluarTallerB2B` justo después del registro inmutable. Lee la fila fresca de `distribuidores` (no la de la sesión) y **no acumula** durante el mes gratis (`isInFreeMonth`, el mismo de los talleres) ni en cuentas `is_test`. No bloquea: si falla solo queda en el log, la consulta ya está registrada. Los intentos fallidos (taller no disponible, confirmación inválida) no cobran.
- Columnas nuevas en `distribuidores` (migración `2026-10-10_b2b_distribuidor_free_month_fees.sql`): `free_month_until` y `pending_b2b_query_fees`. La migración deja a las existentes con `created_at + 30 días`. Todavía no hay forma de poner en cero el pendiente cuando la distribuidora paga (pendiente: botón "Registrar pago").
- Administración (`/admin/distribuidores`, `AdminDistribuidores.jsx`): cada fila muestra "Xd de mes gratis" (mismos colores de urgencia que Talleres, `utils/freeMonth.js`) o "En modo de pago", más consultas evaluadas (conteo sobre `log_consultas_b2b`), tarifa y "Nos debe". Pestañas Suspendidas, Mes gratis y Prueba (filtro en el cliente), buscador, "Mes gratis" (otro mes, un endpoint interno de administración, se rechaza con 400 si todavía hay uno vigente), "Enviar acceso", suspender o activar, contraseña y desbloquear. No se copió "Ocultar": en Talleres se revirtió.
- **No hay un tope de consultas por día o mes** (se revisó el 2026-10-10): lo único implementado es el freno antirráfaga de 10 por minuto y 15 en 5 minutos con bloqueo de 30 minutos (ver arriba). Si se quiere un tope por periodo es un cambio nuevo y, si va en el contrato, un cambio legal a clasificar.

## Panel de la cuenta, contrato v2.1 con PDF firmado y consultas sin puntaje (2026-10-12)

- **Resumen de la cuenta (`ResumenCuentaPanel.jsx`)**: al inicio del portal del mayorista y en la pestaña "Mi cuenta" del minorista. Cuatro tarjetas: mes gratis (días y fecha, o "terminó" con lo pendiente), consultas del Puntaje (del mes, total y cuántas fueron a talleres sin puntaje), método de pago y contrato firmado con botón de descarga. Endpoints `GET /api/b2b/distribuidor/resumen` y `PATCH /api/b2b/distribuidor/metodo-pago`.
- **Método de pago**: `distribuidores.metodo_pago` (`transferencia | pse | tarjeta | efectivo`). Solo guarda la elección; todavía no hay pasarela que cobre. Migración `2026-10-12_distribuidores_metodo_pago.sql`.
- **Contrato v2.1_B2B** (`distributorContractClauses.js`): agrega la cláusula de penalización por consultas no autorizadas o reiteradas (consultar en forma continua en un lapso corto sin solicitud del taller es uso no autorizado; umbral real del portal de más de 10 por minuto o 15 en 5 minutos con bloqueo de 30 minutos; posible suspensión o terminación y responsabilidad por perjuicios; toda consulta queda registrada y el taller puede reportarla). Es un cambio sustancial: quien firmó v2.0 debe firmar v2.1 al entrar. El texto lo revisa el CEO; no se definió ningún monto de multa.
- **PDF firmado igual que el de los talleres**: al firmar se genera un PDF (`distributorContractPdfTemplate.js`, mismo diseño y trazabilidad: firmante, fecha, IP, código de verificación) que se guarda en el bucket privado `efisco-contracts` con su hash SHA-256; `contractPdfStorage.service.js` sirve a las dos tablas (`generateAndStoreDistributorContractPdf`). `GET /api/b2b/distribuidor/contrato/firmado` entrega un enlace temporal y, si el PDF no se generó al firmar (Chromium caído), lo genera al pedirlo. Migración `2026-10-12_distribuidor_contract_pdf.sql` (`pdf_storage_path`, `document_hash`, `pdf_generated_at` en `distribuidor_contract_acceptances`). Al subir de versión, conservar en la plantilla el texto de las versiones anteriores (como `contractTextsForVersion` con los talleres) para regenerar fielmente.
- **Consultas sin puntaje no se cobran**: si el taller consultado aún no tiene puntaje B2B (`score_b2b` nulo, radar en calibración), la consulta queda en el registro inmutable como `EVALUADO` con score nulo pero `evaluarTallerB2B` no llama a `acumularCobroConsultaB2B`. El admin de distribuidoras muestra debajo de "Consultas" cuántas fueron sin puntaje (`consultas_sin_puntaje`). Misma regla para los talleres con los clientes: ver [reception-and-quotes.md](../negocio/reception-and-quotes.md#consultas-de-puntaje-sin-puntaje-disponible-no-se-cobran-2026-10-12).

## Pestañas para ambos tipos y directorio en tarjetas (2026-10-17)
Mayorista y minorista comparten la misma barra de pestañas arriba (Directorio de la red, Despachos y cartera / Ventas y facturación, Perfil tributario, Mi cuenta); el mayorista sigue abriendo en Despachos y el minorista en el Directorio. Ya no hay vista de una sola pantalla para el mayorista. En el directorio de talleres el botón "Consultar crédito B2B" va debajo del nombre a todo el ancho de la tarjeta (en PC el nombre quedaba aplastado). El ancho máximo del portal pasó a 1760 px. El login del portal y el del taller usan la misma estructura de barra superior.

El portal usa hasta 1600 px (antes `max-w-5xl`). Mayorista: las 4 tarjetas de cuenta en una fila (`xl`) y debajo dos columnas (despachos 3/5; consulta de crédito y perfil tributario 2/5). El directorio de talleres va en 2 columnas desde `xl`. La página Score B2B del taller (`ScoreB2B.jsx`) también ocupa 1600 px, con score a la izquierda e insights a la derecha desde `xl`. En móvil y tablet nada cambia.

## Deudas previas con una proveedora que entra a la red + escaneo de la factura (2026-10-18)

- **Enlace por NIT con confirmación del taller** (`services/b2bProviderLinks.service.js`). Si un taller ya tenía cargado a mano a un proveedor (Proveedores) con facturas a crédito y cuotas, y ese NIT se registra como distribuidora (mismo `is_test`), en Proveedores > Compras de la red aparece "Proveedores que ya son de la red" con lo que se enlazaría. Se calcula en el momento (`GET /api/b2b/vinculos-pendientes`), sin propuestas guardadas; propone proveedores con facturas sin enviar (a crédito o de contado; ver [Facturas a mano por enviar](#facturas-a-mano-por-enviar-contado-incluido-2026-10-18)). Dueño o administrador confirma (`POST /api/b2b/vinculos/:providerId/confirmar`) o responde "No es la misma empresa" (`.../rechazar`, se guarda en `providers.distribuidor_rechazado_id` y no se vuelve a preguntar).
- **Qué hace confirmar**: pone `providers.distribuidor_id` (y alinea el perfil tributario) y crea, por cada factura aún no enlazada (a crédito o de contado), una fila `compras_red_b2b` en estado **CONFIRMADA** con `payload.origen = 'IMPORTADA_PROVEEDOR'`, `supplier_purchase_id` apuntando a la compra existente y `supplier_purchases.compra_red_id` de vuelta. **No se contabiliza nada otra vez** (ni inventario ni libro mayor). Idempotente; si la distribuidora ya despachó una factura con el mismo número se deja la que existe. Si falla sin importar ninguna, se deshace el enlace.
- **Portal de la distribuidora**: la lista de despachos trae `cuotas` (pendientes y pagadas: los pagos ya hechos quedan como historial) e `importada`; cada factura muestra sus cuotas y la etiqueta "Enlazada desde las compras que el taller ya tenía registradas contigo".
- **Score B2B**: ya contaba toda cuota pendiente de cualquier proveedor (`getNetworkDebt`); al enlazar la deuda pasa de "proveedor fuera de la red" a deuda con la distribuidora, sin duplicarse (prueba incluida).
- **Aviso**: al crear o aprobar una distribuidora (`crearDistribuidora`) se avisa por la cascada de mensajería a los talleres que ya la tenían como proveedor (`avisarVinculosPendientes`, no bloquea el alta).
- **Escanear la factura** (`POST /api/b2b/distribuidor/despachos/escanear`, `services/b2bInvoiceScan.service.js`): mismo AWS Textract (`ocr.service.js`) que usa el taller en Proveedores, solo imágenes de hasta 5 MB (los PDF se adjuntan pero no se leen). Llena número de factura, ítems (referencia, categoría, unidad y medida deducidas del nombre; IVA, descuentos y retenciones según [Escaneo: IVA incluido...](#escaneo-iva-incluido-descuentos-y-retenciones-2026-10-18)) y reconoce al taller por el NIT impreso (`detected_nits`, solo talleres que esa distribuidora puede ver). No crea nada: la distribuidora revisa y envía; la foto queda adjunta al despacho. Aviso si el total leído no cuadra con los ítems. Tope de 20 escaneos por hora por distribuidora (en memoria).
- Migración: `scripts/migrations/2026-10-18_providers_distribuidor_rechazado.sql` (la corre el usuario; sin ella solo falla "No es la misma empresa"). Staging no tiene claves de AWS: el OCR real no se prueba ahí.
- Pruebas: `tests/b2b/b2bProviderLinks.test.js`, `tests/b2b/b2bInvoiceScan.test.js`.

### Retenciones de ley por defecto en el perfil tributario y botón Actualizar (2026-10-18)

- `PerfilTributarioForm.jsx` (compartido por el registro `DistribuidorAuth`, el panel `PerfilTributarioPanel` y administración): al elegir un régimen se precargan Retefuente, ReteIVA y ReteICA con las tarifas por defecto del motor y del panel del contador (`TARIFAS_POR_DEFECTO` en `frontend/src/utils/b2bTaxProfile.js`): persona natural 3.5 / 15 / 9.66 por mil, ordinario 2.5 / 15 / 9.66, simple 0 / 15 / 0, gran contribuyente 0 / 0 / 0. Siguen editables y cambiar de régimen vuelve a cargar los de ley. Si un campo queda vacío el motor usa la tarifa del taller; con valores guardados, la distribuidora fija los suyos y un ajuste posterior del contador en las tarifas del taller ya no le afecta. Si cambian las tarifas por defecto en `financialEngine.js`, actualizar también ese mapa.
- Botón Actualizar de Despachos (`DespachosPanel.jsx`): ahora gira mientras llega la lista y pide sin caché (`cache: 'no-store'`); antes volvía a pedir pero no se notaba si no había cambios.

### Facturas a mano por enviar, contado incluido (2026-10-18)

Caso real que lo motivó: un proveedor que un taller de prueba ya tenía **enlazado** con una distribuidora de prueba (producción) seguía recibiendo facturas hechas a mano en Proveedores y nunca llegaban al portal, porque el enlace solo importaba lo que existía al confirmar. Además la sección de confirmación solo listaba proveedores sin enlazar, así que el taller no veía nada que confirmar.

- `buscarVinculosPendientes` devuelve ahora dos clases de filas, ambas por confirmar del taller: proveedores **sin enlazar** que coinciden por NIT (`ya_enlazado: false`, "Confirmar enlace" y "No es la misma empresa") y proveedores **ya enlazados** con facturas a mano que aún no pasaron a la red (`ya_enlazado: true`, solo "Enviar a la distribuidora"; "No es la misma empresa" no aplica y el backend responde 409 `YA_ENLAZADO` si lo intentan).
- Se importan también las facturas **de contado** (sin cuotas): entran como `condicion_pago = 'CONTADO'`, estado CONFIRMADA y cartera PAGADA, solo como registro de la venta. Pedido del CEO.
- **No hay importación automática.** Se probó (importar sola la factura de un proveedor enlazado al registrarla) y se quitó: el CEO espera confirmarlas. Toda factura a mano espera la confirmación del taller, esté o no enlazado el proveedor.
- Frontend: la sección de Compras de la red se llama "Facturas tuyas por enviar a la red" y el modal cambia de texto según `ya_enlazado`.
- Las compras a mano anteriores quedan por confirmar tal cual; no hay migración de datos.

### Escaneo de la factura: referencia, categoría, unidad y medida (2026-10-18)

Pedido del CEO. **En el formulario de inventario "Número" es el número de la medida** (el 55 de "55 mm") y va junto a "Unidad Medida"; no es la referencia del artículo. Una primera versión puso en "Número" el código impreso de la factura y estaba mal. Quedó así (`utils/b2bItemInference.js`, todo se deduce del **nombre**; la distribuidora revisa):

- **Unidad de medida y Número**: se llenan juntos y solo si el nombre dice una medida ("Tornillo 55 mm" → Milímetros (mm) + 55; "Aceite 1 L" → Litros (L) + 1; "12V", "1,5 mm"). Si no dice una, ambos quedan vacíos ("Aceite 10W40" no es una medida).
- **Referencia**: el código de producto que lee Textract (`PRODUCT_CODE`) va a su propio campo `referencia`, nunca al número de la medida. Columna nueva `inventory.reference` (ver [Inventario](../negocio/inventory-and-providers.md)).
- **Categoría**: una de las 10 del catálogo por palabras clave (freno, filtro, bujía, correa, amortiguador...). Si ninguna encaja se **crea una** con la primera palabra con sentido del nombre ("Tornillo especial M8" → "Tornillo", aparece como "Otra"). Si el nombre no tiene ninguna palabra útil queda **vacía y obligatoria** (decisión del CEO: antes caía en "Frenos" sin avisar).
- **Unidad de inventario**: Kit ("kit", "juego", "set", "combo"), Par ("par", "x2") o Unidad por defecto.
- Un repuesto que ya existía en el inventario del taller **no recibe** la referencia del despacho: solo se guarda al crear el repuesto al recibir (pendiente si se quiere completar cuando esté vacía).

### Escaneo: IVA incluido, descuentos y retenciones (2026-10-18)

Bug reportado por el CEO: el escáner tomaba el valor de la factura como "sin IVA" y le sumaba otro 19%. `utils/b2bInvoiceTotals.js` (`normalizarMontos`) compara la suma de las líneas con el total y el IVA que lee Textract:

- **Valores sin IVA** (subtotal + IVA = total): quedan como están. **Valores con IVA incluido** (las líneas suman el total): se separa la base (`valor / (1 + tarifa)`) y el IVA no se vuelve a sumar. La tarifa (19, 5 o 0 %) sale del IVA leído; sin IVA leído se asume 19. Si nada cuadra se deja lo leído sin modo (`iva_modo: null`) y el aviso de diferencia de siempre. El total impreso puede venir antes o después de restar retenciones.
- **Descuentos** (cambiado 2026-10-20, ver la sección siguiente): por línea (valor o porcentaje, o deducido de que el total de la línea sea menor que cantidad × valor) o de toda la factura. Ya **no se restan de cada ítem**: se suman en `descuento_total_cop` y el formulario los pone en su campo "Descuento total". Arriba salen los avisos "ya incluían el IVA" y "descuentos por $X".
- **Retenciones de la factura**: `ocr.service.js` las lee por su etiqueta (ReteFuente, ReteICA, ReteIVA u otra) y el escaneo las devuelve en `retenciones_factura`. Se muestran **junto a las que calcula EFISCO** en "Factura y retenciones", con aviso si difieren. **No reemplazan el cálculo**: el neto por cobrar usa siempre las de EFISCO (decisión del CEO).
- Textract no marca descuentos ni retenciones con un tipo estándar: se reconocen por la etiqueta impresa ("Descuento", "Dcto", "ReteFuente"). Una factura con otro rótulo no los detectaría. Lo agregado a `ocr.service.js` es solo campos nuevos del resultado: el escaneo del taller en Proveedores no cambia. **No se probó con facturas reales** (staging no tiene claves de AWS).
- Pruebas: `tests/b2b/b2bInvoiceTotals.test.js`, `b2bInvoiceScan.test.js`, `b2bItemInference.test.js`, `tests/ocr.test.js`.

### Descuento total, IVA incluido por el valor neto y prueba con Textract (2026-10-20)

Pedido del CEO: el despacho no tenía dónde poner el descuento a mano y el escáner lo repartía ítem por ítem.

- **Campo "Descuento total (opcional)"** al final de "Nuevo despacho" (`DespachosPanel.jsx`, `MoneyInput`). Es un descuento comercial en pesos **antes del IVA**: `subtotal = bruto - descuento` y el IVA se calcula sobre lo descontado. El resumen muestra "Descuento -$X · Subtotal · IVA · Total". Al enviar, el frontend **reparte** el descuento entre los ítems en proporción a su valor (`factorDescuento` sobre `precio_costo_unidad_cop`): el servidor sigue recibiendo un valor unitario por ítem, sin cambios de API ni migración. El reparto redondea cada valor unitario al peso, así que el total puede quedar corrido unos pocos pesos.
- **Escaneo**: `normalizarMontos` devuelve los ítems a **precio lleno** (ya sin IVA si venía incluido) y `descuento_total_cop` en base (el descuento impreso dividido por `1 + tarifa` cuando el IVA venía incluido). El formulario lo carga en el campo; el aviso dice "quedaron en el campo Descuento total". Se quitó la línea "Descuento ya restado" de cada ítem (`descuento_cop` ya no existe en los ítems).
- **IVA incluido por el valor neto**: si el total y el IVA impresos no cuadran con las líneas, `normalizarMontos` usa también `subtotal` (`ocr.subtotal_amount`, que ya leía Textract y no se usaba, ahora lo pasa `b2bInvoiceScan.service.js`): si las líneas valen `subtotal × (1 + tarifa)` (19 o 5 %) marca `INCLUIDO`. Caso real: una factura con líneas que suman $199.200 y "Valor neto" $167.393 (= 199.200 / 1,19) cuyo total impreso no cuadra con nada devolvía `iva_modo: null` y la pantalla sumaba el 19 % encima.
- **Limitación conocida**: Textract rotula el "Valor neto" como total en ese formato, así que el aviso "La factura dice un total de $X y los ítems suman $Y" sale engañoso aunque los ítems estén bien. Pendiente decidir si se suprime cuando el "total" leído coincide con el valor neto.
- **Probado con Textract real en staging** (la factura de prueba del CEO): separó la base del IVA, los 8 ítems quedan con el valor impreso y el descuento va en el campo. La distribuidora igual revisa todo antes de enviar.
- **Entorno**: para probar el escáner en local hay que copiar `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` y `AWS_REGION` de `backend/las variables de entorno a `backend/las variables de entorno.staging` (por defecto van vacías para no exponer claves de producción). Ojo con Docker: `docker compose restart` **no relee** el `env_file`; hay que `up -d --force-recreate backend`, y un contenedor recreado vuelve a la copia de la imagen (solo `docker compose watch` sincroniza archivos), así que después de recrear se necesita `up -d --build backend`.

### Despacho a una sede del taller (2026-10-19)

El taller y sus sucursales comparten NIT. `GET /api/b2b/distribuidor/despachos/datos-taller?nit=` devuelve `sedes` (la principal primero y las sucursales activas, con razón social, NIT, dirección, barrio, ciudad y teléfono; `datos` sigue siendo la primera sede por compatibilidad). `POST /despachos` y `/simular` aceptan `sede_id`: `resolverSedeDestino` valida que sea la principal o una sucursal **activa del mismo taller** (si no, 400 `SEDE_INVALIDA`; sin `sede_id`, la principal).

- El despacho, el inventario y la cuenta por pagar quedan en la **sede elegida** (`workshop_id = sede.id`). El **perfil tributario** (retenciones) sigue siendo el del taller principal. El aviso por mensajería usa la sede y, si esta no tiene dueño o teléfono, cae a los de la principal.
- La deuda y la mora de la red cuentan en el puntaje del **grupo** (principal + sucursales): `b2bNetworkDebt.service.js` consulta `providers`, `supplier_installments` y `compras_red_b2b` con `idsDelGrupo`.
- **Dónde se ve**: en "Nuevo despacho", la tarjeta "Datos del taller para tu factura" (con el botón Copiar todo) aparece al escribir un NIT completo de un taller registrado, activo y con contrato B2B; el selector "Sede que recibe el despacho" solo sale si ese NIT tiene más de una sede. Con un taller no registrado no aparece nada de esto.
- Pruebas: `tests/b2b/b2bSedeDestino.test.js`. **No se probó en staging con sucursales reales.** Sin migración.

## Distribuidor minorista: inventario propio, número automático y comprobante en PDF (2026-10-18)

El minorista que no tiene software de facturación usa el de EFISCO. Decisiones del usuario: el comprobante es un **PDF de EFISCO que NO es factura electrónica DIAN** (la factura electrónica real exigiría resolución y proveedor tecnológico por distribuidor, no se construyó), y el inventario **descuenta al despachar**. Solo aplica a `tipo = 'minorista'`; el mayorista no cambia.

- **Número automático**: con "No. de factura" vacío, `crearDespacho` asigna `VD-000001`, `VD-000002`... (máximo existente + 1 del distribuidor; la unicidad la garantiza `UNIQUE (distribuidor_id, factura_id)` y reintenta si chocan). Si escribe el suyo se respeta. La vista previa (`simular`) usa un número de relleno.
- **Comprobante PDF**: `GET /api/b2b/distribuidor/despachos/:id/documento` (`services/distribuidorSaleDoc*.js`, Puppeteer como los demás PDF, todo dato escapado). Trae las dos partes, ítems con referencia y medida, IVA, retenciones y la nota de que no es factura DIAN. Solo el dueño del despacho lo puede pedir (404 si no). Botón "Comprobante de venta" en cada despacho.
- **Inventario** (`distribuidor_inventario`, migración `2026-10-18_distribuidor_inventario.sql`, RLS activada sin políticas): los ítems de cada despacho entran solos con nombre, categoría, medida, número, referencia, unidad y último valor, con cantidad 0 ("él solo pone la cantidad"). `clave` = nombre normalizado | número | medida: el mismo repuesto con otras mayúsculas o tildes no se duplica. Pestaña "Mi inventario" (`InventarioDistribuidorPanel.jsx`): editar cantidad, agregar a mano, quitar, buscar. Endpoints `GET · POST /api/b2b/distribuidor/inventario` y `PATCH · DELETE .../inventario/:id`.
- **Descuenta al despachar** (`registrarDespacho`): al crear el despacho se resta lo despachado **sin bajar de 0**; si se despacha más de lo que había no se bloquea, la respuesta trae `inventario.excesos` y la pantalla avisa. Lo realmente descontado se guarda en `payload.inventario_descontado` y, si el taller **rechaza** el despacho, `rechazarCompraRed` lo devuelve exactamente (el exceso no se devuelve). Es best-effort: si el inventario falla el despacho ya existe y queda un aviso en el log.
- **En "Nueva venta"**: el nombre autocompleta con su inventario (sin pisar lo ya escrito), cada ítem tiene "Cantidad que tengo (opcional)" que fija la cantidad antes de descontar (`stock_actual`, no se guarda en el payload) y la línea del ítem muestra "En inventario: N" con aviso si despacha de más.
- **Lección**: `requireDistribuidor` no seleccionaba la columna `tipo`, así que `req.distribuidor.tipo` llegaba vacío y nada de lo anterior se activaba en el servidor; las pruebas unitarias no lo vieron porque pasaban el tipo a mano y el helper en memoria no filtra columnas. Lo detectó la prueba en el navegador. Ahora hay una prueba que falla si `tipo` sale de ese `select` (`b2b.routes.http.test.js`). Un campo nuevo de `distribuidores` que use el servidor debe agregarse a ese `select`.
- Sin la migración el despacho se crea igual y solo falla el inventario. Pruebas: `tests/b2b/b2bDistribuidorInventario.test.js` y el caso minorista de `b2b.routes.http.test.js`.

## Escrow de crédito: reserva cautelar de cupo (2026-10-20)

Problema: un despacho a crédito queda PENDIENTE_CONFIRMACION y no genera cuotas hasta que el taller confirma; otra distribuidora que consultara al mismo taller veía el cupo intacto y le despachaba también (arbitraje de latencia).

- `crearDespachoUnico` (solo si `condicion_pago = 'CREDITO'`): evalúa el cupo del taller (`evaluateWorkshopB2B` sin persistir; con bloqueo duro por mora el cupo es 0) y llama a `reservarCupo` (`services/creditEscrow.service.js`) ANTES de insertar, con el id de la compra (generado en el servicio) como referencia. Si el insert falla se libera; `confirmarCompraRed` la pasa a `consumida` y `rechazarCompraRed` a `liberada`.
- La función SQL `reservar_cupo_credito` (migración `2026-10-20_cupo_reservas_escrow.sql`, tablas `cupo_cuentas` y `cupo_reservas`, RLS sin políticas, solo el rol de servicio del backend) bloquea la fila de la cuenta con `SELECT ... FOR UPDATE` y lee DENTRO del bloqueo la deuda activa y las reservas vigentes (disponible = cupo − deuda − reservas); es idempotente por referencia y las reservas vencen a los 7 días. Probada contra un PostgreSQL real: con 24 solicitudes simultáneas al límite del IEA, sin bloqueo rompían el límite entre 16 y 27 de cada 50 sujetos; con el bloqueo, 0.
- La consulta de la distribuidora (`evaluarTallerB2B`) resta lo reservado del cupo disponible (`cupo_reservado_cop`) y la pantalla lo explica.
- **Política** `B2B_ESCROW_ENFORCE`: por defecto solo REGISTRA la reserva (marca `excedido`); con `true` rechaza el despacho que excede el cupo (409 `CUPO_INSUFICIENTE`). Apagado porque un taller nuevo tiene cupo 0 hasta tener historial.
- Falla abierta: sin migración o con un error de base el despacho sigue y se advierte en el log. **No está conectado al crédito B2C** (la liquidación a crédito no consulta ningún cupo hoy); la función ya soporta `scope 'B2C'`.
- Pruebas: `tests/b2b/creditEscrow.test.js`, los casos "escrow" de `b2bPurchases.test.js` y de `b2bDistributor.service.test.js`.
- Estado (2026-10-21): la migración está corrida en staging y en producción (se comprobó solo leyendo que `cupo_cuentas` y `cupo_reservas` existen) y el código se subió junto con las salvas defensivas del puntaje. `B2B_ESCROW_ENFORCE` sigue en `false`.

## Etiquetas del directorio: especialidad del taller y línea de la distribuidora (2026-10-21)
Pedido del CEO: en la red todo se veía mezclado y la distribuidora no sabía quién es de mecánica, de latonería y pintura o mixto (y los talleres no sabían qué vende cada distribuidora). Solo es una etiqueta: no cambia puntaje, catálogo ni contabilidad. Migración `2026-10-21_especialidad_taller_linea_distribuidora.sql` (corrida por el usuario en staging y en producción; el código se subió el 2026-10-21).
- **Taller:** `workshop_config.especialidad` = `mecanica` (default) | `latoneria_pintura` | `mixta`. Independiente de `vehicle_type` (qué vehículo atiende). Se elige al registrarse (`Auth.jsx`, obligatorio salvo taller de bicicletas, que queda `mecanica`) y se edita en Ajustes > Datos del taller (el dueño, con PIN). Al crear una sucursal se elige o hereda la de la principal. El panel admin de talleres la muestra como insignia.
- **Distribuidora:** `distribuidores.linea_productos` (y `distribuidor_solicitudes.linea_productos`) = `repuestos` (default) | `pinturas` | `ambos`. Obligatoria en el formulario público de solicitud (`/api/b2b/distribuidor/solicitar-acceso`); el admin la corrige al aprobar o con un endpoint interno de administración. El taller la ve como insignia en Compras de la red y en los enlaces con proveedores.
- **Directorio:** `GET /api/b2b/distribuidor/talleres?especialidad=` filtra; un taller mixto aparece también al filtrar por mecánica y por latonería y pintura (hace las dos cosas). Cada taller trae `especialidad` (también en `taller_identificacion` de la consulta). Constantes y etiquetas: `backend/utils/workshopSpecialty.js` (espejo `frontend/src/utils/workshopSpecialty.js`).
- **Datos existentes:** todos quedan en el default; la migración marca como `mixta` a GR Racing y sus sucursales (EM Moto Cali, J Eléctricos y Mecánica Automotriz JG SAS son solo mecánica). Los talleres nuevos de latonería y pintura se eligen al registrarse.
- **Pruebas:** `tests/b2b/workshopSpecialty.test.js` y los casos de especialidad de `b2bDistributor.service.test.js`, `distribuidorSolicitudes.test.js` y `workshopRegistration.alerts.test.js`. Verificado además a mano en staging (2026-10-21): chips y filtro del directorio (un taller mixto sale en Mixto y en Latonería y pintura), selector en Ajustes con confirmación de contraseña, selector obligatorio del registro, "¿Qué vendes?" del formulario de solicitud, solicitud visible en el panel admin con su insignia, select de línea por fila (`PATCH`) y modal de aprobación con la línea ya rellena, y sucursal que hereda la especialidad de la principal o con la elegida. **No se probó en navegador** aprobar una solicitud (crea la cuenta y manda un enlace por correo) ni el envío completo del registro de un taller; solo los cubren las pruebas automáticas.
- **Detalle de la sucursal:** el formulario de "Crear sucursal" muestra preseleccionada la especialidad de la principal; sin tocarla hereda, y se puede cambiar.

## Panel de puntaje B2C del mostrador: consulta, QR y ventas a clientes (2026-10-21)
Documento "Panel de puntaje B2C para distribuidor minorista y mayorista". Subido el 2026-10-21 (backend `0b66c8e`, frontend `25cf4fb`); migración `2026-10-21_distribuidor_clientes_b2c.sql` corrida en staging y producción.

- **Barra "Consultar el puntaje de un cliente"** arriba de las pestañas (mayorista y minorista, `ConsultaClienteB2C.jsx`). `POST /api/b2b/distribuidor/clientes/buscar`:
  - Ficha propia o de la **Red EFISCO** (última decisión del cliente con versión >= 1.1.0 aceptada): se copia a la distribuidora (`clients` con `workshop_id` NULL y `distribuidor_id`, rastro en `audit_logs`) y se pide la **fecha de nacimiento** (mismo modal de Recepción).
  - Si no está (o le falta la fecha): solicitud en `client_registration_requests` (`distribuidor_id`) y **QR** a `/cliente/registro-comercio/:distribuidorId/:requestId`, la misma página y los mismos campos y textos legales del QR de Venta Directa (`ClienteRegistroVenta.jsx`). El portal pregunta cada 3 s (`GET .../clientes/solicitud/:id`) y al completarse ofrece **Despachar** ahí mismo (el puntaje tarda 3 meses).
  - **Sin dirección** en el QR (decisión del usuario: sería un dato nuevo, cambio legal sustancial).
- **Consulta del puntaje** (`POST .../clientes/:cedula/puntaje`, `services/b2cDistribuidorClientes.service.js`): fecha de nacimiento con 5 intentos / 15 min, registro inmutable `log_consultas_b2c` (CONSULTA, SIN_PUNTAJE_DISPONIBLE, FECHA_NO_COINCIDE, SIN_FECHA_NACIMIENTO, BLOQUEADO_INTENTOS; nunca la fecha tecleada), habilitación a 90 días con `getScoreUnlockInfo`. Devuelve local (solo esta distribuidora) y global, y el cupo con `buildCreditAssessment` (sin techo de política: "Cupo calculado por EFISCO"). **Cobro 100 COP** como al taller (`acumularCobroConsultaB2B`), no en mes gratis, cuentas de prueba ni puntaje en construcción.
- **Panel del puntaje** (`ClienteScoreVista.jsx`): otra pantalla con el **mismo formato de Recepción** (componente compartido `components/ClienteScoreDetalle.jsx`), visible 15 min, botón "Despachar a este cliente". Semáforo y recomendaciones con el puntaje **global** (el local casi siempre está vacío) y los 4 niveles y colores de la app; textos de mostrador en `scoreRecommendations.js` (`contexto 'comercio'`).
- **Ventas a clientes** (pestaña nueva, `VentasClientesPanel.jsx`, `VentaClienteForm.jsx`, `services/b2cDistribuidorVentas.service.js`): datos del cliente con "Copiar todo", arrastrar el PDF (mismo escáner de los despachos), ítems con el editor compartido `ItemsDespachoEditor.jsx`, contado o **fiado** de 1 a 24 cuotas (primera desde mañana hasta un año, las demás al mismo intervalo), número automático `VC-000001`, comprobante PDF (no DIAN) y descuento de inventario del minorista. "Registrar pago" en dos pasos, siempre con la hora del servidor. Tablas `distribuidor_ventas_clientes` y `distribuidor_cuotas_clientes`.
- **Entra al EFISCO Score B2C** (`services/distributorSalesHistory.js`): cada venta es una orden (Pilar 3: frecuencia y ticket) y cada cuota una cuota del motor (Pilar 1: mora y retorno de caja; Pilar 2: IEA y aceleración de deuda) con la distribuidora como acreedor (`workshop_id` = 'DIST-...'). Se suma en Recepción, Registro Seguro, PDF, Mi Puntaje (también como puntaje local por comercio) y en el cupo. `loadScoringContext` ya no manda esos ids a `workshop_config` (uuid).
- **Red EFISCO**: `findNetworkClient` excluye la ficha propia en JS (`workshop_id <> X` en SQL descartaba las fichas con `workshop_id` NULL), así un taller también reconoce a un cliente registrado en una distribuidora. La renovación de autorización inserta una fila por taller o distribuidora.
- **Decisión mía a revisar:** un cliente con autorización de red se consulta desde la distribuidora solo con la fecha de nacimiento (el taller exige la aceptación en su propio taller). Pendiente del CEO: los términos del QR hablan de "El Taller" y el contrato de distribuidoras no menciona consultas B2C.
- Pruebas: `tests/b2b/b2cDistribuidorClientes.test.js`, casos nuevos de `tests/clients/networkClients.test.js`. Verificado en staging por API y con capturas (`nuevos cambios/cambios realizados/capturas panel B2C distribuidor 2026-10-21/`).
