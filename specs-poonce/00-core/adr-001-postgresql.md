# ADR-001 · PostgreSQL como base de datos de deposito-service

| Campo | Valor |
|---|---|
| Estado | Aceptada |
| Fecha | 2026-10-13 |
| Decisores | Equipo depósito |

> Un ADR (*Architecture Decision Record*) registra **una** decisión importante, por qué se tomó y qué alternativas se descartaron. Si la decisión cambia, no se edita: se escribe un ADR nuevo que la reemplaza.

## Contexto

deposito-service registra movimientos de dinero. Sus necesidades de datos son:

1. **Consistencia fuerte**: un depósito y los datos de sus módulos (conversión) se guardan juntos o no se guardan.
2. **Dinero exacto**: ningún error de redondeo binario.
3. **Idempotencia atómica**: dos peticiones simultáneas con la misma `Idempotency-Key` → solo una gana, sin condiciones de carrera.
4. **Guardar la respuesta HTTP** de la primera petición para devolverla igual en los reintentos.
5. **Estados protegidos**: la base debe rechazar estados inválidos aunque el código tenga un bug.
6. **Relaciones**: auditoría, conversiones y alertas apuntan a un depósito.
7. Stack del equipo: **Node.js + Express**.

## Decisión

Usar **PostgreSQL 16** con el driver `pg` (node-postgres), una base exclusiva para el servicio (`depositos_db`).

## Por qué PostgreSQL cubre cada necesidad

| Necesidad | Característica de PostgreSQL usada en el diseño |
|---|---|
| 1. Consistencia | Transacciones ACID; DDL transaccional (una migración falla completa o no se aplica) |
| 2. Dinero exacto | `NUMERIC(15,2)`; `pg` lo entrega como string, sin pasar por `float` |
| 3. Idempotencia atómica | `INSERT … ON CONFLICT DO NOTHING RETURNING *` en una sola sentencia |
| 4. Respuesta guardada | `JSONB` |
| 5. Estados protegidos | `CHECK` y actualizaciones condicionales `UPDATE … WHERE estado = $x` |
| 6. Relaciones | FKs, índices parciales (`WHERE estado = 'PROCESANDO'`) |
| Extra | `UUID` nativo + `gen_random_uuid()`, `TIMESTAMPTZ`, `GENERATED ALWAYS AS IDENTITY` |
| Extra (fase 2) | `SELECT … FOR UPDATE SKIP LOCKED` para que varias instancias reconcilien sin pisarse |

## Alternativas consideradas

### MySQL 8
- **Viable.** Tiene transacciones, `DECIMAL`, `JSON` y `CHECK` (aplicados desde 8.0.16).
- En contra para este diseño: `INSERT … ON DUPLICATE KEY UPDATE` no devuelve la fila (no hay `RETURNING`), así que la reserva de clave necesita dos sentencias; no hay tipo `UUID` nativo; no hay índices parciales.
- Conclusión: funcionaría con ajustes, pero PostgreSQL encaja de forma más directa.

### MongoDB
- Tiene transacciones multi-documento y `Decimal128`, pero:
- El modelo es relacional (depósito → auditoría, conversión, alerta) y las restricciones de estado viven en la base. En Mongo no hay FKs ni `CHECK` equivalentes; todo queda en el código.
- Conclusión: descartado para el núcleo transaccional.

### Redis para la idempotencia (junto a PostgreSQL)
- Patrón común en la industria (`SET clave NX EX 86400`).
- En contra: añade otra pieza que operar y la clave queda en un sistema distinto al depósito, así que no se pueden guardar ambos en la misma transacción.
- Conclusión: no se justifica a la escala del proyecto. Se puede revisar si el volumen lo exige.

## Consecuencias

- **Positivas**: la base protege las reglas críticas aunque el código falle; menos piezas que operar; el SQL es estándar y conocido por el equipo.
- **A cuidar**:
  - No convertir `NUMERIC` a `Number` en Node (`01-arquitectura.md §5`).
  - Pool de conexiones acotado (`DB_POOL_MAX`) y transacciones cortas, **sin** llamadas HTTP dentro.
  - Migraciones versionadas en `db/` (scripts SQL numerados; herramienta sugerida: `node-pg-migrate`).
- **Entorno de clase**: los cuatro servicios pueden compartir **un contenedor** PostgreSQL con **una base por servicio** y un usuario por base. Lo que no se permite es que un servicio use la base de otro (RN-ARQ-01).
