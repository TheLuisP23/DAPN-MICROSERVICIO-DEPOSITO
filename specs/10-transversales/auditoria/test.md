# Pruebas: auditoría

| Id | Prueba | Esperado |
|---|---|---|
| A1 | Depósito válido | 1 fila, EXITO, deposito_id lleno, correlation_id igual al de la respuesta |
| A2 | Monto -5 | 1 fila, ERROR_422, deposito_id nulo |
| A3 | 3 POST con la misma clave | 3 filas CREAR_DEPOSITO, 1 solo depósito |
| A4 | UPDATE y DELETE directos en SQL | Ambos lanzan excepción |
| A5 | Forzar error al insertar auditoría | Respuesta 201 normal; error en log sin cuenta completa |
| A6 | GET /depositos/{id} | 1 fila CONSULTAR_DEPOSITO |

Nota: como la auditoría se escribe después de responder, esperar a que termine (`await` de un pequeño flush o reintentar la consulta) antes de verificar.
