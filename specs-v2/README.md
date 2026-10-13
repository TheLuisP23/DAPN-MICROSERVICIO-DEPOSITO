# Specs de deposito-service

Las personas editan las specs; los subagentes escriben el código, las pruebas y la revisión. Si algo no está en una spec, no se programa: el agente se detiene y lo reporta (PRY-R12).

Fuentes: `arquitectura-deposito-service.pdf` y las specs v1 de `specs/`.

## Specs, en orden de implementación

Cada spec solo depende de las anteriores. Implementarlas en este orden.

- [00-proyecto](00-proyecto.md) · PRY · Contexto, stack, reglas de oro, infraestructura, pruebas y revisión.
- [01-dominio](01-dominio.md) · DOM · Depósito, estados, tabla `depositos`, objeto JSON y errores.
- [02-integraciones](02-integraciones.md) · INT · Contratos con billetera y usuario, mocks y datos de prueba.
- [03-seguridad](03-seguridad.md) · SEG · Correlation id, log por petición, JWT, roles y permisos.
- [04-auditoria](04-auditoria.md) · AUD · Registro inmutable de cada intento.
- [05-idempotencia](05-idempotencia.md) · IDE · `Idempotency-Key` y reintentos seguros.
- [06-pipeline-y-cargador](06-pipeline-y-cargador.md) · PIP · Huecos del pipeline, contrato de módulos y `crearApp`.
- [07-registrar-deposito](07-registrar-deposito.md) · RD · `POST /depositos`.
- [08-consultar-deposito](08-consultar-deposito.md) · CD · `GET /depositos/{id}`.
- [09-historial-depositos](09-historial-depositos.md) · HD · `GET /cuentas/{numero}/depositos`.
- [10-limites-por-canal](10-limites-por-canal.md) · LC · Validador: monto máximo por canal.
- [11-conversion-moneda](11-conversion-moneda.md) · CM · Transformador: PEN ↔ USD.
- [12-alerta-uif](12-alerta-uif.md) · UIF · Post-proceso: alerta antilavado.
- [manifest.yaml](manifest.yaml) · Qué features están activas.

Todas tienen las mismas 6 secciones: **1.** Metadatos y Tech Stack · **2.** Archivos Involucrados · **3.** Flujo del Sistema · **4.** Reglas de Negocio · **5.** El Contrato · **6.** Instrucciones de Ejecución para el Agente.

## Cómo trabajar

1. Editas una spec y subes su "Versión spec" (1.1 → 1.2).
2. Pides: `desarrolladorback, aplica la SPEC-10`.
3. Luego `tester, SPEC-10` y luego `revisor, SPEC-10`.
4. Revisas `tests/reporte.md` y el veredicto del revisor.

## Reglas para editar una spec

- Máximo **150 líneas**. Si crece más, hay que dividirla.
- Una regla por línea con su id (`LC-R03`) y un caso de prueba por línea con el suyo (`LC-T03`). El test se llama igual.
- No reutilices un id borrado: marca la regla `(eliminada)` para que el agente sepa qué código quitar.
- No repitas información: cita la otra spec ("formato de error: SPEC-01 §5").
- Lo que no se sabe se escribe `[POR ACORDAR]`; nunca un valor inventado.
- Si agregas un archivo, agrégalo en §2: el agente solo toca lo que está ahí.

## Agregar una feature

1. Copia la spec más parecida (validador: 10, transformador: 11, post-proceso: 12, endpoint: 08) con el siguiente número libre.
2. Llena las 6 secciones. Su SQL va en `db/<número>-<id>.sql`.
3. Agrega su entrada en `manifest.yaml`.

## Quitar una feature

1. Pon `estado: inactivo` en `manifest.yaml`: el código queda, pero no se carga.
2. Para borrarla del todo, pide al agente borrar su carpeta en `src/modules/` y su `.sql`.

## Decisiones tomadas (donde el PDF y las specs v1 se contradecían o no decían nada)

- **D1** · Un reintento con la misma clave tras un fallo de billetera reusa el mismo `depositoId` (estado de clave `REINTENTABLE`), para que billetera no acredite dos veces. → 01, 05, 07
- **D2** · La auditoría va antes de `autenticar` para registrar también los 401 (v1 lo exigía, pero el orden del PDF no lo permitía). → 04, 06
- **D3** · `monto` viaja como string (`"150.00"`), porque v1 prohíbe `Number` para dinero. → 00, 07
- **D4** · Formato inválido → 400; monto con valor inválido → 422. → 07
- **D5** · La clave de idempotencia es única por usuario. → 05
- **D6** · Los transformadores guardan sus datos con `persistir(deposito, tx, contexto)` dentro de la transacción del depósito. → 06, 11
- **D7** · `depositos` guarda `monto_acreditar` y `moneda_acreditar` para que el reintento no dependa de tablas de features. → 01, 07
- **D8** · Como en el PDF, el service de registrar-deposito orquesta el flujo y el pipeline solo ejecuta los huecos. → 06, 07
- **D9** · UIF compara el monto depositado en su moneda original. → 12
- **D10** · El `orden` de las extensiones vive en el manifest. → 06
- **D11** · Express 4 con `manejar(fn)` para errores `async`. → 01

## Pendientes ([POR ACORDAR])

- **P1** · Rutas y campos de billetera (consultar cuenta, acreditar). Equipo billetera. → 02
- **P2** · ¿Billetera usa `depositoId` como clave de idempotencia y responde 409? Equipo billetera. → 02
- **P3** · Autenticación entre servicios (`BILLETERA_TOKEN`) y valores posibles de `estado` de una cuenta. Equipo billetera. → 02
- **P4** · Algoritmo JWT, `iss`, `aud`, nombres de roles y si un token trae varios roles. Equipo usuario. → 02, 03
- **P5** · Fuente del tipo de cambio. → 11
- **P6** · Umbral UIF vigente según la normativa SBS. → 12
- **P7** · Límites reales por canal. → 10
- **P8** · Stack del frontend y herramienta de API Gateway. → 00

## Riesgos conocidos (fuera de alcance)

- Si el proceso se cae con una clave `EN_PROCESO`, esa clave responde 409 hasta que se borre (IDE-R10).
