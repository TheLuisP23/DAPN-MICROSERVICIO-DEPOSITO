# 00 · Convenciones de los specs

| Campo | Valor |
|---|---|
| Versión | 0.2 |
| Aplica a | Todos los specs de `specs/` |
| Lo usan | Personas del equipo y agentes de `.claude/agents/` |

## 1. Para qué sirven los specs

Un spec describe **qué** debe hacer una parte del servicio y **cómo se comprueba** que lo hace. El código se escribe a partir del spec, los tests se escriben a partir de sus criterios de aceptación y la revisión compara el código contra el spec.

Regla de oro: **si el código y el spec no coinciden, uno de los dos está mal**. Se corrige el que esté mal en el mismo PR; nunca se deja la diferencia.

## 2. Orden de lectura

1. `00-core/` — reglas que valen para todo el servicio.
2. `10-transversales/` — lo que aplica a todas las peticiones (seguridad, idempotencia, auditoría).
3. `20-features/` — cada funcionalidad.
4. `30-integraciones/` — contratos con otros microservicios.

Un spec de un número mayor puede depender de uno menor, nunca al revés.

## 3. Marcas de estado

| Marca | Significado | Quién lo resuelve |
|---|---|---|
| `[POR ACORDAR]` | Depende de otro equipo (billetera, usuario, gateway). | Se acuerda con ese equipo y se actualiza el spec. |
| `[DECISIÓN EQUIPO]` | Propuesta interna que cambia el documento de arquitectura v0.1 y falta confirmar. | Equipo depósito. |
| `[PENDIENTE]` | Dato que falta conseguir (valor, normativa). | Responsable asignado en `8. Pendientes` de la arquitectura. |
| `[FASE 2]` | Fuera del alcance de la primera entrega. | — |

Mientras un punto tenga marca, el código usa el **valor provisional** que indica el spec y lo deja configurable.

## 4. Identificadores

Cada regla y cada criterio llevan un ID estable para poder citarlos en código, tests y PRs.

| Prefijo | Uso | Ejemplo |
|---|---|---|
| `RN-<FEAT>-NN` | Regla de negocio | `RN-REG-03` |
| `CA-<FEAT>-NN` | Criterio de aceptación (se convierte en test) | `CA-REG-07` |
| `ERR-<CODIGO>` | Código de error del catálogo | `ERR-LIMITE_CANAL_EXCEDIDO` |

Abreviaturas de feature: `REG` registrar, `CON` consultar, `HIS` historial, `LIM` límites, `CNV` conversión, `UIF` alerta UIF, `SEG` seguridad, `IDE` idempotencia, `AUD` auditoría, `BIL` billetera, `USR` usuario, `PIP` pipeline.

Los tests citan el ID en su nombre:

```js
test('CA-REG-07 · rechaza monto con más de 2 decimales', async () => { /* ... */ });
```

## 5. Plantilla de un spec de feature

```markdown
# <Nombre>

| Campo | Valor |
|---|---|
| Versión | |
| Tipo | feature base / validador / transformador / post-proceso |
| Carpeta | src/modules/<nombre>/ |
| Depende de | specs que necesita |

## 1. Propósito          — una o dos frases
## 2. Contrato           — endpoint o firma del módulo
## 3. Reglas de negocio  — RN-XXX-NN
## 4. Datos              — tablas que lee/escribe
## 5. Errores            — códigos del catálogo que puede producir
## 6. Criterios de aceptación — CA-XXX-NN en formato Dado/Cuando/Entonces
## 7. Fuera de alcance   — lo que NO hace (evita que el agente invente)
## 8. Pendientes         — marcas abiertas
```

## 6. Flujo de trabajo con los agentes

| Agente | Lee | Produce |
|---|---|---|
| `desarrolladorback` | spec de la feature + `00-core` + transversales que apliquen | Código en `src/` siguiendo las capas |
| `tester` | sección **Criterios de aceptación** | Tests en `tests/` con el ID `CA-…` en el nombre |
| `revisor` | spec + diff | Lista de diferencias código ↔ spec y violaciones de capas |
| `desarrolladorfront` | sección **Contrato** de cada feature | Consumo de la API (el front es compartido; este agente solo usa los contratos) |

Al pedir trabajo a un agente se le indica **el spec exacto**, por ejemplo: *"Implementa `specs/20-features/limites-por-canal.md`"*.

## 7. Cambios a un spec

- Se sube la versión del spec (0.2 → 0.3) y se añade una línea en su historial.
- Si el cambio rompe un contrato con otro equipo, se avisa antes de mergear.
- Los criterios de aceptación no se borran: si dejan de aplicar, se marcan `~~tachado~~ (obsoleto en 0.x)`.

## 8. Formato de datos en los contratos

- JSON con **camelCase** en la API; **snake_case** en PostgreSQL.
- Montos como **string decimal** con 2 decimales en las respuestas (`"150.50"`). Ver `01-arquitectura.md §5`.
- Fechas en ISO-8601 con zona horaria (`2026-10-13T14:49:00.000Z`).
- IDs de depósito: UUID v4.
