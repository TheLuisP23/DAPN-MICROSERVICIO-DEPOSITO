# Estrategia de pruebas

Reglas generales para el agente tester. Los casos concretos están en el `test.md` de cada feature y transversal.

## Herramientas

- Jest para unitarias, Supertest para la API.
- PostgreSQL de prueba levantado con Docker (no se mockea la base de datos en pruebas de integración).
- billetera-service y usuario-service se simulan con mocks HTTP (`nock`) según `30-integraciones/`.

## Niveles

| Nivel | Qué prueba | Ubicación |
|---|---|---|
| Unitaria | Un service o módulo de extensión aislado | `tests/unit/<id>/` |
| Integración | Endpoint completo con BD real y billetera mockeada | `tests/integration/<id>/` |
| Pipeline | Core con módulos activados/desactivados | `tests/core/` |

## Reglas

1. Cada escenario de un `feature.md` tiene al menos una prueba. El nombre de la prueba cita el escenario: `test('E1.3 clave repetida con otros datos → 409')`.
2. Para cada feature enchufable, probar el pipeline con la feature ACTIVA e INACTIVA.
3. Probar dinero con valores límite: 0.01, 0, -1, 10.001.
4. Ninguna prueba depende de otra ni del orden de ejecución.
5. Se limpia la BD entre pruebas.
6. Cobertura mínima: 80 % en `src/core/` y `src/modules/`.

## Reporte

El tester entrega `tests/reporte.md` con: escenarios cubiertos, escenarios sin prueba, pruebas fallidas y su causa.
