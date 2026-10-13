# Front: registrar-deposito

Pantalla: **Nuevo depósito**. Roles: CLIENTE (app), AGENTE, CAJERO.

## Campos

| Campo | Control | Validación en pantalla |
|---|---|---|
| Cuenta destino | Texto con máscara `0000-0000-000000` | Formato |
| Monto | Numérico, 2 decimales | > 0 |
| Moneda | Selector PEN / USD | Obligatorio |
| Canal | Fijo según el rol (no editable) | |
| Referencia | Texto | Máx. 140 |

## Comportamiento

1. Al abrir la pantalla, generar una `Idempotency-Key` (UUID) para ESE depósito.
2. Al presionar "Depositar", deshabilitar el botón y enviar.
3. Si hay error de red o 502, mostrar "Reintentar" que reenvía con la MISMA clave.
4. Solo se genera una clave nueva al empezar otro depósito.
5. En 201 mostrar comprobante: id, monto, moneda, cuenta enmascarada, fecha y estado.
6. Si `extensiones` trae datos (ej. conversión), mostrarlos en el comprobante; si no, no mostrar nada.

## Mensajes por código

| Código | Mensaje |
|---|---|
| MONTO_INVALIDO | El monto debe ser mayor a 0 y tener hasta 2 decimales |
| CUENTA_NO_ENCONTRADA | La cuenta no existe |
| CUENTA_INACTIVA | La cuenta no está activa |
| MONEDA_NO_SOPORTADA | No se puede depositar en esa moneda a esta cuenta |
| LIMITE_CANAL_EXCEDIDO | El monto supera el máximo permitido para este canal |
| SIN_PERMISO | No tiene permiso para depositar a esta cuenta |
| BILLETERA_NO_DISPONIBLE | No se pudo completar. Intente de nuevo |
| SOLICITUD_EN_PROCESO | Su depósito se está procesando, espere un momento |
| Otro | Ocurrió un error. Código de seguimiento: {correlationId} |
