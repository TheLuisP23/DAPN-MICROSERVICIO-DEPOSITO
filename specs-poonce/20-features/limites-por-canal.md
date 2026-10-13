# Límites por canal

| Campo | Valor |
|---|---|
| Versión | 0.2 |
| Tipo | Módulo de extensión · `validador` · `orden: 10` |
| Carpeta | `src/modules/limites-por-canal/` (`index.js`, `repository.js`) |
| Tabla | `limites_canal` (solo lectura) |
| Depende de | `02-pipeline-y-modulos.md` |

## 1. Propósito

Rechazar depósitos cuyo monto supera el máximo permitido para el canal por el que entran (APP, AGENTE, VENTANILLA) en esa moneda.

## 2. Contrato

```js
module.exports = {
  id: 'limites-por-canal',
  tipo: 'validador',
  orden: 10,
  ejecutar: async (deposito, contexto) => { /* lanza ErrorNegocio o no devuelve nada */ },
};
```

## 3. Reglas de negocio

- **RN-LIM-01** Se busca el límite por `(deposito.canal, deposito.moneda)` — la moneda **original** del depósito, no la de la cuenta.
- **RN-LIM-02** Si `deposito.monto > monto_maximo` → `ErrorNegocio('LIMITE_CANAL_EXCEDIDO')` con `detalles: { canal, moneda, montoMaximo }`. Un monto **igual** al máximo se acepta.
- **RN-LIM-03** La comparación se hace con `Decimal`, nunca con `Number`.
- **RN-LIM-04** Si no hay fila para `(canal, moneda)` → el depósito **pasa** y se registra un log `error` "límite no configurado". **[DECISIÓN EQUIPO]** La alternativa es rechazar (más segura, pero un error de configuración bloquearía todos los depósitos de ese canal).
- **RN-LIM-05** El límite se lee en cada depósito (sin caché) para que un cambio en la tabla aplique de inmediato. **[FASE 2]** caché de 60 s si el volumen lo pide.

## 4. Datos

Lee `limites_canal`. Valores de ejemplo en `03-modelo-de-datos.md §4`.

## 5. Errores

`LIMITE_CANAL_EXCEDIDO` 422.

## 6. Criterios de aceptación

Con la semilla de ejemplo (`APP/PEN = 5000.00`):

- **CA-LIM-01** APP, PEN, `4999.99` → pasa.
- **CA-LIM-02** APP, PEN, `5000.00` → pasa (igual al máximo).
- **CA-LIM-03** APP, PEN, `5000.01` → 422 `LIMITE_CANAL_EXCEDIDO` con `detalles.montoMaximo = "5000.00"`.
- **CA-LIM-04** VENTANILLA, PEN, `5000.01` → pasa (otro canal, otro límite).
- **CA-LIM-05** APP, USD con límite `1500.00` y monto `1500.01` → 422 (se usa el límite de la moneda original).
- **CA-LIM-06** Sin fila para el par → pasa y hay un log de nivel `error`.
- **CA-LIM-07** Con el módulo en `activo: false`, APP, PEN, `9000.00` → no se rechaza por límite.

## 7. Fuera de alcance

- Límites acumulados (diarios/mensuales por cuenta o por usuario). **[FASE 2]** — se haría como otro validador.
- Límite mínimo por depósito.
