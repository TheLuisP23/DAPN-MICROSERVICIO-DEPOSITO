# Transversal: auditoría

Middleware: `auditoria(accion)`, por ruta. Archivo: `src/core/middlewares/auditoria.js`.

## Qué hace

Registra cada operación al terminar la respuesta: quién, cuándo, desde dónde, qué, cuánto y con qué resultado. Solo observa: nunca bloquea ni cambia la respuesta.

## Acciones

| Ruta | accion |
|---|---|
| POST /depositos | CREAR_DEPOSITO |
| GET /depositos/{id} | CONSULTAR_DEPOSITO |
| GET /cuentas/{numero}/depositos | LISTAR_DEPOSITOS |

## Reglas

1. Se registra TODO intento, incluso rechazados (401, 403, 422) y reintentos idempotentes.
2. La tabla es solo de inserción. Un trigger en BD bloquea UPDATE y DELETE.
3. Si falla el registro de auditoría, se escribe en el log (sin datos sensibles) y la respuesta al cliente no cambia.
4. El registro se hace después de enviar la respuesta (`res.on('finish')`), para no agregar latencia.
5. El controller deja `res.locals.depositoId` cuando hay un depósito involucrado.

## Tabla

```sql
CREATE TABLE auditoria_depositos (
    id              BIGSERIAL PRIMARY KEY,
    deposito_id     UUID REFERENCES depositos(id),
    usuario_id      VARCHAR(50)  NOT NULL,
    accion          VARCHAR(40)  NOT NULL,
    ip              VARCHAR(45),
    canal           VARCHAR(20),
    monto           NUMERIC(15,2),
    resultado       VARCHAR(20)  NOT NULL,  -- EXITO | ERROR_<codigo http>
    correlation_id  VARCHAR(100),
    creado_en       TIMESTAMPTZ  NOT NULL DEFAULT now()
);

CREATE FUNCTION bloquear_cambios_auditoria() RETURNS trigger AS $$
BEGIN RAISE EXCEPTION 'auditoria_depositos es solo de inserción'; END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER auditoria_inmutable BEFORE UPDATE OR DELETE ON auditoria_depositos
FOR EACH ROW EXECUTE FUNCTION bloquear_cambios_auditoria();
```

## Escenarios

**A1.** Depósito exitoso → 1 fila con resultado EXITO y deposito_id.
**A2.** Depósito rechazado por 422 → 1 fila con resultado ERROR_422 y deposito_id nulo.
**A3.** Mismo depósito reintentado 3 veces con la misma clave → 3 filas.
**A4.** Intentar `UPDATE auditoria_depositos` → la BD lanza error.
**A5.** BD de auditoría falla → el cliente igual recibe su respuesta normal.
