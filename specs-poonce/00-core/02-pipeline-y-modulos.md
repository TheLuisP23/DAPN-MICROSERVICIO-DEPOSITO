# 02 · Pipeline de depósito y módulos de extensión

| Campo | Valor |
|---|---|
| Versión | 0.2 |
| Archivos | `src/core/cargador.js`, `src/core/pipeline.js` |
| Depende de | `01-arquitectura.md`, `05-errores.md` |

## 1. Propósito

Permitir agregar o quitar reglas al registro de depósitos (límites, conversión, UIF, y las que vengan) **sin modificar el core**. El core define huecos; los módulos se enchufan en ellos al arrancar.

## 2. Huecos del pipeline

| # | Hueco | Tipo de módulo | Qué puede hacer | Qué NO puede hacer |
|---|---|---|---|---|
| 1 | Validadores | `validador` | Lanzar `ErrorNegocio` (422) | Modificar el depósito |
| 2 | Transformadores | `transformador` | Modificar y devolver el depósito | Rechazarlo por negocio |
| 3 | Acreditar | **fijo** (core) | Llamar a billetera | — |
| 4 | Post-procesos | `post-proceso` | Reaccionar al resultado | Cambiar el resultado o la respuesta |

Entre el hueco 2 y el 3 el core hace la **verificación final** (RN-PIP-07) y guarda el depósito.

## 3. Contrato del módulo

Cada módulo vive en `src/modules/<id>/index.js`:

```js
module.exports = {
  id: 'limites-por-canal',        // igual al nombre de la carpeta y al id del manifest
  tipo: 'validador',              // validador | transformador | post-proceso
  orden: 10,                      // entero; menor = se ejecuta antes dentro del hueco
  ejecutar: async (deposito, contexto) => { /* ... */ },

  // Opcional (cambio respecto a v0.1, ver RN-PIP-08):
  persistir: async (deposito, { tx }) => { /* guarda en la tabla propia del módulo */ },
};
```

### Qué devuelve `ejecutar`

| Tipo | Retorno esperado |
|---|---|
| `validador` | `undefined`. Si rechaza, lanza `ErrorNegocio`. |
| `transformador` | El objeto `deposito` (modificado o no). Si devuelve `undefined`, el pipeline lanza `ErrorInterno` (bug del módulo). |
| `post-proceso` | Se ignora. |

## 4. Objeto `deposito` dentro del pipeline

```js
{
  id: 'uuid',                    // generado por el core (crypto.randomUUID) antes del pipeline
  cuentaDestino: '1923456789012',
  monto: '150.50',               // string decimal, NUNCA number
  moneda: 'USD',
  canal: 'APP',
  referencia: 'Pago alquiler',
  usuarioId: 'u-123',

  // Lo que efectivamente se acreditará. El core los inicializa = monto/moneda.
  montoAcreditar: '150.50',
  monedaAcreditar: 'USD',

  // Datos que un módulo calcula y luego persiste en su tabla.
  extensiones: {
    // 'conversion-moneda': { tipoCambio: '3.7120', operacion: 'COMPRA', ... }
  },
}
```

- **RN-PIP-01** Un transformador solo puede cambiar `montoAcreditar`, `monedaAcreditar` y su propia entrada en `extensiones[id]`. Los campos originales (`monto`, `moneda`, `cuentaDestino`, `canal`…) son de solo lectura. El pipeline pasa una copia y verifica al terminar cada transformador que los campos originales no cambiaron; si cambiaron, lanza `ErrorInterno`.

## 5. Objeto `contexto`

```js
{
  cuenta: { numero, moneda, estado, titularId },  // lo que devolvió billetera
  usuario: { id, roles },                          // del JWT
  resultado: null,                                 // solo en post-procesos: { estado, error? }
  correlationId: '...',
  logger,                                          // pino hijo con correlationId
  config,                                          // config del servicio (solo lectura)
}
```

Los módulos **no** reciben el pool de base de datos en `ejecutar`. Leen sus tablas a través de su `repository.js`.

## 6. Reglas del pipeline

