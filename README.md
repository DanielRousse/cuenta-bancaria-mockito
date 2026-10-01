# Cajero automático — pruebas con JUnit 5 y Mockito

**Autor:** Jonathan Daniel Reyes Gordillo

## Cómo correr las pruebas

```bash
./probar.sh final
```

## Qué prueba cada clase

| Clase de prueba | Qué prueba | Qué dobles usa | Cuántas pruebas |
|---|---|---|---|
| CajeroConDoblesAManoTest | Retiro exitoso y cuenta inexistente implementando dobles manualmente para comprender el funcionamiento interno de stubs y spies | Fakes hechos a mano (`RepositorioEnMemoria`, `NotificadorFalso`) y una lambda (`ServicioAntifraude`) | 2 |
| StubsTest | Configuración de respuestas programadas en dependencias externas: `when(...).thenReturn(...)`, lanzamiento de excepciones con `thenThrow(...)`, matchers de argumentos (`eq`, `any`), valores por defecto de mocks y límites diarios | Mocks creados con `mock(...)` y configurados con `when(...)` | 7 |
| InteraccionesTest | Verificación exhaustiva de llamadas a servicios colaboradores: frecuencia (`times`, `never`), ausencia de llamadas (`verifyNoInteractions`), captura de parámetros con `ArgumentCaptor`, secuencia estricta con `InOrder` y excepciones en métodos void con `doThrow` | Mocks creados con `mock(...)`, verificados con `verify`, `inOrder` y `ArgumentCaptor` | 7 |
| AnotacionesTest | Uso de sintaxis declarativa limpia para inicializar dependencias y análisis práctico de dos trampas comunes: stubs innecesarios (`UnnecessaryStubbingException`) y dependencias no mockeadas con `@InjectMocks` | `@Mock`, `@Captor` y extensión JUnit `@ExtendWith(MockitoExtension.class)` | 3 |
| QuejasCajeroTest | Reproducción y corrección de las 5 quejas de clientes y seguridad: envío de SMS condicionado a saldo suficiente, aislamiento de saldo ante fraudes, guardado atómico único, principio fail-closed ante caídas de seguridad y límite exacto de $8,000.00 | Mocks configurados con `mock(...)` y stubs base en `@BeforeEach` | 5 |
| RelojTest | Validación de lógica temporal y límites por fecha sin mockear el tiempo, comprobando la fecha local de CDMX, el comportamiento en zonas horarias límite (21:30 en CDMX vs UTC) y el reinicio de cupo al día siguiente | Doble nativo de Java `Clock.fixed(Instant, ZoneId)` y mocks de repositorio | 3 |

## Boleto de salida

1. **¿Qué diferencia hay entre un stub y un mock? Da un ejemplo de cada uno de hoy.**
   - **Stub:** Es un doble enfocado en proveer **entradas indirectas** al sistema bajo prueba. Su propósito es responder con datos prefijados o lanzar excepciones simuladas cuando el código lo invoque (`when(...).thenReturn(...)` o `thenThrow(...)`), permitiendo controlar el escenario de ejecución.  
     *Ejemplo de hoy:* En `StubsTest`, cuando se programó al repositorio para simular el saldo acumulado del día:
     ```java
     when(repositorio.totalRetiradoEn(eq("001"), any(LocalDate.class))).thenReturn(new BigDecimal("7600.00"));
     ```
   - **Mock:** Es un doble enfocado en registrar e inspeccionar las **salidas indirectas** o efectos colaterales del sistema bajo prueba. Su propósito es permitir verificar a posteriori si un método fue ejecutado, cuántas veces, en qué secuencia y con cuáles argumentos (`verify(...)`, `never()`, `InOrder`).  
     *Ejemplo de hoy:* En `InteraccionesTest`, cuando se comprobó que el notificador envió el SMS a la cuenta correcta con el mensaje exacto:
     ```java
     verify(notificador).enviarSms(eq("001"), mensaje.capture());
     ```
   En la práctica, Mockito permite que un mismo objeto funcione simultáneamente como stub (configurando respuestas) y como mock (verificando interacciones).

2. **¿Por qué la prueba inútil de la MP-6 salía en verde con el cajero roto?**
   Porque en `PruebaInutilTest` se mockeó la clase `CuentaBancaria` y se programó artificialmente:
   ```java
   when(cuentaFalsa.getSaldo()).thenReturn(new BigDecimal("1500.00"));
   ```
   Al comentar la instrucción `cuenta.retirar(monto);` en `CajeroAutomatico.java`, el cajero dejó de descontar saldo de la cuenta real. Sin embargo, al consultar el saldo en la prueba (`cajero.consultarSaldo("001")`), el mock simplemente devolvió el `$1500.00` que la propia prueba le había ordenado responder. La prueba nunca evaluó el comportamiento real del cajero ni de la cuenta; solo evaluó la respuesta fija inyectada en el mock, creando un "verde mentiroso" que da falsa seguridad.

3. **¿Qué NO mockearías en un proyecto de Spring Boot como el de la Semana 3, y por qué?**
   - **Entidades del dominio, DTOs y Records:** Clases como `Empleado`, `Direccion`, `Habilidad`, `EmpleadoRequest` o `EmpleadoResponse`, así como tipos de valor de Java (`BigDecimal`, `LocalDate`, `List`, `Map`). Son clases puras, rápidas, sin efectos colaterales y contienen el estado y la lógica de validación esencial. Mockearlas desvirtúa la prueba y oculta fallos reales en getters, setters, cálculos o constructores.
   - **Utilerías estándar y tiempo:** Objetos como el reloj deben sustituirse por dobles deterministas nativos de la plataforma (como `Clock.fixed`), evitando sobrecargar Mockito con clases del SDK que no lo necesitan.
   - **La clase bajo prueba (el SUT):** El servicio que se está evaluando (por ejemplo, `EmpleadoService`) jamás debe ser un mock; debe ser una instancia real que reciba sus dependencias externas (como `EmpleadoRepository` o clientes de APIs terceras) mediante dobles de prueba.
