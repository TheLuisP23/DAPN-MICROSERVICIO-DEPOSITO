# Pruebas: registrar-deposito

Billetera mockeada con nock según `30-integraciones/billetera.md`.

| Id | Prueba | Esperado |
|---|---|---|
| E1.1 | Depósito válido 150.00 PEN | 201 COMPLETADO; 1 fila en depositos; billetera recibió monto "150.00" y depositoId |
| E1.2a | monto 0 | 422 MONTO_INVALIDO; 0 filas; billetera no llamada |
| E1.2b | monto -1 | 422 MONTO_INVALIDO |
| E1.2c | monto 10.001 | 422 MONTO_INVALIDO |
| E1.2d | monto 0.01 | 201 |
| E1.3 | Sin cuentaDestino | 400 FORMATO_INVALIDO |
| E1.4 | Billetera 404 en consulta | 422 CUENTA_NO_ENCONTRADA |
| E1.5 | Cuenta BLOQUEADA | 422 CUENTA_INACTIVA |
| E1.6a | Billetera 500 al acreditar | 502; depósito FALLIDO |
| E1.6b | Billetera tarda 6 s al acreditar | 502; depósito FALLIDO |
| E1.7 | Billetera 500 al consultar | 502; 0 filas |
| E1.8 | conversion-moneda inactiva, USD a cuenta PEN | 422 MONEDA_NO_SOPORTADA |
| E1.9 | Todas las extensiones inactivas | 201, extensiones = {} |
| E1.10 | Estados | Se pasa por PENDIENTE y PROCESANDO antes de COMPLETADO (verificar con spy del repository) |
