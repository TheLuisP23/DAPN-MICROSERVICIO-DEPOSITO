# Integración con usuario-service

| Campo | Valor |
|---|---|
| Versión | 0.2 |
| Archivos | `src/core/middlewares/autenticar.js`, `src/core/roles.js` |
| Contraparte | Equipo usuario |
| Estado | **[POR ACORDAR #3]** |
| Depende de | `10-transversales/seguridad.md` |

## 1. Propósito

Definir qué necesita deposito-service del JWT que emite usuario-service. **No hay llamadas HTTP en cada petición**: el token se verifica localmente con la clave pública.

## 2. Propuesta de depósito para el JWT

| Aspecto | Propuesta | Por qué |
|---|---|---|
| Algoritmo | **RS256** (o ES256) | Asimétrico: usuario firma con la privada; los demás servicios solo tienen la pública y **no pueden** fabricar tokens. Con HS256 cada servicio tendría el secreto y cualquiera podría firmar tokens. |
| Distribución de la clave | Endpoint JWKS (`GET /.well-known/jwks.json`) o archivo PEM compartido por variable de entorno | JWKS permite rotar claves sin redeploy |
| `iss` | `usuario-service` | |
| `aud` | `banco-api` (una audiencia común) o una por servicio | |
| `exp` | 15 minutos | Token de acceso corto; el refresh lo maneja usuario-service |
| `sub` | ID del usuario (string) | Se guarda en `depositos.usuario_id` |
| `roles` | arreglo de strings: `["CLIENTE"]` | Un usuario puede tener varios roles |

Ejemplo de payload:

```json
{
  "iss": "usuario-service",
  "aud": "banco-api",
  "sub": "u-123",
  "roles": ["CLIENTE"],
  "iat": 1792094940,
  "exp": 1792095840
}
```

## 3. Lo que deposito-service hace con el token

1. Verificación según `seguridad.md §3`.
2. Mapea `roles` a los roles internos con `src/core/roles.js` (un solo lugar para adaptarse si usuario-service usa otros nombres, por ejemplo `ROLE_CAJERO`).
3. Ignora roles desconocidos (no fallan la petición; simplemente no dan permisos).

## 4. Lo que deposito-service **no** hace

- No llama a usuario-service para validar cada token.
- No consulta datos personales del usuario (nombre, DNI): no los necesita.
- No maneja login, refresh ni revocación.

> **Limitación:** sin llamada a usuario-service, un token revocado sigue siendo válido hasta su `exp`. Por eso se propone `exp` corto (15 min).

## 5. Tokens para tests

`tests/helpers/tokens.js` genera un par de claves RSA al iniciar la suite y firma tokens con cualquier combinación de claims (válido, expirado, otra clave, `alg: none`, otra audiencia). La clave pública se inyecta en `config` del entorno de test.

## 6. Criterios de aceptación

Los de `seguridad.md §8`, más:

- **CA-USR-01** Rol `ROLE_CAJERO` configurado como alias → se trata como `CAJERO`.
- **CA-USR-02** Rol desconocido `MARKETING` → no da permisos; si es el único rol, `POST /depositos` responde 403.

## 7. Lista para la reunión con usuario

1. Algoritmo y forma de publicar la clave pública.
2. Valores de `iss` y `aud`.
3. Nombre del claim de roles y lista de roles.
4. Duración del token.
5. Si emitirán tokens de servicio (para depósito → billetera).
