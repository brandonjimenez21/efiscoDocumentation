# Seguridad: resumen de controles

Resumen de alto nivel de cómo se protege el sistema. Describe los controles que existen; el detalle operativo (configuración, rutas internas y procedimientos) se mantiene en la documentación privada.

## Aislamiento entre talleres
- **Multi-tenant a nivel de aplicación**: cada operación valida el taller y el rol de quien la pide a partir de la sesión, nunca de un parámetro que mande el cliente.
- **Seguridad a nivel de fila (RLS) en las tablas de datos, con negación por defecto**: las tablas nuevas nacen con RLS activado y sin políticas públicas; el acceso pasa por el backend. Cada migración que crea tablas se audita con una consulta de verificación.

## Autenticación y sesiones
- **Sesión en cookie `httpOnly`** con un JWT propio; el token nunca queda disponible para el JavaScript de la página.
- **Autenticación del panel de administración separada** de la de los talleres (otro secreto y otra cookie).
- **Roles con permisos finos**: dueño, administrador, mecánico y contador, con vistas restringidas por empleado.
- **Segundo factor (TOTP, RFC 6238)** disponible para todas las cuentas, con códigos de recuperación de un solo uso. Su obligatoriedad se activa por política del negocio.
- **Política de contraseñas** con lista de requisitos en pantalla y **PIN de bloqueo por inactividad**.
- **Enlaces de restablecimiento de un solo uso**: se invalidan al usarse y no sirven como sesión.

## Datos sensibles
- **Cifrado en reposo (AES-256-GCM)** para secretos de terceros, con llave fuera del código y rotación soportada.
- **Contraseñas con hash y sal** (scrypt) en las cuentas propias.
- **Archivos privados**: comprobantes y contratos en almacenamiento privado, con URLs firmadas de corta vida.
- **Consentimiento versionado de datos personales (Ley 1581 de 2012)**: cada texto legal tiene versión, y los cambios sustanciales exigen una nueva autorización.

## Abuso y trazabilidad
- **Límites de intentos** en el inicio de sesión, los códigos de un solo uso, el restablecimiento de contraseña y un límite global por IP. La IP del cliente se resuelve de forma que no se pueda falsificar con encabezados.
- **Registro de auditoría inmutable con cadena de hashes**: las acciones críticas quedan encadenadas y cualquier alteración se detecta.
- **Alertas automáticas** ante patrones anómalos (varias IPs contra una misma cuenta, ráfagas de consultas).
- **Webhooks de pago con verificación en tiempo constante** y rechazo por defecto cuando falta la configuración.
- **HTML escapado** en todos los documentos que se convierten a PDF o correo.

## Proceso
- **Cambios en la base de datos**: migraciones con comentario y verificación, ejecutadas de forma manual por el responsable; nada de esquema se aplica desde el código.
- **Scripts que escriben datos reales**: nunca se ejecutan sin confirmación explícita y llevan modo de prueba por defecto.
- **Pruebas automáticas** de los controles de seguridad (sesiones, permisos, límites, cifrado, segundo factor) dentro de la suite de integración continua.
