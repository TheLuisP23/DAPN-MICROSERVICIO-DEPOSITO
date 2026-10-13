# Contexto del proyecto

## El sistema

Proyecto de curso: un banco formado por 4 microservicios que se integran al final.

| Microservicio | Equipo | Responsabilidad |
|---|---|---|
| deposito-service | Nuestro equipo | Registrar depósitos y pedir su acreditación |
| billetera-service | Otro equipo | Cuentas y saldos (único dueño del saldo) |
| usuario-service | Otro equipo | Usuarios, inicio de sesión, emisión de JWT con roles |
| pagos-service | Otro equipo | Pagos a terceros |

Todas las peticiones externas entran por un API Gateway.

## Responsabilidad de deposito-service

Recibe la solicitud, la valida, registra la operación y le pide a billetera-service que acredite el dinero.

- SÍ: guarda el registro de cada depósito (quién, cuánto, cuándo, por qué canal, en qué estado).
- NO: guarda ni modifica saldos.
- NO: accede a la base de datos de otro microservicio. Solo HTTP.

## Alcance actual

Nivel 1: arquitectura interna del microservicio.

Fuera de alcance por ahora: sagas, broker de eventos, outbox, blockchain, integración de nivel 2.
