# Checklist de revisión

El agente revisor aplica esta lista a todo cambio, más el `review.md` de la feature revisada.

## Arquitectura
- [ ] Respeta capas: controller sin lógica, service sin SQL ni HTTP directo.
- [ ] La feature no importa código de otra feature.
- [ ] No se modificó `src/core/` (salvo cambio aprobado en `00-core`).
- [ ] La carpeta en `src/modules/` tiene el mismo nombre que en `specs/20-features/`.

## Contrato
- [ ] Rutas, campos y códigos HTTP coinciden con `api.yaml`.
- [ ] Los errores usan el formato y los códigos de `contrato-base.yaml`.

## Dinero y datos
- [ ] No hay `float` ni `Number` para operar montos.
- [ ] Toda consulta SQL usa parámetros.
- [ ] No se lee ni escribe en tablas de otra feature.

## Seguridad
- [ ] La ruta exige JWT y el rol correcto.
- [ ] No se registran tokens, cuentas completas ni datos personales en logs.
- [ ] No hay secretos ni URLs en el código.

## Pruebas
- [ ] Cada escenario del `feature.md` tiene su prueba y pasa.
- [ ] Si es enchufable, hay prueba con la feature activa e inactiva.

## Resultado

El revisor responde con uno de: APROBADO, APROBADO CON OBSERVACIONES, RECHAZADO. Cada observación cita el archivo, la línea y la regla incumplida.
