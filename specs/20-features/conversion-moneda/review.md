# Revisión: conversion-moneda

- [ ] Multiplicación y redondeo con decimal.
- [ ] No rechaza el depósito; si no puede convertir, no modifica nada.
- [ ] Inserta en la misma transacción que el depósito (usa la conexión del contexto).
- [ ] Filtra `vigente_desde <= now()`.
