# ⚠️ IMPORTANTE – Guía de Práctica Sugerida

Este documento tiene **dos partes**:

1. **Guía de práctica sugerida** (primera parte): paso a paso para aprender haciendo. **NO es lo que se entrega.**
2. **El Trabajo Práctico entregable** (al final): escenario, tareas, entregables y defensa oral. **Eso es lo que se evalúa.**

## 📦 Un solo repositorio para todo el semestre

**Todos los prácticos de la materia se hacen sobre el mismo repositorio: el que creaste en el TP1.** No se crea uno nuevo por práctico, y no se arranca de cero en cada uno — cada TP agrega una capa sobre lo que ya está.

Cada TP se cierra con su **tag y su release**, y el número mayor es el número del práctico: **TP1 → `v1.0.0`, TP2 → `v2.0.0`, TP3 → `v3.0.0`**, y así hasta el TP9. Así cada entrega queda con su **estado congelado**: en la defensa se navega el punto exacto en el que cerraste cada una, y podés volver a cualquiera con `git checkout v2.0.0`.

El archivo `decisiones.md` también es **único**: no se rehace por práctico — se le agrega abajo la sección del TP nuevo.

## Sobre las herramientas en este TP

En esta materia hay dos **rieles** — caminos guiados con soporte de la cátedra — más la opción libre:

- La guía paso a paso usa **GitHub Actions**, el mismo riel del TP4: no se da de alta ningún servicio nuevo esta semana. Todo lo que se agrega son paquetes de tu propio proyecto y pasos en el `ci.yml` que ya tenés.
- Si preferís el riel **Azure Pipelines**, al final tenés la **tabla de equivalencias** con los mismos checkpoints. Vale la advertencia 2026 del TP4 sobre los minutos hosted de ADO.
- El **análisis estático** aparece en §2.5 como **teoría**, no como tarea: se practica en el **TP9**, que lo tiene como tema titular y con CodeQL ya integrado en tu repo. Este práctico exige la otra capa — que tus tests midan, y que ese número frene un merge.

📌 Este TP trabaja **sobre tu app del semestre**, extendiendo el pipeline del TP4: mismos repo, mismas protecciones, mismo workflow — hoy le sumamos capas de calidad.

