# Pruebas: limites-por-canal

Unitarias (módulo aislado, límites cargados en BD de prueba):

| Id | Prueba | Esperado |
|---|---|---|
| L1 | AGENTE 1500.00 PEN | No lanza error |
| L2 | AGENTE 2500.00 PEN | Lanza LIMITE_CANAL_EXCEDIDO |
| L3 | AGENTE 2000.00 PEN | No lanza error |
| L3b | AGENTE 2000.01 PEN | Lanza LIMITE_CANAL_EXCEDIDO |
| L4 | Canal sin fila en limites_canal | No lanza error; logger.warn llamado |

Integración (pipeline):

| Id | Prueba | Esperado |
|---|---|---|
| L2-int | POST 2500 por AGENTE | 422; 0 filas en depositos; billetera no llamada |
| L5 | Feature inactiva en manifest, POST 2500 por AGENTE | 201 |
