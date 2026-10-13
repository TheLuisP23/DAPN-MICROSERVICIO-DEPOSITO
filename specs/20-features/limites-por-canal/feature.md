# Feature: limites-por-canal

Tipo: validador · Extiende: registrar-deposito · Orden: 10 · Código: `src/modules/limites-por-canal/`

## Descripción

Rechaza un depósito si su monto supera el máximo permitido para el canal por el que se hace. Ej.: un agente (bodega) no maneja tanto efectivo como una ventanilla.

## Reglas

| Id | Regla |
|---|---|
| LC-01 | El límite se busca por canal y moneda del depósito (antes de cualquier conversión). |
| LC-02 | Si monto > monto_maximo → `ErrorNegocio('LIMITE_CANAL_EXCEDIDO')` → 422. |
| LC-03 | Monto igual al límite → se permite. |
| LC-04 | Si no hay límite configurado para ese canal y moneda → se permite y se registra una advertencia en el log. |
| LC-05 | Los límites se leen de la tabla, no están en el código. |

## Tabla

```sql
CREATE TABLE limites_canal (
    canal         VARCHAR(20)   NOT NULL,
    moneda        CHAR(3)       NOT NULL,
    monto_maximo  NUMERIC(15,2) NOT NULL CHECK (monto_maximo > 0),
    PRIMARY KEY (canal, moneda)
);

-- Valores de EJEMPLO. El equipo debe definir los reales.
INSERT INTO limites_canal VALUES
  ('AGENTE',     'PEN',  2000.00), ('AGENTE',     'USD',   500.00),
  ('APP',        'PEN',  5000.00), ('APP',        'USD',  1500.00),
  ('VENTANILLA', 'PEN', 50000.00), ('VENTANILLA', 'USD', 15000.00);
```

## Escenarios

**L1.** AGENTE deposita 1,500 PEN (límite 2,000) → continúa.
**L2.** AGENTE deposita 2,500 PEN → 422 LIMITE_CANAL_EXCEDIDO, no se guarda depósito, billetera no recibe acreditación.
**L3.** AGENTE deposita exactamente 2,000 PEN → continúa.
**L4.** Canal sin límite configurado → continúa y advertencia en log.
**L5.** Feature inactiva → un depósito de 2,500 PEN por AGENTE se completa.

## Lo que expone

```js
module.exports = { id: 'limites-por-canal', tipo: 'validador', orden: 10, ejecutar };
```

No guarda datos por depósito, así que no tiene `leerExtension`.