🧪 **Los ejemplos de esta guía están escritos sobre la app de la cátedra** ([`ingsoft3ucc/demo-fullstack`](https://github.com/ingsoft3ucc/demo-fullstack) — .NET 8 + React/Vite + PostgreSQL, la misma de las demos en clase; cómo clonarla y levantarla, base de datos incluida: **TP2 §3.2**). Lo que está ahí de verdad: el validador `TareaValidator` con su suite, `src/lib/tareas.js` con la suya, y los Dockerfiles. Los ejemplos de descuento (`Precios`) y de mock (`ServicioDeTareas`, `INotificador`, `NotificadorEmail` en el backend; `pendientesDe` en el frontend) son **ilustrativos y no están en el sample**: los escribís vos en tu app. 🎬 **Ojo con el video**: se filma sobre una copia del sample a la que ya se le agregaron esas piezas para mostrar el «antes» — por eso la voz dice que «ya venían con el proyecto». Si clonás el sample, no las vas a encontrar. 📌 **Para qué NO sirve el sample: para copiar el pipeline.** Su workflow compila y testea con `dotnet` y `npm` nativos, o sea que tiene una **segunda receta de build** distinta de la del Dockerfile — el anti-patrón de las «dos recetas para lo mismo» — y no construye ninguna imagen. Tu referencia de pipeline es **el que vos construiste en el TP4**, que ya arma la imagen con `docker/build-push-action`: es sobre ése que este práctico agrega la etapa de tests y la publicación. Lo que entregás, siempre, es **tu** app.

---

# Guía Paso a Paso – Testing en el pipeline: unit tests, coverage y el umbral que frena un merge (Práctica sugerida)

## 1- Objetivos de Aprendizaje

- Ubicar los **unit tests** en la pirámide de testing y escribir tests con estructura **AAA**.
- Entender los **mocks** (mocks, stubs, fakes) y qué es diseño testeable.
- Medir **code coverage** en backend y frontend, publicarlo en el pipeline y entender **sus límites**.
- Reconocer el **límite** de la cobertura: por qué un 100 % puede no verificar nada, y qué contestan los **mutantes** que la cobertura no contesta (§2.4 — el práctico no te pide correrlos).
- Convertir esa medición en un **freno**: que un merge se bloquee por calidad, no por compilación.
- Convertir la calidad en un **quality gate que bloquea merges** — la evolución natural del gate del TP4.

## 2- Marco teórico

### 2.1 La pirámide de testing: qué testear y cuánto

No todos los tests cuestan lo mismo. La **pirámide de testing** ordena los tipos por costo y velocidad:

- **Base — Unit tests**: prueban **una unidad de lógica aislada** (una función, una clase) sin tocar red, disco ni base de datos. Corren en milisegundos, fallan señalando exactamente dónde está el problema, y por eso son la base: **muchos**, rápidos y baratos.
- **Medio — Integration tests**: prueban que **varias piezas colaboran** (tu API contra una base de datos real, tu servicio contra otro servicio). Más lentos, más frágiles, más caros de diagnosticar: **menos**.
- **Punta — End-to-end (e2e)**: prueban el **sistema completo desde afuera**, como un usuario (un browser automatizado clickeando tu app desplegada). Los más valiosos como red final de seguridad y los más caros: **pocos**.

La forma importa: una "pirámide invertida" (todo e2e, casi nada unitario) produce suites lentas, frágiles y que no dicen *dónde* está el error — el anti-patrón conocido como *ice cream cone*. La estrategia de la materia sigue la pirámide: hoy consolidamos la **base** (unit tests + coverage); las pruebas **e2e contra un entorno desplegado** llegan en el TP7, cuando haya un entorno donde correrlas.

¿Dónde estás parado? El TP4 te dejó un pipeline que **construye** tus imágenes en cada PR y bloquea el merge si el build falla — pero todavía no verifica ni una línea de **comportamiento**: que compile no dice que ande. Hoy nace la suite, se mide, y la medición se vuelve gate.

### 2.2 Anatomía de un unit test: AAA

La estructura universal de un unit test legible es **AAA — Arrange, Act, Assert**:

```csharp
[Fact]
public void TituloQueSuperaElLargoMaximo_EsRechazado()
{
    // Arrange: preparar el mundo
    var titulo = new string('a', TareaValidator.LargoMaximo + 1);

    // Act: ejecutar LA acción bajo prueba
    var resultado = TareaValidator.Validar(titulo);

    // Assert: verificar el resultado
    Assert.False(resultado.EsValida);
    Assert.Contains($"{TareaValidator.LargoMaximo}", resultado.Error);
}
```

Tres bloques, en ese orden, idealmente separados por una línea en blanco — **cuando hay algo que
preparar**. Si el dato entra directo en la llamada (`TareaValidator.Validar("")`), el Arrange no
existe y está perfecto así: la triple A es una forma de pensar el test, no un molde, y un Arrange
inventado para que sean tres es peor que ninguno. El mismo patrón en el frontend con vitest: `it('rechaza títulos que superan el largo máximo', () => { … })` con `expect(...)` como assert. Convenciones que hacen la diferencia cuando la suite crece:

- **El nombre del test describe el comportamiento esperado**, no el método invocado: `TituloVacio_EsRechazado` dice qué regla protege; `TestValidar2` no dice nada. Cuando falle en el pipeline a las 3 AM, el nombre ES el diagnóstico.
- **Un comportamiento por test**: si tu test verifica cinco cosas y falla, ¿cuál de las cinco se rompió? Mejor cinco tests chicos con nombre propio.
- **Determinismo**: un test que a veces pasa y a veces falla (*flaky*) es peor que no tener test — entrena al equipo a ignorar el rojo. Enemigos típicos: depender de `DateTime.Now`, del orden de ejecución, de la red.
- En xUnit, `[Theory]` + `[InlineData(...)]` parametriza el mismo comportamiento con varios datos sin duplicar el test (en vitest: `it.each`).

### 2.3 «Mockear»: aislar la unidad de sus dependencias

"Unit test" implica **aislamiento**: si tu test de lógica de negocio toca la base de datos real, es lento, frágil… y ya es un test de integración. Para aislar, se reemplazan las dependencias por **mocks** (*test doubles* — como los dobles de riesgo del cine). Los tres nombres que importan:

- **Stub**: un doble que **devuelve respuestas fijas** ("cuando te pidan el usuario 5, devolvé este"). Provee datos; no verifica nada.
- **Mock**: un doble sobre el que el test **verifica la interacción** — qué llamadas recibió y cuántas veces ("¿se llamó a `EnviarEmail` exactamente una vez?"). La diferencia con el stub no es el objeto, es el assert: si el test comprueba cómo se usó la dependencia, es un mock. (En la literatura, el doble que sólo *registra* las llamadas para que el test las mire después se llama *spy*; Moq y `vi.fn()` hacen las dos cosas.)
- **Fake**: una **implementación funcional simplificada** (un repositorio en memoria con un diccionario en lugar de la BD). Útil cuando el stub se queda corto.

En .NET los frameworks típicos son **Moq** o **NSubstitute** (generan dobles de una interfaz en una línea); en vitest, `vi.fn()` y `vi.mock()`. Dos reglas de sanidad:

1. **Mockeá lo que es tuyo de frontera** (tu interfaz de repositorio, tu cliente HTTP), no lo que no controlás directamente (mockear `HttpClient` a pelo es pelearse con la herramienta).
2. **No sobre-mockees**: si tu test tiene 40 líneas de setup de mocks y 2 de asserts, el test está gritando que el diseño acopla demasiado.

Y la lección de diseño más importante del tema: **el código testeable se diseña, no se descubre**. La lógica pura — funciones que reciben valores y devuelven valores, sin tocar infraestructura — se testea sin ningún doble. Es exactamente el patrón del sample de la cátedra: `TareaValidator` (backend) y `src/lib/tareas.js` (frontend) concentran las reglas en funciones puras, y por eso sus tests no necesitan ni base de datos ni mocks ni DOM. Cuanta más lógica vive en el núcleo puro y menos en los bordes con infraestructura, más barata es tu suite. Si algo es difícil de testear, el problema suele ser el diseño, no el test.

### 2.4 Code coverage: qué mide — y qué NO dice

**Coverage** mide qué porcentaje de tu código **se ejecutó** durante los tests. Las dos métricas que vas a ver:

- **Line coverage**: % de líneas ejecutadas.
- **Branch coverage**: % de **ramas de decisión** ejercitadas (un `if` tiene dos ramas; ejecutar solo la rama `true` te da 100% de línea… y 50% de branch). Branch es la métrica más honesta.

Ahora, la advertencia que vale más que la métrica: **coverage mide ejecución, no verificación**. Este test deja `ConDescuento` al 100% de coverage —es la función del descuento de la clase, una cuenta sin ningún `if` (`precio - precio * descuento / 100`), así que ejecutarla una vez la cubre entera— y no verifica absolutamente nada:

```csharp
[Fact]
public void CoberturaSinVerdad()
{
    Precios.ConDescuento(1000, 10);   // se ejecutó todo… y no hay ningún Assert
}
```

Coverage alto con tests malos es **falsa confianza medida con precisión**. Por eso:

- **Coverage bajo SÍ es señal confiable** de problema (hay código que ningún test ejercita — ahí puede esconderse cualquier cosa).
- **Coverage alto NO es señal confiable** de calidad (pudo lograrse con tests sin asserts).
- La métrica sirve como **detector de agujeros y como tendencia** ("el código nuevo entra testeado"), no como trofeo.

De acá sale la **ley de Goodhart** aplicada: *cuando una métrica se convierte en objetivo, deja de ser buena métrica*. Si el equipo cobra bonus por "90% de coverage", va a conseguir 90%… escribiendo tests que ejecutan sin verificar. El umbral correcto es el que el equipo puede defender: en esta materia **elegís tu umbral y lo justificás en `decisiones.md`** — un 70% razonado vale más que un 95% de cargo cult. Y una configuración práctica: el umbral se aplica **sobre lo que tiene sentido testear** — tu lógica —, y hay dos familias que
conviene dejar afuera de la cuenta:

- **El arranque**: el archivo que cablea la aplicación (`Program.cs`, `Startup`, el `main`), la
  configuración de servicios, las rutas. No hay reglas de negocio ahí; y si está mal, la app no
  levanta y te enterás enseguida sin necesidad de un test.
- **Lo que no tiene comportamiento**: las clases de datos —`Tarea`, `Usuario`, el `DbContext`—, que
  sólo tienen propiedades y ninguna regla; y **lo generado**, código que escribió una herramienta y
  no vos —migraciones de base de datos, clientes de API generados desde una especificación, archivos
  `.designer`, stubs de protobuf—. Testear lo primero es testear que una propiedad guarda un valor;
  testear lo segundo es testear al generador. (En la app de la cátedra no hay código generado: las
  que quedan afuera son el arranque y las clases de datos — lo vas a ver en el §3.4.)

> ✍️ **Cómo se ve una justificación que se defiende** (dos frases, y son las que te van a pedir):
> *«Puse 70 porque mi lógica de negocio hoy mide 74 y quiero que el umbral me frene si baja, no que sea
> inalcanzable. Excluí `Program.cs` y las migraciones porque no son código mío ni tienen reglas que
> verificar. Para subirlo a 85 tendría que testear los handlers, que hoy no tienen tests.»*
>
> Fijate lo que hace: **ancla el número en tu medición real**, dice qué queda afuera y por qué, y
> nombra qué haría falta para subirlo. Un «puse 80 porque es lo normal» no es una decisión.

🔴 **Y una trampa que no da ningún error.** Si sacaste UNA regla a su clase y dejaste otras tres
adentro de `Program.cs`, y después excluís `Program.cs` de la cobertura, tu porcentaje pasa a medir
**una sola clase** — y va a dar altísimo. Excluir el arranque vale cuando ahí no queda lógica: si
todavía tenés reglas adentro, primero sacalas, y recién después excluís.

**Excluir eso no es hacer trampa: es medir lo que importa.** Hacer trampa sería excluir tu lógica de
negocio porque no la testeaste.

> 📌 **Lo que no se evalúa es *cómo* lo configuraste** — el atributo, el filtro o el `include` son
> mecánica. Va acá porque es práctico: si tu número te da absurdamente bajo, casi siempre es que
> estás midiendo el arranque de la app y código que no escribiste vos. Ajustá qué medís y seguí.
> 🔴 **Lo que sí va en `decisiones.md` es la lista**: **qué** dejaste afuera de la cuenta, de los dos
> lados, y **por qué** cada cosa — porque con esa lista se puede distinguir «medir lo que importa»
> de «esconder lo que no testeé». Y, por supuesto, el **número** que elegiste y por qué.

**Y cómo se excluye, en concreto.** En el backend, con el atributo `[ExcludeFromCodeCoverage]`
sobre la clase que dejás afuera (`using System.Diagnostics.CodeAnalysis;`) — coverlet lo respeta.
🔴 **¿Tu `Program.cs` no tiene ninguna clase?** Es lo normal desde .NET 6: son *top-level
statements*, y ponerle el atributo arriba no compila (`error CS1001` si lo metés entre los `using`,
`error CS7014` si va después de ellos). La salida es una línea **al final** de `Program.cs`, que
además es la que te va a pedir el TP7 para los tests end-to-end:

```csharp
using System.Diagnostics.CodeAnalysis;   // ⚠️ ARRIBA DE TODO: con top-level statements, los
                                         //    using van antes que cualquier otra cosa (si no, CS1529)

// … el resto de tu Program.cs, hasta app.Run();

[ExcludeFromCodeCoverage]
public partial class Program { }         // esto sí, al FINAL del archivo
```

En el frontend, con el `include:` de la configuración de coverage en `vite.config.js` (§3.3): ahí no excluís, decís qué SÍ entra en la cuenta.

🔴 **Ojo con lo que el atributo NO excluye**: marca **la clase**, no el archivo. Si en tu
`Program.cs` quedaron los `record` del template (`WeatherForecast`) o tus DTOs, el atributo sobre
`public partial class Program` **no los saca** y siguen contando en tu número. Las dos salidas:
ponerle el atributo a cada clase que quede ahí, o —mejor— mover esos `record` a su propio archivo,
que es donde van. Anotá en `decisiones.md` **qué** excluiste y **por qué**, no dónde lo
configuraste.

#### La pregunta que la cobertura no contesta: los mutantes

Quedó dicho que la cobertura mide **ejecución**, no **verificación**. La pregunta obvia es: *¿cómo sé
si mis tests comprueban de verdad?* Hay una técnica que la contesta, y la viste en clase: **le
cambiás el código a propósito y mirás si algún test se pone en rojo.** A esa versión modificada se le
dice un **mutante**. Si tus tests la detectan, lo mataron. Si todos siguen pasando, el mutante
**sobrevivió** — y ahí hay un comportamiento que nadie está comprobando.

Es la mejor demostración de por qué un porcentaje alto de cobertura puede no significar nada: un test
que ejecuta tu código sin comprobar nada llena la cobertura y no mata un solo mutante.

> 📌 **Este práctico NO te pide correr mutantes.** Va acá porque explica el límite de la cobertura, y
> porque es la respuesta a *«¿y entonces para qué me sirve el número?»*. Si te da curiosidad, las
> herramientas existen (Stryker para .NET y para JavaScript) y corren **en tu máquina, no en el
> pipeline**: cada mutante vuelve a correr los tests que pasan por ese código, y son cientos de
> mutantes, así que lo que la cobertura mide en segundos, esto cuesta minutos u horas. Es un diagnóstico, no un freno de cada Pull Request.

### 2.5 Análisis estático: leer el código sin ejecutarlo

Los tests analizan el programa **ejecutándolo** (análisis dinámico). El **análisis estático** examina
el **código fuente sin correrlo**, buscando patrones problemáticos: bugs probables (un `null` que se
desreferencia, una condición que siempre da falso), *code smells* (duplicación, funciones
inabarcables, complejidad alta) y vulnerabilidades. Atrapa cosas que los tests no ven, **porque los
tests sólo verifican lo que a alguien se le ocurrió testear**.

Y la inversa también vale, y es pregunta de defensa: **los tests verifican lo que el análisis
estático no puede saber** — que tu código hace lo que *debe* hacer. Ninguna herramienta sabe que el
descuento tenía que ser 10% y no 15%: eso sólo lo custodian tus tests. 🔴 Y ojo con la trampa: tu
test custodia lo que **vos entendiste** del requisito, no que el requisito esté bien entendido. Si
leíste mal la regla, el test la congela mal y queda en verde para siempre.

🤖 **Y acá hay que ser honestos, porque en 2026 la respuesta fácil ya no sirve**: no es que «ninguna
máquina puede revisar esto». Un agente de IA **sí** evalúa diseño, legibilidad y nombres, y muchas
veces mejor y más rápido que un humano apurado — de hecho esta materia te **pide** que uses IA y que
lo declares. Lo que cambia son dos cosas, y las dos importan.

La primera es la que da nombre a este práctico: **un revisor no es un guardián.** El revisor —humano
o IA— *opina*; el guardián *bloquea*. Y para bloquear hace falta una regla **medible y repetible**:
tu umbral da el mismo veredicto sobre el mismo código todas las veces, y un modelo puede darte dos
lecturas distintas del mismo Pull Request. Por eso lo que frena el merge es un número, y la revisión
—venga de quien venga— suma como lectura extra.

La segunda es más de fondo: **si el requisito se entendió mal, la información que falta no está en
el repositorio.** Está en lo que dijo el cliente, en lo que se habló en una reunión, en lo que el
negocio prioriza esta semana. Ninguna herramienta —ni un humano que no estuvo ahí— puede detectar
ese error leyendo el código, porque el dato no está escrito. Y todavía queda una tercera, que no es
técnica: **alguien tiene que responder por la decisión.** Eso no se delega en una herramienta.

> 📌 **Esto se practica en el TP9, no acá.** El TP9 (DevSecOps) tiene el análisis estático como tema
> titular, con **CodeQL** ya integrado en tu repo y sin dar de alta nada: ahí vas a clasificar
> hallazgos, decidir si son explotables en tu app y vigilar las dependencias. Va allá y no acá
> porque el análisis estático necesita su propia teoría de seguridad para no quedar en «la
> herramienta dijo algo». Lo que **este** práctico exige es la otra capa: que tus tests midan, y que
> ese número frene un merge.

### 2.6 Quality gate: la calidad como requisito, no como reporte

Un **quality gate** es una condición medible que el código tiene que cumplir para avanzar. Si no se
cumple → el check falla → el Pull Request no mergea. Es la generalización del gate del TP4: allá la
condición era *«la imagen se construye»*; hoy se suma *«y los tests pasan, con la cobertura arriba
del número que declaraste»*.

Y ahí está el salto de esta semana, que es más grande de lo que parece: **hasta hoy lo único que
podía frenarte era que algo no compilara.** Desde ahora te puede frenar un número que elegiste vos.
Compila todo, los tests pasan todos, y el merge queda bloqueado igual porque la cobertura quedó
corta. Es la primera vez en la materia que la **calidad**, y no la corrección sintáctica, decide si
un cambio entra.

> 📌 **Que sea TU número no lo hace más blando, lo hace más exigible.** Un umbral impuesto desde
> afuera se apaga el día que molesta; uno que elegiste y justificaste, no — y por eso el práctico te
> pide defenderlo, no acertarle. La regla que hace adoptable a cualquier gate es la misma: **lo que
> tocás, lo dejás limpio.** Exigirle a un repo con deuda histórica un 80% el día uno es condenar el
> gate a que alguien lo apague.

Con esto, tu `main` queda protegido por **tres guardianes**: el Pull Request obligatorio (TP1) + el
build verde (TP4) + los tests con su umbral de cobertura (TP5). Cada uno atrapa una clase de
problema que los otros no ven. 📌 **Sobre el primero**: trabajando solo, tu repo no te pide
aprobaciones — el Pull Request te da el lugar donde mirar, y el que mira sos vos (con la IA que
quieras al lado: se usa, se declara y se supervisa). Es el único de los tres que puede detectar que
el **requisito** se entendió mal, porque para eso hay que saber qué se quiso pedir — un dato que no
vive en el código. El cuarto guardián —el análisis estático— llega en el TP9.

## 3- Desarrollo de la guía (riel GitHub Actions)

> Trabajás sobre **tu app del semestre**, extendiendo el `ci.yml` del TP4. Los ejemplos asumen .NET en `./backend` y Vite/vitest en `./frontend` — adaptá a tu stack; los conceptos (coverage, umbral, gate) son lo transferible.
>
> 📬 **Logística**: ~2-3 PRs esta semana. Podés apilar §3.1–§3.3 en un solo PR (la cobertura y su umbral). El §3.4 conviene que vaya en el suyo: es el que estrena el freno.

### 🔧 Tu stack, de un vistazo

🔧 **Toda esta guía está escrita sobre la app de la cátedra (.NET + vitest). Si tu app es otra cosa,
esta tabla es tu índice: buscá tu fila y seguí.** Nada de lo que se evalúa depende de estas
herramientas — se evalúa que **logres** cada cosa, no con qué la lograste.

| Lo que tenés que lograr | .NET (los ejemplos de la guía) | Java / Kotlin | Python | JS / TS |
|---|---|---|---|---|
| **Dónde viven los tests** | proyecto aparte, referenciando la app | `src/test/java` | carpeta `tests/` | al lado del código, `algo.test.js` |
| **Un test parametrizado** | `[Theory]` + `[InlineData]` | `@ParameterizedTest` | `@pytest.mark.parametrize` | `test.each` / `it.each` |
| **Que la dependencia entre desde afuera** | interfaz + constructor | interfaz + constructor | parámetro de la función o del `__init__` | parámetro, o el módulo inyectado |
| **Fabricar el doble (mock)** | **Moq** · NSubstitute | **Mockito** | `unittest.mock` | `vi.fn()` · `jest.fn()` |
| **Medir la cobertura** | `--collect:"XPlat Code Coverage"` | **JaCoCo** | `pytest --cov` | `vitest run --coverage` |
| 🔴 **Un umbral que ROMPE el build** | `coverlet.msbuild` con `/p:CollectCoverage /p:Threshold /p:ThresholdType` | `jacoco:check` con un `<rule>` y su `<limit>` | `pytest --cov-fail-under=NN` | `coverage.thresholds` de vitest · `coverageThreshold` de jest |
| 🔴 **Qué ENTRA en la cuenta** | `/p:Exclude=[Ens]Namespace.*` (§3.4) | `<excludes>` en el plugin | `omit =` en `.coveragerc` | `include` / `exclude` de `coverage` |
| **Reporte legible del resultado** | ReportGenerator | el `site` que genera JaCoCo | `--cov-report=html` | reporter `lcov` / `html` |
| 🔴 **Que las herramientas de test ENTREN a la etapa de tests del Dockerfile** | el SDK ya las trae (`FROM build`) | que el build no saltee las dependencias con `<scope>test</scope>` | instalar también el `requirements` de tests (`requirements-dev.txt`) | `npm ci` **sin** `--omit=dev` |

📌 **Las filas marcadas son las que más se equivocan**, y no por casualidad. Medir la cobertura
es un flag en todos lados; hacer que un número bajo **frene** el build es una configuración aparte,
en todos lados. Y **decir qué entra en la cuenta** es la que nadie recuerda: sin eso, el umbral se
calcula sobre el arranque y los archivos generados, el número se desploma, y el gate falla siempre
por una razón que no tiene nada que ver con tus tests.

📌 **Si tu stack no está en la tabla** —Go, PHP, Ruby, Rust—, las tres filas existen igual pero la
del umbral suele no ser una bandera: en Go, por ejemplo, se resuelve leyendo el total de
`go tool cover` y comparándolo en el script del pipeline. Averiguar cómo se hace en el tuyo es parte
del trabajo, y va contado en `decisiones.md`.

📌 **¿Tu stack no está?** El criterio es el mismo, y averiguarlo **es parte del trabajo**: todos los
lenguajes tienen **cada fila** de esta tabla. Contá en `decisiones.md` qué usaste. Lo que no se
acepta es «mi lenguaje no tiene mocks» o «no se puede poner un umbral».

---

### 3.0 La suite que pide la Tarea 1

> 🎯 **Qué tenés que lograr** — una suite que, si alguien invierte una regla de tu app, se ponga en rojo: con un test parametrizado, uno de caso de error, y uno que reemplace una dependencia por un doble.
>
> *Lo que sigue es **cómo se hace en el ejemplo de la cátedra**. Si tu stack es otro, traducilo con la tabla «Tu stack, de un vistazo» — se corrige el logro de arriba, no la herramienta.*

Antes de medir nada hay que tener qué medir. Esta sección muestra los tres tests que el enunciado
nombra y no se ven en ningún otro lado: el parametrizado, el caso de error y el que usa un mock.

> 🔧 **Esta sección está escrita en .NET, y tu backend puede no serlo.** Lo que cambia es **dónde
> viven los tests y cómo se crea el proyecto**; lo que se pide —un parametrizado, un caso de error y
> uno con mock— es igual en todos lados. La traducción, para que no la busques:
>
> | Stack | Dónde van los tests | Cómo se arranca |
> |---|---|---|
> | **.NET** | Proyecto **aparte**, referenciando al de la app | `dotnet new xunit` (lo de abajo) |
> | **JavaScript / TypeScript** | Al lado del código, `algo.test.js` | Ya lo tenés si usás vitest o jest (§3.3) |
> | **Java / Kotlin** | `src/test/java`, hermano de `src/main/java` | Maven y Gradle ya lo traen |
> | **Python** | Carpeta `tests/` en la raíz | `pytest`, sin proyecto aparte |
>
> Si tu stack no está acá, el criterio es el mismo: buscá cuál es el marco de tests estándar y dónde
> espera encontrarlos. **Eso es parte del trabajo**, y contalo en `decisiones.md`.

> 🪜 **Si tu backend todavía no tiene un proyecto de tests** — lo normal, hasta acá ningún práctico
> pidió escribir uno. En .NET los tests **no van al lado del código**: van en un proyecto aparte,
> que se crea una sola vez. **Parate en `backend/`** —la carpeta que el Dockerfile usa como
> contexto— y corré:
>
> ```bash
> dotnet new xunit -o MiApi.Tests -f net8.0          # ⚠️ el net8.0 tiene que ser el MISMO
>                                                    #    <TargetFramework> que usa tu app
>                                                    #    (.NET 8 sale de soporte el 10/11/2026;
>                                                    #    si arrancás de cero, net10.0 y sdk:10.0.
>                                                    #    Lo que importa es que coincidan)
> # ↓ la ruta del .csproj de TU APP. Buscala con:  find . -maxdepth 2 -name "*.csproj"
> dotnet add MiApi.Tests/MiApi.Tests.csproj reference MiApi.csproj
> #                                                  ↑ si tu app está en su propia
> #                                                    carpeta, va MiApi/MiApi.csproj
> ```
>
> 🔴 **Todavía no corras `dotnet test`**: si tu `.csproj` está en la raíz de `backend/`, falla — y el
> arreglo es el recuadro que sigue.
>
> 🔴 **Si tu `.csproj` está en la raíz de `backend/`** (o sea, tu app NO vive en su propia
> subcarpeta — es lo que sale de `dotnet new webapi`), **hacé esto AHORA, antes de seguir**: agregale
> a ese `.csproj` este bloque, que excluye al proyecto de tests de las dos formas en que tu app
> podría tragárselo:
>
> ```xml
> <ItemGroup>
>   <Compile Remove="MiApi.Tests/**" />
>   <Content Remove="MiApi.Tests/**" />
> </ItemGroup>
> ```
>
> **Las dos líneas, no una.** Sin la primera tu app intenta compilar los tests y se rompe el build
> que venías teniendo verde (`error CS0246: no se encontró 'Fact'` en `dotnet build` y en
> `dotnet publish`). Sin la segunda **no da error**, y por eso es la que se olvida: el glob de
> `Content` del SDK web se lleva los JSON del proyecto de tests a tu `bin/`, y ahí quedan. No
> revienta nada — la salida queda sucia, nada más. Se pone y listo.
>
> Ahora sí, la comprobación: `dotnet test MiApi.Tests/MiApi.Tests.csproj` tiene que correr **1 test
> de ejemplo**.
>
> Cambiá `MiApi` por el nombre de tu proyecto. Los bloques escritos para vos dicen `MiApi`; los que
> se copian del video —el YAML, el `ENTRYPOINT` con el umbral— dicen `DemoApi`, el de la app de la
> cátedra. En los dos casos va el **tuyo**. El template ya trae `coverlet.collector`, así que la cobertura del §3.1 va a
> funcionar sin instalar nada más. **Si tu backend tiene un `.sln`**, agregá también el proyecto:
> `dotnet sln add MiApi.Tests/MiApi.Tests.csproj`. (Si no tenés uno y lo creás ahora, ojo: el SDK
> actual genera un `.slnx`, no un `.sln` — usá el nombre que te haya quedado en los comandos de
> abajo.)
>
> 🔴 **Antes que nada: ¿tu lógica está adentro de un lambda de `Program.cs`?** Es lo que sale de
> `dotnet new webapi`, y así **no hay nada que un test pueda llamar** — ni te va a dar error, te vas
> a quedar mirando la pantalla. Y encima el §2.4 te dice de excluir `Program.cs` de la cobertura, o
> sea que estarías excluyendo toda tu lógica: la trampa exacta que el práctico no acepta. Sacala a
> una clase pública primero, que es un movimiento de dos minutos:
>
> ```csharp
> // ANTES, adentro de Program.cs:
> app.MapPost("/tareas", (TareaDto dto) => {
>     if (string.IsNullOrWhiteSpace(dto.Titulo)) return Results.BadRequest("...");
>     ...
> });
>
> // DESPUÉS: la regla en su propia clase, y el endpoint sólo la llama
> public static class TareaValidator          // ← esto es lo que vas a testear
> {
>     public static Resultado Validar(string? titulo) { ... }
> }
> ```
>
> Recién ahí el endpoint queda finito (pide, delega, responde) y la regla es testeable. Es la misma
> lección de la filmina 5: **si algo es difícil de testear, sospechá del diseño antes que del test.**
>
> 📌 **De dónde salen `Resultado`, `EsValida` y `LargoMaximo`** que vas a ver en los tests: son de
> la clase que se está probando, la del repo de la cátedra. Si escribís la tuya, adaptá los nombres
> a los tuyos — lo que importa es la forma, no el nombre:
>
> ```csharp
> public static class TareaValidator
> {
>     public const int LargoMaximo = 100;
>     public record Resultado(bool EsValida, string? Error, string? TituloNormalizado);
>     public static Resultado Validar(string? titulo) { ... }   // ← lo que probás
> }
> ```
>
> **Los tests de acá abajo son MÉTODOS y van adentro de una clase.** El archivo se llama como lo que
> probás con `Tests` al final (`MiApi.Tests/TareaValidatorTests.cs`), el `UnitTest1.cs` del template
> lo podés borrar, y se ve así:
>
> ```csharp
> using Xunit;
> using MiApi.Logica;             // ⚠️ el namespace EXACTO de la clase que probás — ver la nota
>
> namespace MiApi.Tests;
>
> public class TareaValidatorTests
> {
>     // ── acá adentro van los tests del validador de abajo ──
> }
> ```
>
> 📌 **Un archivo por clase que probás.** El test con mock del servicio (más abajo) **no** va en
> este archivo: va en el suyo, `ServicioDeTareasTests.cs`, con sus propios `using` — así lo vas a
> ver en el video.
>
> 📌 **Sobre ese `using MiApi.Logica;`**: va el `namespace` **exacto** que declara la clase que
> probás, no el nombre del proyecto — en el sample, `TareaValidator` está en `DemoApi.Logica`, y un
> `using DemoApi;` da `CS0246` igual. Y va sólo si tu clase declara un `namespace`: las apps que
> salen de `dotnet new webapi` **no declaran ninguno**, así que si copiás la línea tal cual te va a
> dar `CS0246`. Dos salidas, y la primera es mejor: ponele un `namespace` a la clase que vas a
> testear (es una línea, y vas a querer tenerlo igual), o borrá el `using`.
>
> ⚠️ **Tres formas de que esto salga mal, y cómo se ven** — leelas ahora, te ahorran la tarde:
>
> | Lo que ves | Qué pasó |
> |---|---|
> | `error NETSDK1045` en el pipeline (y en tu máquina anda) | El `-f` no coincide con el .NET de tu app: el template usó el más nuevo que tenés instalado y el contenedor compila con otro |
> | `error CS0246` | Mirá **qué nombre** dice que no encuentra. Si es `Mock<>`, falta `using Moq;` (y el paquete: está más abajo). Si es el **namespace de tu app**, es al revés: tu código no declara ninguno — ponele uno o sacá el `using`. Si es una clase **tuya**, falta el `using`. Si son tipos de tests (`Xunit`, `Fact`), el proyecto quedó **adentro** del de la app: agregale `<Compile Remove="MiApi.Tests/**" />` al `.csproj` de la app |
> | `error MSB4068: The element <Solution> is unrecognized` **sólo en el pipeline** | Creaste un `.slnx` (es lo que genera el SDK 10) y el contenedor compila con el SDK 8, que no lo entiende. En tu máquina anda y en la corrida no. Salidas: apuntá los comandos al `.csproj` de tests en vez de a la solución, o subí la imagen del Dockerfile a un SDK que lo soporte |
> | El pipeline en **verde** y ningún test en el log | `dotnet test` sobre una solución sin proyecto de tests **devuelve 0 y no dice nada**: es el escenario con el que abre este práctico. Sumá el `COPY` del proyecto de tests al `backend/Dockerfile` (`COPY MiApi.Tests/MiApi.Tests.csproj MiApi.Tests/`, antes del `RUN dotnet restore`) — si no, el `restore` del contenedor no lo encuentra. 🔴 **Va tengas `.sln` o no**: sin eso el `docker run` igual corre los tests, pero baja los paquetes **en cada corrida** (~10 s por Pull Request que no se cachean nunca) y tu imagen de tests deja de ser autosuficiente — necesita red al CORRER, no sólo al construirse |
>
> 📌 **Si tu backend NO tiene `.sln`**: donde la guía escribe `Backend.sln` va la ruta de tu
> `.csproj` de tests. Vale para el `dotnet test` del §3.1, el `ENTRYPOINT` del §3.2 y el §3.4.
>
> El equivalente para el frontend está en el §3.3.

#### Un test parametrizado y un caso de error

Mirá la suite del sample: `TituloVacio_EsRechazado` y `TituloSoloEspacios_EsRechazado` son **dos
tests para el mismo comportamiento** —«un título sin contenido se rechaza»— con dos datos distintos.
Eso no se escribe dos veces: se escribe una y se le pasan los datos desde arriba. En xUnit es
`[Theory]` + `[InlineData]` (en vitest, `it.each`), y **reemplaza a los dos**:

```csharp
[Theory]
[InlineData("")]            // vacío
[InlineData("   ")]         // sólo espacios
[InlineData("\t")]          // un tabulador
public void TituloSinContenido_EsRechazado(string? titulo)
{
    var resultado = TareaValidator.Validar(titulo);

    Assert.False(resultado.EsValida);
}
```

> 🔎 **Una tarea para cuando midas la cobertura (§3.1): volvé acá, abrí tu reporte y elegí UNA rama
> de código sin cubrir.** 🔴 *Rama de código*, no rama de git: es uno de los dos caminos que abre una
> decisión —un `if`, un `?.`, un `??`— y que ningún test recorre. Es lo que mide el *branch coverage*
> del §2.4, y no tiene nada que ver con las ramas del repositorio. Contá tres cosas en
> `decisiones.md`:
>
> 1. **Qué línea es** — el reporte te la marca. En ReportGenerator, entrá a la clase desde la tabla
>    y fijate los colores, que son la leyenda de todo el reporte: **verde**, ejecutada; **naranja**,
>    ejecutada por un solo camino (con el ícono de rama al costado: **ésa es la que buscás**);
>    **rojo**, nunca ejecutada.
> 2. **Qué entrada la recorrería.** Un valor concreto: un nulo, una cadena vacía, un número en el
>    borde. Si creés que **ninguna** entrada puede recorrerla, explicá por qué.
> 3. **Qué decidiste**: agregar ese test, o no agregarlo — con el motivo.
>
> Las tres respuestas valen, incluida «no lo agregué». Lo que se evalúa es que **abriste el reporte y
> miraste el código**, no el número: la cobertura te señala dónde mirar, no qué hacer. A veces lo que
> corresponde no es un test más, es simplificar el código.
>
> 📌 **Una pista para cuando no encuentres el `if`.** Hay ramas que ninguna línea declara: las abren
> operadores como `?.` y `??`, que comprueban el nulo por su cuenta. Si la línea marcada no tiene
> ningún `if` a la vista, mirá si hay uno de ésos — y probá con un nulo.

Y así se ve un **caso de error** —el otro test que el enunciado te pide—: no comprueba que algo
salga bien, sino que ante una entrada inválida el sistema reaccione como debe. Acá, que el mensaje
de rechazo **diga cuál es el límite**: un rechazo que no explica por qué obliga al usuario a
adivinar, así que el mensaje también es comportamiento y se testea. (El sample ya trae uno parecido,
`TituloQueSuperaElLargoMaximo_EsRechazado`; en tu app escribís el tuyo.)

```csharp
[Fact]
public void TituloDemasiadoLargo_ExplicaElLimiteEnElMensaje()
{
    var titulo = new string('a', TareaValidator.LargoMaximo + 1);   // 101: uno más que el tope

    var resultado = TareaValidator.Validar(titulo);

    Assert.False(resultado.EsValida);
    Assert.Contains(TareaValidator.LargoMaximo.ToString(), resultado.Error);
}
```

> 📌 **Cada `[InlineData]` corre como un test propio.** Cambiaste dos tests por uno… y el reporte va
> a mostrar **tres**, uno por dato. Para verlo, pedile el detalle:
> `dotnet test Backend.sln --logger "console;verbosity=detailed"` — cada `[InlineData]` sale en su
> propia línea (con `dotnet test` a secas sólo ves el total). En el sample, después de este cambio:
> 4 métodos, 6 tests. 🔴 **Pero para el mínimo de ocho del enunciado cuentan los
> MÉTODOS, no los datos**: un `[Theory]` con ocho `[InlineData]` es **un** test de los ocho, no los
> ocho. Los ocho tienen que repartirse sobre al menos cuatro reglas distintas (Tarea 1). Y es una
> razón más para no confundir «cantidad de tests» con «calidad de la suite».

#### El test con mock, y el refactor que lo hace posible

La Tarea 1 pide **un test con mock, sí o sí**. Va acá porque casi siempre implica **tocar el
código**, no sólo escribir un test.

> 🔧 **Esto también está escrito en .NET, y la mecánica cambia según tu stack.** Lo que NO cambia es
> la idea, y es la que se evalúa: **si tu clase fabrica su dependencia adentro, no hay forma de
> reemplazarla desde el test; tiene que entrar desde afuera.** Cómo se escribe eso, y con qué se
> fabrica el impostor:
>
> | Stack | Cómo entra la dependencia | Con qué se fabrica el doble |
> |---|---|---|
> | **.NET** | Interfaz + parámetro del constructor (lo de abajo) | **Moq** o **NSubstitute** |
> | **JavaScript / TypeScript** | Parámetro de la función, o el módulo inyectado | `vi.fn()` y `vi.mock()` en vitest · `jest.fn()` |
> | **Java / Kotlin** | Interfaz + constructor, igual que .NET | **Mockito** |
> | **Python** | Parámetro de la función o del `__init__` | `unittest.mock` (`Mock`, `patch`) |
>
> 🔴 **Si tu stack no está en la tabla, averiguarlo es parte del trabajo** — y contá en
> `decisiones.md` qué usaste. Lo que no se acepta es «mi lenguaje no tiene mocks»: todos tienen
> alguna forma de reemplazar una dependencia, y si de verdad no la hubiera, el camino es el mismo que
> acá — sacar el `new` de adentro y recibirla por parámetro.

> 🔴 **Paso cero, y es el que falta en casi todas las apps: la interfaz.** El ejemplo de abajo ya
> tiene una — en .NET lleva una `I` adelante, y dice **qué** se le puede pedir, no **cómo** se hace:
>
> ```csharp
> public interface INotificador
> {
>     void Enviar(string mensaje);
> }
> // y la clase que lo hace de verdad:  public class NotificadorEmail : INotificador { … }
> ```
>
> Si en tu código hay una clase concreta —`private readonly EmailSender
> _sender = new EmailSender();`—, **Moq no puede fabricar un doble de eso** y el error que te tira
> no dice nunca la palabra «interfaz». Primero extraés la interfaz: en tu IDE, botón derecho sobre
> la clase → *Extract Interface* (en Rider y en Visual Studio; en VS Code, con el refactor de
> C# Dev Kit), o a mano — una `public interface IEmailSender` con los métodos que usás, y
> `public class EmailSender : IEmailSender`. Recién entonces seguí con lo de abajo.

> 🔴 **Y después del refactor, un paso que los tests NO te van a reclamar.** Al pasar de fabricar la
> dependencia adentro a recibirla, tu app deja de saber qué instancia usar: hay que **registrarla en
> `Program.cs`** — la dependencia **y** el servicio que la recibe, si un endpoint lo usa. Van antes
> de `var app = builder.Build();`:
>
> ```csharp
> builder.Services.AddScoped<INotificador, NotificadorEmail>();   // o AddSingleton, según el caso
> builder.Services.AddScoped<ServicioDeTareas>();                 // el servicio también
>
> var app = builder.Build();
> ```
>
> Si te lo olvidás, **los tests pasan igual y el pipeline queda verde** —el test le pasa el doble a
> mano—, pero la app real queda rota. 🔴 **Y dónde se nota depende de dónde corras.** Con `dotnet run`
> (entorno *Development*) .NET valida las dependencias al arrancar: si registraste el servicio pero
> no su `INotificador`, la app **no levanta** y te dice cuál falta. En el contenedor (*Production*)
> esa validación está apagada: la app levanta normal, no loguea nada, y la primera llamada de verdad
> **falla con un 500**. Y si no registraste **nada**, ni siquiera en *Development* revienta al
> arrancar: .NET no sabe que ese parámetro es un servicio y lo trata como el cuerpo del pedido. En un
> POST eso da 400 o 500 según lo que mandes; en un GET, la primera llamada a **cualquier** ruta de la
> app —hasta `/health`— da 500 con *«Body was inferred but the method does not allow inferred body
> parameters»*. O sea que «probá que arranque» no sirve de comprobación. Lo que sirve, en los dos
> lados: **un `curl` al endpoint que usa esa dependencia**, esperando 200.

**El problema, que es de diseño y no de testing.** Mirá una pieza típica de una app que avisa cuando
pasa algo:

```csharp
public class ServicioDeTareas
{
    private readonly INotificador _notificador = new NotificadorEmail();   // ← acá está el problema

    public Tarea Crear(string titulo)
    {
        var validacion = TareaValidator.Validar(titulo);            // ← la regla de negocio
        if (!validacion.EsValida) throw new ArgumentException(validacion.Error);

        var tarea = new Tarea { Titulo = validacion.TituloNormalizado!, CreadaEl = DateTime.UtcNow };
        _notificador.Enviar($"Nueva tarea: {tarea.Titulo}");
        return tarea;
    }
}
```

Así como está, **no hay forma de testear `Crear` sin mandar un mail de verdad**: el `new` está
adentro y desde afuera no se puede reemplazar. No es difícil de testear — es imposible.

**El arreglo: que la dependencia entre desde afuera.**

```csharp
public class ServicioDeTareas
{
    private readonly INotificador _notificador;

    public ServicioDeTareas(INotificador notificador)     // ← ahora la RECIBE
    {
        _notificador = notificador;
    }

    public Tarea Crear(string titulo)                     // ← IGUAL que antes
    {
        var validacion = TareaValidator.Validar(titulo);  // ← la regla SIGUE acá
        if (!validacion.EsValida) throw new ArgumentException(validacion.Error);

        var tarea = new Tarea { Titulo = validacion.TituloNormalizado!, CreadaEl = DateTime.UtcNow };
        _notificador.Enviar($"Nueva tarea: {tarea.Titulo}");
        return tarea;
    }
}
```

🔴 **Mirá qué cambió y qué NO.** Lo único que cambió es de dónde viene el notificador. El método
hace exactamente lo mismo, y **la validación sigue adentro**. Abrir una clase para poder testearla
no es licencia para sacarle lo que hace: si el «después» pierde una regla, tu test con mock queda
verde sobre un servicio que dejó de validar — y eso es peor que no tener el test.

Quien lo construye decide qué le pasa: la aplicación real le pasa el notificador que manda mails, y
el test le pasa un impostor. Esa abertura se llama **inyección de dependencias**, y es la diferencia
entre un código que se testea en dos líneas y uno que no se testea nunca.

**Compilá antes de seguir** (`dotnet build`, sobre tu `.sln` si tenés, o sobre el `.csproj` de la app): si en algún lugar de tu app había un
`new ServicioDeTareas()` sin argumentos, el error aparece acá y no en el pipeline.

**El test.** La librería que fabrica impostores en .NET es **Moq** (en vitest, `vi.fn()`; en cada
stack hay una equivalente). Va **al proyecto de tests**, no al de la aplicación — es una herramienta
de tus tests, no algo que tu app se lleve a producción:

```bash
cd backend
dotnet add MiApi.Tests/MiApi.Tests.csproj package Moq
```

Va en **su propio archivo**, `MiApi.Tests/ServicioDeTareasTests.cs` — uno por clase que probás —,
con el `using` del namespace donde vive el servicio:

```csharp
// MiApi.Tests/ServicioDeTareasTests.cs
using MiApi.Servicios;          // ⚠️ el namespace de TU servicio (en el video, DemoApi.Servicios)
using Moq;
using Xunit;

namespace MiApi.Tests;

public class ServicioDeTareasTests
{
    [Fact]
    public void CrearUnaTarea_NotificaUnaSolaVez()
    {
        // Arrange: el impostor, en lugar del notificador real
        var notificador = new Mock<INotificador>();
        var servicio = new ServicioDeTareas(notificador.Object);

        // Act
        servicio.Crear("Comprar café");

        // Assert: no mira un valor devuelto — mira la INTERACCIÓN
        notificador.Verify(n => n.Enviar(It.IsAny<string>()), Times.Once);
    }
}
```

> ⚠️ **En el video vas a ver `servicio.Crear(new Tarea { Titulo = "Comprar café" })`**: es una
> versión anterior del ejemplo, que no compila contra el `Crear(string titulo)` de arriba. Con el
> servicio de esta guía va `servicio.Crear("Comprar café")`, como está acá.

> 📌 **Lo que distingue a un mock**: el assert no comprueba un valor de retorno, comprueba **cómo se
> usó la dependencia** — por eso éste es un **mock** y no un stub (§2.3). En el frontend, más abajo,
> vas a ver las dos cosas con `vi.fn()`: en el primer test actúa como stub (sólo contesta) y en el
> segundo como mock (`toHaveBeenCalledWith`). Si mañana alguien duplica el envío sin querer, el usuario recibe dos mails y
> este test se pone rojo antes de que eso llegue a nadie. Y el test no toca la red, ni la base, ni un
> servidor de correo: corre en milisegundos y no falla los días que el correo anda mal.

> 🔴 **¿Y si tu app no tiene ninguna dependencia externa?** Entonces el ejercicio es éste: buscá una
> pieza que hable con afuera —la base, un cliente HTTP, el reloj del sistema, el sistema de
> archivos—, abrila como acabás de ver, y testeala con un mock. No se acepta *«mi lógica es pura y no
> lo necesité»*: mockear es una técnica que hay que tener en las manos. Contá en `decisiones.md` qué
> refactorizaste y por qué.

#### El mismo mock, ahora en el frontend

> 📌 **No es el camino alternativo.** El mock va en los **dos** lados, igual que el parametrizado y
> el caso de error: esta sección no reemplaza a la de arriba, la completa.

Primero, las otras dos técnicas del lado del front, que también se piden (Tarea 1). En
`src/lib/tareas.test.js`, el **parametrizado** es `it.each` —el equivalente del `[Theory]` con
`[InlineData]`— y el **caso de error** es un `it` que comprueba el rechazo y su mensaje (el sample
trae un solo `it` que prueba dos datos; en el video queda partido en uno por dato, así):

```js
it.each([
  ['vacío', ''],
  ['sólo espacios', '   '],
  ['un tabulador', '\t'],
  ['nulo', null],
])('rechaza un título %s', (_caso, entrada) => {   // %s: el primer dato de cada fila va al nombre
  const resultado = validarTitulo(entrada)
  expect(resultado.valido).toBe(false)
  expect(resultado.error).toBe('El título es obligatorio.')
})

it('rechaza títulos que superan el largo máximo', () => {
  const resultado = validarTitulo('a'.repeat(LARGO_MAXIMO + 1))
  expect(resultado.valido).toBe(false)
  expect(resultado.error).toContain(String(LARGO_MAXIMO))
})
```

Y ahora el mock: es la **misma** lección con otro lenguaje, y no hace falta traducirla de memoria.
Partimos de una función **típica** que le pide las tareas a la API (no está en el sample: es el
«antes» que armamos para el ejemplo):

```js
// ❌ así no se puede testear: la función se fabrica su dependencia adentro
export async function pendientesDe(usuario) {
  const r = await fetch(`/api/tareas?u=${usuario}`)
  const tareas = await r.json()
  return tareas.filter((t) => !t.hecha)
}
```

Es el mismo problema del `new` del backend con otra cara: no hay por dónde meterle un impostor, así
que para probarla necesitás la API levantada. El arreglo también es el mismo — **lo que la función
necesita entra desde afuera**:

```js
// ✅ el cliente entra por parámetro
export async function pendientesDe(usuario, traer) {
  const tareas = await traer(`/api/tareas?u=${usuario}`)
  return tareas.filter((t) => !t.hecha)
}
```

Y el test. Vitest trae el doble incorporado, no hay que instalar nada: `vi.fn()` fabrica la función
impostora y `mockResolvedValue` le dice qué contestar.

```js
import { describe, expect, it, vi } from 'vitest'
import { pendientesDe } from './tareas.js'

describe('pendientesDe', () => {
  it('devuelve sólo las tareas que no están hechas', async () => {
    const traer = vi.fn().mockResolvedValue([
      { id: 1, titulo: 'hecha', hecha: true },
      { id: 2, titulo: 'una', hecha: false },
      { id: 3, titulo: 'otra', hecha: false },
    ])
    const pendientes = await pendientesDe('ana', traer)
    expect(pendientes.map((t) => t.titulo)).toEqual(['una', 'otra'])
  })

  it('le pide a la API la ruta del usuario', async () => {
    const traer = vi.fn().mockResolvedValue([])
    await pendientesDe('ana', traer)
    expect(traer).toHaveBeenCalledWith('/api/tareas?u=ana')   // ← el equivalente del Verify
  })

  it('devuelve una lista vacía si la API no trae nada', async () => {
    const traer = vi.fn().mockResolvedValue([])
    expect(await pendientesDe('ana', traer)).toEqual([])
  })
})
```

El segundo test es el que se parece al `Verify` de Moq: **no mira lo que la función devolvió, mira
qué le pidió a la API**. Si mañana alguien cambia la ruta sin querer, se pone rojo.

Corré `npm test -- --run`: en el ejemplo del video quedan **10 tests en verde** (en tu app, el número es el tuyo) (los 7 de `validarTitulo` y
`ordenarTareas` —el `it.each` cuenta uno por dato— más los 3 de `pendientesDe`).

**¿Y en la app de verdad, quién le pasa el cliente?** Es el mismo paso que el registro en
`Program.cs` del backend, y tampoco te lo reclaman los tests: si cambiaste la firma de
`pendientesDe` y no actualizaste a quien la llama, la suite queda verde y la pantalla falla con
`traer is not a function`. Son dos archivos — el cliente real, en un solo lugar:

```js
// src/api/cliente.js
export const traerJson = async (url) => (await fetch(url)).json()
```

y el código que la usa de verdad, que le pasa **ése**:

```js
// src/api/tareas-de-la-pantalla.js
import { pendientesDe } from '../lib/tareas.js'
import { traerJson } from './cliente.js'

export async function refrescarPendientes(usuario) {
  return pendientesDe(usuario, traerJson)   // en producción entra el real; en el test, el impostor
}
```

> 🔴 **El límite, que es tan importante como el ejemplo**: esto prueba **tu** código, no la conexión.
> Un frontend con la cobertura al cien se rompe igual si el backend cambió el contrato, porque tu
> impostor sigue contestando lo de siempre. Que la conexión ande de verdad se verifica con pruebas de
> punta a punta, con todo levantado, y eso es el **TP7**. Acá probamos la lógica; allá, que las
> piezas se entiendan.

**✅ Checkpoint:** al menos un test que reemplaza una dependencia por un mock y verifica la
interacción, y el código de producción recibiendo esa dependencia desde afuera.

### 3.1 Coverage del backend con coverlet

> 🎯 **Qué tenés que lograr** — que tu suite de backend emita un reporte de cobertura en un formato que otra herramienta pueda leer.
>
> *Lo que sigue es **cómo se hace en el ejemplo de la cátedra**. Si tu stack es otro, traducilo con la tabla «Tu stack, de un vistazo» — se corrige el logro de arriba, no la herramienta.*

> 🔧 **La herramienta de cobertura depende de tu stack** — los ejemplos de esta guía son los del
> proyecto de la cátedra. Buscá el tuyo acá y usá ése; lo que se pide es el **número**, no la
> herramienta.
>
> | Tu stack | Herramienta | Cómo se activa |
> |---|---|---|
> | C# / .NET | **coverlet** (viene con el template de xUnit) | `dotnet test --collect:"XPlat Code Coverage"` |
> | JavaScript / TypeScript | **vitest** con `@vitest/coverage-v8`, o **jest** | `vitest run --coverage` · `jest --coverage` |
> | Java / Kotlin | **JaCoCo** | Plugin de Maven o Gradle |
> | Python | **coverage.py**, normalmente vía **pytest-cov** | `pytest --cov` |
> | PHP | **PHPUnit** con Xdebug o PCOV | `phpunit --coverage-html coverage` |
> | Go | viene en el propio `go test` | `go test -cover` |
> | Ruby | **SimpleCov** | Se activa desde el archivo de tests |
>
> Todas miden cobertura de **línea** y dan un reporte navegable. 🔴 **La de rama, en varias hay que
> pedirla**: `pytest --cov --cov-branch` en Python, `enable_coverage :branch` en SimpleCov, Xdebug (no
> PCOV) en PHPUnit; y en Go no existe (`-cover` mide sentencias: contalo en `decisiones.md`). Como la
> Tarea 2 pide reportar siempre la de rama, fijate que la tuya la esté midiendo. Y casi todas
> exportan a los mismos formatos —`cobertura`, `lcov`, `opencover`—, que es lo que después leen los
> generadores de reportes.

El template de xUnit ya trae el paquete **coverlet.collector** (miralo en el `.csproj` de tu proyecto de tests). Activarlo es un flag en `dotnet test`:

```bash
cd backend
find . -type d -name TestResults -prune -exec rm -rf {} +   # cada corrida deja una carpeta NUEVA con
rm -rf coveragereport                                        # otro <guid>: si no borrás las viejas, el
                                                             # reporte de abajo las junta todas
# (PowerShell: Get-ChildItem -Recurse -Directory -Filter TestResults | Remove-Item -Recurse -Force;
#  Remove-Item -Recurse -Force coveragereport -ErrorAction SilentlyContinue)
dotnet test Backend.sln --collect:"XPlat Code Coverage"
# → <TuProyecto>.Tests/TestResults/<guid>/coverage.cobertura.xml
#   (el XML aparece bajo el PROYECTO DE TESTS, en una subcarpeta con un identificador
#   que cambia en cada corrida; formato Cobertura, el default)
```

Ese XML es ilegible a ojo — conviene un reporte navegable. El **ReportGenerator** (herramienta .NET estándar) lo convierte:

```bash
dotnet tool install -g dotnet-reportgenerator-globaltool
# (si la terminal no encuentra "reportgenerator" tras instalar: cerrala y abrila
#  de nuevo — ~/.dotnet/tools entra al PATH en la próxima shell. Si ni así aparece,
#  esa carpeta no está en tu PATH: la salida de `dotnet tool install` te dice cómo agregarla)
reportgenerator -reports:"**/TestResults/**/coverage.cobertura.xml" \
  -targetdir:coveragereport -reporttypes:Html
# 🔴 Acá el patrón arranca con `**/` porque corrés desde `backend/` y la carpeta
#    TestResults cuelga del PROYECTO DE TESTS. En el pipeline (§3.2) la carpeta
#    está en la raíz y va `TestResults/*/`: un solo nivel, y con UN asterisco —
#    allá los tests corren con `--logger trx`, que deja una COPIA del XML en otra
#    subcarpeta, y con dos asteriscos ReportGenerator lee las dos: el Summary sale
#    con «Parser: MultiReport (2x Cobertura)». Acá, sin trx, no pasa — verificá
#    igual que tu Summary diga «Parser: Cobertura».
open coveragereport/index.html   # (Windows: start …)
```

> 💻 **PowerShell**: las continuaciones `\` son de bash — en PowerShell escribí el comando de `reportgenerator` en **una sola línea** (o usá el backtick `` ` `` como continuación). El glob de `-reports:` lo expande ReportGenerator, no el shell, así que funciona igual en todos lados.

Mirá tu reporte: ¿qué % de línea y de **branch** tenés? ¿Qué archivos están en rojo? ¿Cuáles de esos vale la pena testear y cuáles corresponde **excluir** (arranque, config, clases de datos, código generado)?

> 📌 **Para comparar, lo que da la app de la cátedra** con la suite de §3.0 (lo que muestra el
> video): el proyecto entero, **~30 % de línea y 75 % de rama** —arrastrado por `Program`,
> `AppDbContext` y `NotificadorEmail`, en 0 %—; `TareaValidator`, **100 % de línea y 83 % de rama**. La rama que le
> falta es el `?.` de `titulo?.Trim()`: justo el ejercicio del recuadro 🔎 de §3.0.

**✅ Checkpoint:** ves tu coverage real (línea y branch) en un reporte HTML local, y podés nombrar qué está cubierto y qué no.

### 3.2 Coverage del backend EN el pipeline

> 🎯 **Qué tenés que lograr** — que esa medición corra **en el pipeline**, no sólo en tu máquina, y que su resultado se vea en la página de la corrida y se pueda descargar.
>
> *Lo que sigue es **cómo se hace en el ejemplo de la cátedra**. Si tu stack es otro, traducilo con la tabla «Tu stack, de un vistazo» — se corrige el logro de arriba, no la herramienta.*

El TP4 dejó tu Dockerfile con dos etapas —`build` y `final`— y un pipeline que sólo construye la
segunda. Ahora le sumamos **una etapa en el medio, que corre los tests**, y el pipeline la usa. Es
la promesa que dejó abierta el TP4, y se cumple sin duplicar nada: la etapa de tests reutiliza el
`build` que ya tenías.

> 🔴 **No vas a crear ningún archivo nuevo en esta sección.** Los dos archivos que tocás ya existen
> y los escribiste vos: `backend/Dockerfile` (TP2) y `.github/workflows/ci.yml` (TP4). A los dos les
> **agregás** cosas; nada se reemplaza.

**Uno — abrí `backend/Dockerfile`** (el del TP2, el que tu pipeline viene construyendo desde el TP4)
y agregale una etapa **en el medio**. Te queda así:

```
FROM …sdk… AS build        ← ya estaba, no se toca (compila tu app)
        ⬇
FROM build AS test         ← ESTO es lo que agregás  (2 instrucciones)
        ⬇
FROM …aspnet… AS final     ← ya estaba, no se toca (la imagen que desplegás)
```

La etapa nueva, completa (y renumerá el comentario de la de abajo a `# ---- Etapa 3: runtime …`: es
sólo un comentario, pero así no te quedan dos «Etapa 2»):

```dockerfile
# ---- Etapa 2: tests (parte del build, con el SDK adentro) ----
FROM build AS test
ENTRYPOINT ["dotnet", "test", "Backend.sln", \
            "--logger", "trx;LogFileName=tests.trx", \
            "--results-directory", "/out"]
```

> 🔴 **La consulta más frecuente de la semana: `FROM build` sólo sirve si `build` se copió TODA la
> solución, tests incluidos.** Si en el TP2 escribiste la etapa `build` copiando sólo el proyecto de
> la app —para aprovechar mejor el cache—, tu proyecto de tests no está adentro de la imagen y
> `dotnet test` no lo va a encontrar. Se arregla **en la etapa `build`**: copiando también el
> `.csproj` de tests antes del `restore` (`COPY MiApi.Tests/MiApi.Tests.csproj MiApi.Tests/`) y la
> carpeta de tests con el resto del código (un `COPY . .` ya la trae). Y de paso mirá tu
> `.dockerignore`: **si ahí excluiste la carpeta de tests, no entra por más que la pidas.** La imagen
> final no se lleva nada de eso: copia sólo lo publicado.

> 📌 **Por qué `FROM build` y no una imagen nueva**: los tests necesitan el SDK y el código fuente,
> y eso es exactamente lo que ya tiene la etapa `build`. Partir de ella no cuesta nada — Docker
> reutiliza esas capas — y garantiza que testeás **el mismo código fuente y las mismas
> dependencias** que compilaste. 🔴 **No los mismos binarios**: los tests compilan en `Debug` y la
> imagen se publica en `Release`. Lo que desaparece es la divergencia de **recetas** —una sola forma
> de construir, la del Dockerfile—, no la de configuración; y eso es lo que se contesta si en la
> defensa te preguntan «¿testeás lo mismo que desplegás?». La etapa `final` sigue intacta: tu imagen
> de producción no se lleva ni el SDK ni los tests adentro.
>
> 📌 **`/out` es una carpeta del contenedor**, y el contenedor se muere al terminar. Lo que la salva
> es el `-v` del paso siguiente: monta una carpeta del *runner* —la máquina prestada de GitHub que
> corre el job y se destruye al terminar; así la vas a ver nombrada en los logs— ahí, así que lo que el contenedor
> escribe en `/out` aparece en el disco de afuera.

#### La etapa, línea por línea

| Línea | Qué es | Qué pasa si la tocás |
|---|---|---|
| `FROM build AS test` | Arranca una etapa nueva **a partir de** `build`, y la bautiza `test`. Hereda todo lo que `build` tenía: el SDK, tus fuentes, las dependencias restauradas | `FROM mcr.microsoft.com/dotnet/sdk` arrancaría de cero: habría que volver a copiar el código y restaurar los paquetes. Más lento, y ya no sería *lo mismo* que compilaste |
| *(no va `WORKDIR`)* | 🔴 **No lo pongas.** `FROM build AS test` **hereda** la carpeta de trabajo de `build`, que es donde tu código ya está — sea `/src`, `/app` o la que uses | Escribir un `WORKDIR` fijo es lo que rompe: si tu `build` usa otra carpeta, apuntás a una vacía y `dotnet test` no encuentra la solución (`MSB1009: Project file does not exist`). Verificalo: `grep -n WORKDIR backend/Dockerfile` tiene que mostrar sólo el de `build` y el de `final`, ninguno entre `FROM build AS test` y su `ENTRYPOINT` |
| `ENTRYPOINT [...]` | El comando que corre el contenedor **cuando lo arrancás**. Ojo: no corre durante el `docker build` — la etapa se construye, y los tests recién corren en el `docker run` del pipeline | Con `RUN dotnet test` en vez de `ENTRYPOINT`, los tests correrían **al construir**, y sacar el reporte de ahí pide otro mecanismo (`--output type=local`) que no es el que usa el resto del pipeline. Con `ENTRYPOINT` los resultados salen por el mismo puente que ya conocés — un volumen — y el paso siguiente los lee sin nada nuevo |
| `"dotnet", "test", "Backend.sln"` | Qué se ejecuta y sobre qué. La forma con corchetes y comillas (*exec form*) es la que permite agregarle argumentos desde afuera | Con la forma de string (`ENTRYPOINT dotnet test`) los argumentos que le pases al `docker run` **se ignoran**, y el `--collect` del coverage nunca llega. Lo engañoso es cómo se ve: **los tests corren y salen verdes**, y el que se pone rojo es el paso *Reporte de coverage*, con *«found no matching files»* — un mensaje que no dice que la causa son los corchetes |
| `--logger "trx;LogFileName=tests.trx"` | Pide el resultado de cada test en un archivo `.trx` (el formato de reportes de .NET), con nombre fijo | Sin esto sólo queda la salida de consola: no hay archivo que publicar como evidencia |
| `--results-directory /out` | Dónde deja esos archivos **dentro** del contenedor | Si lo cambiás, cambialo también en el `-v` del paso siguiente — si no, el `-v` monta una carpeta que nadie escribe y salís con las manos vacías |

**Dos — abrí `.github/workflows/ci.yml`** y agregale pasos al job `build-backend` que ya tenías.
**No es un job nuevo ni un archivo nuevo**: son cuatro pasos más, al final de los que ya están.

| Tu job `build-backend` hoy (TP4) | Cómo queda después de esta sección |
|---|---|
| 1 · `Qué estamos verificando` (el `echo`) | 1 · *(igual)* |
| 2 · `actions/checkout` | 2 · `actions/checkout` *(igual)* |
| 3 · `docker/setup-buildx-action` (el constructor que sabe cachear) | 3 · *(igual)* |
| 4 · construir la imagen (`push: false`) | 4 · construir la imagen (`push: false`) *(igual)* |
| — | 5 · **construir la etapa `test`** ← nuevo |
| — | 6 · **`docker run`: correr los tests** ← nuevo |
| — | 7 · **armar el reporte y el resumen** ← nuevo |
| — | 8 · **publicar el reporte como artefacto** ← nuevo |

Si tu job del TP4 tiene algún paso más o de menos, no importa la numeración: lo que importa es que
los cuatro nuevos van **después** de construir la imagen. (En clase los numeramos del 3 al 6 sobre un
job resumido; en el `ci.yml` del video quedan entre las líneas 33 y 64.)

El primero de los pasos nuevos:

```yaml
      - name: Construir la etapa de tests
        uses: docker/build-push-action@v7
        with:
          context: ./backend
          target: test          # ← se detiene en la etapa de tests, no llega a final
          load: true            # ← deja la imagen en el Docker local, para poder correrla
          push: false
          tags: backend-test:ci
          cache-from: type=gha,scope=backend-test
          cache-to: type=gha,mode=max,scope=backend-test
```

> 🔴 **Cada build tiene su propio `scope`, y ahora son cuatro.** El TP4 ya avisó que dos builds sin
> `scope` comparten estante y se pisan. Con esta sección el repo pasa a construir **cuatro** cosas
> distintas —la etapa de tests y la imagen final, del back y del front—, así que son cuatro estantes:
> `backend-test`, `frontend-test`, `backend` y `frontend`. Sin eso vas a ver `CACHED` aparecer y
> desaparecer sin patrón, que es el síntoma exacto que el TP4 describe.

> 🔴 **`load: true` no es opcional acá.** Con buildx (el constructor del TP4) la imagen queda por
> default en el cache del constructor, **no** en el Docker de la máquina — y el `docker run` del
> paso siguiente falla con *"Unable to find image 'backend-test:ci' locally"*. `load: true` la baja
> al Docker local. Es el error más común de este paso, y el mensaje no dice qué falta.

Y los tres que siguen. El primero es el que **corre los tests de verdad**: arranca un contenedor con
esa imagen, y adentro se ejecuta el `ENTRYPOINT` que escribiste en el Dockerfile. Todo lo que pongas
después del nombre de la imagen se le **agrega** a ese comando:

```yaml
      - name: Correr los tests con coverage
        run: |
          docker run --rm -v "${{ github.workspace }}/TestResults:/out" backend-test:ci \
            --collect:"XPlat Code Coverage"

      - name: Reporte de coverage
        if: ${{ !cancelled() }}        # ← también cuando los tests fallaron: es la corrida que se entrega
        run: |
          dotnet tool install -g dotnet-reportgenerator-globaltool
          reportgenerator -reports:"TestResults/*/coverage.cobertura.xml" -sourcedirs:backend \
            -targetdir:coveragereport "-reporttypes:Html;MarkdownSummaryGithub" \
            "-classfilters:-Program*;-DemoApi.Data.*;-DemoApi.Models.*"
          cat coveragereport/SummaryGithub.md >> $GITHUB_STEP_SUMMARY

      - name: Publicar coverage y resultados
        uses: actions/upload-artifact@v6   # la del video; ya existe la v7, y cualquiera de las dos anda
        if: ${{ !cancelled() }}
        with:
          name: coverage-report
          path: |
            coveragereport
            TestResults/tests.trx
```

> 📌 **Por qué el reporte se arma afuera del contenedor**: los archivos de coverage ya están en el
> disco del runner gracias al `-v` que montaste arriba. ReportGenerator sólo los lee y produce el HTML — no
> necesita tu código ni tu SDK adentro de la imagen. Y como el `docker run` devuelve el código de
> salida de los tests, un test en rojo sigue frenando el pipeline **con el coverage ya extraído**.

#### Los pasos, línea por línea

| Línea | Qué es | Qué pasa si la tocás |
|---|---|---|
| `uses: docker/build-push-action@v7` | La **misma** action del TP4. No hay nada nuevo acá: cambian dos parámetros | — |
| `target: test` | Le dice a Docker **hasta qué etapa construir**. Se detiene en `test` y no llega a `final` | Sin `target`, construye hasta la última etapa del archivo: te da la imagen de producción, que no tiene los tests adentro |
| `load: true` | Baja la imagen construida **al Docker de la máquina**, para poder arrancarla con `docker run` | Sin esto el paso siguiente falla con *«Unable to find image `backend-test:ci` locally»*. Es el error más común de esta sección |
| `push: false` | Sigue sin publicar nada. La imagen de tests **no va a ningún registry**: vive y muere en el runner | En `true` pediría credenciales para subir algo que a nadie le sirve |
| `tags: backend-test:ci` | El nombre con el que queda esa imagen en la máquina. Es literal el que escribís en el `docker run` | Si los dos nombres no coinciden, el `docker run` no la encuentra |
| `scope=backend-test` | Su propio estante de cache, distinto del de la imagen final | Ver el recuadro rojo de arriba: sin `scope` propio, los cuatro builds se pisan el cache entre sí |
| `run: \|` | Bloque literal de YAML (la barra vertical). Es lo que permite partir el comando en varias líneas con `\` | Sin la barra, YAML pliega las líneas y el comando llega roto a bash (`MSB1008`) |
| `docker run --rm` | Arranca un contenedor de esa imagen y **lo borra al terminar** (`--rm`). El `ENTRYPOINT` de la etapa es lo que se ejecuta: los tests | Sin `--rm` el contenedor muerto queda ocupando disco en el runner (que igual se destruye, pero es el hábito correcto) |
| `-v "${{ github.workspace }}/TestResults:/out"` | El **puente** entre el contenedor y la máquina: monta la carpeta `TestResults` del runner sobre `/out` del contenedor. Lo que los tests escriben adentro aparece afuera | Sin el `-v`, los tests corren igual… y sus resultados se van con el contenedor. Los tests salen **verdes** y el que falla es el paso *Reporte de coverage* (*«found no matching files»*): el job queda rojo por el reporte, no por los tests, y el mensaje no dice que falta el `-v` |
| `backend-test:ci \` | Qué imagen arrancar. La barra final continúa el comando en la línea de abajo | — |
| `--collect:"XPlat Code Coverage"` | Se le **agrega** al `ENTRYPOINT`: es como si hubieras escrito `dotnet test … --collect:…`. Pide medir la cobertura (formato Cobertura, el default) | Sin esto los tests corren igual y no se mide nada — y el paso del reporte falla con *«found no matching files»* |
| `"-classfilters:-Program*;…"` | Recorta **el reporte** al mismo código que vas a exigir en el §3.4: saca el arranque, los datos y los modelos. **No frena nada** — sólo decide qué se muestra. `DemoApi` es el namespace de la app del ejemplo: poné el **tuyo** | Sin esto el Summary dice ~30 % al lado de un umbral de 70 en verde: dos mediciones distintas de la misma cosa, y no vas a poder explicarlo en la defensa |
| `dotnet tool install -g …reportgenerator…` | Instala la herramienta que convierte los XML de coverage en un HTML legible. Corre **en el runner**, no en el contenedor: los archivos ya están afuera gracias al `-v` | — |
| `-reports:"TestResults/*/coverage.cobertura.xml"` | Qué archivos leer. El `*` es porque coverlet lo deja en una subcarpeta con un identificador aleatorio: es **un** nivel, y por eso va un asterisco y no dos | Con una ruta fija no encuentra nada: ese nombre cambia en cada corrida. Y con `**` encuentra además la **copia** del XML que el logger `trx` deja en `_<máquina>_<fecha>/In/…`: el Summary sale con `Parser: MultiReport (2x Cobertura)`, contando dos veces un reporte que es uno. El porcentaje no cambia, pero el Summary queda engañoso y no lo vas a poder explicar |
| `-reporttypes:"Html;MarkdownSummaryGithub"` | Dos salidas: el sitio HTML navegable, y un resumen en Markdown pensado para GitHub | — |
| `cat …SummaryGithub.md >> $GITHUB_STEP_SUMMARY` | `$GITHUB_STEP_SUMMARY` es un archivo especial: **todo lo que le agregues se muestra en la página de la corrida**. Acá es lo que hace visible el número sin descargar nada | Sin esta línea el coverage existe, pero hay que bajarse un `.zip` para verlo — y un número que nadie ve no cambia decisiones. Es la misma idea que el badge del TP4: poner el estado donde la gente ya está mirando |
| `uses: actions/upload-artifact@v6` | Sube archivos de la corrida para que sobrevivan a la máquina efímera y te los puedas descargar | Sin esto, el reporte HTML se destruye junto con el runner |
| `if: ${{ !cancelled() }}` | Corre el paso **aunque uno anterior haya fallado** (lo que GitHub recomienda hoy en lugar de `always()`, que corre incluso si cancelás la corrida a mano) | Sin esto, cuando los tests fallan no hay reporte — que es justo cuando lo querés mirar |

Y el orden importa: el `docker run` devuelve el **código de salida** de los tests, así que un test en
rojo frena el job ahí mismo. Los pasos de reporte y publicación van después y sólo el `if:` que llevan
los rescata.

Tres detalles con intención:

- ⚠️ El `run: |` no es decorativo — ver la tabla. Es el error de YAML más difícil de leer de esta sección.
- El HTML completo y el `tests.trx` viajan como **artefacto**: es el mecanismo con el que un archivo que produjo la
  corrida sobrevive a la máquina efímera y te lo podés bajar. Ese `if:` es lo que hace que también los
  tengas **cuando los tests fallaron** — que es justo cuando los querés mirar. 📌 Ojo con la palabra:
  en el TP6 «artefacto» va a nombrar algo más grande —el producto que el pipeline entrega—; acá es
  sólo un archivo de la corrida.
- 📌 **En la corrida vas a ver más artefactos de los que publicaste**: unos `….dockerbuild` y unas
  tablas *Build records* en el Summary. Los agrega sola `docker/build-push-action`, uno por cada
  construcción; no los pediste y no hacen falta. Los tuyos son `coverage-report` y el del frontend.

**✅ Checkpoint:** el pipeline **corre tus tests** en cada PR (rojo si alguno falla), muestra el resumen de coverage en el Summary del run, y deja el reporte HTML descargable como artefacto. En la página de la corrida, bajo *build-backend summary*, tiene que aparecer una tabla con `Parser: Cobertura`, `Line coverage` y `Branch coverage` (en el ejemplo, 86,9 % y 75 %). Si dice `MultiReport`, revisá el glob.

### 3.3 Coverage del frontend con umbral que rompe el build

> 🎯 **Qué tenés que lograr** — un número que vos elegís y que **frena el build** cuando la cobertura queda por debajo. Que frene de verdad es la mitad de la nota de esta tarea.
>
> *Lo que sigue es **cómo se hace en el ejemplo de la cátedra**. Si tu stack es otro, traducilo con la tabla «Tu stack, de un vistazo» — se corrige el logro de arriba, no la herramienta.*

En el frontend, vitest trae coverage integrado (provider **V8**).

> 🪜 **Si tu frontend todavía no tiene tests** (lo normal: el TP4 sólo construía), primero el puente:
> `npm i -D vitest` y el script `"test": "vitest"` en `package.json` (es el mismo que usa el sample). Un test de lógica pura
> entero, para que veas dónde va y cómo se corre — el archivo se llama igual que el que prueba, con
> `.test.js` en el medio, y vive **al lado** de él (`src/lib/precios.js` → `src/lib/precios.test.js`):
>
> ```js
> import { describe, it, expect } from 'vitest';
> import { conDescuento } from './precios';
>
> describe('conDescuento', () => {
>   it('un descuento del 10% sobre 1000 da 900', () => {
>     expect(conDescuento(1000, 10)).toBe(900);   // arrange y act en una línea; el assert es toBe
>   });
> });
> ```
>
> Corrés `npm test -- --run` y tenés que ver `1 passed`. (Sin el `--run`, `vitest` se queda mirando
> cambios y no termina — es el modo *watch*, cómodo mientras escribís; salís con `q`.) Ésos son los mismos tres bloques del backend —
> preparar, ejecutar, comprobar— con otros nombres. Escribí así los 4 que pide la Tarea 1 y recién
> después sumá coverage.
>
> 🧪 **¿Y testear componentes React?** Testear componentes requiere además jsdom + Testing Library. Queda como **opcional avanzado** — no lo exige el TP: en esta materia la relación costo/valor favorece testear la **lógica pura** unitariamente, y la UI completa se verifica **end-to-end en el TP7**, contra la app desplegada de verdad.

Instalá el paquete de coverage y declará el **umbral**:

```bash
cd frontend
npm ls vitest                      # anotá el número que te imprime (en el video, 3.2.7)
npm i -D @vitest/coverage-v8@3     # ⚠️ el @3 es el MAJOR de TU vitest (4.x → @4, 5.x → @5): la última
                                   #    de esa línea, no la última a secas
npm ls vitest @vitest/coverage-v8  # 🔴 comprobalo: los DOS tienen que mostrar la MISMA versión
# Si el `npm i` de arriba falla con ERESOLVE (tu vitest quedó fijo en una versión anterior de su
# línea), instalá el número EXACTO de tu vitest —p. ej. npm i -D @vitest/coverage-v8@3.2.4—.
# Nunca lo "arregles" con --force.
```

```js
// vite.config.js — dentro de defineConfig({ … })
test: {
  coverage: {
    provider: 'v8',
    reporter: ['text', 'html', 'lcov', 'json-summary'],
                                    // 🔴 `json-summary` NO es decorativo: deja un
                                    //    `coverage-summary.json` con los totales, y es el
                                    //    archivo que el paso del pipeline lee para armar la
                                    //    tablita del Summary. Sin él ese paso falla con
                                    //    «no está el archivo» y no vas a saber por qué.
    include: ['src/TU-CARPETA/**'], // 🔴 cambiá esto por la carpeta donde vive TU lógica
                                    //    (en la app de la cátedra, 'src/lib/**').
                                    //    Esta línea decide QUÉ entra en la cuenta, y sin ella
                                    //    falla distinto según tu versión: en vitest 3 mide TODO
                                    //    el proyecto —componentes, pegamento de UI— y el número
                                    //    se hunde; desde vitest 4 mide SÓLO lo que tus tests
                                    //    importan, así que un archivo nuevo sin tests ni aparece
                                    //    y el número SUBE: el umbral queda ciego justo para el
                                    //    código que el §3.5 quiere frenar. Con `include`, todo
                                    //    archivo de esa carpeta entra aunque nadie lo pruebe.
                                    //    Y si el patrón no matchea nada, mide CERO archivos y
                                    //    pasa en verde, en silencio. Verificá que el reporte
                                    //    liste los archivos que esperabas
    thresholds: { lines: 80, branches: 80 },   // TU umbral, el que puedas defender
                                              // (80 es el del ejemplo: el número lo elegís y lo justificás)
  },
},
```

Y declará **un solo comando** para correr la suite con cobertura, en el `package.json`. Es el que
vas a usar en tu máquina y el que va a usar el pipeline — **una receta, no dos**:

```json
"scripts": {
  "test": "vitest",
  "test:ci": "vitest run --coverage --coverage.reportsDirectory=${COVERAGE_DIR:-coverage}"
}
```

> 📌 **Qué hace ese `${COVERAGE_DIR:-coverage}`**: si la variable `COVERAGE_DIR` está definida, el
> reporte va ahí; si no, va a `coverage/`, al lado tuyo. Así el **mismo** comando sirve en tu
> máquina —sin definir nada— y adentro del contenedor del pipeline, que sí la define para que el
> reporte caiga en la carpeta montada. El día que cambies cómo se corre la suite, lo cambiás en un
> solo lugar y el pipeline se entera solo.
>
> 💻 **Windows**: esa expansión `${…:-…}` es de `sh`, y en Windows npm corre los scripts con
> `cmd.exe`, que no la entiende — el reporte cae en una carpeta llamada literalmente
> `${COVERAGE_DIR:-coverage}`. En tu máquina corré `npm test -- --run --coverage` (va a `coverage/`);
> el script `test:ci` es el del contenedor, que es Linux, y ahí anda.

```bash
npm run test:ci        # corré local: ¿pasa tu umbral?
                       # (equivale a `npm test -- --run --coverage`, que es lo que ves en el video)
```

Cómo se lee: la tabla de v8 trae, por archivo, `% Stmts`, `% Branch`, `% Funcs`, `% Lines` y las
líneas sin cubrir. **Si pasás el umbral, vitest no dice nada sobre él** y el comando termina bien.
Si no llegás, al final imprime:

```
ERROR: Coverage for branches (66.66%) does not meet global threshold (80%)
```

y sale con error — aunque todos los tests estén en verde. Ése es el rojo que buscás. Si en cambio ves
`Tests 0 passed` o un error de versiones mezcladas, el rojo es de instalación, no del umbral.

🔴 **Los `thresholds` sólo se evalúan si la corrida pide cobertura.** `npm test -- --run` a secas,
sin `--coverage`, no mide nada y pasa en verde aunque estés por debajo (salvo que actives
`coverage.enabled` en la config): por eso el pipeline corre `test:ci`.

Y ahora al pipeline, **con la misma receta que el backend**: una etapa de tests en el Dockerfile del
frontend, que el job construye y corre. Los pasos de abajo van al final de los steps del job
**`build-frontend`** — el que ya tenés del TP4, el hermano del `build-backend` del §3.2.

🔴 **La etapa va EN EL MEDIO, entre la de build y la de nginx.** Si la pegás al final, la última etapa
del archivo pasa a ser la de tests y tu imagen de producción termina siendo el contenedor de tests —
sin que nada se ponga rojo. El `frontend/Dockerfile` completo queda así (es el que se ve en el video,
con los comentarios de etapa renumerados):

```dockerfile
# ---- Etapa 1: build ----
FROM node:22-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci                 # ci, no install: respeta el package-lock.json al pie de la letra
COPY . .
RUN npm run build

# ---- Etapa 2: tests (parte del build: ya tiene Node y las dependencias) ← ESTO es lo que agregás ----
FROM build AS test
ENTRYPOINT ["npm", "run", "test:ci"]

# ---- Etapa 3: nginx sirve los estáticos (ya la tenías, queda ÚLTIMA) ----
FROM nginx:alpine
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 80
```

```yaml
      - name: Construir la etapa de tests del frontend
        uses: docker/build-push-action@v7
        with:
          context: ./frontend
          target: test
          load: true            # ← igual que en el backend: sin esto no se puede correr
          push: false
          tags: frontend-test:ci
          cache-from: type=gha,scope=frontend-test
          cache-to: type=gha,mode=max,scope=frontend-test

      - name: Correr los tests del frontend con coverage
        run: |
          docker run --rm -e COVERAGE_DIR=/salida/reporte \
            -v "${{ github.workspace }}/frontend-coverage:/salida" frontend-test:ci

      - name: Resumen del coverage del frontend
        if: ${{ !cancelled() }}
        run: |
          F=frontend-coverage/reporte/coverage-summary.json
          test -f "$F"          # ← si no está, el paso FALLA: mejor rojo que un resumen vacío
          node -e "
            const t = require('./$F').total;
            if (!t.lines.total) { console.error('midio 0 archivos: revisa el include'); process.exit(1); }
            const f = (m) => \`| \${m} | \${t[m].pct}% |\`;
            console.log('### Coverage del frontend');
            console.log('| métrica | % |'); console.log('|---|---|');
            console.log(f('lines')); console.log(f('branches')); console.log(f('functions'));
          " >> $GITHUB_STEP_SUMMARY

      - name: Publicar el coverage del frontend
        uses: actions/upload-artifact@v6   # la del video; ya existe la v7, y cualquiera de las dos anda
        if: ${{ !cancelled() }}
        with:
          name: coverage-frontend
          path: frontend-coverage
```

> 📌 **De dónde sale ese resumen**: el reporter `json-summary` que declaraste arriba en el
> `vite.config.js` deja un `coverage-summary.json` con los totales, y el paso lo convierte en una
> tablita. El reporte navegable sigue yendo como artefacto.
>
> 📌 **Y de dónde salen las rutas.** El contenedor escribe en `/salida/reporte` porque el `docker
> run` le pasa `COVERAGE_DIR` con ese valor, y `/salida` está montado sobre `frontend-coverage/` del
> runner: por eso el archivo aparece afuera en `frontend-coverage/reporte/`. Si cambiás una, cambiá
> las tres — es el mismo puente que el `-v` del backend en el §3.2.
>
> 🔴 **Los dos frenos del paso, y por qué hacen falta los dos.** El `test -f` cubre «no está el
> archivo». El `if (!t.lines.total)` cubre el caso silencioso, que es el que este práctico entero
> viene a cerrar: si tu `include` no matchea nada, el archivo **sí** se escribe, con `total: 0` y
> `pct: "Unknown"` — y con cero archivos medidos, **el umbral ni se evalúa**. Sin ese segundo freno,
> el `include` mal escrito publica una tabla de `Unknown%` y el job queda **verde**.

> ⚠️ **La etapa de tests necesita las dependencias de desarrollo, y las imágenes suelen construirse
> sin ellas a propósito.** Si tu Dockerfile instala con `npm ci --omit=dev`, la etapa de tests no va a
> encontrar vitest: ese flag saltea justamente las dependencias de desarrollo. Sacale el flag a la
> etapa de build (la imagen final igual no las lleva: nginx sólo se copia el resultado compilado).
> La trampa es la misma en cualquier stack: en Python, el `requirements.txt` que no incluye el de
> tests; en Java, las dependencias con `<scope>test</scope>` que tu build saltea. El arreglo es
> siempre el mismo: **dárselas a la etapa que compila para testear, no a la final.** Tu fila está en
> la tabla «Tu stack, de un vistazo».
>
> 📌 **Una sola regla para todo el TP**: los dos jobs testean adentro del contenedor, en la etapa que
> ya tiene las herramientas —el SDK en el backend, Node y tus dependencias de desarrollo en el
> front— y ninguno de los dos las mete en la imagen final. El pipeline sigue sin saber cómo se
> compila ni cómo se testea tu app: se lo sigue pidiendo a tu Dockerfile.

Acá está la mecánica de gate más simple de toda la guía: si el coverage cae bajo el umbral, **`vitest` sale con error → el `docker run` devuelve ese error → el job falla → el required check del TP4 bloquea el merge**. Sin herramientas nuevas: el gate del TP4 ya sabía bloquear; ahora tiene una condición más que vigilar.

**Probalo en el pipeline**: comentá un test, pusheá, y mirá el job del frontend ponerse rojo por
umbral — esa corrida es la evidencia del checkpoint. 🔴 **Después descomentalo y volvé a pushear
antes de mergear**: esa rama era una demostración, y si entra con el test apagado tu suite entregada
tiene un agujero.

> 📌 **Qué test comentar para verlo frenar — no cualquiera sirve.** Si comentás uno que es **el único
> que ejecuta una función**, bajan las **líneas** y, en vitest 3, no las ramas: esa versión no cuenta
> los caminos de una función que nunca se ejecuta (desde vitest 4 sí, y bajan las dos). Si comentás uno **cuya función ya tocan otros tests**
> pero que era el único que recorría un camino de un `if`, las líneas casi no se mueven y bajan las
> **ramas**. Con `thresholds: { lines: 80, branches: 80 }` frena el que deje corta cualquiera de las
> dos (en el video, el segundo: 94,73 % de líneas, 66,66 % de ramas). Y si tu suite está muy por
> arriba del umbral, comentar uno puede no alcanzar: en ese caso subí el umbral por encima de tu
> número actual para la prueba, o agregá un archivo sin tests (§3.5), y volvé todo atrás.

> 💡 **En el backend el equivalente existe**, y lo armás en el §3.4: el paquete `coverlet.msbuild` con `dotnet test /p:CollectCoverage=true /p:Threshold=NN /p:ThresholdType=line` rompe el build bajo el umbral. 🔴 Y aparte va `/p:Exclude`, que no arma el umbral sino que dice **qué entra en la cuenta** — sin él el número es inalcanzable; el §3.4 los explica juntos. Los tres de acá hacen falta igual: sin `CollectCoverage` el threshold no se evalúa, y sin `ThresholdType=line` te exige línea, rama **y método** a la vez — vas a fallar por método sin entender por qué.

**✅ Checkpoint:** el job de front falla si el coverage cae del umbral declarado (evidencia: una corrida roja por umbral) y pasa con la suite completa.

### 3.4 Que el umbral rompa el build **de los dos lados**

> 🎯 **Qué tenés que lograr** — que un Pull Request cuya cobertura no llega a tu número **no se
> pueda mergear**. Compilando bien y con todos los tests en verde.
>
> *Lo que sigue es **cómo se hace en el ejemplo de la cátedra**. Si tu stack es otro, traducilo con
> la tabla «Tu stack, de un vistazo» — se corrige el logro de arriba, no la herramienta.*

> ✅ **Y acá hay una buena noticia que conviene entender, porque explica todo el diseño de la
> materia: no vas a tener que configurar ningún freno nuevo.** El freno ya está puesto desde el TP4
> —`build-backend` y `build-frontend` son *required checks* de `main` desde entonces— y la cobertura
> que agregaste corre **adentro de esos mismos dos jobs**, no en jobs nuevos. Así que el día que tu
> umbral ponga uno de esos jobs en rojo, el merge se bloquea **solo**. Todo tu trabajo de esta
> sección es uno: **que el número rompa de verdad**.

El §3.3 ya lo dejó funcionando en el frontend. Falta el backend, que es donde casi todos entregan
incompleto: **el §3.1 mide, pero no frena** — `coverlet.collector`, el que
trae el template, produce el archivo de cobertura y nada más. Para que un número bajo ponga el job en
rojo hace falta **otro paquete**:

```bash
dotnet add DemoApi.Tests/DemoApi.Tests.csproj package coverlet.msbuild   # con el nombre de TU proyecto de tests
```

y **tres** parámetros en el `ENTRYPOINT` de tu etapa de tests —los mismos tres de la clase—, más un
cuarto que no es del umbral sino de **qué se mide** (abajo). 🔴 **Cada uno es un elemento del array,
con su coma**: pegados como uno solo, `dotnet test` los ignora **sin decir nada** y volvés a tener un
verde que no exige nada.

```dockerfile
ENTRYPOINT ["dotnet","test","Backend.sln", \
            "/p:CollectCoverage=true", "/p:Threshold=70", "/p:ThresholdType=line", \
            "/p:Exclude=[DemoApi]Program*%2c[DemoApi]DemoApi.Data.*%2c[DemoApi]DemoApi.Models.*", \
            "--logger","trx;LogFileName=tests.trx","--results-directory","/out"]
```

Los tres hacen falta: sin `CollectCoverage` el threshold **se ignora**, y sin `ThresholdType` te
exige línea, rama **y método** a la vez — vas a fallar por método sin entender por qué. 📌 **El
`--collect` del §3.2 se queda donde está**: hacen cosas distintas y conviven — aquél produce el
archivo que lee el reporte, y estos `/p:` son los que **frenan**. Verificado: con los dos puestos,
por debajo del umbral el comando devuelve error y dice cuál fue.

> 🔴 **El que dice QUÉ SE MIDE es el que hace alcanzable el 70, y por eso va en el mismo bloque.** Sin
> él, coverlet mide **el ensamblado entero**: `Program.cs`, el `DbContext`, los modelos, el arranque.
> Nada de eso tiene tests ni debería tenerlos, y arrastra el número al piso. Son las **dos familias**
> del §2.4 en esta app: **el arranque** (`Program*`) y **las clases de datos** —el `DbContext`
> (`*.Data.*`) y los modelos que sólo tienen propiedades, `Tarea` y compañía (`*.Models.*`)—. Medido sobre la app de
> la cátedra, con la suite de la Tarea 1 completa y en verde: **30,55 % midiendo todo, y 86,95 %
> sacando el arranque, los datos y los modelos**. Con el umbral en 70, el primero **falla siempre**,
> y el alumno no tiene forma de saber si se equivocó en algo.
>
> 🔴 **Es `Exclude` —lo que se saca— y no `Include` —lo que se deja—, y la diferencia importa.** Con
> `Exclude`, la clase nueva que escribas mañana **entra a la cuenta sola**: si no tiene tests, el
> número baja y el gate te avisa. Con `Include` pasaría lo contrario — lo que te olvides de nombrar
> desaparece de la medición **en silencio**, y cada archivo nuevo nace invisible para el umbral. Un
> control que falla hacia el número alto no es un control.
>
> 🔴 **La coma va escrita `%2c`** porque MSBuild parte los valores por coma. Y ojo con el nombre del
> ensamblado entre corchetes: es el de **tu** proyecto, no el del ejemplo.
>
> 🔴 **Y `Program*` filtra por nombre de CLASE, no por archivo.** El `record` que hayas dejado suelto
> al final de tu `Program.cs` no se llama `Program`, así que sigue contando — es la trampa del §2.4
> otra vez, con el otro mecanismo. Es literal lo que separa el 86,95 % de arriba de un 90,9 %:
> movelo a su propio archivo (lo correcto) o nombralo también en el filtro.
>
> 📌 **Verificalo, no lo supongas** — es el mismo cuidado que el §3.3 le pide al frontend. Corré la
> etapa de tests y mirá la tabla que imprime coverlet: si el porcentaje **saltó al 100 %** o si el
> `Total` bajó a un puñado de líneas, tu filtro se comió más de lo que debía. Y si no se movió nada,
> no matcheó nada.
>
> 🔴 **Y lo que sacás del umbral tiene que ser lo mismo que sacás del reporte**, o vas a entregar un
> Summary que dice 30 % al lado de un umbral de 70 en verde — dos mediciones distintas de la misma
> cosa, que se lee como trampa y no vas a poder explicar. En el §3.2 ya lo pusiste; **volvé a mirarlo
> ahora** y comprobá que las dos listas nombren lo mismo, porque es el mismo recorte escrito de otra
> forma:
>
> ```
> reportgenerator ... "-classfilters:-Program*;-DemoApi.Data.*;-DemoApi.Models.*"
> ```

> 🔴 **Dos cosas que sorprenden, y las dos son pregunta de defensa.** Con `ThresholdType=line` el
> umbral mira **sólo líneas** — justo la métrica que este práctico llama la menos honesta; si querés
> que también mire ramas, poné `"/p:ThresholdType=line%2cbranch"`. 🔴 **Con la coma de verdad no
> anda**: `"line,branch"` muere con `MSB1006: Property is not valid. Switch: branch` **antes de
> correr un solo test**, y el error no menciona ni cobertura ni comas — es el mismo `%2c` de arriba.
> Con un solo `Threshold` alcanza: se aplica a las dos métricas y frena por la que quede corta
> (medido). Si querés números distintos, `"/p:Threshold=80%2c60"`. Y coverlet, por default, evalúa
> el umbral **por ensamblado, no sobre el total** (`ThresholdStat=minimum`): con dos proyectos y uno
> en cero, falla aunque el promedio dé bien. Si querés que mire el total,
> `"/p:ThresholdStat=total"`. Elijas lo que elijas, decilo en `decisiones.md` con su porqué.

**Y eso es todo lo que hay que hacer.** No se agrega ningún job, ningún check y ninguna
configuración de rama: la protección que armaste en el TP4 ya cubre esto.

> 📌 **Verificalo en 20 segundos, y no lo saltees** — es lo único que puede fallar acá: *Settings →
> Branches →* la regla de `main` → en **Require status checks to pass before merging** tienen que
> figurar `build-backend` y `build-frontend`. Si están, listo. Si por algún motivo se perdieron
> —cambiaste el nombre de un job, rehiciste la protección—, volvelos a tildar: es el mismo lugar y
> el mismo gesto del TP1 y del TP4. 🔴 Y si el buscador no los encuentra, no está roto: GitHub sólo
> ofrece checks que corrieron en los últimos 7 días. Abrí un Pull Request, dejá que corra, y volvé.

**✅ Checkpoint:** un job se pone rojo **por cobertura** —no sólo por compilación— en los dos lados,
y ese job figura como *Required* en un Pull Request nuevo. En el Pull Request el check aparece como
`CI / build-frontend (pull_request)` con la etiqueta **Required**; con él en rojo, el recuadro dice
*Some checks were not successful* y, para vos como dueño, el botón de merge queda gris.

### 3.5 Romper el gate (a propósito) y ver la calidad frenar un merge

> 🎯 **Qué tenés que lograr** — la evidencia de que el freno funciona: un Pull Request bloqueado por
> **calidad**, no por compilación, y su secuencia hasta el merge.
>
> *Lo que sigue es **cómo se hace en el ejemplo de la cátedra**. Si tu stack es otro, traducilo con
> la tabla «Tu stack, de un vistazo» — se corrige el logro de arriba, no la herramienta.*

Ésta es la evidencia central del práctico, y la razón por la que todo lo anterior valió la pena.

1. Partí de una rama que **ya tenga** la suite, la cobertura y el umbral: tu rama de trabajo, o
   `main` si ya la mergeaste. Si partís de un `main` que todavía no tiene el umbral, no hay nada que
   frene. (En el video se parte de la rama de trabajo, así que ese Pull Request lleva también todo lo
   de la semana — y al mergearlo, entra a `main`.)

   Ahí agregá **código nuevo sin tests** — un método con varios caminos que nadie ejercita. Lo
   importante es que sea código que **compila perfecto**: si rompés la compilación, estás
   demostrando el gate del TP4, no el de esta semana. 🔴 **Y que tenga el tamaño suficiente para
   bajar tu número.** El porcentaje es de todo lo medido: si hoy tenés 45 líneas cubiertas de 45 y tu
   umbral es 80, cinco líneas nuevas sin tests no alcanzan (45 de 50 = 90 %). Hacé la cuenta con tu
   reporte antes de subir. En el video, un método de veinte líneas —quince que ningún test toca—
   bajó el frontend de 100 % a 74,13 %.
2. Abrí el Pull Request. Los tests pasan todos, no hay un solo error… y el job igual se pone
   **rojo**, porque la cobertura cayó por debajo de tu número. En el log del paso de tests está dicho
   con todas las letras cuál era el umbral y cuánto dio — en el frontend, una línea así:

   ```
   ERROR: Coverage for lines (74.13%) does not meet global threshold (80%)
   ```

   Alcanza con que se ponga rojo **uno** de los dos jobs —en el video, el del frontend, porque el
   código nuevo está ahí—; el otro puede seguir verde. Los dos figuran como *Required* en la lista de
   checks del Pull Request, y con uno solo en rojo el merge ya queda bloqueado. 🔴 **Guardá la
   dirección de esa corrida roja** (`…/actions/runs/<id>`, no la del job verde) **y la del Pull
   Request**: las dos van a `decisiones.md` cuando termines el paso 3.
3. Escribí los tests que faltaban, pusheá, mirá el check pasar a verde, y **mergealo**. Su
   historial queda contando la secuencia completa. **¿Cuántos tests?** No lo inventás vos: uno por
   cada camino que el código nuevo declara — en el video, cinco: sin fecha, nueva, de esta semana, de
   este mes, vieja. No es «escribí muchos tests»: es «cubrí lo que declaraste». Antes de pushear,
   corré `npm run test:ci` (o `npm test -- --run --coverage`, como en el video) y mirá que termine sin
   `ERROR: Coverage…`.
4. 🔴 **Y ahora abrí un SEGUNDO Pull Request, chiquito, con el mismo problema — y dejalo ahí, abierto
   y en rojo, hasta la defensa.** No lo arregles. Abrilo desde `main`, **después** del merge del
   paso 3, con código nuevo sin tests —otro método, no hace falta que sea el mismo— y verificá que
   quede rojo: «chiquito» es de un archivo, no de dos líneas (vale la cuenta del paso 1). Son dos a
   propósito: el primero *cuenta* la historia, el segundo la *prueba*. La pantalla de configuración de
   la protección sólo la ve quien administra el repositorio, y lo que dice no prueba que el freno
   funcione; un Pull Request frenado y visible —con su check en rojo y el motivo— sí, y la corrección
   lo comprueba por su cuenta.

> 🔴 **Por qué frena, y en qué métrica — lo vas a tener que contar, y depende de tu versión de
> vitest.** En **vitest 3** (el del video) una función que ningún test llama **no suma ramas**: sólo
> suma líneas sin cubrir. Por eso el ejemplo frena por **líneas** (74,13 % contra 80) mientras las
> ramas siguen en 90 %, y las ramas recién se mueven (a 94,44 %) cuando los tests entran por cada
> camino. **Desde vitest 4** esas ramas sí cuentan desde el principio: el mismo código cae en las dos
> métricas (medido: 54,54 % de líneas y 36,84 % de ramas). Por las dos cosas, poné umbral en `lines`
> **y** en `branches`: con uno solo de ramas, en vitest 3 la demostración no se pone roja. Para contar
> los caminos del paso 3, no dependas de cómo los muestre el reporte: leé los `if` del código nuevo,
> cada salida distinta pide una entrada que la recorra. En `decisiones.md` decí en qué métrica frenó
> el tuyo y con qué versión de vitest: el log lo dice.

<details>
<summary>📎 El ejemplo del video, para releerlo: el método sin tests y sus cinco tests</summary>

```js
// src/lib/tareas.js — agregado al final

/**
 * Devuelve la prioridad de una tarea segun cuanto hace que se creo.
 * (Tiene varios caminos adentro y —a proposito— ni un solo test.)
 */
export function prioridadDe(tarea, ahora = new Date()) {
  if (!tarea || !tarea.creadaEl) {
    return 'sin-fecha'
  }
  const dias = (ahora - new Date(tarea.creadaEl)) / 86400000
  if (dias < 1) {
    return 'nueva'
  }
  if (dias < 7) {
    return 'esta-semana'
  }
  if (dias < 30) {
    return 'este-mes'
  }
  return 'vieja'
}
```

```js
// src/lib/prioridad.test.js — un test por camino. `ahora` entra por parámetro,
// así el test no depende del reloj (la regla de determinismo del §2.2)
import { describe, expect, it } from 'vitest'
import { prioridadDe } from './tareas.js'

describe('prioridadDe', () => {
  const ahora = new Date('2026-07-01T12:00:00Z')

  it('devuelve sin-fecha si la tarea no tiene fecha de creación', () => {
    expect(prioridadDe({}, ahora)).toBe('sin-fecha')
  })

  it('devuelve nueva dentro del primer día', () => {
    expect(prioridadDe({ creadaEl: '2026-07-01T06:00:00Z' }, ahora)).toBe('nueva')
  })

  it('devuelve esta-semana antes de los siete días', () => {
    expect(prioridadDe({ creadaEl: '2026-06-28T12:00:00Z' }, ahora)).toBe('esta-semana')
  })

  it('devuelve este-mes antes de los treinta días', () => {
    expect(prioridadDe({ creadaEl: '2026-06-15T12:00:00Z' }, ahora)).toBe('este-mes')
  })

  it('devuelve vieja pasados los treinta días', () => {
    expect(prioridadDe({ creadaEl: '2026-05-01T12:00:00Z' }, ahora)).toBe('vieja')
  })
})
```

</details>

> 🔴 **Mirá bien el paso 2, porque es el punto de toda la materia hasta acá.** No hay ningún error.
> Nada está roto. El código compila, los tests pasan, y el merge está bloqueado igual — por un
> número que elegiste vos hace dos secciones. Es la primera vez en el semestre que lo que te frena
> no es la máquina diciendo «esto no anda», sino un criterio de calidad que vos mismo escribiste.

**✅ Checkpoint:** la secuencia completa documentada — check rojo **por cobertura** con su motivo en
el log → los tests que faltaban → verde → merge.

## 4- Riel alternativo: Azure Pipelines

Mismos conceptos y checkpoints; mapa de equivalencias:

| Concepto | GitHub Actions (canónico) | Azure Pipelines |
|---|---|---|
| Coverage backend | `--collect:"XPlat Code Coverage"` (coverlet) | Igual (mismo `dotnet test`) + task `PublishCodeCoverageResults@2` sobre el `coverage.cobertura.xml` (pestaña Code Coverage) y `PublishTestResults@2` con `testResultsFormat: VSTest` sobre el `tests.trx` (pestaña Tests: con los tests adentro de un contenedor, ADO no los recoge solo) |
| Reporte visible | `$GITHUB_STEP_SUMMARY` + artefacto | Pestañas **Tests** y **Code Coverage** del run (punto fuerte de ADO) |
| Umbral frontend | `coverage.thresholds` de vitest (rompe el job) | Igual (misma config de vitest) |
| Umbral backend | `coverlet.msbuild` con `/p:Threshold` en el `ENTRYPOINT` | Igual (mismo `dotnet test` adentro del contenedor) |
| El check como obligatorio | Required status check en la regla de `main` | Branch policy **Build validation** (con Azure Repos); con repo en GitHub, sigue siendo el required check de GitHub (combo híbrido del TP4) |

**Checkpoints riel Azure:** los mismos 6 (mock → coverage local → coverage en pipeline con reporte → umbral rompiendo el build en los dos lados → checks required → secuencia rojo→fix→verde).

> 📌 Recordatorio 2026 del TP4: los minutos hosted de ADO requieren una organización vinculada a una suscripción de Azure; sin tarjeta, agente self-hosted.

---
---

# 📋 Trabajo Práctico 05 – Calidad automatizada: tests, coverage y el umbral que frena un merge (2026)

## ⚠️ Este es el TP que debés entregar y defender

## 🎯 Objetivo

Convertir la calidad de tu app en algo **medido y exigido por el pipeline**: una suite de unit tests significativa, la cobertura publicada en cada corrida, y un umbral elegido por vos que **bloquea el merge** cuando no se cumple. Al final de la semana, un cambio que compila perfecto y pasa todos los tests puede quedar frenado igual — y vas a poder explicar por qué eso es bueno.

Este trabajo se aprueba **solo si podés explicar qué hiciste, por qué lo hiciste y cómo lo resolviste**.

## 🧩 Escenario

Tu pipeline del TP4 verifica que el código compila y que la imagen se construye… pero no ejecuta un solo test. La semana pasada entró a `main` un PR con un bug en una función que nadie verificaba — el pipeline, verde. En la retro, el equipo acordó tres cosas: la suite tiene que crecer sobre la lógica que importa, la cobertura tiene que ser **visible**, y —la que cambia el juego— tiene que haber un **número que frene el merge** cuando no se llega. Como responsable de calidad, te toca cablearlo.

## 📋 Tareas que debés cumplir

### 1. Suite de unit tests significativa
- **Backend**: mínimo **8 métodos de test** con estructura AAA, repartidos sobre **al menos 4 reglas distintas** de tu app (no 8 datos de la misma). Entre ellos, al menos un `[Theory]`/parametrizado y al menos un caso de error (input inválido, borde). 📌 **El criterio para saber si un test vale**: si cambiás la regla que prueba (invertís un `&&` por `||`, corrés un borde de `>` a `>=`), algún test tiene que ponerse en rojo — el del borde, en el segundo caso: por eso los casos de borde importan. Si no, no está verificando nada.
- **Al menos UN test con mock — obligatorio**, y **cuenta dentro de los 8** del backend (no es un
  noveno aparte). Elegí una pieza de tu app que dependa de algo
  externo (la base, un servicio, un cliente HTTP) y testeala con un doble. **No se acepta
  «mi lógica es pura y no lo necesité»**: mockear es una técnica que hay que tener en las manos.
  Si tu código no tiene por dónde meterlo, ahí está el ejercicio: refactorizá para que la
  dependencia entre desde afuera y contalo en `decisiones.md`. **Cómo se hace, paso a paso, está en §3.0.**
- **Frontend**: mínimo **4 unit tests** (sin DOM), y **las mismas tres técnicas que el backend**:
  al menos un **parametrizado** (`it.each`), al menos un **caso de error**, y al menos un **test
  con mock**. Para el mock, si tu lógica de front habla con la API, pasale un doble al cliente en
  lugar de salir a la red — el ejemplo completo está en §3.0. **No vale «mi front no tiene nada que
  mockear»**: si no lo tiene, abrí el código para que lo tenga, igual que en el backend.
  📌 **¿Tu app tiene un solo Dockerfile, o no tenés frontend separado?** Entonces estos cuatro no
  aplican y las tres técnicas se cumplen en el backend; decilo en `decisiones.md` en una línea. Lo
  que **no** vale es inventar tests vacíos para llegar al número.

> 📌 **Las tres técnicas se piden de los DOS lados.** No es que se cuentan una vez y elegís dónde:
> el parametrizado, el caso de error y el mock van en el backend **y** en el frontend. Son la misma
> idea en dos lenguajes, y el video las muestra en los dos (§3.0).

### 2. Cobertura medida, y un umbral que rompe el build
- Coverage de **backend y frontend** corriendo en el pipeline.
- Reporte **visible en el run** (summary) y **descargable**, **de los dos lados**: backend y frontend.
- **Umbral definido y justificado** en `decisiones.md` — el número, **sobre qué métrica** (línea, rama, o las dos) y por qué ése —, aplicado **en tu pipeline** con **fallo real del build** si no se cumple: `thresholds` de vitest en el front (§3.3), `coverlet.msbuild` en el back (§3.4). Y reportá siempre el de rama, aunque el umbral lo pongas sobre líneas.

### 3. El umbral bloqueando un merge
- Los checks del pipeline —los mismos del TP4— como **required checks** de `main`, ahora capaces de
  ponerse en rojo **por cobertura** y no sólo por compilación (§3.4).
- **Demostración documentada**: un Pull Request con código nuevo sin tests, que **compila perfecto y
  tiene todos los tests en verde**, bloqueado igual porque la cobertura no llega a tu número → los
  tests que faltaban → verde → merge (§3.5).
- 🔴 **Son DOS Pull Requests, y esto es lo que más se equivoca.** El primero lo hacés entero —rojo,
  los tests que faltaban, verde, merge— y su historial cuenta la secuencia. El segundo es chiquito,
  tiene el mismo problema **sin arreglar**, y queda **abierto y en rojo hasta la defensa**. Uno
  cuenta la historia; el otro la prueba: la pantalla de configuración de los required checks sólo la
  ve quien administra el repo, y no prueba que el freno funcione; un Pull Request frenado y visible
  sí, y la corrección lo puede comprobar sola.

> 📌 **¿Tu app tiene un solo Dockerfile, o no tenés frontend separado?** No se descuenta nada:
> los mínimos del frontend (los 4 unit tests con sus tres técnicas, su cobertura y su umbral) no
> aplican, y las tres técnicas se cumplen en el backend. Decilo en
> `decisiones.md` en una línea. Lo que **no** vale es inventar tests vacíos para llegar al número.
## 📄 Entregables

1. **URL del repositorio público** (formulario de la cátedra) con la suite, el workflow extendido y los PRs. **Es lo único que se pega en el formulario**: todo lo demás se llega desde ahí.
   - Y como en todos los prácticos, el cierre: **tag `v5.0.0` y su release**, sobre el commit que
     querés que se corrija. En la defensa se navega ese punto exacto, no lo que quedó después.
2. **`decisiones.md`** (acumulativo) — escribilo **mientras trabajás**, no la noche anterior: el
   porqué se recuerda con la discusión fresca, y reconstruirlo tres semanas después es otro trabajo.
   Explicando:
   - Qué lógica elegiste testear y por qué ESA (¿dónde duele un bug en tu app?).
   - Tu umbral de coverage: el número, **sobre qué métrica** (línea, rama o las dos) y por qué ése — y el número de **rama** que te da hoy, lo hayas usado o no como umbral.
   - **Qué dejaste afuera de la cuenta de cobertura**, backend y frontend, y por qué cada cosa (§2.4): el arranque, las clases de datos, lo generado — o lo que corresponda en tu app.
   - Por qué coverage alto no garantiza calidad (con TU ejemplo).
   - Tu Pull Request bloqueado: qué check se puso en rojo, **en qué métrica** (el log lo dice), por qué, y qué escribiste para arreglarlo.
   - **Si refactorizaste para poder mockear**: qué cambiaste y por qué no se podía testear antes.
   - **Si tu app no tenía qué testear**: qué reglas de negocio le agregaste.
   - **Si tu stack no es el de la cátedra** (.NET + vitest): qué herramienta usaste para cada fila de
     la tabla «Tu stack, de un vistazo» — el parametrizado, el doble, el medidor de cobertura, el
     umbral que frena y el filtro de qué entra en la cuenta. Se pide en §3 y **se cuenta acá**.
   - **El ejercicio del camino sin cubrir** (en la filmina de entregables de la Clase 5 figura
     como *«el ejercicio de la rama sin cubrir»*: es el mismo) — la *rama de código*, no una rama de git: uno de los
     dos caminos que abre un `if` (o un `?.`, o un `??`) y que ningún test recorre. Está en
     §3.0 › *Un test parametrizado y un caso de error*, en el recuadro 🔎, y se hace cuando medís la
     cobertura (§3.1). Van **tres cosas**: qué línea es, qué entrada la recorrería —un valor
     concreto— y qué decidiste hacer. 📌 Las tres respuestas valen, **incluida «no lo agregué»**: lo
     que se evalúa es que abriste el reporte y miraste el código, no el número. Es media pantalla, y
     **no es opcional**: sin él la entrega está incompleta aunque el resto esté perfecto.
   - Problemas encontrados y cómo los resolviste.
   - Declaración de uso de IA.
3. **Los enlaces que prueban cada decisión, pegados a la decisión que prueban.** Van adentro de
   `decisiones.md`, no en un archivo aparte, y **como URL**: una dirección se abre y se comprueba.
   **No se piden capturas** — todo lo que este práctico evalúa tiene una dirección propia.

   > 📌 **«¿URL de qué, si todo está en mi repo?»** De la **corrida** o del **Pull Request
   > concretos**, no del repositorio. Es la diferencia entre entregar algo comprobable y hacer que
   > quien corrige bucee entre cuarenta corridas para encontrar la tuya. Las direcciones que vas a
   > pegar tienen esta forma:
   >
   > | Lo que probás | El link |
   > |---|---|
   > | El resumen de cobertura y su reporte descargable | `…/actions/runs/<id>` — la corrida, no *Actions* |
   > | La corrida **roja por umbral**, con el número en el log | `…/actions/runs/<id>` de ESA corrida |
   > | La secuencia rojo→tests→verde→merge | `…/pull/<n>` del **primer** Pull Request, el **mergeado**: la conversación la muestra entera |
   > | El freno vigente, en rojo | `…/pull/<m>` del **segundo** Pull Request, el que queda **abierto** |
   >
   > 🔴 **Son dos direcciones de Pull Request distintas, y es lo que más se confunde.** El primero ya
   > no está frenado —lo arreglaste y lo mergeaste—: su valor es que cuenta la historia completa. El
   > que se deja **abierto hasta la defensa** es el segundo: en su pantalla se ven los checks
   > obligatorios en rojo y **por qué** (y vos, como dueño, ves además el botón de merge apagado). Es
   > lo que le permite a la corrección comprobar por su cuenta que el freno existe de verdad, sin
   > depender de una captura tuya.

> 📌 **En este TP tampoco hay `evidencias.md`**, igual que en el TP3 y el TP4 y por el mismo motivo:
> tu repositorio es público, así que quien corrige abre *Actions*, el Pull Request y los checks y lo
> ve — sacar capturas de eso es duplicar lo que ya está a la vista. Y esta semana **no hay ningún
> servicio de afuera** que haya que enlazar: todo lo que se evalúa vive en tu repositorio.

## 🗣️ Defensa Oral Obligatoria

Se realiza en **P2**, junto con los TPs 6 a 9. Vas a mostrar tu trabajo y responder preguntas como:
- Dibujá la pirámide de testing. ¿Por qué esa forma? ¿Qué pasa si se invierte? ¿Dónde están parados tus tests hoy y qué falta?
- Mostrame un test tuyo y señalá el Arrange, el Act y el Assert. ¿Por qué el nombre del test importa?
- ¿Qué diferencia hay entre un mock y un stub? Mostrame **tu** test con mock: ¿qué dependencia reemplazaste y qué verifica el assert? Si tuviste que refactorizar para poder inyectarla, contame qué cambiaste.
- Line coverage vs branch coverage: ¿cuál puede mentirte más y por qué?
- Tu coverage es del X%: ¿eso prueba que tu código funciona? Escribime (conceptualmente) un test que sume coverage sin verificar nada.
- ¿Por qué elegiste TU umbral? ¿Qué pasaría si mañana lo subís diez puntos?
- ¿Qué es un quality gate? ¿Cuál es la condición del tuyo, y qué pasa si alguien la baja?
- Mostrame el Pull Request que quedó bloqueado. **Compilaba bien y los tests pasaban todos: ¿por qué no se podía mergear?**
- ¿Qué tipo de problemas encontraría un análisis estático que tus tests no ven? ¿Y al revés?
- Tu cobertura dice 80%. ¿Eso quiere decir que el 80% de tu código está verificado? ¿Cómo lo
  comprobarías?
- Dos de tus tres guardianes de `main` son automáticos. ¿Qué cosas de un cambio NINGUNO de ellos puede detectar? 🤖 Y si me contestás «lo que sólo ve un humano»: ¿qué de eso **sí** podría revisarte hoy un agente de IA, y qué queda afuera aunque lo uses?

## ✅ Evaluación

| Criterio | Peso |
|---|---|
| Configuración técnica (suite, coverage, umbral que frena, evidencias reproducibles) | 25% |
| Claridad y justificación en `decisiones.md` (cada decisión con su prueba al lado) | 25% |
| Defensa oral: comprensión y argumentación | 50% |

> ⚖️ Peso orientativo de este TP en la nota de **P2**: **25%** (la ponderación completa de los 9 TPs está en el reglamento, §5).

> 🎯 **Se corrige el logro, no la herramienta.** Los ejemplos están escritos sobre la app de la
> cátedra (.NET + vitest) porque un ejemplo tiene que estar escrito en *algo*; la tabla «Tu stack,
> de un vistazo», arriba de §3.0, traduce **cada una** de las cosas que se piden. Contá en `decisiones.md`
> cuáles usaste.

**Cómo se distingue un 4 de un 8.** Los mínimos de las tareas te dan el 4: están o no están. Lo que
mueve la nota de ahí para arriba es el criterio, y se ve en estas tres cosas:

| | **Suficiente (4-5)** | **Bien (6-7)** | **Muy bien (8-10)** |
|---|---|---|---|
| **La suite** | Llega a los números. Los tests prueban lo que era fácil de probar | Cubre las reglas que importan, con sus casos de borde | Cada test verifica un comportamiento y se entiende sin leer el código. Invertir cualquier regla pone algo en rojo |
| **El umbral** | Hay un número y rompe el build | El número está justificado contra tu medición real, no copiado | Además explica qué haría falta para subirlo, y qué mide de verdad (línea vs. rama) |
| **El gate** | Los checks están como required y hay un Pull Request frenado | El Pull Request está frenado **por cobertura**, no por compilación, y la secuencia completa está documentada | Además explica por qué ese freno es distinto del del TP4, y qué clase de error deja pasar igual |
| **La defensa** (50%) | Contesta lo conceptual: qué es cobertura, qué es un mock | Muestra **su** repo y explica **sus** decisiones cuando se le pregunta | Sostiene la repregunta: por qué ESE número, qué mide y qué no, y qué haría distinto la próxima vez |

📌 **Cómo se usa esta tabla**: la columna «Suficiente» es lo que ya piden las tareas — con eso se
aprueba. Las otras dos no piden **más trabajo**, piden **más criterio** sobre el mismo trabajo, y
casi todo se ve en `decisiones.md` y en la defensa. Un 70% que sabés defender vale más que un 95%
que no.

## ⚠️ Uso de IA

Podés usar IA (ChatGPT, Copilot, Claude), pero **deberás declarar en `decisiones.md` qué parte fue asistida por IA** y justificar cómo la verificaste. En este TP en particular: si la IA te escribió tests, tenés que poder explicar **qué verifica cada assert y qué caso NO está cubierto**. Si no podés defenderlo, **no se aprueba**.
