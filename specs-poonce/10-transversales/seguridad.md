# Seguridad: autenticación, roles y canal

| Campo | Valor |
|---|---|
| Versión | 0.2 |
| Archivos | `src/core/middlewares/autenticar.js`, `src/core/middlewares/requiereRol.js` |
| Depende de | `30-integraciones/usuario.md`, `05-errores.md` |

## 1. Propósito

Asegurar que solo usuarios autenticados por usuario-service usen el servicio, que cada rol solo haga lo que le corresponde y que el **canal** del depósito no lo pueda elegir el cliente.

## 2. Orden de middlewares

| # | Middleware | Alcance | Nota |
|---|---|---|---|
| 1 | `correlationId` | global | |
| 2 | `auditoria` | global (excepto `/health`, `/ready`) | **[DECISIÓN EQUIPO]** En v0.1 iba después de `requiereRol`. Se adelanta para que los intentos 401/403 también queden auditados. Solo registra un listener `res.on('finish')`, así que lee `req.usuario` al terminar. |
| 3 | `autenticar` | global (excepto `/health`, `/ready`) | |
| 4 | `requiereRol(...roles)` | por ruta | |
| 5 | `idempotencia` | solo `POST /depositos` | |

## 3. Autenticación (`autenticar`)

- **RN-SEG-01** Se exige `Authorization: Bearer <jwt>`. Sin header o con otro esquema → 401 `NO_AUTENTICADO`.
- **RN-SEG-02** El token se verifica **localmente** con la clave pública de usuario-service (`jsonwebtoken` o `jose`). No se llama a usuario-service en cada petición.
- **RN-SEG-03** Se verifican: firma, `alg` igual a `JWT_ALGORITMO` (lista blanca; nunca se acepta `none`), `exp`, `nbf` (si viene), `iss = JWT_ISSUER`, `aud = JWT_AUDIENCE`. Tolerancia de reloj: 30 s.
- **RN-SEG-04** Cualquier fallo de verificación → 401 `NO_AUTENTICADO` con mensaje genérico ("Token inválido o expirado"). El motivo exacto solo va al log.
- Resultado en la petición:

```js
req.usuario = { id: payload.sub, roles: payload.roles };  // formato de roles [POR ACORDAR]
```

## 4. Roles **[POR ACORDAR]** con usuario-service

Nombres provisionales. Si usuario-service usa otros, se mapean en un solo lugar (`src/core/roles.js`).

| Rol | Quién es | Canal que le corresponde |
|---|---|---|
| `CLIENTE` | Titular que usa la app | `APP` |
| `AGENTE` | Agente corresponsal | `AGENTE` |
| `CAJERO` | Personal de ventanilla en agencia | `VENTANILLA` |
| `SUPERVISOR` | Supervisa operaciones | — |
| `AUDITOR` | Solo lectura para control | — |

## 5. Matriz de permisos

| Operación | CLIENTE | AGENTE | CAJERO | SUPERVISOR | AUDITOR |
|---|---|---|---|---|---|
| `POST /depositos` | sí | sí | sí | no | no |
| `GET /depositos/:id` | propios¹ | propios¹ | propios¹ | todos | todos |
| `GET /depositos?cuenta=` | si es titular² | no **[DECISIÓN EQUIPO]** | no **[DECISIÓN EQUIPO]** | todos | todos |

¹ *Propios* = los que registró (`usuario_id = sub`) o, para `CLIENTE`, los hechos a una cuenta de la que es titular.
² Titularidad según `titularId` que devuelve billetera al consultar la cuenta. Ver `30-integraciones/billetera.md`.

- **RN-SEG-05** `requiereRol` responde 403 `PROHIBIDO` si ningún rol del usuario está en la lista de la ruta.
- **RN-SEG-06** Cuando un usuario pide un depósito que existe pero no puede ver, se responde **404** `DEPOSITO_NO_ENCONTRADO` (no 403), para no revelar que el ID existe.

## 6. Canal del depósito — **[DECISIÓN EQUIPO]**

En v0.1 el canal venía en el cuerpo. Eso permite que un cliente de la app envíe `canal: "VENTANILLA"` y use el límite más alto.

- **RN-SEG-07** El canal se **deriva del rol** según la tabla de §4.
- **RN-SEG-08** Si el usuario tiene **un solo** rol con canal, ese es el canal; el campo `canal` del cuerpo se ignora si coincide y se rechaza con 403 `PROHIBIDO` si no coincide.
- **RN-SEG-09** Si el usuario tiene **varios** roles con canal (ej. AGENTE y CAJERO), el cuerpo debe traer `canal` y debe ser uno de los permitidos; si falta → 400 `VALIDACION_FALLIDA`.

## 7. Otras medidas

- `helmet()` activado; `express.json({ limit: '10kb' })`.
- CORS desactivado en el servicio: lo maneja el gateway.
- **[POR ACORDAR]** Si billetera exige autenticación servicio-a-servicio (ver `30-integraciones/billetera.md §5`).

## 8. Criterios de aceptación

- **CA-SEG-01** Sin header `Authorization` → 401 `NO_AUTENTICADO`.
- **CA-SEG-02** Token firmado con otra clave → 401.
- **CA-SEG-03** Token con `alg: none` → 401.
- **CA-SEG-04** Token expirado hace 1 minuto → 401; expirado hace 10 segundos → aceptado (tolerancia).
- **CA-SEG-05** Token con `aud` distinto → 401.
- **CA-SEG-06** Usuario `AUDITOR` hace `POST /depositos` → 403 `PROHIBIDO`.
- **CA-SEG-07** Usuario `CLIENTE` envía `canal: "VENTANILLA"` → 403 `PROHIBIDO`; el depósito no se crea.
- **CA-SEG-08** Usuario `CLIENTE` sin `canal` en el cuerpo → el depósito se guarda con `canal = 'APP'`.
- **CA-SEG-09** Usuario `CLIENTE` pide `GET /depositos/:id` de un depósito ajeno a una cuenta ajena → 404.
- **CA-SEG-10** Un intento 401 queda registrado en `auditoria_depositos` con `usuario_id` NULL y `status_http = 401`.
