# 01 · Arquitectura interna

| Campo | Valor |
|---|---|
| Versión | 0.2 |
| Stack | Node.js 20 LTS + Express 4 + PostgreSQL 16 (driver `pg`) |
| Estilo | Capas + core con puntos de extensión |
| Depende de | `00-convenciones.md` |

## 1. Contexto

deposito-service es uno de cuatro microservicios del banco (usuario, billetera, depósito, pagos). Registra depósitos a cuentas y los acredita llamando a **billetera-service**, que es el dueño de los saldos.

Reglas de contexto (no negociables):

- **RN-ARQ-01** deposito-service solo habla con otros servicios por **HTTP**. Nunca abre conexión a la base de datos de otro servicio.
- **RN-ARQ-02** deposito-service **no guarda saldos**. El saldo de una cuenta solo existe en billetera.
- **RN-ARQ-03** `cuenta_destino` y `usuario_id` son referencias a otros servicios: se guardan como texto, **sin FK**.
- **RN-ARQ-04** Toda petición entra por el API Gateway. El servicio confía en `X-Forwarded-For` solo del gateway (`app.set('trust proxy', 1)`).

## 2. Capas

Una petición baja siempre en este orden. **Ninguna capa se salta a otra** y ninguna capa llama a una superior.

| Capa | Carpeta | Responsabilidad | Prohibido |
|---|---|---|---|
| 0 · Middlewares | `src/core/middlewares/` | correlationId → autenticar → requiereRol → auditoría → idempotencia (solo POST) | Lógica de negocio |
| 1 · Presentación | `src/modules/*/routes.js`, `controller.js` | Leer petición, validar **forma** (tipos, formato), llamar al service, responder | Reglas de negocio, SQL, HTTP saliente |
| 2 · Negocio | `src/modules/*/service.js`, `src/core/pipeline.js` | Reglas de negocio, orquestar el flujo, transiciones de estado | SQL, `fetch`/`axios`, conocer URLs |
| 3 · Datos e integración | `src/core/repositories/`, `src/modules/*/repository.js`, `src/core/clients/` | SQL parametrizado, llamadas HTTP a otros servicios | Reglas de negocio |

- **RN-ARQ-05** Solo la capa 3 importa `pg` o un cliente HTTP.
- **RN-ARQ-06** Solo la capa 3 conoce nombres de tablas y URLs de otros servicios.
- **RN-ARQ-07** Los controllers no tienen `if` de negocio: validan forma, llaman al service y traducen el resultado a HTTP.
- **RN-ARQ-08** Todo SQL usa parámetros (`$1, $2…`). Prohibido concatenar valores en SQL.

### Validación de forma vs. validación de negocio

| Tipo | Ejemplo | Dónde | Código |
|---|---|---|---|
| Forma | `monto` no es número, `moneda` no es PEN/USD, falta `cuentaDestino` | Controller (capa 1) | 400 `VALIDACION_FALLIDA` |
| Negocio | Cuenta inactiva, supera límite del canal | Service / validadores del pipeline (capa 2) | 422 |

## 3. Estructura de carpetas

> **[DECISIÓN EQUIPO]** Cambio respecto a v0.1: `DepositoRepository` se mueve a `src/core/repositories/` porque lo usan tres features (registrar, consultar, historial). Si se queda en `registrar-deposito/`, consultar e historial dependerían de otro módulo.

