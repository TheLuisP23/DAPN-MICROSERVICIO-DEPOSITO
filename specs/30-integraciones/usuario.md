# Integración: usuario-service

Estado: **[POR ACORDAR]** con el equipo de usuario.

## Qué necesitamos

Solo el JWT. deposito-service no llama a usuario-service en cada petición.

## Formato propuesto del token

```json
{
  "sub": "usr-123",
  "roles": ["CLIENTE"],
  "iss": "usuario-service",
  "aud": "banco-api",
  "exp": 1760000000
}
```

| Campo | Uso en depósito |
|---|---|
| sub | `usuario_id` en depósitos y auditoría |
| roles | Autorización: CLIENTE, AGENTE, CAJERO |
| iss / aud | Validar emisor y destinatario |
| exp | Expiración |

## Clave para validar

- Propuesta: firma RS256. usuario-service publica su clave pública (por ejemplo en `GET /.well-known/jwks.json`) y depósito la obtiene al arrancar.
- Alternativa: HS256 con secreto compartido en variable de entorno (más simple, menos seguro).

## Preguntas para el equipo de usuario

- [ ] ¿RS256 o HS256?
- [ ] Valores exactos de `iss` y `aud`.
- [ ] Nombres exactos de los roles.
- [ ] ¿El token de un cajero/agente trae algún dato de su agencia?
