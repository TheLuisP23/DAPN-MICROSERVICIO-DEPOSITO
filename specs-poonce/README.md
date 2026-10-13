# Specs de deposito-service

Especificaciones del servicio. Empieza por `00-core/00-convenciones.md`; el índice completo está en `manifest.yaml`.

```
specs/
├── manifest.yaml                     índice de specs + módulos activos (lo lee el Cargador)
├── 00-core/
│   ├── 00-convenciones.md            cómo se escriben y usan los specs
│   ├── 01-arquitectura.md            capas, carpetas, dinero, transacciones
│   ├── 02-pipeline-y-modulos.md      huecos, contrato de módulo, Cargador
│   ├── 03-modelo-de-datos.md         DDL PostgreSQL + semillas + consultas clave
│   ├── 04-estados-deposito.md        máquina de estados
│   ├── 05-errores.md                 formato común y catálogo
│   ├── 06-configuracion-y-observabilidad.md
│   └── adr-001-postgresql.md         por qué PostgreSQL
├── 10-transversales/
│   ├── seguridad.md                  JWT, roles, matriz de permisos, canal
│   ├── idempotencia.md
│   └── auditoria.md
├── 20-features/
│   ├── registrar-deposito.md
│   ├── consultar-deposito.md
│   ├── historial-depositos.md
│   ├── limites-por-canal.md          validador
│   ├── conversion-moneda.md          transformador
│   └── alerta-uif.md                 post-proceso
└── 30-integraciones/
    ├── billetera.md                  propuesta para negociar con billetera
    └── usuario.md                    propuesta para negociar con usuario
```

## Decisiones pendientes de confirmar por el equipo (v0.2)

| # | Decisión | Spec |
|---|---|---|
| D1 | Timeout de billetera deja el depósito en `PROCESANDO` (no `FALLIDO`) y la clave queda `REINTENTABLE` atada al mismo depósito | `04-estados-deposito.md`, `idempotencia.md` |
| D2 | Historial en `GET /depositos?cuenta=` en vez de `GET /cuentas/:numero/depositos` | `historial-depositos.md` |
| D3 | El canal se deriva del rol del JWT | `seguridad.md §6` |
| D4 | Tipo de cambio con `compra` y `venta` | `conversion-moneda.md §3` |
| D5 | `DepositoRepository` en `src/core/repositories/` | `01-arquitectura.md §3` |
| D6 | Auditoría antes de autenticar (para auditar 401/403) | `seguridad.md §2` |
| D7 | Límite no configurado → deja pasar y loguea | `limites-por-canal.md` |
| D8 | AGENTE/CAJERO no ven historial por cuenta | `seguridad.md §5` |

## Pendientes con otros equipos

Ver `30-integraciones/billetera.md §9` y `30-integraciones/usuario.md §7`.
