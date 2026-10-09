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
- No hay módulo propio de ventas directas ni facturación para el minorista: usa el mismo panel de despachos. Si se quiere uno, hay que definirlo.

## Contraseña de las distribuidoras: igual que el taller (2026-10-09)

- Al **aprobar una solicitud** (o crear una distribuidora sin contraseña temporal) la cuenta nace sin acceso (hash aleatorio) y le llega un correo (`sendDistribuidorPasswordEmail`, modo `setup`) con un enlace `FRONTEND_URL/distribuidor#token=...&type=setup` para crear su contraseña. El enlace vale 72 horas y es de un solo uso.
- El login tiene **"¿Olvidaste tu contraseña?"** (`POST /api/b2b/distribuidor/olvide-contrasena`): respuesta siempre genérica y enlace de 30 minutos (modo `reset`). La pantalla de crear contraseña es la misma para los dos modos (`POST /api/b2b/distribuidor/definir-contrasena`).
- El enlace es un JWT propio (secreto de la distribuidora) con `pwv`, la huella del password_hash vigente: al usarlo el hash cambia y el mismo enlace deja de servir. No sirve como sesión ni la sesión sirve como enlace. Definir la contraseña también quita un bloqueo vigente.
- Administración: la contraseña temporal pasó a ser opcional en el alta manual; cada fila tiene "Enviar acceso" (un endpoint interno de administración) para reenviar el enlace si el correo no salió.
- Rate limit: comparte el de restablecimientos por IP (8 por 15 minutos).

## Perfil tributario y retenciones de los despachos (2026-10-09)

- Distribuidora: `supplier_regime`, `responsable_iva`, `es_autorretenedor`, `es_declarante` (NULL = según régimen) y `tarifa_ica_por_mil`. Se pide en la solicitud, viaja a la cuenta al aprobar y se cambia en el portal (`PATCH /api/b2b/distribuidor/perfil-tributario`). Validación y traducción al motor en `utils/b2bTaxProfile.js` (un autorretenedor se trata como gran contribuyente).
- Taller: `GET/PUT /api/workshop/tax-profile` (dueño o contador) cambia solo el régimen y el carácter de agente retenedor; el resto de la configuración fiscal sigue siendo del Contador. Tarjeta en Ajustes > Datos del Taller.
- Nuevo despacho: `POST /api/b2b/distribuidor/despachos/simular` y `calcularRetencionesDespacho` (mismo motor que la compra del taller) devuelven subtotal, IVA, Retefuente, ReteIVA, ReteICA y neto por cobrar, con el motivo si no aplican. El despacho guarda `payload.retenciones` y la lista expone `retenciones` y `neto_por_cobrar_cop`.
- Paneles por tipo: mayorista = una pantalla de control de cartera (sin pestañas); minorista = pestañas Directorio, Ventas y facturación y Perfil tributario.
