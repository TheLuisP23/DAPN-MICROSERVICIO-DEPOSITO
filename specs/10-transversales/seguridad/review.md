# Revisión: seguridad

- [ ] `jwt.verify` fija `algorithms`, `issuer` y `audience`.
- [ ] No hay llamada HTTP a usuario-service por petición.
- [ ] `requiereRol` se aplica en todas las rutas de negocio.
- [ ] La regla rol ↔ canal está implementada.
- [ ] Existe la función de enmascarar cuenta y se usa en todos los logs.
- [ ] `/health` es la única ruta sin token.
