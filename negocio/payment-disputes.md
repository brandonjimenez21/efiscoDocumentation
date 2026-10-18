# Disputas de pago y auditoría inmutable (2026-10-07)

Documento "Puntos pendientes para hacer en la plataforma", pendiente 1 (FEAT-DISPUTE-01) y el log de auditoría del pendiente 4. Migración: `backend/scripts/migrations/2026-10-07_payment_disputes_audit.sql` (la corre el usuario en el SQL Editor; el código que depende de ella no se sube hasta que lo confirme).

## Flujo
1. **Cliente (Mi Puntaje).** "Tus movimientos" lista sus cuotas de toda la red (`GET /api/clients/score-portal/movements`). En cada cuota pendiente sin reclamo hay un enlace discreto "¿Reportar inconsistencia o pago no reflejado?". El formulario pide tipo (`pago_no_registrado`, `monto_incorrecto`, `otro`), comprobante (JPG, PNG, WEBP o PDF, máx. 5 MB; obligatorio salvo en `otro`), fecha y monto pagado (opcionales) y una observación (5 a 500 caracteres). `POST /api/clients/score-portal/disputes` (multipart, sesión del portal, máx. 5 reclamos por cédula cada 15 min).
2. **Ticket e inmunidad.** Se crea `payment_disputes` con ticket `DISP-0001` (secuencia de Postgres; puede haber saltos si un insert falla), `status = EN_DISPUTA`, `client_installments.dispute_status = 'EN_DISPUTA'` y plazo `due_at` = 5 días hábiles (`DISPUTE_SLA_BUSINESS_DAYS`, lunes a viernes sin festivos). Mientras esté abierta, el motor trata esa cuota como `en_disputa`: no cuenta como mora, ni cartera castigada, ni aceleración (la deuda sí cuenta para el IEA) y el puntaje trae la bandera `DISPUTA_ABIERTA`. Una sola disputa abierta por cuota (índice único parcial) y una cuota con reclamo rechazado no se puede volver a reclamar.
3. **Taller (`/pagos-por-verificar`, `/api/disputes`).** Contador "X pagos por verificar" en el menú [2026-10-17: el rol contador ya no ve este ítem ni la ruta; solo dueño y admin] (insignia) y en las alertas del Dashboard. El dueño o un administrador (`role` `owner` o `admin`) abre el comprobante (URL firmada 10 min) y **aprueba** (registra el cobro con la misma lógica de `billing.controller.js:payInstallment`, deja `paid_at` en la fecha que declaró el cliente para que su mora no cuente la demora del taller, y recalcula el puntaje solo al volver a consultarlo) o **rechaza con motivo** (la cuota vuelve a contar). El contador y el mecánico solo consultan. Un reclamo `monto_incorrecto` con monto menor al saldo abona solo ese monto; `otro` se cierra sin tocar la caja.
4. **Vencimiento.** `sweepDisputes` corre cada hora (`setInterval` en `server.js`): con 2 días hábiles o menos avisa una vez (`DISPUTE_ESCALATED`, correo al contador y alerta de Telegram); vencido el plazo aprueba solo (`AUTO_APPROVED_TIMEOUT`, medio `banco`), registra el cobro y avisa al contador.

## Auditoría inmutable (`audit_events`)
- Solo INSERT: triggers rechazan UPDATE, DELETE y TRUNCATE. Un trigger BEFORE INSERT toma un lock, calcula `prev_hash` y `entry_hash` (SHA-256 de los campos). `verify_audit_chain()` recorre la cadena y devuelve la primera fila rota; `verifyAuditChain()` en `services/auditLog.service.js` la llama.
- Eventos: `DISPUTE_CREATED`, `DISPUTE_PROOF_VIEWED`, `DISPUTE_APPROVED`, `DISPUTE_REJECTED`, `DISPUTE_ESCALATED`, `AUTO_APPROVED_TIMEOUT`. Cada uno con timestamp, actor (`client` / `workshop_user` / `system`), `workshop_id`, `transaction_id` (la cuota) y el SHA-256 del comprobante en `evidence_sha256`. Los eventos críticos fallan en modo estricto (si no se puede escribir el rastro, la acción no se completa); después de mover dinero solo se avisa por Telegram.
- Comprobantes: bucket privado `efisco-dispute-proofs` (lo crea el backend al primer archivo), ruta `{taller}/{reclamo}/{hash16}.{ext}`, tipo validado por firma de bytes (no por el nombre) y hash guardado en `payment_disputes.proof_sha256`.

## Movimientos de prueba y botón Actualizar
En Mi Puntaje, "Tus movimientos" tiene un botón **Actualizar** (recarga la lista sin cerrar la sesión del portal). Los movimientos de talleres de prueba (`is_test_mode`) solo los ve el cliente demo (1000000001) y las cédulas de `INTERNAL_TEST_CEDULAS`, marcados "Movimiento de prueba, no afecta tu puntaje"; para un cliente real se ocultan. Un reclamo sobre una cuota de prueba queda marcado como de prueba (el puntaje no lo ve de todos modos).

## Pruebas
`backend/tests/disputes/` (servicio con un Supabase en memoria que reproduce la migración, controladores, flujo HTTP real con multer y días hábiles). La migración se probó contra un Postgres 16 desechable (cadena, inmutabilidad, ticket, índice único). El cobro real (`payInstallment`) está mockeado en las pruebas: no se ejecutó contra la base real.

## Endpoints
| Método | Ruta | Acceso | Qué hace |
|---|---|---|---|
| GET | `/api/clients/score-portal/movements` | Sesión del portal | Cuotas del cliente con estado del reclamo y `can_report` |
| POST | `/api/clients/score-portal/disputes` | Sesión del portal | Crea el reclamo (multipart: `proof`, `installment_id`, `report_type`, `observation`, `claimed_amount?`, `claimed_paid_at?`) |
| GET | `/api/disputes?status=` | Taller | Lista con días hábiles restantes |
| GET | `/api/disputes/pending-count` | Taller | `{count}` (0 si la migración no corrió) |
| GET | `/api/disputes/:id/proof` | Taller | URL firmada del comprobante (audita la vista) |
| POST | `/api/disputes/:id/approve` | owner/admin | `{cash_channel?: 'banco'\|'efectivo'}` |
| POST | `/api/disputes/:id/reject` | owner/admin | `{reason}` (5 a 500 caracteres) |
