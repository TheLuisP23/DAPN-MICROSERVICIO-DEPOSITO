# Cómo están organizadas las specs

Este archivo es para personas. La lista de qué está activo vive SOLO en `manifest.yaml`.

| Carpeta | Qué contiene | ¿Cambia al agregar una feature? |
|---|---|---|
| `00-core/` | Reglas globales, dominio, contrato base, puntos de extensión, reglas de pruebas y revisión | No |
| `10-transversales/` | Middlewares que aplican a todas las peticiones | No |
| `20-features/` | Una carpeta por funcionalidad, enchufable | Se agrega o borra una carpeta |
| `30-integraciones/` | Contratos con otros microservicios | Solo si la feature usa un servicio nuevo |
| `manifest.yaml` | Qué specs están activas | Una línea |

## Archivos de cada feature

| Archivo | Lo lee |
|---|---|
| `feature.md` | desarrolladorback, tester, revisor |
| `api.yaml` | desarrolladorback, desarrolladorfront |
| `front.md` | desarrolladorfront |
| `test.md` | tester |
| `review.md` | revisor |

Una feature siempre trae los 5 archivos, aunque alguno diga "No aplica".

## Orden de lectura para cualquier agente

1. `manifest.yaml`
2. `00-core/contexto.md` y `00-core/constitucion.md`
3. Los archivos de `00-core/` que indique su rol
4. Los archivos de su rol en cada feature con `estado: activo`

## Agregar una feature nueva

1. Copiar la carpeta de una feature parecida y renombrarla.
2. Llenar sus 5 archivos.
3. Agregar su línea en `manifest.yaml`.
4. Si es un validador, transformador o post-proceso, cumplir `00-core/puntos-extension.md`.

## Quitar una feature

1. Cambiar su `estado` a `inactivo` en `manifest.yaml` (o borrar su línea y su carpeta).
2. Si tenía tabla propia, agregar su script de eliminación en `db/`.
