# Pruebas: conversion-moneda

| Id | Prueba | Esperado |
|---|---|---|
| C1 | 100.00 USD a cuenta PEN, tipo 3.7500 | billetera recibe "375.00" PEN; conversiones_deposito con monto_final 375.00 |
| C2 | 150.00 PEN a cuenta PEN | Sin fila de conversión; sin clave en extensiones |
| C3 | USD a PEN sin tipo cargado | 422 MONEDA_NO_SOPORTADA |
| C4 | Tipos de ayer (3.70) y hoy (3.75) | Usa 3.75 |
| C5 | Tipo vigente mañana | Se ignora |
| C6 | 10.01 × 3.7555 | 37.59 |
| C7 | Feature inactiva, USD a PEN | 422 MONEDA_NO_SOPORTADA |
| C8 | Billetera falla después de convertir | Depósito FALLIDO; la fila de conversión queda (pertenece al depósito) |
