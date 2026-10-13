# Pruebas: consultar-deposito

| Id | Prueba | Esperado |
|---|---|---|
| E2.1 | CLIENTE consulta depósito de su cuenta | 200 con todos los campos |
| E2.2 | UUID que no existe | 404 DEPOSITO_NO_ENCONTRADO |
| E2.3 | id "abc" | 400 FORMATO_INVALIDO |
| E2.4 | CLIENTE consulta depósito de otra cuenta | 404 DEPOSITO_NO_ENCONTRADO |
| E2.4b | AGENTE consulta depósito registrado por otro agente | 404 |
| E2.5 | Depósito USD→PEN con conversion-moneda activa | extensiones.conversion-moneda presente |
| E2.6 | Mismo depósito, conversion-moneda inactiva | extensiones sin conversion-moneda |
