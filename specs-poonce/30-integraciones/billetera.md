# Integración con billetera-service

| Campo | Valor |
|---|---|
| Versión | 0.2 |
| Archivo | `src/core/clients/billeteraClient.js` |
| Contraparte | Equipo billetera |
| Estado | **[POR ACORDAR]** — esta es la **propuesta** de depósito para negociar con billetera |
| Depende de | `04-estados-deposito.md`, `05-errores.md` |

## 1. Propósito

Definir las dos (más una opcional) llamadas que deposito-service hace a billetera-service, y cómo se interpretan sus respuestas. Este spec es el documento que se lleva a la reunión con el equipo billetera.

## 2. Reglas generales del cliente

- **RN-BIL-01** Solo `billeteraClient.js` conoce `BILLETERA_URL` y las rutas de billetera.
- **RN-BIL-02** Toda llamada envía `X-Correlation-Id` y tiene timeout (`AbortController`).
- **RN-BIL-03** El cliente traduce las respuestas de billetera a resultados del dominio de depósito; el service nunca ve códigos HTTP de billetera.
- **RN-BIL-04** Se usa la red interna entre servicios, no el API Gateway público. **[POR ACORDAR]**

## 3. Consultar cuenta — **[POR ACORDAR #1]**

```http
GET {BILLETERA_URL}/cuentas/{numero}
X-Correlation-Id: ...
```

Respuesta esperada `200`:

```json
{
  "numero": "1923456789012",
  "moneda": "PEN",
  "estado": "ACTIVA",
  "titularId": "u-456"
}
```

| Campo | Para qué lo usa depósito |
|---|---|
| `moneda` | Verificación final del pipeline y conversión |
| `estado` | `ACTIVA` / `INACTIVA` / `BLOQUEADA` **[POR ACORDAR]** valores |
| `titularId` | Autorizar a `CLIENTE` en consulta e historial. **Se pide a billetera agregarlo** (debe coincidir con el `sub` del JWT de usuario-service) |

Depósito **no** necesita (ni debe recibir) el saldo.

| Respuesta de billetera | Resultado en el cliente |
|---|---|
| 200 | `{ ok: true, cuenta }` |
| 404 | `{ ok: false, motivo: 'NO_EXISTE' }` |
| 5xx, timeout (`BILLETERA_TIMEOUT_CONSULTA_MS`), red | lanza `ErrorIntegracion('BILLETERA_NO_DISPONIBLE')` |

- **RN-BIL-05** La consulta es de solo lectura: se reintenta **una vez** ante error de red o 503 (con 200 ms de espera).

## 4. Acreditar — **[POR ACORDAR #2]**

```http
POST {BILLETERA_URL}/cuentas/{numero}/acreditaciones
Idempotency-Key: {depositoId}
X-Correlation-Id: ...
Content-Type: application/json

{
  "depositoId": "0b6f4d2a-3c1e-4a8b-9f7d-2e5c6a1b0d93",
  "monto": "556.80",
  "moneda": "PEN",
  "concepto": "DEPOSITO",
  "referencia": "Pago de alquiler octubre"
}
```

**Requisito clave para billetera:** la operación debe ser **idempotente por `depositoId`**. Si recibe el mismo `depositoId` dos veces, la segunda vez **no** acredita y responde igual que la primera.

| Respuesta de billetera | Significado | Resultado en el cliente | Estado del depósito |
|---|---|---|---|
| 201 | Acreditado ahora | `{ resultado: 'ACREDITADO' }` | `COMPLETADO` |
| 200 | Ya estaba acreditado (repetido) | `{ resultado: 'ACREDITADO' }` | `COMPLETADO` |
| 4xx de negocio (422, 409 por cuenta bloqueada, etc.) | No acreditó y no lo hará | `{ resultado: 'RECHAZADO', motivo: <codigo de billetera> }` | `FALLIDO` |
| 400 | Bug nuestro (cuerpo mal formado) | `{ resultado: 'RECHAZADO', motivo: 'SOLICITUD_INVALIDA' }` + log `error` | `FALLIDO` |
| 5xx, timeout (`BILLETERA_TIMEOUT_ACREDITAR_MS`), red | **Desconocido** | `{ resultado: 'INCIERTO' }` | sigue `PROCESANDO` |

- **RN-BIL-06** `acreditar` **no** se reintenta automáticamente dentro de la misma petición (se responde rápido al cliente con 502). El reintento lo hace el cliente con su Idempotency-Key o la reconciliación.
- **RN-BIL-07** Los montos viajan como **string** con 2 decimales.

## 5. Consultar acreditación (opcional, para reconciliación) — **[FASE 2] [POR ACORDAR]**

```http
GET {BILLETERA_URL}/acreditaciones/{depositoId}
```

`200` con el resultado si existe; `404` si billetera nunca la recibió. Permite cerrar depósitos `PROCESANDO` sin volver a llamar a `acreditar`. No es indispensable: como `acreditar` es idempotente, reintentarlo es seguro.

## 6. Autenticación entre servicios — **[POR ACORDAR]**

Opciones a definir con billetera y usuario:

1. Reenviar el JWT del usuario (simple; billetera valida lo mismo que depósito).
2. Token de servicio emitido por usuario-service (`client_credentials`), con rol `SERVICIO_DEPOSITO`.
3. Red interna sin autenticación (aceptable solo en el entorno del curso).

Propuesta: **opción 2**; para la primera entrega, **opción 1**.

## 7. Billetera simulada para tests

`tests/fakes/billeteraFake.js`: servidor Express en un puerto aleatorio que permite configurar por test:

- cuentas existentes (número, moneda, estado, titularId);
- respuesta de `acreditar` (201, 200, 422, 500, demora en ms);
- registro de las llamadas recibidas (para verificar `depositoId` e idempotencia).

## 8. Criterios de aceptación

- **CA-BIL-01** Consulta 200 → el cliente devuelve `moneda`, `estado`, `titularId`.
- **CA-BIL-02** Consulta con error de red la primera vez y 200 la segunda → éxito (un reintento).
- **CA-BIL-03** Acreditar con demora mayor al timeout → `INCIERTO` en ≤ timeout + 100 ms.
- **CA-BIL-04** Acreditar 200 (repetido) → `ACREDITADO`.
- **CA-BIL-05** Acreditar 422 → `RECHAZADO` con el motivo de billetera.
- **CA-BIL-06** Toda llamada lleva `X-Correlation-Id`; `acreditar` lleva `Idempotency-Key = depositoId`.

## 9. Lista para la reunión con billetera

1. Ruta y forma de **consultar cuenta**; agregar `titularId`; valores posibles de `estado`.
2. Ruta y forma de **acreditar**; confirmar **idempotencia por `depositoId`** y que el repetido responda 200.
3. Códigos de error de negocio que devuelve y su formato.
4. Formato del número de cuenta (longitud, solo dígitos).
5. Autenticación entre servicios.
6. (Opcional) consultar acreditación por `depositoId`.
