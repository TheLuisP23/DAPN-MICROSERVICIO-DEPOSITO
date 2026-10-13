# Alerta UIF

| Campo | Valor |
|---|---|
| Versión | 0.2 |
| Tipo | Módulo de extensión · `post-proceso` · `orden: 10` |
| Carpeta | `src/modules/alerta-uif/` (`index.js`, `repository.js`) |
| Tablas | `alertas_uif` (escritura), `tipos_cambio` (lectura) |
| Depende de | `02-pipeline-y-modulos.md` |

## 1. Propósito

Marcar los depósitos que igualan o superan el umbral de reporte a la UIF-Perú (Unidad de Inteligencia Financiera, adscrita a la SBS), para que el área de cumplimiento los revise y reporte.

El módulo **no** reporta a la UIF: solo deja la alerta registrada en `PENDIENTE`.

## 2. Contrato

```js
module.exports = {
  id: 'alerta-uif',
  tipo: 'post-proceso',
  orden: 10,
  ejecutar: async (deposito, contexto) => { /* inserta alerta si corresponde */ },
};
```

## 3. Umbral — **[PENDIENTE #5]**

- Se configura con `UIF_UMBRAL` y `UIF_UMBRAL_MONEDA` (por defecto `USD`).
- **El valor debe verificarse en la normativa vigente de la SBS** sobre registro de operaciones antes de la entrega. No se escribe en el código.
- Si `alerta-uif` está activo y falta `UIF_UMBRAL`, el servicio no arranca.

## 4. Reglas de negocio

- **RN-UIF-01** Solo corre para depósitos `COMPLETADO` (lo garantiza el pipeline).
- **RN-UIF-02** Se compara el **monto original depositado** (`deposito.monto` en `deposito.moneda`), no el acreditado.
- **RN-UIF-03** Equivalente en la moneda del umbral:
  - Misma moneda → `montoEquivalente = monto`.
  - PEN con umbral en USD → `monto ÷ compra` (se usa el precio menor para no subestimar el equivalente; criterio conservador).
  - USD con umbral en PEN → `monto × venta` (mismo criterio).
- **RN-UIF-04** Si `montoEquivalente >= umbral` → INSERT en `alertas_uif` con `estado_reporte = 'PENDIENTE'`.
- **RN-UIF-05** Si no hay tipo de cambio para calcular el equivalente → INSERT con `monto_equivalente = NULL` y `estado_reporte = 'REVISAR'` (ante la duda, se alerta).
- **RN-UIF-06** `deposito_id` es `UNIQUE` en `alertas_uif`: si por un reintento ya existe la alerta, el INSERT usa `ON CONFLICT (deposito_id) DO NOTHING`.
- **RN-UIF-07** Como todo post-proceso, si falla se loguea y la respuesta al cliente no cambia. El log incluye `depositoId` y el texto `ALERTA_UIF_NO_REGISTRADA` para poder buscarlo.
- **RN-UIF-08** El cliente **no** se entera de la alerta (no va en la respuesta ni en el historial).

## 5. Criterios de aceptación

Con `UIF_UMBRAL = 10000`, `UIF_UMBRAL_MONEDA = USD` (valor **solo para tests**) y compra `3.7120`:

- **CA-UIF-01** USD `9999.99` → sin alerta.
- **CA-UIF-02** USD `10000.00` → alerta `PENDIENTE` (igual al umbral cuenta).
- **CA-UIF-03** PEN `37120.00` → equivalente `10000.00` → alerta.
- **CA-UIF-04** PEN `37119.00` → equivalente `9999.73` → sin alerta.
- **CA-UIF-05** PEN alto sin tipo de cambio → alerta `REVISAR` con `monto_equivalente` NULL.
- **CA-UIF-06** Depósito `FALLIDO` → el módulo no se ejecuta.
- **CA-UIF-07** `alertas_uif` inaccesible (simulado) → respuesta 201 igual; log con `ALERTA_UIF_NO_REGISTRADA`.
- **CA-UIF-08** La respuesta del POST no contiene ninguna mención a la alerta.

## 6. Fuera de alcance

- **[FASE 2]** Operaciones múltiples: varios depósitos menores al umbral que, sumados en un período, lo superan (fraccionamiento). La normativa de prevención de lavado suele contemplarlas; se haría como un post-proceso adicional que suma los depósitos del mes por cuenta.
- Envío del reporte a la UIF.
