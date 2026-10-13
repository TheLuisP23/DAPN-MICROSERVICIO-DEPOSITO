# Pruebas: idempotencia

| Id | Prueba | Esperado |
|---|---|---|
| I1 | POST con clave nueva | 201; fila TERMINADO con status 201 |
| I2 | Repetir el mismo POST | 201 idéntico, header Idempotency-Replayed true, billetera mock llamada 1 sola vez |
| I3 | Misma clave, monto distinto | 409 IDEMPOTENCY_KEY_REUTILIZADA |
| I4 | Billetera mock con demora de 2 s; segundo POST a los 100 ms | Segundo responde 409 SOLICITUD_EN_PROCESO |
| I5 | 5 POST simultáneos con la misma clave (Promise.all) | 1 depósito creado, 1 acreditación |
| I6 | Billetera responde 500, luego se reintenta con la misma clave y billetera OK | Primer intento 502, segundo 201 |
| I7 | Sin header | 400 IDEMPOTENCY_KEY_REQUERIDA |
| I8 | Clave de 7 caracteres | 400 IDEMPOTENCY_KEY_REQUERIDA |
