# 06 · Configuración y observabilidad

| Campo | Valor |
|---|---|
| Versión | 0.2 |
| Archivos | `src/core/config.js`, `src/core/logger.js`, `src/core/middlewares/correlationId.js` |

## 1. Variables de entorno

`config.js` las lee y valida **al arrancar**. Si una obligatoria falta o es inválida, el proceso termina (RN-CFG-01).

| Variable | Oblig. | Por defecto | Descripción |
|---|---|---|---|
| `PORT` | no | `3002` | Puerto HTTP |
| `NODE_ENV` | no | `development` | `development` / `test` / `production` |
| `DATABASE_URL` | **sí** | — | `postgres://usuario:clave@host:5432/depositos_db` |
| `DB_POOL_MAX` | no | `10` | Conexiones máximas del pool |
| `BILLETERA_URL` | **sí** | — | URL base de billetera-service (vía red interna, no vía gateway) |
| `BILLETERA_TIMEOUT_CONSULTA_MS` | no | `3000` | Timeout de consultar cuenta |
| `BILLETERA_TIMEOUT_ACREDITAR_MS` | no | `5000` | Timeout de acreditar |
| `JWT_PUBLIC_KEY` o `JWT_JWKS_URL` | **sí (uno)** | — | Clave pública para verificar tokens. **[POR ACORDAR]** con usuario-service |
| `JWT_ALGORITMO` | no | `RS256` | **[POR ACORDAR]** |
| `JWT_ISSUER` | **sí** | — | **[POR ACORDAR]** |
| `JWT_AUDIENCE` | **sí** | — | **[POR ACORDAR]** |
| `IDEMPOTENCIA_TTL_HORAS` | no | `24` | Vida de una clave de idempotencia |
| `IDEMPOTENCIA_BLOQUEO_SEG` | no | `60` | Tras este tiempo, una clave `EN_PROCESO` se considera abandonada |
| `UIF_UMBRAL` | **sí** si `alerta-uif` activo | — | Monto umbral. **[PENDIENTE]** verificar normativa SBS |
| `UIF_UMBRAL_MONEDA` | no | `USD` | Moneda del umbral |
| `MANIFEST_PATH` | no | `specs/manifest.yaml` | Ruta del manifest |
| `LOG_LEVEL` | no | `info` | `debug` / `info` / `warn` / `error` |

- **RN-CFG-01** Fallar rápido: mejor no arrancar que arrancar mal configurado.
- **RN-CFG-02** Ningún secreto en el repositorio. Se entrega un `.env.example` con todas las variables y valores ficticios.
- **RN-CFG-03** El resto del código no lee `process.env`; usa el objeto `config`.

## 2. Correlation ID

- **RN-OBS-01** El middleware `correlationId` es el **primero** de la cadena.
- Si llega el header `X-Correlation-Id` (lo pone el gateway) y tiene 8–64 caracteres `[A-Za-z0-9-]`, se usa; si no, se genera un UUID v4.
- Se guarda en `req.correlationId`, se devuelve en el header de respuesta `X-Correlation-Id`, va en todos los logs, en `auditoria_depositos.correlation_id` y se **propaga** a billetera en el mismo header.

## 3. Logs

- Formato JSON con `pino` (`pino-http` para las peticiones).
- Cada línea incluye `correlationId`, `usuarioId` (si hay), `metodo`, `ruta`, `status`, `duracionMs`.
- **RN-OBS-02** Nunca se loguean: el token JWT, el header `Authorization`, contraseñas ni el cuerpo completo de respuestas de billetera. Se usa `redact` de pino para `req.headers.authorization`.
- Las llamadas a billetera se loguean con: operación, `depositoId`, status, duración.

## 4. Salud

| Endpoint | Auth | Responde | Uso |
|---|---|---|---|
| `GET /health` | no | `200 {"estado":"ok"}` si el proceso vive | Liveness (Docker / orquestador) |
| `GET /ready` | no | `200` si `SELECT 1` funciona; `503` si no | Readiness; el gateway no envía tráfico si falla |

`/ready` **no** consulta a billetera: si billetera cae, depósito igual puede responder consultas e historial.

Estos endpoints no se exponen a través del API Gateway (solo red interna). **[POR ACORDAR]** con quien configure el gateway.

## 5. Docker

- Imagen base `node:20-alpine`, usuario no root, `npm ci --omit=dev`.
- Copia `src/`, `package*.json` y `specs/manifest.yaml` (lo lee el Cargador).
- `HEALTHCHECK` contra `/health`.

## 6. Criterios de aceptación

- **CA-OBS-01** Dada una petición sin `X-Correlation-Id`, entonces la respuesta trae uno con formato UUID.
- **CA-OBS-02** Dada una petición con `X-Correlation-Id: abc12345`, entonces la respuesta trae el mismo valor y la llamada a billetera también lo lleva.
- **CA-OBS-03** Dado un log de una petición autenticada, entonces no contiene el token.
- **CA-OBS-04** Dada la base caída, `GET /ready` responde 503 y `GET /health` responde 200.
