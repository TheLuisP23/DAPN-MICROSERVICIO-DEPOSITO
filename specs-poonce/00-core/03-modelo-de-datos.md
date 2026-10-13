# 03 · Modelo de datos (PostgreSQL)

| Campo | Valor |
|---|---|
| Versión | 0.2 |
| Motor | PostgreSQL 16 (mínimo 13, por `gen_random_uuid()` nativo) |
| Base | `depositos_db` — exclusiva de deposito-service |
| Scripts | `db/` (numerados: `001_esquema.sql`, `002_semillas.sql`, …) |
| Depende de | `adr-001-postgresql.md` |

## 1. Reglas generales

- **RN-DAT-01** Cada módulo de extensión tiene su propia tabla con FK a `depositos`.
- **RN-DAT-02** `cuenta_destino` y `usuario_id` **no son FK**: pertenecen a billetera-service y usuario-service.
- **RN-DAT-03** Todos los estados y catálogos se protegen con `CHECK`; no se confía solo en el código.
- **RN-DAT-04** Todas las fechas son `TIMESTAMPTZ` (se guardan en UTC).
- **RN-DAT-05** `depositos.id` lo genera la aplicación (`crypto.randomUUID()`), porque se necesita antes del INSERT (pipeline, `extensiones`, `depositoId` hacia billetera).
- **RN-DAT-06** `auditoria_depositos` es **solo inserción**: el usuario de base de datos de la aplicación no tiene permisos `UPDATE`/`DELETE` sobre ella.

## 2. Cambios respecto al documento v0.1

| Tabla | Cambio | Motivo |
|---|---|---|
| `claves_idempotencia` | PK `(usuario_id, clave)` en vez de `clave` | Dos usuarios podían chocar con la misma clave. |
| `claves_idempotencia` | `deposito_id` **nullable** | La clave se reserva antes de que exista el depósito. |
| `claves_idempotencia` | Nuevo estado `REINTENTABLE`; columnas `actualizado_en`, `expira_en` | Reintentos seguros tras timeout de billetera y caducidad de claves. Ver `10-transversales/idempotencia.md`. |
| `auditoria_depositos` | `deposito_id` **nullable**; nuevas `moneda`, `status_http` | Se auditan también los rechazos (422) que nunca se guardan. |
| `depositos` | Nueva `motivo_fallo` | Saber por qué quedó `FALLIDO`. |
| `limites_canal` | PK `(canal, moneda)` | Con PK = canal solo cabía una moneda por canal. |
| `tipos_cambio` | `compra` y `venta` en vez de `valor` **[DECISIÓN EQUIPO]** | El banco compra y vende la divisa a precios distintos. |
| `conversiones_deposito` | Nuevas `moneda_origen`, `operacion`, `tipo_cambio_id` | Trazabilidad de qué tasa se usó. |
| `alertas_uif` | Nuevas `moneda`, `monto_equivalente`, `umbral_moneda`; estado `REVISAR` | Comparar contra el umbral en una sola moneda. |

## 3. DDL