```
src/
├── app.js                       # arma Express, registra middlewares y rutas, arranca el Cargador
├── server.js                    # escucha el puerto, apagado ordenado (SIGTERM)
├── core/
│   ├── db.js                    # Pool de pg + helper transaccion(fn)
│   ├── cargador.js              # lee manifest.yaml y registra módulos en el pipeline
│   ├── pipeline.js              # huecos: validadores, transformadores, acreditar, post-procesos
│   ├── errores.js               # jerarquía de errores + manejador común
│   ├── dinero.js                # helpers con decimal.js (parsear, sumar, redondear, formatear)
│   ├── config.js                # lee y valida variables de entorno al arrancar
│   ├── logger.js                # pino con correlationId
│   ├── middlewares/
│   │   ├── correlationId.js
│   │   ├── autenticar.js
│   │   ├── requiereRol.js
│   │   ├── auditoria.js
│   │   └── idempotencia.js
│   ├── repositories/
│   │   ├── depositoRepository.js
│   │   ├── idempotenciaRepository.js
│   │   └── auditoriaRepository.js
│   └── clients/
│       └── billeteraClient.js
└── modules/
    ├── registrar-deposito/      # routes, controller, service
    ├── consultar-deposito/      # routes, controller, service
    ├── historial-depositos/     # routes, controller, service
    ├── limites-por-canal/       # index.js (contrato de módulo) + repository.js
    ├── conversion-moneda/       # index.js + repository.js
    └── alerta-uif/              # index.js + repository.js
```

Los módulos de extensión tienen su **propio** `repository.js` para su propia tabla. Nunca escriben en `depositos` directamente.

## 4. Arranque

1. `config.js` valida variables de entorno. Si falta una obligatoria, el proceso **termina** con un mensaje claro (fallar rápido).
2. `db.js` crea el pool y hace `SELECT 1`.
3. `cargador.js` lee `manifest.yaml`, carga los módulos `activo: true`, valida su contrato y los registra en el pipeline. Ver `02-pipeline-y-modulos.md`.
4. `app.js` monta middlewares, rutas, el manejador de errores y `/health`, `/ready`.
5. `server.js` escucha `PORT`. En `SIGTERM` deja de aceptar conexiones, espera las peticiones en curso (máx. 10 s) y cierra el pool.

## 5. Dinero

- **RN-ARQ-09** Nunca se usa `Number` para operar montos. Se usa `decimal.js` a través de `src/core/dinero.js`.
- **RN-ARQ-10** El driver `pg` devuelve `NUMERIC` como string; se mantiene como string hasta convertirlo a `Decimal`.
- **RN-ARQ-11** Montos: máximo 13 dígitos enteros y **2 decimales** (`NUMERIC(15,2)`). Se rechaza un monto con más de 2 decimales; **no** se redondea en silencio.
- **RN-ARQ-12** Redondeo de resultados calculados (conversiones): `ROUND_HALF_UP` a 2 decimales.
- **RN-ARQ-13** La API responde montos como string con exactamente 2 decimales: `"150.50"`.
- La API acepta `monto` como string (recomendado) o como number; en ambos casos se valida contra `^\d{1,13}(\.\d{1,2})?$` después de convertir a string.

## 6. Transacciones

`db.js` expone:

```js
await transaccion(async (tx) => {
  await depositoRepository.insertar(tx, deposito);
  await modulo.persistir?.(deposito, { tx });
});
```

- **RN-ARQ-14** Una transacción **nunca** envuelve una llamada HTTP a otro servicio (mantendría conexiones y bloqueos abiertos durante segundos).
- Los repositories reciben `tx` (cliente de la transacción) o usan el pool si no hay transacción.

## 7. Criterios de aceptación

- **CA-ARQ-01** Dado el código de `src/modules/*/controller.js`, cuando se busca `require('pg')`, `fetch(` o `axios`, entonces no hay coincidencias. (Lo comprueba el agente revisor o un test de arquitectura con `grep`.)
- **CA-ARQ-02** Dado `src/modules/*/service.js`, cuando se busca SQL (`SELECT `, `INSERT `, `UPDATE `), entonces no hay coincidencias.
- **CA-ARQ-03** Dado que falta `DATABASE_URL`, cuando arranca el servicio, entonces termina con código distinto de 0 y un log que nombra la variable.
- **CA-ARQ-04** Dado un cálculo `0.10 + 0.20` a través de `dinero.js`, entonces el resultado es exactamente `"0.30"`.
