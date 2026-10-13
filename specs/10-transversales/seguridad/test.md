# Pruebas: seguridad

| Id | Prueba | Esperado |
|---|---|---|
| S1 | POST /depositos sin header Authorization | 401 TOKEN_REQUERIDO |
| S2 | Token con exp en el pasado | 401 TOKEN_EXPIRADO |
| S3a | Token firmado con otra clave | 401 TOKEN_INVALIDO |
| S3b | Token con alg none | 401 TOKEN_INVALIDO |
| S4 | Token con aud distinto | 401 TOKEN_INVALIDO |
| S5 | CLIENTE, cuenta con titularId distinto | 403 SIN_PERMISO, billetera no recibe acreditación |
| S6 | CLIENTE con canal VENTANILLA | 403 SIN_PERMISO |
| S7 | Sin X-Correlation-Id | Respuesta con header X-Correlation-Id no vacío |
| S8 | Con X-Correlation-Id "abc" | Respuesta con "abc"; billetera mock recibió "abc" |
| S9 | Error 500 forzado | El log no contiene el token ni la cuenta completa |

Tokens de prueba: generarlos en el test con un par de claves propio de pruebas.
