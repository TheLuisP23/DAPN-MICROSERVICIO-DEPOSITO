# Feature: historial-depositos

Tipo: endpoint · Ruta: `GET /api/v1/cuentas/{numero}/depositos` · Código: `src/modules/historial-depositos/`

## Descripción

Lista los depósitos de una cuenta, del más reciente al más antiguo, paginados.

## Middlewares

`correlationId` → `autenticar` → `requiereRol('CLIENTE','AGENTE','CAJERO')` → `auditoria('LISTAR_DEPOSITOS')` → controller.

## Parámetros

| Parámetro | Por defecto | Regla |
|---|---|---|
| pagina | 1 | >= 1 |
| tamano | 20 | 1 a 100 |
| estado | (todos) | Opcional, uno de los 4 estados |
| desde / hasta | (sin límite) | Opcional, fecha ISO |

## Reglas

| Id | Regla |
|---|---|
| HD-01 | CLIENTE solo ve el historial de cuentas propias; si no, 403 SIN_PERMISO. |
| HD-02 | AGENTE y CAJERO solo ven los depósitos que ellos registraron en esa cuenta. |
| HD-03 | Orden: creado_en descendente. |
| HD-04 | No incluye `extensiones` (para que la lista sea liviana); el detalle se ve con consultar-deposito. |

## Escenarios

**E3.1 Historial de cuenta propia** → 200 con lista y total.
**E3.2 Cuenta sin depósitos** → 200 con lista vacía y total 0.
**E3.3 Paginación** → con 25 depósitos y tamano 20, la página 2 trae 5.
**E3.4 Filtro por estado** → solo depósitos con ese estado.
**E3.5 CLIENTE consulta cuenta ajena** → 403 SIN_PERMISO.
**E3.6 tamano 500** → 400 FORMATO_INVALIDO.
