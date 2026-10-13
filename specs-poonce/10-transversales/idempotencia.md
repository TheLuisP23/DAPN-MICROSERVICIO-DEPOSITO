# Idempotencia

| Campo | Valor |
|---|---|
| Versión | 0.2 |
| Archivos | `src/core/middlewares/idempotencia.js`, `src/core/repositories/idempotenciaRepository.js` |
| Tabla | `claves_idempotencia` |
| Aplica a | Solo `POST /depositos` |
| Depende de | `03-modelo-de-datos.md`, `04-estados-deposito.md` |

## 1. Propósito

Que una misma solicitud de depósito, enviada varias veces (doble clic, reintento de red, reintento automático del front), produzca **un solo depósito** y **la misma respuesta**.

Hay dos niveles de protección:

1. **Entrada** (este spec): `Idempotency-Key` del cliente → un solo depósito por clave.
2. **Salida** (`30-integraciones/billetera.md`): `depositoId` hacia billetera → una sola acreditación por depósito.

## 2. Contrato con el cliente

- Header **obligatorio** `Idempotency-Key` en `POST /depositos`.
- Formato: 8–100 caracteres `[A-Za-z0-9_-]`. Se recomienda un UUID v4 generado por el front **una vez por intención de depósito** (no por clic).
- Vida: `IDEMPOTENCIA_TTL_HORAS` (24 h por defecto). Pasado ese tiempo, la clave puede reutilizarse y se trata como nueva.
- La clave es **por usuario**: la misma clave de dos usuarios distintos son dos claves distintas.

## 3. Huella de la solicitud (`hash_solicitud`)

SHA-256 en hex del cuerpo **canonicalizado**: JSON con claves ordenadas alfabéticamente, sin espacios, y `monto` normalizado a string con 2 decimales (así `150.5` y `"150.50"` dan la misma huella).

## 4. Estados de la clave

| Estado | Significado |
|---|---|
| `EN_PROCESO` | Una petición la tomó y está trabajando. |
| `REINTENTABLE` | La petición terminó con resultado incierto (billetera no respondió). El depósito ya existe en `PROCESANDO`; el próximo reintento puede retomarlo. |
| `TERMINADO` | Hay respuesta definitiva guardada (`status_http` + `respuesta`). |

## 5. Algoritmo

```
1. Validar header (falta o mal formado → 400 IDEMPOTENCY_KEY_REQUERIDA).
2. Calcular hash.
3. INSERT … ON CONFLICT (usuario_id, clave) DO NOTHING RETURNING *.
   3a. Devolvió fila → clave nueva, la tomamos: continuar al controller.
   3b. No devolvió fila → leer la fila existente y decidir:

      expira_en < now()            → borrar la fila y volver a 3 (clave caducada = nueva).
      hash distinto                → 422 IDEMPOTENCIA_CONFLICTO.
      TERMINADO                    → devolver status_http + respuesta guardados,
                                     con header  Idempotent-Replayed: true.
      EN_PROCESO y actualizado_en
        hace < IDEMPOTENCIA_BLOQUEO_SEG → 409 IDEMPOTENCIA_EN_PROCESO.
      EN_PROCESO abandonada (más antigua)
        o REINTENTABLE             → intentar tomarla:
                                     UPDATE … SET estado='EN_PROCESO', actualizado_en=now()
                                     WHERE usuario_id=$1 AND clave=$2 AND estado=<el leído>
                                       AND actualizado_en=<el leído>
                                     · rowCount=1 → continuar como REINTENTO con su deposito_id.
                                     · rowCount=0 → otra petición la tomó: 409 IDEMPOTENCIA_EN_PROCESO.
```

El middleware deja en la petición:

```js
req.idempotencia = { clave, esReintento: boolean, depositoId: uuid | null };
```

### Al terminar (lo llama RegistrarDepositoService a través del repository)

| Resultado del registro | Qué se guarda en la clave |
|---|---|
| Se creó el depósito (aún sin respuesta de billetera) | `deposito_id = <id>` (en la misma transacción que el INSERT del depósito) |
| 201 depósito completado | `TERMINADO`, status 201, respuesta |
| 422 de negocio (validador, cuenta inactiva, billetera rechazó…) | `TERMINADO`, status 422, respuesta |
| 400 de forma | Se **borra** la clave (el cliente puede corregir y reenviar con la misma clave) |
| 502 `BILLETERA_NO_DISPONIBLE` | `REINTENTABLE` (no se guarda respuesta) |
| 503 / 500 antes de crear el depósito | Se **borra** la clave |
| 503 / 500 después de crear el depósito | `REINTENTABLE` |

> **[DECISIÓN EQUIPO]** v0.1 decía "clave liberada para reintento". Aquí la clave **no se libera**: queda `REINTENTABLE` y **atada al mismo `deposito_id`**. Así un reintento retoma el mismo depósito en lugar de crear otro (evita el doble abono descrito en `04-estados-deposito.md §2`).

### Reintento (`esReintento = true` y `depositoId` no nulo)

RegistrarDepositoService **no** vuelve a ejecutar validadores ni transformadores. Lee el depósito:

- `PROCESANDO` → vuelve a llamar a `acreditar` con el mismo `depositoId` y sigue el flujo normal desde ahí.
- `COMPLETADO` / `FALLIDO` (lo resolvió la reconciliación) → construye la respuesta correspondiente y marca la clave `TERMINADO`.
- `PENDIENTE` (se cayó entre guardar y pasar a PROCESANDO) → pasa a `PROCESANDO` y llama a `acreditar`.

## 6. Limpieza

**[FASE 2]** Tarea programada que borra claves con `expira_en < now() - INTERVAL '1 day'` y estado `TERMINADO`. Mientras tanto, la caducidad se resuelve al vuelo (paso 3b).

## 7. Criterios de aceptación

- **CA-IDE-01** POST sin `Idempotency-Key` → 400 `IDEMPOTENCY_KEY_REQUERIDA`; no se crea depósito.
- **CA-IDE-02** Dos POST secuenciales con la misma clave y el mismo cuerpo → la segunda devuelve el **mismo** status y cuerpo que la primera, con `Idempotent-Replayed: true`; existe **un** solo depósito; billetera recibió **una** llamada a acreditar.
- **CA-IDE-03** Dos POST **simultáneos** con la misma clave → uno responde 201 y el otro 409 `IDEMPOTENCIA_EN_PROCESO`; existe un solo depósito.
- **CA-IDE-04** Misma clave, distinto `monto` → 422 `IDEMPOTENCIA_CONFLICTO`.
- **CA-IDE-05** Misma clave, `monto: 150.5` y luego `monto: "150.50"` → se trata como la misma solicitud (replay).
- **CA-IDE-06** Usuarios A y B con la misma clave → dos depósitos distintos.
- **CA-IDE-07** Primera petición con timeout de billetera (502) → la clave queda `REINTENTABLE` con `deposito_id`. Reintento con la misma clave y billetera OK → 201 con el **mismo** `id` de depósito; existe un solo depósito.
- **CA-IDE-08** Primera petición rechazada con 422 `LIMITE_CANAL_EXCEDIDO` → reintento con la misma clave devuelve el mismo 422 (replay), sin volver a ejecutar el pipeline.
- **CA-IDE-09** Primera petición con 400 `VALIDACION_FALLIDA` → reintento corregido con la misma clave se procesa como nuevo.
- **CA-IDE-10** Clave con `expira_en` vencido → se procesa como nueva.
