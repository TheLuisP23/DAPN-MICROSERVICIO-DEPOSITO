# Feature: consultar-deposito

Tipo: endpoint · Ruta: `GET /api/v1/depositos/{id}` · Código: `src/modules/consultar-deposito/`

## Descripción

Devuelve un depósito por su id, con su estado actual y los datos de extensiones activas.

## Middlewares

`correlationId` → `autenticar` → `requiereRol('CLIENTE','AGENTE','CAJERO')` → `auditoria('CONSULTAR_DEPOSITO')` → controller.

## Reglas

| Id | Regla |
|---|---|
| CD-01 | `id` debe ser un UUID válido; si no, 400 FORMATO_INVALIDO. |
| CD-02 | CLIENTE solo ve depósitos de cuentas propias (se verifica con billetera, titularId). |
| CD-03 | AGENTE y CAJERO solo ven depósitos que ellos registraron (`usuario_id`). |
| CD-04 | Si no tiene permiso, responder 404 (no 403), para no revelar que el depósito existe. |
| CD-05 | `extensiones` incluye los datos de cada feature activa que tenga información de ese depósito. |

## Escenarios

**E2.1 Consultar depósito propio** → 200 con el depósito.
**E2.2 Id inexistente** → 404 DEPOSITO_NO_ENCONTRADO.
**E2.3 Id con formato inválido** → 400 FORMATO_INVALIDO.
**E2.4 CLIENTE consulta depósito de otra persona** → 404 DEPOSITO_NO_ENCONTRADO.
**E2.5 Depósito con conversión** (conversion-moneda activa) → `extensiones.conversion-moneda` con los datos.
**E2.6 Mismo depósito con conversion-moneda inactiva** → `extensiones` sin esa clave.

## Cómo se arman las extensiones

Cada feature que guarda datos de un depósito expone en su `index.js` una función opcional:

```js
leerExtension: async (depositoId, contexto) => objeto | null
```

El core la llama para las features activas y arma `extensiones`. Este módulo no conoce qué features existen.
