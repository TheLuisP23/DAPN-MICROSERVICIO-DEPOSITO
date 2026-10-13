# Feature: conversion-moneda

Tipo: transformador · Extiende: registrar-deposito · Orden: 10 · Código: `src/modules/conversion-moneda/`

## Descripción

Si el depósito llega en una moneda distinta a la de la cuenta (PEN ↔ USD), calcula cuánto acreditar en la moneda de la cuenta según el tipo de cambio vigente.

## Reglas

| Id | Regla |
|---|---|
| CM-01 | Si moneda del depósito = moneda de la cuenta → no hace nada. |
| CM-02 | Si son distintas, busca en `tipos_cambio` la fila con ese par (origen → destino) y la `vigente_desde` más reciente que no sea futura. |
| CM-03 | `montoAcreditar = monto × valor`, redondeado a 2 decimales (redondeo hacia arriba desde 0.005). Se calcula con decimal. |
| CM-04 | Asigna `deposito.montoAcreditar`, `deposito.monedaAcreditar` y `deposito.extensiones['conversion-moneda']`. |
| CM-05 | Si no hay tipo de cambio para ese par → no modifica nada; el core rechaza con 422 MONEDA_NO_SOPORTADA. |
| CM-06 | Guarda la conversión en `conversiones_deposito` en la misma transacción que el depósito. |
| CM-07 | Cada sentido tiene su propia fila (USD→PEN usa el tipo de compra del banco, PEN→USD el de venta). |

## Tablas

```sql
CREATE TABLE tipos_cambio (
    id              UUID PRIMARY KEY,
    moneda_origen   CHAR(3)        NOT NULL,
    moneda_destino  CHAR(3)        NOT NULL,
    valor           NUMERIC(10,4)  NOT NULL CHECK (valor > 0),
    vigente_desde   TIMESTAMPTZ    NOT NULL
);

CREATE TABLE conversiones_deposito (
    deposito_id     UUID PRIMARY KEY REFERENCES depositos(id),
    tipo_cambio_id  UUID           NOT NULL REFERENCES tipos_cambio(id),
    moneda_cuenta   CHAR(3)        NOT NULL,
    tipo_cambio     NUMERIC(10,4)  NOT NULL,
    monto_final     NUMERIC(15,2)  NOT NULL
);
```

Fuente de los tipos de cambio: **[POR DEFINIR]** (carga manual por ahora).

## Datos que agrega a la respuesta

```json
"extensiones": {
  "conversion-moneda": {
    "monedaCuenta": "PEN",
    "tipoCambio": 3.7500,
    "montoAcreditado": 375.00
  }
}
```

## Escenarios

**C1.** 100.00 USD a cuenta PEN con tipo 3.7500 → billetera recibe 375.00 PEN; fila en conversiones_deposito.
**C2.** 150.00 PEN a cuenta PEN → no hay conversión ni fila; `extensiones` sin esta clave.
**C3.** USD a cuenta PEN sin tipo de cambio cargado → 422 MONEDA_NO_SOPORTADA.
**C4.** Dos tipos cargados (ayer y hoy) → se usa el de hoy.
**C5.** Tipo con `vigente_desde` mañana → se ignora.
**C6.** Redondeo: 10.01 USD × 3.7555 = 37.592... → 37.59 PEN.
**C7.** Feature inactiva → USD a cuenta PEN da 422 MONEDA_NO_SOPORTADA.

## Lo que expone

```js
module.exports = { id: 'conversion-moneda', tipo: 'transformador', orden: 10, ejecutar, leerExtension };
```
