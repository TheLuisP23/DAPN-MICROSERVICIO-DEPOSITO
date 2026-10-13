# Transversal: seguridad

Middlewares: `correlationId` y `autenticar` (globales), `requiereRol` (por ruta).
Archivos: `src/core/middlewares/correlationId.js`, `src/core/middlewares/auth.js`.

## Correlation id

- Si la petición trae `X-Correlation-Id`, se usa; si no, se genera un UUID.
- Se devuelve en el header de la respuesta y en todo cuerpo de error.
- Se envía a billetera-service en cada llamada.

## Autenticación (JWT)

- El token lo emite usuario-service (ver `30-integraciones/usuario.md`).
- Se valida localmente con la clave pública; NO se llama a usuario-service en cada petición.
- Se verifican: firma, algoritmo permitido, expiración, `iss` y `aud`.
- Si es válido, se deja en `req.usuario = { id, roles }`.

## Roles y autorización

| Rol | Canal que puede usar | Puede depositar a | Puede consultar |
|---|---|---|---|
| CLIENTE | APP | Solo cuentas propias | Solo depósitos de cuentas propias |
| AGENTE | AGENTE | Cualquier cuenta | Solo depósitos que él registró |
| CAJERO | VENTANILLA | Cualquier cuenta | Solo depósitos que él registró |

- El canal del depósito debe corresponder al rol. Si no, 403 `SIN_PERMISO`.
- "Cuenta propia" se verifica con el `titularId` que devuelve billetera al consultar la cuenta.

## Escenarios

**S1. Sin token** → 401 `TOKEN_REQUERIDO`.
**S2. Token expirado** → 401 `TOKEN_EXPIRADO`.
**S3. Token firmado con otra clave o con `alg: none`** → 401 `TOKEN_INVALIDO`.
**S4. Token válido de otro servicio (aud distinto)** → 401 `TOKEN_INVALIDO`.
**S5. CLIENTE deposita a una cuenta que no es suya** → 403 `SIN_PERMISO`.
**S6. CLIENTE usa canal VENTANILLA** → 403 `SIN_PERMISO`.
**S7. Petición sin correlation id** → la respuesta trae uno generado.
**S8. Petición con correlation id** → la respuesta trae el mismo.

## Logs

- Nunca: token, número de cuenta completo (usar `****-****-456789`), datos personales.
- Siempre: correlationId, usuarioId, ruta, código de respuesta, duración.

## Configuración

`JWT_PUBLIC_KEY`, `JWT_ALGORITHMS`, `JWT_ISSUER`, `JWT_AUDIENCE`.
