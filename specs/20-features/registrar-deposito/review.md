# Revisión: registrar-deposito

- [ ] El service usa el pipeline del core; no llama directamente a ninguna feature de extensión.
- [ ] No hay referencias a limites, conversión ni UIF en este módulo.
- [ ] El monto se maneja con decimal y se envía a billetera como texto.
- [ ] Guardar PENDIENTE y los datos de transformadores ocurre en una sola transacción.
- [ ] Las transiciones de estado son solo las permitidas.
- [ ] `res.locals.depositoId` se asigna antes de responder.
