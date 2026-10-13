# 05 · Errores: formato común y catálogo

| Campo | Valor |
|---|---|
| Versión | 0.2 |
| Archivo | `src/core/errores.js` |
| Depende de | `01-arquitectura.md` |

## 1. Formato común de respuesta

Todas las respuestas de error del servicio tienen esta forma, sin excepción:

```json
{
  "error": {
    "codigo": "LIMITE_CANAL_EXCEDIDO",
    "mensaje": "El monto supera el máximo permitido para el canal APP en PEN.",
    "detalles": { "canal": "APP", "moneda": "PEN", "montoMaximo": "5000.00" },
    "correlationId": "7f3c1e2a-..."
  }
}
```

- `codigo`: estable, en MAYÚSCULAS_CON_GUIONES. El frontend decide por `codigo`, **nunca** por `mensaje`.
- `mensaje`: legible para una persona, en español. Puede cambiar sin aviso.
- `detalles`: opcional. Para `VALIDACION_FALLIDA` es una lista `[{ campo, problema }]`.
- `correlationId`: siempre presente (también va en el header `X-Correlation-Id`).

- **RN-ERR-01** Nunca se devuelve el stack trace, SQL, nombres de tablas ni URLs internas al cliente.
- **RN-ERR-02** Todo error no controlado se convierte en 500 `ERROR_INTERNO` y se loguea completo (con stack) en el servidor.
- **RN-ERR-03** Errores 4xx se loguean en nivel `warn`; 5xx en nivel `error`.

## 2. Jerarquía en `errores.js`

```js
class ErrorApp extends Error { constructor(codigo, mensaje, status, detalles) { ... } }

class ErrorValidacion          extends ErrorApp {}  // 400
class ErrorAutenticacion       extends ErrorApp {}  // 401
class ErrorAutorizacion        extends ErrorApp {}  // 403
class ErrorNoEncontrado        extends ErrorApp {}  // 404
class ErrorConflicto           extends ErrorApp {}  // 409
class ErrorNegocio             extends ErrorApp {}  // 422
class ErrorIntegracion         extends ErrorApp {}  // 502 / 504
class ErrorServicioNoDisponible extends ErrorApp {} // 503
class ErrorInterno             extends ErrorApp {}  // 500
```

El **manejador de errores** (último middleware de Express) es el único lugar que convierte un error en respuesta HTTP.

## 3. Catálogo

| Código | HTTP | Cuándo | Lo produce |
|---|---|---|---|
| `VALIDACION_FALLIDA` | 400 | Cuerpo/params/query con forma inválida | Controllers |
| `IDEMPOTENCY_KEY_REQUERIDA` | 400 | POST sin header `Idempotency-Key` o con formato inválido | Middleware idempotencia |
| `NO_AUTENTICADO` | 401 | Falta token, token inválido o expirado | Middleware autenticar |
| `PROHIBIDO` | 403 | El rol no permite la operación | Middleware requiereRol |
| `DEPOSITO_NO_ENCONTRADO` | 404 | No existe el depósito o el usuario no puede verlo | ConsultarDepositoService |
| `IDEMPOTENCIA_EN_PROCESO` | 409 | Llega la misma clave mientras la primera petición sigue en curso | Middleware idempotencia |
| `IDEMPOTENCIA_CONFLICTO` | 422 | Misma clave con un cuerpo distinto | Middleware idempotencia |
| `CUENTA_NO_ENCONTRADA` | 422 | Billetera dice que la cuenta no existe | RegistrarDepositoService |
| `CUENTA_INACTIVA` | 422 | Cuenta en estado distinto de `ACTIVA` | RegistrarDepositoService |
| `MONEDA_NO_COINCIDE` | 422 | Tras transformadores, la moneda a acreditar ≠ moneda de la cuenta | Pipeline (verificación final) |
| `LIMITE_CANAL_EXCEDIDO` | 422 | Monto mayor que el máximo del canal | Módulo limites-por-canal |
| `DEPOSITO_RECHAZADO` | 422 | Billetera rechazó la acreditación (4xx de negocio) | RegistrarDepositoService |
| `TIPO_CAMBIO_NO_DISPONIBLE` | 503 | No hay tipo de cambio vigente configurado | Módulo conversion-moneda |
| `BILLETERA_NO_DISPONIBLE` | 502 | Timeout, 5xx o error de red con billetera. **Reintentable** con la misma Idempotency-Key | BilleteraClient |
| `SERVICIO_NO_DISPONIBLE` | 503 | Base de datos no disponible | db.js / manejador |
| `ERROR_INTERNO` | 500 | Cualquier error no previsto | Manejador |

> 422 se usa para "la petición está bien formada pero el negocio no la acepta". 400 para "la petición está mal formada".

## 4. Criterios de aceptación

- **CA-ERR-01** Dado un error lanzado con `throw new Error('x')` en un service, entonces la respuesta es 500 con `codigo = ERROR_INTERNO`, sin stack y con `correlationId`.
- **CA-ERR-02** Dada cualquier respuesta de error, entonces el cuerpo tiene `error.codigo`, `error.mensaje` y `error.correlationId`.
- **CA-ERR-03** Dado un error de PostgreSQL (por ejemplo, violación de CHECK), entonces el mensaje al cliente no contiene el nombre de la tabla ni de la restricción.
- **CA-ERR-04** Dado un `ErrorNegocio`, entonces el log es de nivel `warn`; dado un `ErrorInterno`, de nivel `error` con stack.
