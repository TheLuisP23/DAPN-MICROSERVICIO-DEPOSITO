# Front: consultar-deposito

Pantalla: **Detalle del depósito**. Se abre desde el comprobante o desde el historial.

- Mostrar: id, cuenta enmascarada, monto, moneda, canal, referencia, estado, fechas.
- Estado con etiqueta de color: PENDIENTE/PROCESANDO (en curso), COMPLETADO (éxito), FALLIDO (error).
- Si el estado es PENDIENTE o PROCESANDO, ofrecer botón "Actualizar".
- Mostrar cada bloque de `extensiones` que venga; si no viene, no mostrar nada.
- 404: "No encontramos este depósito".
