# Auditoría

| Campo | Valor |
|---|---|
| Versión | 0.2 |
| Archivos | `src/core/middlewares/auditoria.js`, `src/core/repositories/auditoriaRepository.js` |
| Tabla | `auditoria_depositos` |
| Depende de | `seguridad.md`, `06-configuracion-y-observabilidad.md` |

## 1. Propósito

Dejar un registro inalterable de **quién** intentó **qué**, **desde dónde**, **cuándo** y **con qué resultado**, incluidos los intentos rechazados. Es la evidencia ante un reclamo o una revisión del regulador.

> No confundir con los logs: los logs son para depurar y se rotan; la auditoría es un registro de negocio que se conserva.

## 2. Qué se audita

| Ruta | `accion` |
|---|---|
| `POST /depositos` | `REGISTRAR_DEPOSITO` |
| `GET /depositos/:id` | `CONSULTAR_DEPOSITO` |
| `GET /depositos?cuenta=` | `CONSULTAR_HISTORIAL` |

No se auditan `/health` ni `/ready`.

## 3. Cómo funciona

1. El middleware registra `res.on('finish', …)` al **inicio** de la cadena (ver orden en `seguridad.md §2`).
2. Durante la petición, la capa que conoce los datos los deja en `res.locals.auditoria`:
   ```js
   res.locals.auditoria = { depositoId, canal, monto, moneda };
   ```
   (lo llena el controller con lo que devuelve el service, también en los rechazos 422 cuando ya se conoce el canal y el monto).
3. Al terminar la respuesta, inserta una fila:

| Columna | Valor |
|---|---|
| `deposito_id` | `res.locals.auditoria.depositoId` o NULL |
| `usuario_id` | `req.usuario?.id` o NULL (401) |
| `accion` | según la ruta |
| `ip` | `req.ip` (con `trust proxy`, es la IP real que reporta el gateway) |
| `canal`, `monto`, `moneda` | de `res.locals.auditoria` o NULL |
| `resultado` | `'OK'` si status < 400; si no, `error.codigo` de la respuesta |
| `status_http` | `res.statusCode` |
| `correlation_id` | `req.correlationId` |

- **RN-AUD-01** La auditoría **nunca** cambia la respuesta: se escribe después de enviarla. Si el INSERT falla, se loguea con nivel `error` (incluyendo todos los campos de la fila, para poder reconstruirla) y no se reintenta.
- **RN-AUD-02** La tabla es solo inserción (sin permisos UPDATE/DELETE para la aplicación).
- **RN-AUD-03** No se guardan datos sensibles: ni token, ni cuerpo completo, ni datos personales más allá del `usuario_id`.
- **RN-AUD-04** En un replay de idempotencia también se registra una fila (es otro intento del cliente) con el mismo `deposito_id`.

> **Limitación conocida:** si el proceso muere entre enviar la respuesta y escribir la auditoría, esa fila se pierde. Para el alcance del curso es aceptable. En producción se escribiría en la misma transacción del depósito o con un patrón *outbox*.

## 4. Criterios de aceptación

- **CA-AUD-01** POST exitoso → una fila con `accion = REGISTRAR_DEPOSITO`, `resultado = 'OK'`, `status_http = 201`, `deposito_id` no nulo y el `correlation_id` de la respuesta.
- **CA-AUD-02** POST rechazado por límite → fila con `resultado = 'LIMITE_CANAL_EXCEDIDO'`, `status_http = 422`, `deposito_id` NULL, `monto` y `canal` llenos.
- **CA-AUD-03** GET sin token → fila con `usuario_id` NULL y `status_http = 401`.
- **CA-AUD-04** Con la tabla de auditoría inaccesible (simulada), un POST válido sigue respondiendo 201.
- **CA-AUD-05** `GET /health` no genera fila.
