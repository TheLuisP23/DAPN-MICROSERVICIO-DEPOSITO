# Revisión: idempotencia

- [ ] La reserva usa `INSERT ... ON CONFLICT DO NOTHING`, no "SELECT y luego INSERT".
- [ ] El hash incluye el cuerpo completo.
- [ ] En 5xx la clave se libera.
- [ ] Solo se aplica a POST.
- [ ] La respuesta repetida lleva `Idempotency-Replayed: true`.
- [ ] Existe la limpieza de claves antiguas.
