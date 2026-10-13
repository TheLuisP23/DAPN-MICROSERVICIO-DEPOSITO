# Constitución

Reglas que ningún agente puede romper. Si una spec de feature contradice este archivo, gana este archivo y se avisa al equipo.

## Stack

- Backend: Node.js LTS + Express.
- Base de datos: PostgreSQL, propia del servicio.
- Pruebas: Jest + Supertest.
- Empaquetado: Docker.
- Frontend: [POR DEFINIR].

## Arquitectura

1. Arquitectura en capas dentro de cada módulo: `routes` → `controller` → `service` → `repository`.
2. Los controllers no tienen lógica de negocio.
3. Los services no escriben SQL ni hacen llamadas HTTP directas.
4. Solo `repository` conoce PostgreSQL y solo `src/core/clients/` conoce las URLs de otros servicios.
5. El core (`src/core/`) define el pipeline y sus puntos de extensión. Los módulos se enchufan; nunca modifican el core.
6. Una feature solo depende del core. Nunca importa código de otra feature.
7. El Cargador lee `specs/manifest.yaml` al arrancar y registra solo los módulos activos.

## Dinero

8. Nunca `float` para dinero. En BD `NUMERIC(15,2)`; en código, una librería decimal (`decimal.js`) o enteros en céntimos.
9. Monedas permitidas: PEN y USD.
10. Montos con máximo 2 decimales.

## Seguridad

11. Toda ruta requiere JWT válido, salvo `/health`.
12. Nunca registrar en logs: tokens, número de cuenta completo, datos personales.
13. Toda consulta SQL usa parámetros (`$1, $2`), nunca concatenación.
14. Secretos y URLs en variables de entorno, nunca en el código.

## Consistencia

15. Todo POST que crea un depósito es idempotente (`Idempotency-Key`).
16. Todos los errores usan el formato de `contrato-base.yaml`.
17. Toda petición tiene `X-Correlation-Id`.

## Proceso

18. Ningún agente modifica `specs/00-core/` sin aprobación del equipo.
19. Cada escenario de un `feature.md` tiene al menos una prueba.
20. Los datos `[POR ACORDAR]` se implementan con un mock que respete el contrato propuesto, nunca con valores inventados.

## Estructura de código

```
src/
├── app.js
├── core/
│   ├── db.js
│   ├── cargador.js
│   ├── pipeline.js
│   ├── errores.js
│   ├── middlewares/   correlationId, auth, auditoria, idempotencia
│   └── clients/       billeteraClient.js
└── modules/<id-de-feature>/
```

Cada carpeta de `src/modules/` tiene el mismo nombre que su carpeta en `specs/20-features/`.
