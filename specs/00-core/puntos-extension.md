# Puntos de extensión

El core define el flujo de registrar un depósito como un pipeline con huecos. Las features se enchufan en los huecos sin modificar el core.

```
1. Validadores        ← hueco
2. Transformadores    ← hueco
3. Acreditar          ← FIJO (core)
4. Post-procesos      ← hueco
```

Si un hueco está vacío, el pipeline sigue de largo.

## Contrato de un módulo de extensión

Archivo `src/modules/<id>/index.js`:

```js
module.exports = {
  id: 'limites-por-canal',      // igual al id del manifest
  tipo: 'validador',            // validador | transformador | post-proceso
  orden: 10,                    // menor = se ejecuta antes dentro del hueco
  ejecutar: async (deposito, contexto) => { /* ... */ },
  // Opcional: datos de este módulo para mostrar en GET /depositos/{id}
  leerExtension: async (depositoId, contexto) => null,
};
```

`contexto` trae: `{ cuenta, usuario, correlationId, db, logger }`.
- `cuenta`: datos devueltos por billetera (numero, estado, moneda).
- `db`: acceso de solo lectura/escritura a las tablas PROPIAS del módulo.

## Reglas por tipo

| Tipo | Cuándo corre | Puede | No puede | Si falla |
|---|---|---|---|---|
| validador | Antes de guardar el depósito | Lanzar `ErrorNegocio(codigo)` → 422 | Modificar el depósito | Se rechaza, no se guarda nada |
| transformador | Antes de guardar | Devolver el depósito con campos en `deposito.extensiones.<id>` y ajustar `deposito.montoAcreditar` / `deposito.monedaAcreditar` | Rechazar (si no puede transformar, no modifica nada) | Se trata como error interno (500) |
| post-proceso | Después de COMPLETADO | Guardar en sus tablas | Cambiar estado ni respuesta | Se registra en log y se ignora; el depósito sigue COMPLETADO |

## Lo que hace el core en orden

1. Consultar la cuenta en billetera.
2. Ejecutar validadores (orden ascendente). El primero que falla corta.
3. Ejecutar transformadores en cadena.
4. Si `monedaAcreditar` ≠ moneda de la cuenta → 422 MONEDA_NO_SOPORTADA.
5. Guardar depósito PENDIENTE. Los transformadores guardan sus datos en la misma transacción.
6. Cambiar a PROCESANDO y llamar a billetera con `montoAcreditar` y `monedaAcreditar` (por defecto = monto y moneda del depósito).
7. COMPLETADO o FALLIDO.
8. Si COMPLETADO, ejecutar post-procesos.

## Datos en la respuesta

Lo que una feature quiere mostrar al cliente va en `deposito.extensiones.<id>`. Así el core no cambia su respuesta cuando se agrega o quita una feature.
