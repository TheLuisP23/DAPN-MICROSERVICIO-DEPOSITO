# Pruebas: alerta-uif

Usar en pruebas un umbral de prueba (ej. UIF_UMBRAL_PEN=1000.00), nunca el valor real.

| Id | Prueba | Esperado |
|---|---|---|
| U1 | COMPLETADO con 1000.00 PEN | 1 fila PENDIENTE con umbral 1000.00 |
| U2 | COMPLETADO con 999.99 PEN | 0 filas |
| U3 | FALLIDO con 5000.00 PEN | 0 filas |
| U4 | Forzar error al insertar | 201 normal; logger.error con depositoId |
| U5 | GET /depositos/{id} como CLIENTE | Respuesta sin datos de UIF |
| U6 | Feature inactiva, 5000.00 PEN | 201; 0 filas |
