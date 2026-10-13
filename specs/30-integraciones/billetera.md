# Integración: billetera-service

Estado: **[POR ACORDAR]** con el equipo de billetera. Esto es la PROPUESTA de deposito-service. Mientras no se confirme, se implementa con un mock que respete este contrato.

Cliente: `src/core/clients/billeteraClient.js`. Es el único lugar con la URL de billetera (`BILLETERA_URL`).

## Reglas de la integración

- Solo HTTP. Nunca acceso a su base de datos.
- Toda llamada envía `Authorization` (token de servicio) y `X-Correlation-Id`.
- Timeout: 5 s (configurable `BILLETERA_TIMEOUT_MS`).
- Sin reintentos automáticos en `acreditar` (para no acreditar dos veces); la idempotencia la garantiza el `depositoId`.

## 1. Consultar cuenta

`GET /api/v1/cuentas/{numero}`

Respuesta 200:
```json
{
  "numero": "0011-0123-456789",
  "estado": "ACTIVA",
  "moneda": "PEN",
  "titularId": "usr-123"
}
```

| Respuesta de billetera | Qué hace depósito |
|---|---|
| 200 con estado ACTIVA | Continúa |
| 200 con otro estado | 422 CUENTA_INACTIVA |
| 404 | 422 CUENTA_NO_ENCONTRADA |
| 5xx o timeout | 502 BILLETERA_NO_DISPONIBLE (no se guarda depósito) |

## 2. Acreditar

`POST /api/v1/cuentas/{numero}/creditos`

Request:
```json
{
  "depositoId": "7f1c2e9a-...",
  "monto": "150.00",
  "moneda": "PEN",
  "concepto": "DEPOSITO"
}
```

- `monto` va como texto para no perder precisión.
- `depositoId` funciona como clave de idempotencia en billetera: si la recibe dos veces, no acredita dos veces.

| Respuesta de billetera | Estado del depósito | Respuesta al cliente |
|---|---|---|
| 201 / 200 | COMPLETADO | 201 |
| 409 (ese depositoId ya se acreditó) | COMPLETADO | 201 |
| 4xx | FALLIDO | 502 BILLETERA_NO_DISPONIBLE |
| 5xx o timeout | FALLIDO | 502 BILLETERA_NO_DISPONIBLE |

## Preguntas para el equipo de billetera

- [ ] ¿Aceptan estas rutas y campos?
- [ ] ¿Usarán `depositoId` como clave de idempotencia?
- [ ] ¿Cómo nos autenticamos entre servicios?
- [ ] ¿Qué valores puede tener `estado` de la cuenta?
