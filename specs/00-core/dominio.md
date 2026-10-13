# Dominio: el depósito

## Entidad Deposito

| Campo | Tipo | Regla |
|---|---|---|
| id | UUID | Lo genera el servicio |
| cuentaDestino | texto | Formato `0000-0000-000000`. Pertenece a billetera-service; no es FK |
| monto | decimal | > 0, máximo 2 decimales |
| moneda | PEN \| USD | Moneda en que se entrega el dinero |
| canal | AGENTE \| VENTANILLA \| APP | Por dónde se hace el depósito |
| referencia | texto | Opcional, máximo 140 caracteres |
| estado | ver abajo | |
| usuarioId | texto | Sale del JWT. Pertenece a usuario-service; no es FK |
| creadoEn / actualizadoEn | fecha-hora | |

## Estados

```
PENDIENTE → PROCESANDO → COMPLETADO
                       → FALLIDO
```

| Estado | Significado |
|---|---|
| PENDIENTE | Pasó validaciones y se registró |
| PROCESANDO | Se llamó a billetera y se espera respuesta |
| COMPLETADO | Billetera confirmó la acreditación |
| FALLIDO | Billetera rechazó, falló o no respondió a tiempo |

Reglas:
- Solo se permiten las transiciones del diagrama.
- COMPLETADO y FALLIDO son finales.
- Un reintento del cliente no reabre un depósito FALLIDO: crea uno nuevo (la clave de idempotencia se libera en 5xx).

## Tabla del core: depositos

```sql
CREATE TABLE depositos (
    id              UUID PRIMARY KEY,
    cuenta_destino  VARCHAR(20)   NOT NULL,
    monto           NUMERIC(15,2) NOT NULL CHECK (monto > 0),
    moneda          CHAR(3)       NOT NULL CHECK (moneda IN ('PEN','USD')),
    canal           VARCHAR(20)   NOT NULL CHECK (canal IN ('AGENTE','VENTANILLA','APP')),
    referencia      VARCHAR(140),
    estado          VARCHAR(20)   NOT NULL CHECK (estado IN ('PENDIENTE','PROCESANDO','COMPLETADO','FALLIDO')),
    usuario_id      VARCHAR(50)   NOT NULL,
    creado_en       TIMESTAMPTZ   NOT NULL DEFAULT now(),
    actualizado_en  TIMESTAMPTZ   NOT NULL DEFAULT now()
);
CREATE INDEX idx_depositos_cuenta ON depositos (cuenta_destino, creado_en DESC);
```

Esta tabla NO tiene columnas de ninguna feature. Cada feature o transversal que necesita guardar algo tiene su propia tabla con FK a `depositos(id)`:

| Tabla | Dueño | Relación |
|---|---|---|
| claves_idempotencia | 10-transversales/idempotencia | 1 a 0..1 |
| auditoria_depositos | 10-transversales/auditoria | 1 a N |
| conversiones_deposito | 20-features/conversion-moneda | 1 a 0..1 |
| alertas_uif | 20-features/alerta-uif | 1 a 0..1 |
| limites_canal | 20-features/limites-por-canal | configuración, sin FK |
| tipos_cambio | 20-features/conversion-moneda | configuración, sin FK |
