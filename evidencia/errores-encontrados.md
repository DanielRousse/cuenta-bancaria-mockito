# Errores encontrados en el cajero del compañero

Mi suite (MP-1 a MP-4) atrapó sola **3** de los 5 errores:
- `InteraccionesTest.saldoInsuficienteNoGuardaNiMandaSms`
- `InteraccionesTest.sospechosaNoGuardaNiMandaSms`
- `CajeroConDoblesAManoTest.retiroExitoso`, `InteraccionesTest.retiroGuardaYMandaSms`, `InteraccionesTest.orden` y `InteraccionesTest.smsFallaRetiroSeMantiene`

| Queja | Prueba que la reproduce | Qué estaba mal en el código (la línea y por qué) | ¿Mi suite ya lo atrapaba? |
|---|---|---|---|
| 1 | sinSmsSiNoHaySaldo | En `retirar`, se llamaba a `avisar(...)` antes de `cuenta.retirar(monto)`. Si el retiro fallaba por falta de saldo, el SMS ya había sido enviado al cliente. | sí (`InteraccionesTest.saldoInsuficienteNoGuardaNiMandaSms`) |
| 2 | sospechosaNoMueveDinero | En `retirar`, se ejecutaba `cuenta.retirar(monto)` antes de `validarAntifraude(...)`. Si la operación era sospechosa, el dinero ya había sido sustraído de la cuenta. | sí (`InteraccionesTest.sospechosaNoGuardaNiMandaSms`) |
| 3 | seGuardaUnaVez | En `retirar`, la instrucción `repositorio.guardar(numeroCuenta, cuenta);` estaba duplicada, guardando la cuenta dos veces en la base de datos en cada retiro. | sí (`CajeroConDoblesAManoTest` e `InteraccionesTest`) |
| 4 | antifraudeCaidoRechaza | En `validarAntifraude`, al capturar `ServicioNoDisponibleException` se asignaba `sospechosa = false;` (fail-open) en lugar de lanzar `OperacionRechazadaException` (fail-closed). Es uno de los errores más peligrosos porque si el servicio de seguridad cae, el cajero deja pasar cualquier intento de fraude sin validación. | no |
| 5 | limiteDiarioExacto | En `validarLimiteDiario`, la comparación usaba `>=` (`compareTo(LIMITE_DIARIO) >= 0`) en vez de `>`, rechazando retiros que alcanzaban exactamente el límite diario permitido de $8,000.00. | no |