```sql
-- =========================================================================
-- 001_esquema.sql · deposito-service
-- =========================================================================

-- ---------- Tablas de configuración (las leen los módulos; sin relaciones)

CREATE TABLE limites_canal (
    canal         VARCHAR(12)   NOT NULL
                  CHECK (canal IN ('AGENTE', 'VENTANILLA', 'APP')),
    moneda        CHAR(3)       NOT NULL
                  CHECK (moneda IN ('PEN', 'USD')),
    monto_maximo  NUMERIC(15,2) NOT NULL CHECK (monto_maximo > 0),
    PRIMARY KEY (canal, moneda)
);

CREATE TABLE tipos_cambio (
    id             UUID          PRIMARY KEY DEFAULT gen_random_uuid(),
    moneda         CHAR(3)       NOT NULL DEFAULT 'USD'
                   CHECK (moneda IN ('USD')),          -- divisa cotizada en PEN
    compra         NUMERIC(10,4) NOT NULL CHECK (compra > 0),
    venta          NUMERIC(10,4) NOT NULL CHECK (venta > 0),
    vigente_desde  TIMESTAMPTZ   NOT NULL,
    CONSTRAINT ck_tipos_cambio_spread CHECK (venta >= compra),
    CONSTRAINT uq_tipos_cambio_vigencia UNIQUE (moneda, vigente_desde)
);

-- ---------- Tabla principal

CREATE TABLE depositos (
    id              UUID          PRIMARY KEY,
    cuenta_destino  VARCHAR(20)   NOT NULL,                 -- ref. billetera, sin FK
    monto           NUMERIC(15,2) NOT NULL CHECK (monto > 0),
    moneda          CHAR(3)       NOT NULL CHECK (moneda IN ('PEN', 'USD')),
    canal           VARCHAR(12)   NOT NULL
                    CHECK (canal IN ('AGENTE', 'VENTANILLA', 'APP')),
    referencia      VARCHAR(100),
    estado          VARCHAR(12)   NOT NULL DEFAULT 'PENDIENTE'
                    CHECK (estado IN ('PENDIENTE', 'PROCESANDO', 'COMPLETADO', 'FALLIDO')),
    motivo_fallo    VARCHAR(200),
    usuario_id      VARCHAR(64)   NOT NULL,                 -- ref. usuario, sin FK
    creado_en       TIMESTAMPTZ   NOT NULL DEFAULT now(),
    actualizado_en  TIMESTAMPTZ   NOT NULL DEFAULT now(),
    CONSTRAINT ck_depositos_motivo_fallo
        CHECK ((estado = 'FALLIDO') = (motivo_fallo IS NOT NULL))
);

CREATE INDEX ix_depositos_cuenta_fecha  ON depositos (cuenta_destino, creado_en DESC);
CREATE INDEX ix_depositos_usuario_fecha ON depositos (usuario_id, creado_en DESC);
CREATE INDEX ix_depositos_procesando    ON depositos (actualizado_en)
    WHERE estado = 'PROCESANDO';                          -- para reconciliación

-- ---------- Transversales

CREATE TABLE claves_idempotencia (
    usuario_id      VARCHAR(64)  NOT NULL,
    clave           VARCHAR(100) NOT NULL,
    deposito_id     UUID         REFERENCES depositos (id),  -- NULL hasta crear el depósito
    hash_solicitud  CHAR(64)     NOT NULL,                   -- SHA-256 hex del cuerpo canónico
    estado          VARCHAR(12)  NOT NULL DEFAULT 'EN_PROCESO'
                    CHECK (estado IN ('EN_PROCESO', 'REINTENTABLE', 'TERMINADO')),
    status_http     SMALLINT,
    respuesta       JSONB,
    creado_en       TIMESTAMPTZ  NOT NULL DEFAULT now(),
    actualizado_en  TIMESTAMPTZ  NOT NULL DEFAULT now(),
    expira_en       TIMESTAMPTZ  NOT NULL DEFAULT now() + INTERVAL '24 hours',
    PRIMARY KEY (usuario_id, clave),
    CONSTRAINT ck_idem_terminado
        CHECK (estado <> 'TERMINADO' OR (status_http IS NOT NULL AND respuesta IS NOT NULL))
);

CREATE INDEX ix_idem_expira ON claves_idempotencia (expira_en);

CREATE TABLE auditoria_depositos (
    id              BIGINT       GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    deposito_id     UUID         REFERENCES depositos (id),  -- NULL si se rechazó antes de guardar
    usuario_id      VARCHAR(64),                             -- NULL si falló la autenticación
    accion          VARCHAR(30)  NOT NULL
                    CHECK (accion IN ('REGISTRAR_DEPOSITO', 'CONSULTAR_DEPOSITO', 'CONSULTAR_HISTORIAL')),
    ip              VARCHAR(45),                             -- cabe IPv6
    canal           VARCHAR(12),
    monto           NUMERIC(15,2),
    moneda          CHAR(3),
    resultado       VARCHAR(40)  NOT NULL,                   -- 'OK' o código de error
    status_http     SMALLINT     NOT NULL,
    correlation_id  VARCHAR(64)  NOT NULL,
    creado_en       TIMESTAMPTZ  NOT NULL DEFAULT now()
);

CREATE INDEX ix_auditoria_deposito ON auditoria_depositos (deposito_id);
CREATE INDEX ix_auditoria_fecha    ON auditoria_depositos (creado_en);

-- ---------- Tablas de módulos de extensión

CREATE TABLE conversiones_deposito (
    deposito_id     UUID          PRIMARY KEY REFERENCES depositos (id),
    moneda_origen   CHAR(3)       NOT NULL CHECK (moneda_origen IN ('PEN', 'USD')),
    moneda_cuenta   CHAR(3)       NOT NULL CHECK (moneda_cuenta IN ('PEN', 'USD')),
    operacion       VARCHAR(6)    NOT NULL CHECK (operacion IN ('COMPRA', 'VENTA')),
    tipo_cambio     NUMERIC(10,4) NOT NULL CHECK (tipo_cambio > 0),
    monto_final     NUMERIC(15,2) NOT NULL CHECK (monto_final > 0),
    tipo_cambio_id  UUID          NOT NULL REFERENCES tipos_cambio (id),
    CONSTRAINT ck_conversion_monedas CHECK (moneda_origen <> moneda_cuenta)
);

CREATE TABLE alertas_uif (
    id                 UUID          PRIMARY KEY DEFAULT gen_random_uuid(),
    deposito_id        UUID          NOT NULL UNIQUE REFERENCES depositos (id),
    monto              NUMERIC(15,2) NOT NULL,
    moneda             CHAR(3)       NOT NULL,
    monto_equivalente  NUMERIC(15,2),                        -- en umbral_moneda; NULL si no se pudo convertir
    umbral             NUMERIC(15,2) NOT NULL,
    umbral_moneda      CHAR(3)       NOT NULL,
    estado_reporte     VARCHAR(12)   NOT NULL DEFAULT 'PENDIENTE'
                       CHECK (estado_reporte IN ('PENDIENTE', 'REVISAR', 'REPORTADO', 'DESCARTADO')),
    creado_en          TIMESTAMPTZ   NOT NULL DEFAULT now()
);
```

