# Conversión de moneda

| Campo | Valor |
|---|---|
| Versión | 0.2 |
| Tipo | Módulo de extensión · `transformador` · `orden: 10` |
| Carpeta | `src/modules/conversion-moneda/` (`index.js`, `repository.js`) |
| Tablas | `tipos_cambio` (lectura), `conversiones_deposito` (escritura vía `persistir`) |
| Depende de | `02-pipeline-y-modulos.md`, `01-arquitectura.md §5` |

## 1. Propósito

Permitir depositar en una moneda distinta a la de la cuenta (USD a cuenta en soles, o soles a cuenta en dólares), calculando el monto a acreditar con el tipo de cambio vigente del banco.

## 2. Contrato

```js
module.exports = {
  id: 'conversion-moneda',
  tipo: 'transformador',
  orden: 10,
  ejecutar: async (deposito, contexto) => deposito,          // devuelve el depósito
  persistir: async (deposito, { tx }) => { /* INSERT conversiones_deposito */ },
};
```

## 3. Compra y venta — **[DECISIÓN EQUIPO]**

v0.1 tenía un solo `valor` en `tipos_cambio`. Los bancos publican dos precios por divisa y la diferencia (spread) es su ganancia:

| El cliente deposita… | En cuenta en… | El banco… | Precio que se usa | Cálculo |
|---|---|---|---|---|
| USD | PEN | **compra** dólares | `compra` | `montoAcreditar = monto × compra` |
| PEN | USD | **vende** dólares | `venta` | `montoAcreditar = monto ÷ venta` |

Si el equipo prefiere mantener un solo valor, se usa ese en ambos casos y el resto del spec no cambia.

## 4. Reglas de negocio

- **RN-CNV-01** Si `deposito.moneda === contexto.cuenta.moneda` → devuelve el depósito sin cambios y **no** persiste nada.
- **RN-CNV-02** Si son distintas, obtiene el tipo de cambio vigente: la fila de `tipos_cambio` con `moneda = 'USD'` y el mayor `vigente_desde <= now()`.
- **RN-CNV-03** Sin tipo de cambio vigente → lanza `ErrorServicioNoDisponible('TIPO_CAMBIO_NO_DISPONIBLE')` (503). Es un problema técnico de configuración, no un rechazo de negocio (un transformador no rechaza).
- **RN-CNV-04** Calcula con `Decimal` y redondea a 2 decimales `ROUND_HALF_UP`.
- **RN-CNV-05** Solo modifica `montoAcreditar`, `monedaAcreditar` y `extensiones['conversion-moneda']`:
  ```js
  deposito.extensiones['conversion-moneda'] = {
    monedaOrigen: 'USD', monedaCuenta: 'PEN', operacion: 'COMPRA',
    tipoCambio: '3.7120', tipoCambioId: 'uuid', montoFinal: '556.80',
  };
  ```
- **RN-CNV-06** `persistir` inserta en `conversiones_deposito` desde `extensiones['conversion-moneda']`, dentro de la transacción que le pasa el core. Si no hay extensión, no hace nada.
- **RN-CNV-07** El tipo de cambio queda **congelado** al registrar el depósito: un reintento (idempotencia) no recalcula.

## 5. Fuente del tipo de cambio — **[PENDIENTE #4]**

Se implementa leyendo la tabla `tipos_cambio` a través de una interfaz:

```js
// src/modules/conversion-moneda/proveedorTipoCambio.js
async function obtenerVigente(moneda) → { id, compra, venta } | null
```

Si más adelante se decide un servicio externo, solo cambia esta función.

## 6. Errores

`TIPO_CAMBIO_NO_DISPONIBLE` 503.

## 7. Criterios de aceptación

Con la semilla de ejemplo (compra `3.7120`, venta `3.7480`):

- **CA-CNV-01** USD `150.00` a cuenta PEN → `montoAcreditar = "556.80"`, `monedaAcreditar = "PEN"`, operación `COMPRA`.
- **CA-CNV-02** PEN `1000.00` a cuenta USD → `1000.00 ÷ 3.7480 = 266.80896…` → `"266.81"`, operación `VENTA`.
- **CA-CNV-03** PEN a cuenta PEN → depósito sin cambios y sin fila en `conversiones_deposito`.
- **CA-CNV-04** Depósito convertido y completado → una fila en `conversiones_deposito` con el `tipo_cambio_id` usado.
- **CA-CNV-05** Sin filas en `tipos_cambio` y monedas distintas → 503 `TIPO_CAMBIO_NO_DISPONIBLE`; no hay fila en `depositos`.
- **CA-CNV-06** Dos filas de tipo de cambio, una con `vigente_desde` en el futuro → se usa la vigente (la pasada más reciente).
- **CA-CNV-07** El módulo no altera `deposito.monto` ni `deposito.moneda` (los originales).
- **CA-CNV-08** Si el INSERT del depósito falla, tampoco queda fila en `conversiones_deposito` (misma transacción).

## 8. Fuera de alcance

- Monedas distintas de PEN y USD.
- Tipo de cambio preferencial por cliente o por monto.
