# Pruebas: historial-depositos

| Id | Prueba | Esperado |
|---|---|---|
| E3.1 | 3 depósitos en la cuenta | 200, 3 elementos, el más reciente primero |
| E3.2 | Cuenta sin depósitos | 200, datos [], total 0 |
| E3.3 | 25 depósitos, pagina 2 tamano 20 | 5 elementos, total 25 |
| E3.4 | estado=FALLIDO | Solo FALLIDOS |
| E3.5 | CLIENTE con cuenta ajena | 403 SIN_PERMISO |
| E3.6 | tamano=500 | 400 FORMATO_INVALIDO |
| E3.7 | AGENTE | Solo los depósitos con su usuario_id |