## 4. Semillas de desarrollo (`002_semillas.sql`)

> Valores **de ejemplo** para desarrollo y tests. Los reales los define el negocio. **[PENDIENTE]**

```sql
INSERT INTO limites_canal (canal, moneda, monto_maximo) VALUES
    ('APP',        'PEN',   5000.00),
    ('APP',        'USD',   1500.00),
    ('AGENTE',     'PEN',  10000.00),
    ('AGENTE',     'USD',   3000.00),
    ('VENTANILLA', 'PEN', 100000.00),
    ('VENTANILLA', 'USD',  30000.00);

INSERT INTO tipos_cambio (moneda, compra, venta, vigente_desde) VALUES
    ('USD', 3.7120, 3.7480, '2026-10-01T00:00:00-05:00');
```

## 5. Consultas clave (referencia para los repositories)

```sql
-- Reservar clave de idempotencia (atómico; si devuelve 0 filas, ya existía)
INSERT INTO claves_idempotencia (usuario_id, clave, hash_solicitud)
VALUES ($1, $2, $3)
ON CONFLICT (usuario_id, clave) DO NOTHING
RETURNING *;

-- Transición de estado protegida (si rowCount = 0, el estado no era el esperado)
UPDATE depositos
   SET estado = $2, motivo_fallo = $3, actualizado_en = now()
 WHERE id = $1 AND estado = $4;

-- Tipo de cambio vigente
SELECT id, compra, venta
  FROM tipos_cambio
 WHERE moneda = 'USD' AND vigente_desde <= now()
 ORDER BY vigente_desde DESC
 LIMIT 1;

-- Historial de una cuenta (paginado)
SELECT * FROM depositos
 WHERE cuenta_destino = $1
   AND ($2::timestamptz IS NULL OR creado_en >= $2)
   AND ($3::timestamptz IS NULL OR creado_en <  $3)
   AND ($4::varchar     IS NULL OR estado = $4)
 ORDER BY creado_en DESC, id DESC
 LIMIT $5 OFFSET $6;
```

## 6. Usuario de base de datos

```sql
CREATE ROLE deposito_app LOGIN PASSWORD '...';
GRANT SELECT, INSERT, UPDATE ON depositos, claves_idempotencia,
      conversiones_deposito, alertas_uif TO deposito_app;
GRANT DELETE ON claves_idempotencia TO deposito_app;          -- purga de claves expiradas
GRANT SELECT ON limites_canal, tipos_cambio TO deposito_app;
GRANT SELECT, INSERT ON auditoria_depositos TO deposito_app;  -- sin UPDATE/DELETE
```

## 7. Criterios de aceptación

- **CA-DAT-01** Dado el DDL, cuando se ejecuta sobre una base vacía de PostgreSQL 16, entonces termina sin errores.
- **CA-DAT-02** Dado un INSERT en `depositos` con `estado = 'OTRO'`, entonces PostgreSQL lo rechaza.
- **CA-DAT-03** Dado un depósito `FALLIDO` sin `motivo_fallo`, entonces PostgreSQL lo rechaza.
- **CA-DAT-04** Dados dos usuarios distintos con la misma `clave`, entonces ambas filas se insertan.
- **CA-DAT-05** Dado el rol `deposito_app`, cuando ejecuta `UPDATE auditoria_depositos`, entonces recibe error de permisos.