- **RN-PIP-02** Los módulos de un hueco se ejecutan **en secuencia**, ordenados por `orden` ascendente. Empate en `orden` → orden alfabético de `id` (determinista).
- **RN-PIP-03** Validadores: el primero que lanza error detiene el pipeline. El depósito **no** se guarda.
- **RN-PIP-04** Transformadores: cada uno recibe la salida del anterior.
- **RN-PIP-05** Un transformador que no puede hacer su trabajo por un problema **técnico** (por ejemplo, no hay tipo de cambio configurado) lanza `ErrorServicioNoDisponible` (503), no `ErrorNegocio`. Así se respeta "un transformador no rechaza por negocio".
- **RN-PIP-06** Post-procesos: se ejecutan solo si el depósito quedó `COMPLETADO`. Cada uno va envuelto en `try/catch`: un error se registra en log con nivel `error` y **no** se propaga. Un post-proceso que falla no impide que se ejecuten los siguientes.
- **RN-PIP-07** Verificación final del core (después de los transformadores): si `monedaAcreditar !== contexto.cuenta.moneda` → `ErrorNegocio('MONEDA_NO_COINCIDE')`. Así, con `conversion-moneda` desactivado, un depósito en otra moneda se rechaza con un error claro.
- **RN-PIP-08** Persistencia de módulos: después de los transformadores, el core guarda el depósito (`PENDIENTE`) y llama a `persistir` de cada transformador **en la misma transacción**. Esto resuelve que el transformador corre antes de que exista la fila en `depositos` (la FK de su tabla fallaría si insertara por su cuenta).
- **RN-PIP-09** Los post-procesos persisten por su cuenta dentro de `ejecutar` (el depósito ya existe).
- **RN-PIP-10** Un módulo nunca llama a billetera ni a otro servicio. Si lo necesita, se discute y se añade al spec de integración.

## 7. Cargador

Al arrancar:

1. Lee `MANIFEST_PATH` (por defecto `specs/manifest.yaml`).
2. Para cada módulo con `activo: true`: hace `require(<ruta>/index.js)`.
3. Valida el contrato:
   - `id` coincide con el del manifest.
   - `tipo` ∈ {validador, transformador, post-proceso} y coincide con el del manifest.
   - `orden` es entero.
   - `ejecutar` es función; `persistir`, si existe, es función.
4. Si algo falla → el proceso **termina** con un error que nombra el módulo (fallar rápido: mejor no arrancar que arrancar sin una validación de límites).
5. Registra el módulo en su hueco y escribe en log: `módulos cargados: limites-por-canal(validador,10), ...`.

Módulos con `activo: false` no se cargan ni se validan.

## 8. Cómo agregar un módulo nuevo (checklist)

1. Escribir su spec en `specs/20-features/<id>.md` usando la plantilla.
2. Crear `src/modules/<id>/index.js` cumpliendo el contrato.
3. Si necesita tabla, añadir script en `db/` con FK a `depositos` y un `repository.js` propio.
4. Agregarlo a `manifest.yaml`.
5. Tests: unitarios del módulo + al menos un test del pipeline con el módulo activo y otro con el módulo inactivo.
6. **No** se toca nada de `src/core/`.

## 9. Criterios de aceptación

- **CA-PIP-01** Dado un validador con `orden: 20` y otro con `orden: 10`, cuando se ejecuta el pipeline, entonces el de `orden: 10` se ejecuta primero.
- **CA-PIP-02** Dado un validador que lanza `ErrorNegocio`, entonces los transformadores no se ejecutan y no se inserta fila en `depositos`.
- **CA-PIP-03** Dado un transformador que modifica `monto` (campo original), entonces el pipeline lanza `ErrorInterno`.
- **CA-PIP-04** Dado un post-proceso que lanza una excepción, cuando el depósito se completa, entonces la respuesta sigue siendo 201 y se registra un log de error con el `id` del módulo.
- **CA-PIP-05** Dado un módulo en el manifest cuyo `index.js` no exporta `ejecutar`, cuando arranca el servicio, entonces el proceso termina con un error que nombra el módulo.
- **CA-PIP-06** Dado `conversion-moneda` con `activo: false` y un depósito en USD a una cuenta en PEN, entonces la respuesta es 422 `MONEDA_NO_COINCIDE`.
- **CA-PIP-07** Dado que `persistir` de un transformador falla, entonces tampoco queda la fila en `depositos` (rollback) y no se llama a billetera.
