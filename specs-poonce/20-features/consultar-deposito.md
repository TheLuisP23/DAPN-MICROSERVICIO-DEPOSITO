# Consultar depósito

| Campo | Valor |
|---|---|
| Versión | 0.2 |
| Tipo | Feature base |
| Carpeta | `src/modules/consultar-deposito/` (routes, controller, service) |
| Depende de | `10-transversales/seguridad.md`, `03-modelo-de-datos.md` |

## 1. Propósito

Devolver el detalle y el estado actual de un depósito. Es también la forma de saber cómo terminó un depósito que respondió 502.

## 2. Contrato

```http
GET /depositos/{id}
Authorization: Bearer <jwt>
```

Roles: todos (`CLIENTE`, `AGENTE`, `CAJERO`, `SUPERVISOR`, `AUDITOR`), con la visibilidad de `seguridad.md §5`.

### Respuesta 200

```json
{
  "id": "0b6f4d2a-3c1e-4a8b-9f7d-2e5c6a1b0d93",
  "cuentaDestino": "1923456789012",
  "monto": "150.00",
  "moneda": "USD",
  "canal": "APP",
  "referencia": "Pago de alquiler octubre",
  "estado": "COMPLETADO",
  "motivoFallo": null,
  "acreditado": { "monto": "556.80", "moneda": "PEN" },
  "conversion": { "tipoCambio": "3.7120", "operacion": "COMPRA" },
  "registradoPor": "u-123",
  "creadoEn": "2026-10-13T19:49:00.000Z",
  "actualizadoEn": "2026-10-13T19:49:01.200Z"
}
```

- `acreditado` es `null` si el estado no es `COMPLETADO`.
- `conversion` es `null` si no hubo conversión.
- `registradoPor` solo se incluye para `SUPERVISOR` y `AUDITOR`.

### Errores

`VALIDACION_FALLIDA` 400 (id no es UUID) · `NO_AUTENTICADO` 401 · `DEPOSITO_NO_ENCONTRADO` 404 · `BILLETERA_NO_DISPONIBLE` 502 (solo si hace falta verificar titularidad).

## 3. Reglas de negocio

- **RN-CON-01** `id` debe ser UUID; si no, 400 (no se consulta la base).
- **RN-CON-02** Visibilidad:
  - `SUPERVISOR`, `AUDITOR` → cualquier depósito.
  - Cualquier rol → los que registró (`usuario_id = req.usuario.id`).
  - `CLIENTE` → además, los hechos a cuentas de las que es titular. Solo en este caso se consulta billetera (`titularId`); si el depósito lo registró él mismo, no se llama a billetera.
- **RN-CON-03** Sin permiso → 404 `DEPOSITO_NO_ENCONTRADO` (no 403), igual que si no existiera.
- **RN-CON-04** Se lee con un solo `SELECT` con `LEFT JOIN conversiones_deposito`. El service no conoce SQL: pide al repository `obtenerConConversion(id)`.

## 4. Criterios de aceptación

- **CA-CON-01** Depósito existente, consultado por quien lo registró → 200 con todos los campos del contrato.
- **CA-CON-02** `id` = `"abc"` → 400.
- **CA-CON-03** UUID inexistente → 404 `DEPOSITO_NO_ENCONTRADO`.
- **CA-CON-04** `CLIENTE` que no lo registró ni es titular → 404.
- **CA-CON-05** `CLIENTE` titular de la cuenta destino (según billetera fake) → 200.
- **CA-CON-06** `AUDITOR` → 200 e incluye `registradoPor`.
- **CA-CON-07** Depósito `FALLIDO` → `motivoFallo` lleno y `acreditado = null`.
- **CA-CON-08** Depósito con conversión → `conversion` con `tipoCambio` y `operacion`.

## 5. Fuera de alcance

- Consultar varios IDs en una sola petición.
