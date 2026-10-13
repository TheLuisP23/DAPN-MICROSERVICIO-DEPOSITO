# Specs de deposito-service

Las personas editan las specs y los subagentes escriben el código. Si algo no está en una spec, no se programa.

## Qué hay y en qué orden se implementa

| Spec | Qué define | Prefijo de ids |
|---|---|---|
| [00-proyecto](00-proyecto.md) | Contexto, stack, reglas de oro, infraestructura, variables de entorno | PRY |
| [01-core](01-core.md) | Depósito, estados, errores, pipeline con huecos, cargador | CORE |
| [02-seguridad](02-seguridad.md) | Correlation id, JWT, roles y canales | SEG |
| [03-auditoria](03-auditoria.md) | Registro inmutable de cada intento | AUD |
| [04-idempotencia](04-idempotencia.md) | `Idempotency-Key` y reintentos seguros | IDE |
| [05-registrar-deposito](05-registrar-deposito.md) | `POST /depositos` | RD |
| [06-consultar-depositos](06-consultar-depositos.md) | `GET /depositos/{id}` e historial | CON |
| [07-limites-por-canal](07-limites-por-canal.md) | Validador: monto máximo por canal | LC |
| [08-conversion-moneda](08-conversion-moneda.md) | Transformador: PEN ↔ USD | CM |
| [09-alerta-uif](09-alerta-uif.md) | Post-proceso: alerta antilavado | UIF |
| [10-integraciones](10-integraciones.md) | Contratos con billetera y usuario, y sus mocks | INT |
| [manifest.yaml](manifest.yaml) | Qué features están activas | — |

Todas las specs tienen las mismas 6 secciones: **1.** Metadatos y Tech Stack · **2.** Archivos Involucrados · **3.** Flujo del Sistema · **4.** Reglas de Negocio · **5.** El Contrato · **6.** Instrucciones de Ejecución para el Agente.

## Cómo trabajar

1. Editas una spec y subes su "Versión spec" (1.0 → 1.1).
2. Pides: `desarrolladorback, aplica la SPEC-07`.
3. Luego `tester, SPEC-07` y luego `revisor, SPEC-07`.
4. Revisas el reporte del revisor.

## Reglas para editar una spec

- Máximo **150 líneas** por spec. Si crece más, es señal de que hay que dividirla.
- Cada regla tiene un id (`LC-R03`) y cada prueba también (`LC-T03`). El test se llama igual, así un cambio se rastrea hasta el código.
- No reutilices un id borrado: marca la regla como `(eliminada)` para que el revisor sepa qué código quitar.
- No repitas información: cita la otra spec ("formato de error: SPEC-01 §5").
- Lo que no se sabe se escribe `[POR ACORDAR]`; nunca un valor inventado.

## Agregar una feature

1. Copia la spec más parecida (validador: 07, transformador: 08, post-proceso: 09, endpoint: 06) con el siguiente número libre.
2. Llena las 6 secciones.
3. Agrega su entrada en `manifest.yaml`.

## Quitar una feature

1. Pon `estado: inactivo` en `manifest.yaml`. El código queda, pero no se carga.
2. Si se va a borrar del todo, pide al agente borrar su carpeta en `src/modules/` y su `.sql`.

## Decisiones tomadas

| # | Decisión | Spec |
|---|---|---|
| D1 | Un reintento con la misma `Idempotency-Key` tras un fallo de billetera reutiliza el mismo `depositoId`, para que billetera no acredite dos veces | 01, 04 |
| D2 | La auditoría corre antes de `autenticar`, así también registra los 401 | 03 |
| D3 | `monto` viaja como string (`"150.00"`) en request, response y hacia billetera | 00, 05 |
| D4 | Formato inválido da 400; un monto con valor inválido da 422 | 05 |
| D5 | La clave de idempotencia es única por usuario | 04 |
| D6 | Los transformadores guardan sus datos con `persistir(deposito, tx)` dentro de la transacción del depósito | 01, 08 |
| D7 | Rol, canal y titular se verifican en el core después de consultar la cuenta | 01, 02 |
| D8 | Consultar e historial son un solo módulo (`consultar-depositos`) | 06 |
| D9 | UIF compara el monto depositado en su moneda original | 09 |
| D10 | El `orden` de las extensiones vive en el manifest, no en el código | 01 |

## Pendientes ([POR ACORDAR])

| # | Tema | Con quién | Spec |
|---|---|---|---|
| P1 | Rutas y campos de billetera (consultar cuenta, acreditar) | Equipo billetera | 10 |
| P2 | ¿Billetera usa `depositoId` como clave de idempotencia y responde 409? | Equipo billetera | 10 |
| P3 | Autenticación entre servicios (`BILLETERA_TOKEN`) | Equipo billetera | 10 |
| P4 | Valores posibles de `estado` de una cuenta | Equipo billetera | 10 |
| P5 | Algoritmo JWT (RS256 o HS256), `iss`, `aud` y nombres de roles | Equipo usuario | 02, 10 |
| P6 | ¿Un token puede traer varios roles? | Equipo usuario | 02 |
| P7 | Fuente del tipo de cambio | Equipo | 08 |
| P8 | Umbral UIF vigente según la normativa SBS | Equipo | 09 |
| P9 | Límites reales por canal | Equipo | 07 |
| P10 | Stack del frontend | Equipo | 00 |
| P11 | Herramienta de API Gateway (NGINX, Kong, Traefik) | Todos los equipos | 00 |
