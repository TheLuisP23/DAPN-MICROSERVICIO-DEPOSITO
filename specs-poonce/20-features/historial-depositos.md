# Historial de depósitos de una cuenta

| Campo | Valor |
|---|---|
| Versión | 0.2 |
| Tipo | Feature base |
| Carpeta | `src/modules/historial-depositos/` (routes, controller, service) |
| Depende de | `10-transversales/seguridad.md`, `30-integraciones/billetera.md` |

## 1. Propósito

Listar, paginados y filtrables, los depósitos hechos a una cuenta.

## 2. Contrato — **[DECISIÓN EQUIPO]** cambio de ruta

v0.1 proponía `GET /cuentas/:numero/depositos`. En el API Gateway, el prefijo `/cuentas/*` se enrutaría naturalmente a billetera-service (dueño de las cuentas), lo que obliga a reglas de enrutamiento por sufijo. Se propone mantener todo lo de depósito bajo `/depositos`:

```http
GET /depositos?cuenta=1923456789012&desde=2026-10-01&hasta=2026-10-31&estado=COMPLETADO&pagina=1&limite=20
Authorization: Bearer <jwt>
```

| Parámetro | Oblig. | Validación | Por defecto |
|---|---|---|---|
| `cuenta` | sí | mismo formato que `cuentaDestino` | — |
| `desde` | no | fecha ISO-8601 (inclusive) | sin límite |
| `hasta` | no | fecha ISO-8601 (exclusiva); `hasta > desde` | sin límite |
| `estado` | no | `PENDIENTE`, `PROCESANDO`, `COMPLETADO`, `FALLIDO` | todos |
| `pagina` | no | entero ≥ 1 | 1 |
| `limite` | no | entero 1–100 | 20 |

Si `desde`/`hasta` vienen solo como fecha (`2026-10-01`), se interpretan en hora de Lima (UTC-5) a las 00:00. **[DECISIÓN EQUIPO]**

Roles: `CLIENTE` (si es titular), `SUPERVISOR`, `AUDITOR`.

### Respuesta 200

```json
{
  "datos": [
    {
      "id": "0b6f4d2a-...",
      "monto": "150.00",
      "moneda": "USD",
      "canal": "APP",
      "referencia": "Pago de alquiler octubre",
      "estado": "COMPLETADO",
      "acreditado": { "monto": "556.80", "moneda": "PEN" },
      "creadoEn": "2026-10-13T19:49:00.000Z"
    }
  ],
  "paginacion": { "pagina": 1, "limite": 20, "total": 37, "totalPaginas": 2 }
}
```

Sin resultados → 200 con `datos: []` y `total: 0` (no 404).

### Errores

`VALIDACION_FALLIDA` 400 · `NO_AUTENTICADO` 401 · `PROHIBIDO` 403 · `BILLETERA_NO_DISPONIBLE` 502.

## 3. Reglas de negocio

- **RN-HIS-01** Para `CLIENTE` se consulta billetera: si `titularId !== req.usuario.id` → 403 `PROHIBIDO`. Si la cuenta no existe → 403 también (no se revela si existe).
- **RN-HIS-02** `SUPERVISOR` y `AUDITOR` no requieren llamar a billetera.
- **RN-HIS-03** Orden: `creado_en DESC, id DESC` (desempate estable para que la paginación no repita ni salte filas).
- **RN-HIS-04** `total` se obtiene con `COUNT(*)` usando los mismos filtros. **[FASE 2]** Si el volumen crece, cambiar a paginación por cursor.
- **RN-HIS-05** Usa el índice `ix_depositos_cuenta_fecha`.

## 4. Criterios de aceptación

- **CA-HIS-01** Cuenta con 25 depósitos, `limite=20` → página 1 con 20, página 2 con 5, `total = 25`, `totalPaginas = 2`, sin repetidos entre páginas.
- **CA-HIS-02** Orden descendente por fecha.
- **CA-HIS-03** `estado=FALLIDO` → solo fallidos.
- **CA-HIS-04** `desde` y `hasta` filtran correctamente (inclusivo/exclusivo).
- **CA-HIS-05** `limite=500` → 400.
- **CA-HIS-06** `hasta` anterior a `desde` → 400.
- **CA-HIS-07** Falta `cuenta` → 400.
- **CA-HIS-08** `CLIENTE` no titular → 403.
- **CA-HIS-09** `AUDITOR` → 200 sin llamar a billetera.
- **CA-HIS-10** Cuenta sin depósitos → 200 con lista vacía.

## 5. Fuera de alcance

- Historial por usuario registrador ("mis depósitos registrados" para AGENTE/CAJERO). **[FASE 2]**
- Exportar a CSV/PDF.
