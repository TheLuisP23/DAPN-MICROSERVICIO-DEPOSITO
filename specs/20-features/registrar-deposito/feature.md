# Feature: registrar-deposito

Tipo: endpoint · Ruta: `POST /api/v1/depositos` · Código: `src/modules/registrar-deposito/`

## Descripción

Registra un depósito a una cuenta y pide a billetera-service que lo acredite. Ejecuta el pipeline del core (`00-core/puntos-extension.md`), donde se enchufan validadores, transformadores y post-procesos.

## Request

```json
{
  "cuentaDestino": "0011-0123-456789",
  "monto": 150.00,
  "moneda": "PEN",
  "canal": "AGENTE",
  "referencia": "Pago alquiler"
}
```

Headers: `Authorization`, `Idempotency-Key`, `X-Correlation-Id` (opcional).

## Middlewares (en orden)

`correlationId` → `autenticar` → `requiereRol('CLIENTE','AGENTE','CAJERO')` → `auditoria('CREAR_DEPOSITO')` → `idempotencia` → controller.

## Reglas propias (no dependen de ninguna otra feature)

| Id | Regla |
|---|---|
| RD-01 | Campos obligatorios: cuentaDestino, monto, moneda, canal. |
| RD-02 | monto > 0 y máximo 2 decimales. |
| RD-03 | La cuenta debe existir y estar ACTIVA (consulta a billetera). |
| RD-04 | Después de los transformadores, si `monedaAcreditar` es distinta a la moneda de la cuenta, se rechaza con 422 MONEDA_NO_SOPORTADA. |
| RD-05 | Se acredita `montoAcreditar` en `monedaAcreditar` (por defecto = monto y moneda; un transformador puede cambiarlos). |
| RD-06 | Estados según `00-core/dominio.md`. |

## Escenarios

**E1.1 Depósito válido**
Dado una cuenta activa en PEN
Cuando un AGENTE deposita 150.00 PEN por canal AGENTE con una clave nueva
Entonces se responde 201 con estado COMPLETADO
Y billetera recibe una acreditación de 150.00 PEN con el depositoId

**E1.2 Monto inválido**
Cuando el monto es 0, negativo o tiene 3 decimales
Entonces se responde 422 MONTO_INVALIDO
Y no se guarda ningún depósito ni se llama a billetera

**E1.3 Campo faltante**
Cuando falta cuentaDestino
Entonces se responde 400 FORMATO_INVALIDO

**E1.4 Cuenta inexistente**
Cuando billetera responde 404 al consultar la cuenta
Entonces se responde 422 CUENTA_NO_ENCONTRADA

**E1.5 Cuenta inactiva**
Cuando billetera devuelve la cuenta con estado distinto a ACTIVA
Entonces se responde 422 CUENTA_INACTIVA

**E1.6 Billetera falla al acreditar**
Cuando billetera responde 5xx o no responde en 5 s al acreditar
Entonces el depósito queda FALLIDO
Y se responde 502 BILLETERA_NO_DISPONIBLE

**E1.7 Billetera no disponible al consultar**
Cuando billetera falla al consultar la cuenta
Entonces se responde 502 BILLETERA_NO_DISPONIBLE
Y no se guarda ningún depósito

**E1.8 Moneda distinta sin conversión activa**
Dado conversion-moneda inactiva y una cuenta en PEN
Cuando se deposita en USD
Entonces se responde 422 MONEDA_NO_SOPORTADA

**E1.9 Sin extensiones activas**
Dado limites-por-canal, conversion-moneda y alerta-uif inactivas
Cuando se deposita 150.00 PEN a una cuenta en PEN
Entonces el depósito funciona igual que E1.1 y `extensiones` viene vacío

Los escenarios de seguridad, auditoría e idempotencia están en `10-transversales/`.

## Respuesta 201

```json
{
  "id": "7f1c2e9a-...",
  "cuentaDestino": "0011-0123-456789",
  "monto": 150.00,
  "moneda": "PEN",
  "canal": "AGENTE",
  "referencia": "Pago alquiler",
  "estado": "COMPLETADO",
  "creadoEn": "2026-10-13T19:20:00Z",
  "actualizadoEn": "2026-10-13T19:20:01Z",
  "extensiones": {}
}
```

## Dependencias

- Core: pipeline, `depositos`.
- Integración: billetera (consultar cuenta, acreditar).
