# Feature: alerta-uif

Tipo: post-proceso · Extiende: registrar-deposito · Orden: 10 · Código: `src/modules/alerta-uif/`

## Descripción

Marca los depósitos que igualan o superan el umbral de la normativa antilavado, para que el área de cumplimiento los revise y, si corresponde, los reporte a la UIF-Perú (SBS). No bloquea el depósito.

## Reglas

| Id | Regla |
|---|---|
| UIF-01 | Corre solo cuando el depósito quedó COMPLETADO. |
| UIF-02 | Si monto >= umbral de su moneda → inserta una fila en `alertas_uif` con estado_reporte PENDIENTE. |
| UIF-03 | El umbral es configurable por moneda: `UIF_UMBRAL_PEN`, `UIF_UMBRAL_USD`. **El valor debe verificarse en la normativa SBS vigente**; no se fija en el código. |
| UIF-04 | Nunca cambia el estado del depósito ni la respuesta. |
| UIF-05 | La alerta NO se muestra al cliente (no aparece en `extensiones` para el rol CLIENTE). |
| UIF-06 | Si falla al guardar la alerta, se registra en el log como ERROR con el depositoId para reprocesar; el depósito sigue COMPLETADO. |
| UIF-07 | `estado_reporte` sí puede cambiar después (PENDIENTE → REPORTADO o DESCARTADO); por eso no va en la tabla de auditoría, que es inmutable. |

## Tabla

```sql
CREATE TABLE alertas_uif (
    id              UUID PRIMARY KEY,
    deposito_id     UUID          NOT NULL UNIQUE REFERENCES depositos(id),
    monto           NUMERIC(15,2) NOT NULL,
    moneda          CHAR(3)       NOT NULL,
    umbral          NUMERIC(15,2) NOT NULL,
    estado_reporte  VARCHAR(20)   NOT NULL DEFAULT 'PENDIENTE'
                    CHECK (estado_reporte IN ('PENDIENTE','REPORTADO','DESCARTADO')),
    creado_en       TIMESTAMPTZ   NOT NULL DEFAULT now()
);
```

## Escenarios

**U1.** Depósito COMPLETADO con monto igual al umbral → 1 alerta PENDIENTE.
**U2.** Depósito COMPLETADO por debajo del umbral → sin alerta.
**U3.** Depósito FALLIDO por encima del umbral → sin alerta.
**U4.** Falla la inserción de la alerta → respuesta 201 normal; log ERROR con depositoId.
**U5.** CLIENTE consulta el depósito → no ve nada de la alerta.
**U6.** Feature inactiva → sin alertas, depósito normal.

## Fuera de alcance

Detectar fraccionamiento (varios depósitos pequeños que juntos superan el umbral) y la pantalla de cumplimiento.

## Lo que expone

```js
module.exports = { id: 'alerta-uif', tipo: 'post-proceso', orden: 10, ejecutar };
```

No expone `leerExtension` (no se muestra a ningún rol por ahora).
