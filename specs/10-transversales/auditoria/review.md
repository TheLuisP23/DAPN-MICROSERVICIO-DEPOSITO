# Revisión: auditoría

- [ ] Existe el trigger que bloquea UPDATE y DELETE.
- [ ] Se usa `res.on('finish')`, no se espera la auditoría antes de responder.
- [ ] Un fallo de auditoría no cambia la respuesta.
- [ ] Todas las rutas de negocio tienen su `auditoria('<ACCION>')`.
- [ ] No se guarda el cuerpo completo ni datos personales en la tabla.
