# Transversal: idempotencia

Middleware: `idempotencia`, solo en POST /depositos. Archivo: `src/core/middlewares/idempotencia.js`.

## Objetivo

Que un reintento del cliente nunca acredite dos veces.

## Flujo

1. Exigir header `Idempotency-Key` (8 a 100 caracteres). Si falta: 400 `IDEMPOTENCY_KEY_REQUERIDA`.
2. Calcular `hash_solicitud` = SHA-256 de método + ruta + cuerpo.
3. Reservar la clave de forma atómica:
   `INSERT ... VALUES (clave, ..., 'EN_PROCESO') ON CONFLICT (clave) DO NOTHING`.
4. Si se insertó: petición nueva, continúa.
5. Si ya existía:
   - hash distinto → 409 `IDEMPOTENCY_KEY_REUTILIZADA`.
   - estado EN_PROCESO → 409 `SOLICITUD_EN_PROCESO`.
   - estado TERMINADO → devolver el status y cuerpo guardados, con header `Idempotency-Replayed: true`.
6. Al responder:
   - status < 500 → guardar status, cuerpo y deposito_id; estado TERMINADO.
   - status >= 500 → borrar la clave para permitir el reintento.

## Estados de la clave (distintos a los del depósito)

| Estado | Significado |
|---|---|
| EN_PROCESO | La petición original se está atendiendo |
| TERMINADO | Ya hay respuesta guardada |

## Tabla

```sql
CREATE TABLE claves_idempotencia (
    clave           VARCHAR(100) PRIMARY KEY,
    usuario_id      VARCHAR(50)  NOT NULL,
    hash_solicitud  CHAR(64)     NOT NULL,
    estado          VARCHAR(20)  NOT NULL CHECK (estado IN ('EN_PROCESO','TERMINADO')),
    status_http     SMALLINT,
    respuesta       JSONB,
    deposito_id     UUID REFERENCES depositos(id),
    creado_en       TIMESTAMPTZ  NOT NULL DEFAULT now()
);
```

Las claves se conservan 7 días (configurable `IDEMPOTENCIA_DIAS`). Una tarea programada borra las más antiguas.

## Escenarios

**I1.** Clave nueva → se procesa normal.
**I2.** Misma clave y mismos datos, después de terminar → misma respuesta, `Idempotency-Replayed: true`, billetera no recibe otra acreditación.
**I3.** Misma clave, otros datos → 409 `IDEMPOTENCY_KEY_REUTILIZADA`.
**I4.** Misma clave mientras la primera aún se procesa → 409 `SOLICITUD_EN_PROCESO`.
**I5.** Dos peticiones simultáneas con la misma clave → solo una se procesa.
**I6.** Primera respuesta fue 502 → el reintento con la misma clave se procesa como nuevo.
**I7.** Sin header → 400.

## Ejemplo

Ana deposita S/ 100 con clave `k-555`. Se acredita, pero se corta el internet antes de recibir la respuesta. La app reintenta con `k-555`. El servicio devuelve el 201 original sin acreditar de nuevo. Resultado: S/ 100, no S/ 200.
