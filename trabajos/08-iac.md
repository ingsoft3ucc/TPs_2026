# ⚠️ IMPORTANTE – Guía de Práctica Sugerida

Este documento tiene **dos partes**:

1. **Guía de práctica sugerida** (primera parte): paso a paso para aprender haciendo. **NO es lo que se entrega.**
2. **El Trabajo Práctico entregable** (al final): escenario, tareas, entregables y defensa oral. **Eso es lo que se evalúa.**

## 📦 Un solo repositorio para todo el semestre

**Todos los prácticos de la materia se hacen sobre el mismo repositorio: el que creaste en el TP1.** No se crea uno nuevo por práctico, y no se arranca de cero en cada uno — cada TP agrega una capa sobre lo que ya está.

Cada TP se cierra con su **tag y su release**, y el número mayor es el número del práctico: **TP1 → `v1.0.0`, TP2 → `v2.0.0`, TP3 → `v3.0.0`**, y así hasta el TP9. Así cada entrega queda con su **estado congelado**: en la defensa se navega el punto exacto en el que cerraste cada una, y podés volver a cualquiera con `git checkout v2.0.0`.

El archivo `decisiones.md` también es **único**: no se rehace por práctico — se le agrega abajo la sección del TP nuevo.

## Sobre las herramientas en este TP

- La guía usa **Terraform** contra **los mismos proveedores que venís usando desde el TP6**: el
  provider de **Neon** (bases de datos) y el de **Render** (servicios). Sin cuentas nuevas y sin
  tarjeta — las dos cuentas ya son tuyas.
- 🔴 **Y lo declarado es un entorno NUEVO: `preprod`.** Este práctico **no toca** tu QA ni tu
  producción: los dos quedan exactamente como están, con sus URLs y su cadena del TP7 intacta. Lo
  que vas a escribir es un tercer entorno, entre QA y producción, que **no existe hasta que tu
  código lo crea** — y que desaparece cuando ya no lo necesitás. El porqué de esta elección está en
  §2.6.
- El **estado** de Terraform vive **en tu máquina**: es el comportamiento por defecto y no hay nada
  que configurar. Es una simplificación deliberada de este práctico — el §2.3 explica qué es el
  estado, qué pasa si lo perdés, y qué cambia el día que tiene que compartirlo un equipo.
- **Plan C garantizado (§3.7)**: **Terraform con el provider Docker**, 100 % local, sobre tu
  compose. Ahí `preprod` es un proyecto de Compose más. Cumple todos los checkpoints y es la entrega
  recomendada para el que ya venía corriendo sus entornos en compose desde el TP6 porque el free
  tier le falló.
- Dato de contexto 2026: Terraform cambió su licencia a **BSL** en 2023 (gratis para usar, restrictiva para competidores); de ahí nació **OpenTofu**, el fork open source (MPL-2.0) compatible — si preferís `tofu` en lugar de `terraform`, todos los comandos de esta guía funcionan igual.
- Riel **Azure**: **Bicep** + `what-if` contra tu subscription (*Azure for Students* si la tenés) — reusa el material AZ-400 de la cátedra. Tabla de equivalencias en §4.

> 🔴 **Los dos providers se fijan por versión, y hay un motivo concreto.** El de Neon que hoy
> funciona es `kislerdm/neon` **0.18.0**; su repositorio está **archivado** y el equipo de Neon
> anunció que va a publicar el oficial bajo **otro nombre de fuente** (`neondatabase/neon`), que al
> 21-09-2026 todavía no existe en el registro. El de Render es `render-oss/render` **1.9.1**. Si no
> fijás la versión, un día `terraform init` te baja otra cosa y el archivo que ayer andaba deja de
> compilar. Fijarla es la primera regla de higiene de IaC — y migrar de un provider a otro cuando
> salga el oficial es, justamente, un ejercicio de IaC.
>
> 🔧 **Y si algún día `terraform init` no lo encuentra, no te quedes trabado**: el registro no
> despublica versiones ya subidas, y el `.terraform.lock.hcl` que commiteás fija el hash exacto —
> así que lo más probable es que siga bajando. Pero si no baja, **avisá a la cátedra y pasá al plan
> C (§3.7)**, que es 100 % local, no depende de ningún registro de terceros y cumple todos los
> checkpoints. No hay nota perdida por un provider de otro.

📌 Este TP trabaja **sobre la infraestructura de tu app del semestre**: la misma nube, la misma imagen del TP7, la misma forma de entorno que armaste clickeando en el TP6 — pero esta vez escrita, versionada, y creada con un comando.

🧪 **Los ejemplos de esta guía están escritos sobre la app de la cátedra** ([`ingsoft3ucc/demo-fullstack`](https://github.com/ingsoft3ucc/demo-fullstack) — .NET 8 + React/Vite + PostgreSQL, la misma de las demos en clase; cómo clonarla y levantarla, base de datos incluida: **TP2 §3.2**). Si un paso no te sale en tu app, probalo primero ahí —donde sabés que funciona— y después llevalo a la tuya. Lo que entregás, siempre, es **tu** app.

---

# Guía Paso a Paso – Infraestructura como Código (Práctica sugerida)

## 1- Objetivos de Aprendizaje

- Entender **declarativo vs imperativo** e **idempotencia** — los cimientos conceptuales de IaC.
- **Crear un entorno entero escribiéndolo**: la base en Neon y los servicios en Render de un
  `preprod` que no existía, con la imagen del TP7 adentro.
- Dominar el ciclo **`plan` → `apply` → `destroy` → `apply`** y el rol del **estado**: qué es, qué
  guarda, qué pasa si lo perdés, y por qué jamás se commitea.
- Provocar y detectar **drift** (la infra real que se separó del código) — y saber **qué no detecta**.
- *(Opcional, §3.6)* Integrar IaC **al pipeline**: que el `plan` corra y se publique en cada PR que
  toca `infra/`, para que un cambio de infraestructura se revise como se revisa el código.
- Leer la **letra chica del free tier** y convertirla en una decisión: un entorno que existe sólo
  cuando se lo necesita.

## 2- Marco teórico

### 2.1 El problema: la infraestructura artesanal

Hacé memoria del TP6: creaste cuatro servicios en Render **clickeando** — nombre, runtime, instance type, variables de entorno, Auto-Deploy off — y dos bases en Neon del mismo modo. ¿Podrías recrear todo eso, idéntico, mañana? ¿Y tu compañero, sin mirar tu pantalla? ¿Y en seis meses, cuando nadie recuerde qué se clickeó?

Eso es infraestructura **artesanal**, y la industria le puso nombre: *snowflake servers* (servidores únicos e irrepetibles como copos de nieve), configuración que vive "en la cabeza del que la hizo", entornos que nadie se anima a tocar porque nadie sabe reconstruirlos. Sus síntomas son conocidos: "en QA anda y en PROD no" (los entornos divergieron a fuerza de clicks), el pánico a recrear algo, y la imposibilidad de revisar un cambio de infraestructura antes de que ocurra.

**Infraestructura como Código (IaC)** hace con la infraestructura lo mismo que ya hiciste con todo lo demás: describirla en **archivos versionados** que una herramienta materializa. Es la **quinta aparición del patrón de la materia** — y la anunciamos en la clase 1: protecciones (TP1), entorno de ejecución (TP2), plan de trabajo (TP3), proceso de build (TP4)… y ahora la infraestructura entera. Consecuencias inmediatas: la infra se **revisa en PRs**, tiene **historial** (quién cambió qué y cuándo), se **recrea idéntica** cuantas veces haga falta, y deja de vivir en la cabeza de nadie.

> 📌 **Y el contraste vas a tenerlo en la misma cuenta.** Al
> terminar este TP vas a tener, en el mismo dashboard de Render, entornos que existen porque alguien
> los clickeó y un entorno que existe porque está escrito. Uno lo podés recrear con un comando; los
> otros, no. Esa comparación se pregunta en la defensa.

### 2.2 Declarativo vs imperativo — e idempotencia

Dos formas de decirle a una máquina qué querés:

- **Imperativo**: el CÓMO, paso a paso — "creá un proyecto; después una base; después un servicio conectado a esa base…". Es un script. Si lo corrés dos veces, la segunda **falla** ("ya existe") o duplica. Si algo quedó a medias, el script no sabe desde dónde retomar.
- **Declarativo**: el QUÉ — "debe existir un proyecto así, una base así, un servicio así". La herramienta compara lo declarado con lo real y calcula **qué hacer** para converger: crear lo que falta, corregir lo que difiere, no tocar lo que ya está bien.

De lo declarativo se desprende una propiedad: **idempotencia** — aplicar la misma configuración N veces produce el mismo resultado que aplicarla una. La segunda corrida de un `apply` sin cambios dice "No changes" y no toca nada. ¿Por qué importa tanto? Porque convierte a tu configuración en algo **seguro de re-aplicar**: ante la duda, aplicás de nuevo — sin miedo a romper ni duplicar. Y porque es la condición de que un pipeline pueda aplicar solo: un `apply` automático en cada merge sólo es viable si el 99 % de las veces no hace nada.

(¿Te suena la idea? Tu `docker compose up` del TP2 ya era declarativo para *un* stack de contenedores; Terraform generaliza el patrón a cualquier infraestructura y le agrega dos piezas. El **estado** —compose no lo necesita: sus recursos viven en tu máquina y llevan su nombre— y el **plan**. Y con el plan, la precisión: compose tiene un ensayo desde 2023 (`docker compose --dry-run up`), pero lo que simula son las **acciones** que ejecutaría; no compara contra una memoria de lo que creó antes. Esa comparación es lo que Terraform agrega.)

### 2.3 El estado: la memoria de la herramienta

Para calcular "qué cambiar", la herramienta necesita recordar **qué creó**. Terraform lo guarda en el **estado**: un JSON que mapea cada recurso declarado con el recurso real (IDs, atributos, dependencias). Cuatro verdades sobre el estado que van directo a tu defensa:

1. **Es la fuente de correlación, no de verdad**: la verdad de lo que *querés* es el código; la
   verdad de lo que *hay* es la infraestructura real; el estado es el puente que permite compararlas.
2. **JAMÁS se commitea**: contiene TODO lo que la herramienta sabe — incluidos atributos sensibles
   (passwords de base, connection strings) **en texto plano**. En este TP lo vas a ver con tus ojos
   (§3.2).
3. **Si lo perdés, no perdés la infraestructura: perdés la prueba de que es tuya.** Terraform deja de
   saber que ese proyecto de Neon lo creó él, y el próximo `plan` propone **crear todo de nuevo**,
   contra recursos que ya existen. El camino de vuelta es traerlos a la memoria uno por uno,
   **importándolos**. Por eso el archivo de estado no es un temporal: importa tanto como el código,
   aunque no se versione.
4. **En equipo se comparte por otro lado**: los *remote backends* —un almacenamiento compartido, con
   **bloqueo**— existen para que dos personas (o una persona y un pipeline) no apliquen a la vez y se
   pisen. En este práctico no vas a montar uno; la parte opcional (§3.6) muestra, en tu
   propio repositorio, qué te perdés por eso.

> 📌 **Acá tu estado vive en tu máquina, y es una decisión del práctico.** Sin un bloque `backend`,
> Terraform escribe `terraform.tfstate` al lado de tu `main.tf`: cero configuración, y alcanza
> perfectamente, porque **el que aplica sos vos**. Lo que tenés que poder explicar es la consecuencia:
> ese archivo es la única memoria de tu preprod, no se commitea (tiene secretos adentro), y el día que
> tuviera que aplicar alguien más —otra persona, u otra máquina— esa memoria tendría que mudarse a un
> lugar compartido. Ésa es la conversación.

### 2.4 `plan`: el "what-if" — la infraestructura entra por PR

El comando más importante de IaC no es el que aplica: es el que **muestra qué VA a pasar sin hacerlo**. `terraform plan` (en Bicep: `what-if`) compara código ↔ estado ↔ realidad y lista: qué se crea (`+`), qué cambia (`~`), qué se destruye (`-`).

Leelo como lo que es: **el diff de la infraestructura**. Y con eso, el cambio cultural completo: un cambio de infra se propone en una rama, el plan se ejecuta en el pipeline y se revisa como se revisa un PR — *"este cambio destruye y recrea la base de datos"* es algo que querés leer ANTES, en un plan, y no descubrir DESPUÉS, en un incidente. La regla profesional: **nunca apliques lo que no planeaste** — y desconfiá de todo plan que destruya algo que no esperabas destruir.

Dos cosas del plan que desconciertan la primera vez, y que vas a ver en el tuyo:

- **`(known after apply)`**: el valor todavía no existe porque lo asigna el proveedor. La URL de un
  servicio de Render o el id de un proyecto de Neon se conocen recién cuando se crean. No es un
  error ni una incógnita peligrosa: es el plan siendo honesto sobre lo que no puede saber.
- **`# forces replacement`**: ese atributo **no se puede cambiar en caliente**; para cambiarlo, el
  recurso se destruye y se crea de nuevo. Cuando aparece al lado de algo que tiene datos adentro,
  frenás y leés el plan tres veces. Es la línea más importante que un plan puede tener.

Y un uso del plan que casi nadie aprovecha: **el plan es una prueba gratis**. Podés escribir infraestructura, preguntarle a Terraform qué haría, leer la respuesta, y borrar lo que escribiste — sin crear nada y sin gastar un centavo.

### 2.5 Drift: cuando la realidad se separa del código

**Drift** (literalmente, *deriva*) es la divergencia entre lo declarado y lo real, y aparece por la puerta de siempre: alguien tocó la infra **a mano** — "un cambito de emergencia" en la consola, una base borrada, una variable editada al vuelo. El código dice una cosa; la realidad, otra. La documentación acaba de volverse mentira.

La detección es gratis con lo que ya tenés: el próximo `plan` **refresca** el estado contra la realidad y muestra la diferencia — "esto que declarás no está / cambió: propongo repararlo". Aplicás, y la realidad **converge de nuevo al código**. Dos lecturas profesionales: (1) el drift no se "prohíbe" — se **detecta** (un plan periódico en el pipeline es un detector de drift automático); (2) la reparación tiene dirección: la realidad se corrige hacia el código — si el cambio manual era legítimo, el camino correcto es **traerlo al código** por PR, no dejarlo huérfano en la consola.

> 🔴 **Y ahora lo que NO detecta, que es igual de importante y se malentiende siempre.** Terraform
> vigila **lo que declaraste**, no todo lo que existe en la plataforma. Medido por la cátedra el
> 21-09: se borró a mano una base que Terraform administraba y el plan siguiente la detectó
> (`# neon_database.preprod will be created` · `Plan: 1 to add`); se creó a mano **otra** base que
> Terraform no declara, y el plan siguió diciendo **«No changes»**. Traducción: tu plan limpio
> **no** quiere decir "no hay nada raro en mi cuenta". Quiere decir "lo que yo gobierno está como
> pedí". Todo lo que alguien cree por fuera es invisible para vos, y sigue gastando cuota — y en
> este TP la cuota importa (§2.6).
>
> 📌 **Y hay un caso enorme de esto en tu propia cuenta: tu QA y tu producción.** Existen, están
> vivos, y tu `terraform plan` no dice una palabra sobre ellos, porque no los declaraste. Ésa es la
> mejor demostración posible de esta idea, y la tenés gratis.
>
> 📌 Nota de método, porque también se aprende de esto: la primera vuelta de esa prueba dio un falso
> «no detecta drift», porque se había borrado una base que ya no estaba en el estado. Antes de
> concluir hubo que comparar **lo que existe** contra **lo que Terraform cree que existe**. La
> conclusión apurada habría sido la contraria.

### 2.6 El entorno que falta: `preprod`

Acá está la decisión de diseño de este práctico, y conviene entenderla antes de escribir una línea.

**Este TP no lleva a código la infraestructura que ya tenés. Crea una que no existe.** Tu QA y tu producción quedan intactos —mismas URLs, misma cadena del TP7, cero riesgo a una semana de la defensa— y vos escribís un **tercer entorno, `preprod`**, que se mete entre los dos:

```
CI verde → QA (automático) → preprod → [aprobación] → producción
```

> 📌 **Ese dibujo es dónde va un preprod en un equipo. En este práctico no lo conectás a tu
> pipeline**: lo creás y lo probás a mano, desde tu terminal. Conectarlo es la extensión opcional
> del final de §3.4.

Hay cuatro razones, y las cuatro son argumentos que vas a tener que poder dar:

1. **Es el argumento real de IaC, y el único que no se puede enseñar de otra forma.** El valor de la
   infraestructura como código no es «gobernar lo que ya está»: es que cuando alguien dice *"necesito
   un entorno para probar esto"*, la respuesta sea **un archivo y un `apply`** en vez de una tarde de
   clicks. Gobernando algo preexistente nunca vivís ese momento. Creándolo, sí.
2. **Preprod es un patrón de industria de verdad.** Un entorno lo más parecido posible a producción,
   donde se prueba la release antes de promoverla: *staging*, *pre-producción*, *UAT*. Existe
   justamente porque QA suele estar sucio y compartido, y producción no se usa para probar.
3. **Render sólo tiene que CREAR** — que es lo único que el plan gratuito permite hacer bien (§3).
   Con un entorno nuevo, el defecto de la actualización no se toca nunca.
4. **Y el ciclo de vida deja de ser un ejercicio y pasa a ser el uso normal**: preprod existe cuando
   lo necesitás y se va cuando no. Lo cual nos lleva a la letra chica.

> 🔴 **La letra chica del free tier, y esta vez tiene consecuencia sobre tu PRODUCCIÓN.** Las **750
> horas de instancia por mes** del plan gratuito de Render son **del workspace y compartidas entre
> todos tus servicios gratuitos** (fuente: `render.com/docs/free`) — no son 750 por servicio. Con
> este TP pasás de **4 servicios a 6**.
>
> La cuenta, con números: un servicio despierto **todo el mes** se come 720 horas él solo si el mes
> tiene 30 días, y **744** si tiene 31 — o sea que un solo servicio prendido todo el mes te deja
> **seis horas** de margen para los otros cinco. No te va a pasar
> —los servicios gratuitos **se apagan solos a los 15 minutos sin tráfico** y sólo consumen mientras
> corren—, así que **con uso de cursada entra**. Pero si te pasás, **Render suspende TODOS tus
> servicios gratuitos hasta el mes siguiente, producción incluida.** No suspende el que se excedió:
> suspende el workspace. Y tus URLs tienen que seguir vivas hasta la defensa.
>
> **La salida es exactamente lo que este TP enseña**: `terraform destroy` de preprod cuando no lo
> usás, `terraform apply` cuando lo volvés a necesitar. Un entorno efímero
> es la forma de que entren seis servicios en la cuota de cuatro. Mirá el consumo en *Workspace → Billing*.

📌 **Dos palabras de vocabulario, porque el examen las nombra y este práctico no las usa.** Para
infraestructura que **ya existe** y querés pasar a código, el camino es **importarla** — desde
Terraform 1.5 con bloques `import`, declarativos y revisables en un PR. Y el día que tengas tres
entornos y estés copiando el mismo bloque tres veces, lo que vas a querer es un **módulo**: un molde
de "qué es un entorno", que se instancia con parámetros. Ninguna de las dos se pide acá; saber que
existen es parte del tema.

## 3- Desarrollo de la guía (riel Terraform + Neon + Render)

> Trabajás en tu repo del semestre, carpeta nueva `infra/`, sobre las dos cuentas que ya tenés desde
> el TP6. Al terminar, existe un entorno `preprod` completo —su base de datos y sus dos servicios
> corriendo la imagen del TP7— que **no existía**, que nació de un `apply` tuyo, y que podés hacer desaparecer y volver a traer con un comando.
>
> 🟢 **Lo que este práctico NO toca: tu QA y tu producción.** No se migran, no se importan, no se
> borran, no se les cambia nada. Si en algún momento te encontrás editando un servicio existente,
> pará: te fuiste de la consigna.

### 🔴 Lo que la cátedra midió, y lo que no

Antes de escribir esta guía se corrió Terraform v1.15.8 contra las cuentas reales de la cátedra, con los providers **fijados al 21-09-2026 — la última versión publicada de cada uno** (`kislerdm/neon` 0.18.0 y `render-oss/render` 1.9.1). Anotá la procedencia, porque es la clase de distinción que este TP evalúa: **las versiones salen del registro; la corrida, de las cuentas.** **Leé esta tabla antes de empezar**: explica por qué el TP está diseñado como está.

| Operación | Neon (`kislerdm/neon` 0.18.0) | Render (`render-oss/render` 1.9.1) |
|---|---|---|
| **Crear** | ✅ medido (4 s) | ✅ medido (6 s, plan `free`) |
| **Idempotencia** (2do `plan`) | ✅ «No changes» | ✅ «No changes» |
| **Modificar** | ✅ funciona | ❌ **IMPOSIBLE en plan gratuito** |
| **Destruir** | ✅ instantáneo | ✅ instantáneo |
| **Drift** (romper a mano → el plan lo detecta) | ✅ medido | ⚠️ no medido |

> 🔴 **Render no deja actualizar un servicio del plan gratuito.** Cualquier `apply` que modifique un
> servicio `free` muere con:
>
> ```
> Could not update service, unexpected error: could not update service:
> maintenance mode can only be configured for non-free tier services
> ```
>
> El provider manda el campo `maintenance mode` en **toda** actualización y el plan gratuito lo
> rechaza. Se probó el rodeo de declararlo explícitamente y **falla igual**: no es que falte
> declararlo, es que el provider lo manda siempre. Es un defecto conocido y abierto del provider, no
> un error tuyo.
>
> **Consecuencia, y es el diseño del práctico: el ciclo de enseñanza —cambiás el código, leés el
> plan, aplicás, provocás drift, reparás— se da sobre NEON, que lo soporta entero. En Render se CREA
> y se DESTRUYE, nunca se modifica.** Ninguna sección de esta guía te va a pedir que cambies un
> servicio de Render ya creado. Si necesitás que un servicio de preprod quede distinto, el camino es
> el que el TP enseña igual: `terraform destroy` y `terraform apply` — las dos operaciones medidas —,
> o `terraform apply -replace=<dirección del recurso>`. 🔴 **Y ojo con el matiz:
> Terraform NO deduce esa recreación.** Frente a un atributo cambiado planea un `~ update in place`,
> ese update falla con el error de arriba, y el `apply` se corta. La recreación **la pedís vos**
> (§3.4 nota 4). Es la ventaja de que preprod sea **desechable**.

**Y lo que NO está medido** — no significa "no funciona", significa "si te pasa algo raro acá, no es tu culpa, anotalo en `decisiones.md`":

- Si al **recrear** un servicio de Render con el mismo nombre vuelve la **misma URL** o Render le
  agrega un sufijo (el TP6 avisa que lo agrega si el nombre está tomado). Para preprod no es grave
  —es un entorno desechable y su URL sale de un `output`—, pero conviene que lo sepas antes de
  destruirlo la primera vez.
- Los **topes reales del free tier de una cuenta**: si crear/destruir repetido choca contra algún
  límite diario, y hasta dónde llegan de verdad los cupos publicados. Cada alumno usa su propia
  cuenta, así que **no comparten cupo** — lo que importa es el techo de uno solo. Lo que **sí** está
  **documentado por el proveedor** (documentado, no medido por nosotros) son los 100 proyectos de
  Neon del §3.2 y las 750 horas compartidas de Render del §2.6.
- `prevent_destroy` (§3.5) es del **lenguaje** de Terraform, no del provider; su comportamiento es
  el documentado y la cátedra no lo midió en esta corrida.

### 🔧 Tu stack y tu nube, de un vistazo

🔧 **Esta guía está escrita sobre Neon + Render, que es el riel de la materia desde el TP6. Si
usaste otro proveedor, buscá tu fila: lo que se evalúa es que LOGRES cada cosa.**

| Lo que tenés que lograr | Los ejemplos de la guía | Cómo se llama en otro lado |
|---|---|---|
| **Declarar una base gestionada** | `neon_project` + `neon_database` + `neon_role` | providers de Supabase, AWS (`aws_db_instance`), GCP (`google_sql_database`), Azure (`azurerm_postgresql_flexible_server`) |
| **Declarar el servicio que corre tu imagen** | `render_web_service` con `runtime_source.image` | `fly_app` · `google_cloud_run_v2_service` · `azurerm_linux_web_app` · `kubernetes_deployment` |
| **El estado, fuera del repositorio** | `terraform.tfstate` en tu máquina, cubierto por el `.gitignore` | cuando hay que compartirlo: S3 + DynamoDB · Azure Storage · GCS · el estado de GitLab · HCP Terraform |
| **La credencial del provider, fuera del código** | `NEON_API_KEY`, `RENDER_API_KEY`, `RENDER_OWNER_ID` como variables de entorno | igual en todos: el provider lee una variable de entorno, nunca una clave escrita en HCL |
| **Pasarle la conexión a tu app** | `ConnectionStrings__Default` en `env_vars` | `DATABASE_URL` · `SPRING_DATASOURCE_URL` · `DJANGO_DATABASE_URL` — la que tu framework lea del ambiente (TP2 §3.2) |
| **Que el `plan` se lea en el PR** | `terraform plan` + `actions/upload-artifact` | artefactos de GitLab CI · *publish artifacts* de Azure Pipelines |

📌 **¿Tu stack no está?** El criterio es el mismo y averiguarlo es parte del trabajo: las seis filas existen en todos lados. Contá en `decisiones.md` qué usaste para cada una.

> 📬 **Cómo repartirlo.** El práctico tiene **dos mitades que se pueden hacer en días distintos**.
> La primera (§3.1–§3.3) es **todo Neon**: el ciclo completo, la idempotencia, el estado por dentro y
> el drift — es donde está el aprendizaje, y no cuesta un minuto de cuota de Render. La segunda
> (§3.4–§3.5) crea los servicios de preprod y hace el ciclo de vida.
>
> 🔴 **Y una regla operativa para todo el práctico: cuando termines una sesión de trabajo, corré
> `terraform destroy`.** No es ceremonia: es el §2.6 en acción, y es la diferencia entre entrar en
> las 750 horas y que Render te suspenda producción. Al día siguiente, `terraform apply` te lo
> devuelve idéntico en segundos — que es, exactamente, lo que este TP viene a demostrar.

### 3.1 Terraform, las dos llaves, y la carpeta `infra/`

Instalá Terraform — ojo con el detalle 2026: **homebrew-core removió la fórmula `terraform` tras el cambio de licencia** (el de 2023 que cuenta el encabezado — frenándote en el minuto uno):

- macOS: `brew tap hashicorp/tap && brew install hashicorp/tap/terraform` (el tap oficial de HashiCorp)
- Windows: `choco install terraform` (si falla, el zip oficial de developer.hashicorp.com)
- Linux: el repo apt/yum de HashiCorp
- ¿Preferís OpenTofu? `brew install opentofu` (SÍ está en core) y usá `tofu` donde diga `terraform`.
  ⚠️ **No medido**: la corrida de la cátedra fue sólo con Terraform. El riesgo no es el lenguaje sino
  los **providers** — antes de arrancar, confirmá que `kislerdm/neon` y `render-oss/render` estén
  publicados en el registro de OpenTofu, y contalo en `decisiones.md`.

Verificá con `terraform -version`. 📌 **La cátedra midió con 1.15.8; te sirve ésa o cualquiera posterior** — hoy `brew` te va a bajar una más nueva, y no hay incompatibilidad conocida con lo de esta guía.

**Las dos llaves.** Los providers no llevan la credencial escrita en el HCL: la leen del entorno. Eso es deliberado.

| Provider | Qué necesita | De dónde sale |
|---|---|---|
| Neon | `NEON_API_KEY` | consola de Neon → *Account settings → API keys* |
| Render | `RENDER_API_KEY` **y** `RENDER_OWNER_ID` | la clave, en *Settings* del dashboard; el owner id está **en la URL del dashboard** y empieza con `usr-` o `tea-` |

```bash
export NEON_API_KEY="neon_api_key_…"
export RENDER_API_KEY="rnd_…"
export RENDER_OWNER_ID="usr-…"        # o tea-…
```

> 🔴 **El `RENDER_OWNER_ID` es el dato que todos se olvidan**, y el error que tira no habla del
> recurso sino de la configuración del provider: te va a parecer que está mal el `render_web_service`
> y está mal la línea de arriba.
>
> ⚠️ **Y esos tres valores no se escriben en ningún archivo del repo.** Si las exportás en tu terminal
> se van cuando la cerrás (volvé a correr el `export`); si preferís un archivo local, que esté en el
> `.gitignore` **antes** de escribirlo. En el pipeline viajan como secrets (§3.6).

**Higiene inmediata** — en tu `.gitignore`, antes de correr nada:

```
*.tfstate
*.tfstate.*
.terraform/
*.tfvars        # ahí van los valores locales
```

Fijate qué es lo primero de esa lista: **el estado**. En este TP vive acá, en tu máquina (§2.3), así
que ese `.gitignore` es lo único que lo separa de tu repositorio público — y adentro hay contraseñas
(lo vas a ver en §3.2).

Y la excepción que confunde a todos la primera vez: el **`.terraform.lock.hcl`** que va a generar `init` **SÍ se commitea** — fija la versión y los hashes exactos de cada provider para que vos y el CI usen lo mismo. Es la recomendación oficial de HashiCorp, y es la otra mitad del `version =` que vas a escribir en un minuto.

**✅ Checkpoint:** `terraform -version` responde, las tres variables de entorno están exportadas, y el `.gitignore` cubre estado y tfvars **antes** de cualquier apply.

### 3.2 La base de preprod: el ciclo `init → plan → apply`, la idempotencia y el estado

Creá `infra/main.tf`. Lo primero que el archivo tiene que decir es **quién habla con quién**:

```hcl
terraform {
  required_version = ">= 1.5"

  required_providers {
    neon = {
      source  = "kislerdm/neon"
      version = "0.18.0"        # fijada: ver la advertencia del encabezado
    }
    render = {
      source  = "render-oss/render"
      version = "1.9.1"
    }
  }
}

provider "neon" {}              # lee NEON_API_KEY
provider "render" {}            # lee RENDER_API_KEY y RENDER_OWNER_ID
```

> 🔧 **Ese `source` es el punto frágil de esta guía, y conviene saberlo antes de que pase.** El
> provider de Neon que usamos lo mantiene una persona, no la empresa, y su repositorio está
> archivado: el equipo de Neon anunció el oficial bajo **otro** nombre (`neondatabase/neon`) que al
> cierre de esta guía todavía no está publicado. Lo que ya está subido al registro no se
> despublica, y tu `.terraform.lock.hcl` fija el hash — así que esto debería seguir funcionando. Si
> igual `init` te dice que no encuentra el provider: **avisá a la cátedra y pasá al plan C (§3.7)**.
> Y el día que salga el oficial, cambiar el `source` y correr `terraform init -upgrade` es
> exactamente el ejercicio que enseña este práctico.

> 📌 **No hay ningún bloque `backend`, y eso también es una decisión.** Sin él, Terraform guarda el
> estado en `infra/terraform.tfstate`, al lado de este archivo: cero configuración, y alcanza porque
> el que aplica sos vos (§2.3).

Y ahora el primer recurso de verdad: la base de datos de tu entorno nuevo. Agregá a `infra/main.tf`:

```hcl
# --- El proyecto de Neon de PREPROD: no existe hasta que esto se aplica ---

resource "neon_project" "preprod" {
  name = "miapp-preprod"

  # 🔴 SIN esta línea tu PRIMER apply falla. El provider pide por defecto 86400 segundos
  #    (un día) de historial y el tope del plan gratuito es 21600 (seis horas). El error
  #    —"requested history retention seconds exceeds allowed maximum"— no dice qué hacer.
  history_retention_seconds = 21600
}

resource "neon_role" "preprod" {
  project_id = neon_project.preprod.id
  branch_id  = neon_project.preprod.default_branch_id   # 🔴 default_branch_id, NO branch_id
  name       = "app_preprod_owner"
}

resource "neon_database" "preprod" {
  project_id = neon_project.preprod.id
  branch_id  = neon_project.preprod.default_branch_id
  name       = "app_preprod"
  owner_name = neon_role.preprod.name
}
```

Y `infra/outputs.tf`:

```hcl
output "db_host" {
  value = neon_project.preprod.database_host
}

output "database" {
  value = neon_database.preprod.name
}
```

> 🔴 **`default_branch_id`, no `branch_id`.** El recurso `neon_project` expone el id de su rama por
> defecto con **ese** nombre. Un ejemplo copiado de documentación vieja usa `branch_id` y **no
> compila**: `Unsupported attribute`. Medido. (Y es un buen recordatorio de por qué el `version =`
> que fijaste arriba importa: los nombres de atributos cambian entre versiones de un provider.)
>
> 📌 **Un proyecto propio para preprod, y no una base más en el proyecto de tu app.** Dos motivos:
> el de la consigna —el proyecto de tu app lo creaste a mano y este TP no lo toca— y uno práctico,
> que es el que importa: un proyecto aparte se puede **destruir entero** sin rozar nada de QA ni de
> producción. La documentación de Neon dice que el plan gratuito permite hasta 100 proyectos; vos vas
> a usar tres, y ese techo la cátedra **no lo probó** — *documentado, no medido*, que es la distinción
> que esta guía usa todo el tiempo.
>
> 📌 **La cadena de conexión no se saca por `output`.** Un output se imprime en la consola y en el
> log de la corrida; la contraseña del rol es un atributo **sensible** y sale del proveedor, no de
> un tfvars tuyo. Lo que sí sale por output son el host y los nombres, que no son secretos. En §3.4
> vas a ver cómo la cadena llega al servicio sin pasar por ningún lado visible.

Y ahora el ciclo, que es lo central:

```bash
cd infra
terraform init                # baja los providers y escribe el .terraform.lock.hcl
terraform fmt                 # el linter de HCL — corrélo SIEMPRE tras copiar snippets:
                              #   si hacés la parte opcional, su fmt -check falla hasta por alineación de comentarios
terraform validate            # sintaxis y coherencia
terraform plan                # ¿QUÉ va a pasar? (sin hacerlo)
```

**Leé el plan completo, porque esa lectura ES la habilidad del práctico.** Recortado, y con nombres de ejemplo, se ve así:

```
Terraform will perform the following actions:

  # neon_project.preprod will be created
  + resource "neon_project" "preprod" {
      + id                        = (known after apply)
      + name                      = "miapp-preprod"
      + history_retention_seconds = 21600
      + database_host             = (known after apply)
      + default_branch_id         = (known after apply)
    }

  # neon_database.preprod will be created
  + resource "neon_database" "preprod" {
      + id         = (known after apply)
      + name       = "app_preprod"
      + owner_name = "app_preprod_owner"
      + project_id = (known after apply)
    }

Plan: 3 to add, 0 to change, 0 to destroy.
```

Tres lecturas antes de aplicar:

1. **`(known after apply)` no es un problema** (§2.4): el id del proyecto lo asigna Neon, así que
   Terraform todavía no lo sabe — y por eso tampoco sabe el `project_id` de la base, que **sale**
   del proyecto.
2. **Los recursos se referencian entre sí** (`neon_project.preprod.id`, `neon_role.preprod.name`):
   de esas referencias Terraform deduce un **grafo de dependencias** y con él el orden de creación.
   El orden en el archivo no importa; el grafo sí. Y a la hora de destruir, lo recorre **al revés**.
3. **La última línea es el resumen que se lee primero en la vida real**: `3 to add, 0 to change,
   0 to destroy`. El día que diga `to destroy` y no lo esperabas, frenás.

```bash
terraform apply               # pide confirmación: yes
terraform output              # el host y el nombre de tu base, ya reales
```

Andá a la consola de Neon: el proyecto `miapp-preprod` está ahí, con su base. **Hace un minuto no existía, y lo creó tu código.** Ése es el argumento del §2.1, y conviene que te detengas a mirarlo.

> 🔴 **Esa base nace VACÍA — ¿quién crea tus tablas?** La misma pregunta del TP6 §3.1 y con las
> mismas respuestas: si tu app crea el esquema al arrancar (`EnsureCreated()` y equivalentes), las
> tablas aparecen la primera vez que el servicio se conecte (§3.4); si usás migraciones, las corrés
> contra esta base antes del primer deploy. Terraform crea **la base**, no **tu esquema**: ésa
> es la frontera. (Y en preprod la consecuencia es liviana: un entorno que nace
> vacío cada vez es exactamente lo que se espera de un entorno desechable.)

**Y ahora la segunda pasada, que es la que demuestra la idempotencia:**

```bash
terraform plan
# → "No changes. Your infrastructure matches the configuration."
terraform apply
# → aplica en cero acciones
```

Eso es idempotencia **demostrada**: misma configuración, N aplicaciones, mismo resultado. Y pensá el contraste: ¿qué haría un script imperativo (`neon projects create …`) en su segunda corrida? Fallaría, o te crearía un segundo proyecto idéntico.

**Y ahora abrí tu estado, que es la experiencia que convierte una regla en una convicción:**

```bash
terraform state list                                        # los recursos que Terraform cree que existen
grep -i -o 'password[^,]\{0,60\}' terraform.tfstate | head  # el archivo está ahí, al lado de tu main.tf
```

Ahí está, en texto plano: la contraseña del rol que Neon generó. Tres conclusiones:

- **Por eso el estado no se commitea.** Un estado en un repo público es una filtración de
  credenciales con historial incluido — y el historial de git no se borra. Lo
  único que lo separa de tu repositorio es la línea del `.gitignore` que escribiste en §3.1.
- **Y por eso ese archivo vale tanto como el código.** Es la única memoria de que preprod es tuyo: si
  lo borrás, Terraform deja de saber que ese proyecto de Neon lo creó él y el próximo `plan` propone
  crearlo de nuevo, contra algo que ya existe (§2.3). No se versiona, pero no es descartable.
- **Y lo que NO podés hacer con un estado que vive en una sola máquina**: que otra —un compañero, o
  una corrida del pipeline— compare contra él. Si hacés la parte opcional (§3.6) lo vas a ver en tu propio
  repositorio.

**✅ Checkpoint:** el `apply` creó el proyecto y la base en Neon, `terraform output` devuelve el host, **la segunda pasada dice «No changes»**, viste con tus ojos un atributo sensible dentro del estado, y podés explicar qué hizo `init`, qué significan `(known after apply)` y la última línea del plan.

### 3.3 Drift: romper la realidad a mano y verla repararse

Simulá el "cambito de emergencia" clásico — **por fuera de Terraform**, desde la consola de Neon: borrá la base `app_preprod` (*Databases → Delete*).

```bash
terraform plan
# → # neon_database.preprod will be created
#   Plan: 1 to add, 0 to change, 0 to destroy.
```

**Eso es drift detectado**, y esas dos líneas son las que devolvió la corrida de la cátedra. Terraform refrescó el estado contra la realidad, vio que lo que declarás ya no está, y propone converger.

```bash
terraform apply     # la realidad vuelve al código
```

Y ahora el experimento que casi nadie hace, y que es la mitad más valiosa de esta sección: **creá a mano, desde la consola de Neon, una base que tu código NO declara** (llamala `base_intrusa`) y volvé a planear.

```bash
terraform plan
# → "No changes."     ← y NO es un error
```

> 🔴 **Terraform vigila lo que declaraste, no todo lo que existe.** Un plan limpio dice "lo que yo
> gobierno está como pedí", no "no hay nada raro en tu cuenta". La `base_intrusa` sigue ahí,
> gastando cuota, invisible para tu código y para tu pipeline. Esto es medido, y es un malentendido
> clásico. Borrala a mano cuando termines — con la misma mano que la creó,
> que es justamente el punto.
>
> 📌 **Y el caso grande lo tenés gratis: tus cuatro servicios de QA y producción tampoco aparecen en
> ningún plan tuyo**, porque no los declaraste. Sin embargo existen, corren y consumen de las mismas
> 750 horas. Que tu `terraform plan` esté limpio no te dice **nada** sobre ellos. Decilo así en la
> defensa.

> 🔴 **Y ojo con una tentación razonable: el plan que corre en tu pipeline NO puede detectar esto.**
> No porque esté mal armado, sino porque no tiene tu estado — la memoria vive en tu máquina. Está
> explicado en la parte opcional (§3.6), y muestra para qué sirve un estado compartido.

**✅ Checkpoint:** secuencia de drift completa — cambio manual → plan que lo detecta → apply que repara → verificación en la consola de Neon — **y** la comprobación de que un recurso no declarado no produce ningún cambio en el plan.

### 3.4 Render: los dos servicios de preprod, escritos

La base ya existe. Ahora los dos servicios que la usan — y con ellos preprod deja de ser "una base" y pasa a ser **un entorno**.

Primero `infra/variables.tf`, para lo que es configurable:

```hcl
variable "image_repo_api"   { default = "ghcr.io/<owner>/<tu-repo>-backend" }    # SIN la etiqueta
variable "image_repo_front" { default = "ghcr.io/<owner>/<tu-repo>-frontend" }   # SIN la etiqueta
variable "image_tag"        { default = "sha-<los 40 caracteres de un merge del TP7>" }
variable "region"           { default = "oregon" }   # frankfurt | ohio | oregon | singapore | virginia
```

> 📌 **La etiqueta la sacás de la pestaña *Packages* de tu repositorio**: es la `sha-…` que tu
> pipeline del TP7 le puso a la imagen cuando mergeaste.
>
> 📌 **Estos cuatro valores NO son secretos** —el nombre de una imagen pública, una etiqueta, una
> región—, así que van con `default` en un archivo que **sí** se commitea. Los `*.tfvars` que ignoraste en §3.1 quedan para lo que no se publica.

Y a `infra/main.tf`, los dos servicios:

```hcl
locals {
  # La cadena de conexión se ARMA acá, con lo que devolvió Neon. Nadie la copia y pega:
  # ése era el error más caro del TP6 (un entorno apuntando a la base de otro).
  conn = "Host=${neon_project.preprod.database_host};Database=${neon_database.preprod.name};Username=${neon_role.preprod.name};Password=${neon_role.preprod.password};SSL Mode=Require"
}

resource "render_web_service" "api" {
  name   = "miapp-api-preprod"
  plan   = "free"
  region = var.region

  runtime_source = {
    image = {
      image_url = var.image_repo_api   # 🔴 SIN la etiqueta pegada
      tag       = var.image_tag        #    la etiqueta va en su propio campo
    }
  }

  env_vars = {
    ConnectionStrings__Default = { value = local.conn }
  }
}

resource "render_web_service" "front" {
  name   = "miapp-front-preprod"
  plan   = "free"
  region = var.region

  runtime_source = {
    image = {
      image_url = var.image_repo_front
      tag       = var.image_tag
    }
  }

  env_vars = {
    BACKEND_URL  = { value = render_web_service.api.url }   # ← la URL sale de Terraform, no de un copy-paste
    DNS_RESOLVER = { value = "8.8.8.8" }
  }
}
```

Y a `infra/outputs.tf`, las dos direcciones:

```hcl
output "api_url"   { value = render_web_service.api.url }
output "front_url" { value = render_web_service.front.url }
```

Cinco cosas de este bloque, y **tres de ellas son errores medidos**:

> 🔴 **1 — `image_url` va SIN la etiqueta, y la etiqueta va en `tag`.** Medido: si pegás
> `…-backend:sha-abc`, el apply muere con `must not contain the tag or digest`. El error es claro, y
> de paso te refuerza la distinción imagen/etiqueta del TP7 §2.2.
>
> 🔴 **2 — `plan = "free"` y la `region` de la lista corta.** El provider **no valida** el plan, así
> que `"free"` pasa derecho a la API (y se midió que crea). La región, en cambio, tiene validador
> estricto: sólo `frankfurt`, `ohio`, `oregon`, `singapore`, `virginia`.
>
> 🔴 **3 — NO declares `APP_COMMIT` en `env_vars`.** El TP7 §3.1 lo dejó dicho: esa variable la pone
> **sólo** tu pipeline, como argumento de build de la imagen. Una variable del servicio **pisa** a la
> de la imagen, y tu smoke —que compara el commit— dejaría de probar nada mientras sigue en verde.
>
> 🔴 **4 — El `image_tag` es fijo, y es a propósito. Y la recreación la pedís VOS.** Preprod corre
> **la imagen de un merge concreto**, escrita en el código: si mañana querés que corra otra, cambiás
> esa línea y hacés el PR. **Pero mirá bien qué va a planear Terraform**, porque acá casi todo el
> mundo contesta mal en la defensa: cambiar `tag` es, para él, una **modificación** del servicio
> (`~ update in place`), no un reemplazo — y ese update es exactamente el que muere con el
> `maintenance mode can only be configured for non-free tier services` de más arriba. **Terraform
> no recrea un recurso porque el update le falle: el `apply` se corta con error.** Así que la
> recreación se pide a mano, con una de estas dos:
>
> ```bash
> terraform destroy && terraform apply              # el ciclo que el TP te hace hacer igual (§3.5)
> terraform apply -replace=render_web_service.api   # o sólo ese recurso
> ```
>
> La idea de fondo no cambia, y es la buena: **un entorno desechable no se "actualiza": se vuelve a
> hacer** — es lo que hace la industria con los entornos efímeros. Lo que cambia es **quién lo
> pide**: vos, no la herramienta.
>
> 📌 **5 — Y lo que ganás, que es el argumento entero.** La cadena de conexión se **arma** con la
> contraseña que Neon generó, y la `BACKEND_URL` del front **sale del atributo `url` de su propia
> api**. Las dos cosas que en el TP6 se copiaban y pegaban a mano —y que producían los dos errores
> más caros del práctico, un entorno contra la base de otro y la barra final de la URL— ahora no se
> copian: se derivan.
>
> ⚠️ Comprobá igual el detalle de la barra: si el `url` que devuelve el provider viniera con `/` al
> final, tu nginx mandaría todo a `/` y cada llamada daría 404 (TP6 §3.2). Lo ves en el output, en un
> segundo, y si hiciera falta se corta con `trimsuffix(render_web_service.api.url, "/")`.

```bash
terraform fmt && terraform validate
terraform plan      # 2 to add, 0 to change, 0 to destroy
terraform apply
terraform output    # las dos URLs de tu entorno nuevo
```

**Y ahora comprobalo como un entorno de verdad, no como dos recursos creados.** Los servicios nacieron con la imagen declarada, así que ya están corriendo tu app:

```bash
curl -fsS "$(terraform output -raw api_url)/health"      # tu api de preprod, viva
curl -fsS "$(terraform output -raw api_url)/api/tareas"  # …y hablando con SU base
open "$(terraform output -raw front_url)"                # el front, contra esa api
```

Acordate del contrato del free tier del TP6 §2.7: el primer request después de un rato despierta el servicio y puede tardar hasta un minuto. Si el `/api/tareas` falla la primera vez, es la base creando el esquema o el servicio despertando: reintentá.

**✅ Checkpoint:** existe un entorno `preprod` completo —base, api y front— que **no existía**, creado por código; `/health` y el endpoint que toca la base responden; el front carga contra su propia api; y podés mostrar en el HCL de dónde sale la cadena de conexión y de dónde sale la `BACKEND_URL`.

> 📌 **Extensión opcional (no se exige, y suma): meté preprod en la cadena del TP7.** Copiá su deploy
> hook (Render → cada servicio → *Settings → Deploy Hook*) a dos secrets nuevos y agregá un job
> `deploy-preprod` entre `e2e` y `deploy-prod`, calcado del de QA pero con el `imgURL` de la corrida.
> La cadena queda `QA → preprod → [aprobación] → producción`, que es el patrón que §2.6 describe.
>
> 🔴 **Dos avisos antes de hacerlo.** (1) **El deploy hook no sale de Terraform**: el provider de
> Render no lo expone (hay un pedido de función abierto desde 2025), así que se copia del dashboard
> — y ése es tu primer «esto sigue siendo artesanal» con nombre y apellido. La
> alternativa profesional es disparar el deploy por la **API de Render**
> (`POST /v1/services/{id}/deploys`) con el `id` que sale de un output. (2) **Si después destruís
> preprod, sacá el job del workflow en el mismo PR**: un job que le pega a un hook de un servicio que
> ya no existe pone la corrida en rojo y te frena la promoción a producción. Que tu pipeline dependa
> de infraestructura que puede no existir es una tensión real de los entornos efímeros.

### 3.5 El ciclo de vida completo: preprod nace, muere, y vuelve idéntico

Esto es lo principal del práctico, y es lo que vas a hacer en la defensa.

```bash
terraform destroy      # baja TODO preprod: los dos servicios y la base. Pide yes.
```

Mirá el plan del destroy antes de confirmar: ahí se leen **las cinco cosas que se van** y la última línea, `5 to destroy`. 🔴 **Lo que el plan NO te muestra es el orden**: Terraform lista los recursos ordenados por su dirección, alfabéticamente, no en el orden en que los va a tocar. **El orden se ve cuando confirmás**, en las líneas de `Destroying...` que van saliendo — y ahí vas a ver que recorre el grafo de dependencias **al revés**: primero el front (que depende de la api), después la api (que depende de la base), después la base, el rol y el proyecto. Nadie escribió ese orden; sale de las referencias que escribiste en §3.4. Que el plan se lea alfabético y la ejecución siga el grafo es una distinción fina, y es justo la clase de detalle que separa a quien leyó un plan de quien lo miró.

Verificá en las dos consolas: en Render no están los dos servicios de preprod (y **sí** están los cuatro de QA y producción, intactos). En Neon no está el proyecto `miapp-preprod` (y **sí** está el de tu app, intacto). Y tu `terraform.tfstate` sigue en tu máquina, ahora casi vacío: la memoria de que no queda nada.

```bash
terraform apply        # …y renace
terraform output
curl -fsS "$(terraform output -raw api_url)/health"
```

> ⚠️ **La URL sacala del `output`, no de tu marcador.** Si al recrear un servicio de Render con el
> mismo nombre vuelve **la misma** URL o Render le agrega un sufijo, la cátedra **no lo midió** (y el
> TP6 avisa que lo agrega cuando el nombre está tomado). Para preprod no es grave —es desechable y su
> dirección se **deriva**—, pero es una razón más para leerla del `output` cada vez en vez de
> guardarla.

**Eso responde la pregunta con la que abrió el práctico** —*¿podrías recrear tu infraestructura idéntica mañana?*—: con IaC la respuesta es un comando y unos segundos. Y responde una mejor todavía, que es la del §2.6: *¿podés permitirte tener un entorno más?* Sí, si sabés hacerlo desaparecer.

> 🔴 **La regla operativa del resto del semestre, y no es un consejo: `terraform destroy` cuando no
> estés usando preprod.** Las 750 horas son del **workspace** y las comparten tus seis servicios
> gratuitos; si te pasás, Render **suspende todos, producción incluida**, hasta el mes siguiente — y
> tus URLs tienen que seguir vivas hasta la defensa (§2.6). Destruir preprod no es perder trabajo:
> el trabajo está en el repositorio. Esa frase es la tesis del práctico.

**Y ahora la red de seguridad, que es lo que separa un entorno desechable de uno que no lo es.** Agregá temporalmente, en `infra/main.tf`, dentro de `resource "neon_database" "preprod"`:

```hcl
  lifecycle {
    prevent_destroy = true      # ninguna base con datos se destruye por accidente
  }
```

```bash
terraform apply           # sin cambios: la línea no altera nada, sólo agrega una prohibición
terraform destroy         # → Terraform SE NIEGA, y te nombra el recurso
```

Sacá la línea cuando termines (preprod tiene que poder morir). Tres cosas para saber:

- **Qué te salva**: que un plan destructivo sobre algo con datos **no se puede aplicar**, ni por vos
  apurado ni por el pipeline. Es la línea que **iría en tu base de producción**, no en preprod.
- **De qué NO te salva**: si borrás el bloque del recurso del código, también se va el
  `prevent_destroy` — y con él la protección. Un `rm` de diez caracteres en un PR desarma la red.
  Por eso la protección de verdad es **el PR revisado** más el backup, no una línea de HCL.
- **Qué pasaría si igual se destruyera**: en Neon un proyecto borrado se va **con los datos
  adentro**, y no hay papelera. En preprod eso es gratis, porque la base nace vacía y se llena sola.
  **Decidir qué entorno es desechable y cuál no, es una decisión de diseño** — y es la razón por la
  que este TP eligió preprod y no tu producción.

**Y subilo al repositorio, que hasta acá vive sólo en tu máquina.** Como en todos los prácticos,
`main` está protegido: va por una rama y un pull request.

```bash
cd ..
git switch -c infra-preprod
git add -A && git commit -m "Preprod, declarado en infra/"
git push -u origin infra-preprod
gh pr create --fill
gh pr merge --merge          # cuando tu pipeline quede en verde
git switch main && git pull
```

Mirá qué viaja en ese commit: `infra/` con sus `.tf` y el `.terraform.lock.hcl`, y el `.gitignore`.
**El estado no**: lo ignoraste en §3.1. Si `git status` te lo muestra, frená y revisá el
`.gitignore` antes de commitear.

**✅ Checkpoint:** `infra/` está en `main`, sin el estado; el ciclo `destroy → apply` está hecho y preprod volvió (el `/health` responde, y las URLs las leíste del `output` — no las diste por iguales); viste el orden inverso **en la ejecución** del destroy (no en su plan, que va alfabético); comprobaste que QA y producción no se tocaron; y el `prevent_destroy` rechazó un destroy antes de que lo sacaras.

### 3.6 ➕ Opcional — El plan en cada PR: la infraestructura se revisa como se revisa el código

> ➕ **Esta sección es opcional**: no es tarea obligatoria del práctico ni se evalúa para aprobar.

La infraestructura ya es código. Le falta el tratamiento de código: que **cada cambio se verifique y se muestre antes de pasar**, en el mismo lugar donde se revisa todo lo demás. Eso es lo que hace este workflow: en cada PR que toca `infra/` corre el formateador, valida, ejecuta `terraform plan` y **publica el plan como artefacto de la corrida**, para que se lea antes de aprobar el PR.

**Las tres claves, como secrets del repositorio** (el patrón del TP6 §3.4):

```bash
gh secret set NEON_API_KEY        # sin --body: así no quedan en el historial de tu terminal
gh secret set RENDER_API_KEY
gh secret set RENDER_OWNER_ID
```

> 🔴 **Y la contracara hay que decirla.** Con esas
> tres claves, el job que planea puede **leer toda tu cuenta** —incluida la producción que este TP no
> toca—, y viven en los secrets de un repositorio público. La de Render, además, no tiene permisos
> granulares: es todo o nada sobre tu workspace. Tenelo presente.
>
> ⚠️ Y un límite que conviene conocer: GitHub **no le pasa secrets** a un `pull_request` que viene de
> un fork. En el flujo de la materia sos el único contribuidor, así que no te va a pasar; en un
> repositorio con gente de afuera, el job de plan de un PR ajeno falla — y está bien que falle.

Workflow nuevo `.github/workflows/infra.yml`:

```yaml
name: Infra

on:
  pull_request:
    paths: ["infra/**"]        # corre sólo cuando cambia la infraestructura
  workflow_dispatch:           # …y a pedido, desde la pestaña Actions

env:                           # los dos providers las leen solos
  NEON_API_KEY:    ${{ secrets.NEON_API_KEY }}
  RENDER_API_KEY:  ${{ secrets.RENDER_API_KEY }}
  RENDER_OWNER_ID: ${{ secrets.RENDER_OWNER_ID }}

jobs:
  infra-check:
    runs-on: ubuntu-latest     # fmt y validate no necesitan hablar con nadie
    steps:
      - uses: actions/checkout@v6
      - uses: hashicorp/setup-terraform@v4
      - name: Formato
        run: terraform -chdir=infra fmt -check
      - name: Init (sin backend) + validate
        run: |
          terraform -chdir=infra init -backend=false
          terraform -chdir=infra validate

  infra-plan:
    runs-on: ubuntu-latest
    needs: infra-check
    steps:
      - uses: actions/checkout@v6
      - uses: hashicorp/setup-terraform@v4
      - name: Plan
        run: |
          terraform -chdir=infra init
          terraform -chdir=infra plan -no-color | tee plan.txt
      - uses: actions/upload-artifact@v6
        if: ${{ !cancelled() }}
        with:
          name: tf-plan
          path: plan.txt
```

**Tres decisiones de diseño de este archivo:**

- **El filtro `paths` del disparador** — no tiene sentido planear infraestructura porque cambió un
  test del front. Es el mismo criterio de alcance que venís usando desde el TP4.
- **Dos jobs, porque no necesitan lo mismo.** `fmt` y `validate` son análisis del archivo: no hablan
  con nadie y no llevan credenciales. El `plan` sí las lleva, porque tiene que hablar con Neon y con
  Render. Separarlos hace que el error barato (un comentario mal alineado) falle en diez segundos y
  sin claves.
- **El plan se publica como artefacto, no queda enterrado en el log.** Se descarga, se lee entero, y
  queda adjunto al PR — que es exactamente lo que hace un reviewer de infraestructura.

> 📌 **Y acá va la parte honesta, que es tan contenido como el resto: en este práctico el `apply` lo
> hacés VOS, a mano. En la industria lo hace el pipeline, y así sería.**
>
> **Lo que tu cadena hace hoy**: cada PR que toca `infra/` verifica que el código es válido y publica
> lo que haría. **Lo que no hace**: aplicarlo. Después de mergear, el `terraform apply` lo corrés
> desde tu terminal.
>
> **Por qué queda así**: porque para que una corrida aplique hacen falta dos piezas que este TP no
> monta. **(1) El estado compartido.** La memoria de lo que existe está en tu notebook (§2.3), así
> que la corrida no tiene con qué comparar. Habría que mudarlo a un *backend* remoto con bloqueo —una
> base de Postgres, un bucket de S3, Azure Storage—, y **crearlo por fuera de Terraform**: si el
> lugar donde vive el estado estuviera declarado en el mismo código que destruís, el primer `destroy`
> se llevaría su propia memoria. **(2) Credenciales con permiso de escritura en el pipeline, y un
> portón humano**: un *environment* de GitHub con revisor obligatorio, para que nadie aplique sin que
> alguien haya leído el plan. La cadena completa de un equipo real se ve así:
> `PR → plan → merge → ⏸ aprobación → apply de la corrida`. Es el mismo patrón del gate de producción
> que ya armaste en el TP6, aplicado a la infraestructura.
>
> 🔴 **Y hay una consecuencia que podés ver HOY, en tu propia corrida, y que es la mejor prueba de
> todo esto**: el plan del pipeline **no arranca de tu estado**, así que no dice «esto va a cambiar»,
> dice **«esto crearía, si no existiera nada»**. Sirve —y mucho— para ver que el código compila, que
> los atributos existen y qué recursos declara. **No** sirve para revisar un cambio incremental ni
> para detectar drift. Ese hueco **es** el argumento del estado compartido, y lo tenés medido en tu
> propio repositorio.

> 🔴 **Y una trampa del propio diseño, que conviene que sepas antes de abrir el PR.** El disparador
> filtra por `paths: ["infra/**"]`, así que **un PR que sólo agrega el workflow no dispara nada** —
> el archivo nuevo vive en `.github/workflows/`, no en `infra/`. Metele en el mismo PR un cambio
> chico y real de `infra/` (una salida nueva en `outputs.tf` alcanza) y resolvés las dos cosas: el plan
> corre, y el `terraform apply` de después del merge tiene algo que hacer. Si no, tu primera corrida
> no existe y tu primer apply dice «No changes», y parece que nada funcionó.

**El flujo resultante**, que es el práctico entero en dos líneas:

```
PR que toca infra/  →  fmt + validate + PLAN publicado como artefacto  →  se lee antes de aprobar
merge a main        →  terraform apply, desde tu terminal              →  preprod cambia
```

> 📌 **¿Y el detector de drift automático?** El `workflow_dispatch` de arriba te deja correr la
> verificación cuando quieras, y un `schedule: [{ cron: "0 9 * * 1" }]` la volvería semanal. Pero un
> detector de drift de verdad necesita el estado compartido, por lo mismo de recién: sin él, el plan
> programado te diría siempre «crearía todo». Otra consecuencia de la misma decisión — y otra
> respuesta que podés dar entera en la defensa.

**✅ Checkpoint:** un PR que toca `infra/` corre fmt+validate+plan, descargaste el artefacto `tf-plan` y lo leíste, y podés explicar **qué te dice y qué NO te dice ese plan**, por qué el job de `fmt` no necesita credenciales y el de `plan` sí, y qué haría falta para que el `apply` lo ejecutara la corrida.

### 3.7 Plan C: Terraform + provider Docker, 100 % local (entrega válida)

**Para quién es este plan C.** Para el que en TP6/TP7 fue por el fallback local —su free tier falló, o eligió no usar la nube— y hoy corre sus entornos QA y PROD con `compose.qa.yml` y `compose.prod.yml` sobre su self-hosted runner. También sirve si a mitad de camino uno de los dos free tiers muta: **cumple todos los checkpoints del TP y es entrega válida**, sin descuento.

**La consigna es la misma: declarás un entorno NUEVO, `preprod`, que no existe.** Acá es un tercer proyecto de Compose —su red, su volumen, su base y tu backend con la imagen del TP7— que convive con los dos que ya tenés y que se levanta y se baja con Terraform. Todo lo demás —el ciclo, el estado, el plan, el drift— es idéntico, y la verificación desde el pipeline es opcional también acá. De hecho el plan C tiene una ventaja: no arrastra el defecto del plan gratuito de Render, así que **también podés modificar** recursos y ver el `~` que el riel canónico no te muestra.

> 🚨 **Plan C NO es «levanto el compose a mano y lo muestro andando».** Eso es el TP2. Acá lo que se
> evalúa es que la infraestructura esté **declarada, aplicada y gobernada por código**.

`infra/main.tf`:

```hcl
terraform {
  required_version = ">= 1.5"
  required_providers {
    docker = {
      source  = "kreuzwerker/docker"
      version = "~> 4.0"
    }
  }
  backend "local" {}          # el path se pasa por -backend-config en el init: ver abajo
}

provider "docker" {}          # habla con tu Docker local (el daemon del TP2)

resource "docker_network" "app" {
  name = "miapp-preprod"
}

resource "docker_volume" "db_data" {
  name = "miapp-preprod-db-data"
}

resource "docker_image" "postgres" {
  name = "postgres:16-alpine"
}

resource "docker_container" "db" {
  name  = "miapp-preprod-db"
  image = docker_image.postgres.image_id
  env = [
    "POSTGRES_PASSWORD=${var.db_password}",
    "POSTGRES_DB=app",
  ]
  networks_advanced { name = docker_network.app.name }
  volumes {
    volume_name    = docker_volume.db_data.name
    container_path = "/var/lib/postgresql/data"
  }
}

resource "docker_image" "api" {
  name = "ghcr.io/<owner>/<tu-repo>-backend:${var.api_tag}"   # ← LA IMAGEN DEL TP7 (TP7 §3.1)
}

resource "docker_container" "api" {
  name    = "miapp-preprod-api"
  image   = docker_image.api.image_id
  restart = "unless-stopped"   # ver la nota: depends_on espera la CREACIÓN de la BD, no su readiness
  env = [
    "ConnectionStrings__Default=Host=${docker_container.db.name};Database=app;Username=postgres;Password=${var.db_password}",
  ]
  networks_advanced { name = docker_network.app.name }
  ports {
    internal = 8080
    external = 8090          # 🔴 un puerto que NO usen tus QA (8080/3000) ni PROD (8081/3001)
  }
  depends_on = [docker_container.db]
}
```

`infra/variables.tf` + `infra/outputs.tf`:

```hcl
variable "db_password" {
  type      = string
  sensitive = true            # no se imprime en logs ni en el plan
}

variable "api_tag" {
  type = string               # sin default: va un sha-<commit> del TP7 (tu pipeline no publica otra etiqueta)
}

output "api_url" {
  value = "http://localhost:${docker_container.api.ports[0].external}"
}
```

El ciclo es el mismo del §3.2, con el `-var-file`:

```bash
cd infra
terraform init -backend-config="path=$HOME/tfstate/miapp.tfstate"
terraform fmt && terraform validate
echo 'db_password = "super-secreto-local"' > local.tfvars
echo 'api_tag = "sha-<commit>"' >> local.tfvars   # los 40 caracteres de un merge con imagen publicada
# Windows PowerShell 5.1: el echo escribe UTF-16 y Terraform no lo parsea — usá el editor
#   o: Set-Content local.tfvars 'db_password = "…"', 'api_tag = "sha-…"' -Encoding utf8
terraform plan  -var-file=local.tfvars
terraform apply -var-file=local.tfvars
curl http://localhost:8090/health
docker ps --filter name=miapp-preprod
```

Los cinco avisos que te ahorran la tarde:

> ⚠️ **1 — `depends_on` espera que el contenedor de la BD se *cree*, no que postgres esté *listo*.**
> ¿Te suena? Es «arrancó ≠ está listo» del TP2, que compose resolvía con `healthcheck` +
> `service_healthy`. Por eso la API lleva `restart = "unless-stopped"`: si arranca antes y crashea,
> Docker la reintenta. Dale unos segundos antes del curl.
>
> ⚠️ **2 — La imagen del TP7 tiene que ser pública** (o `docker login ghcr.io` antes del apply): si
> el apply muere con `denied` al pullear, es el nombre mal escrito o el package privado.
>
> 🔴 **3 — El estado va FUERA del workspace, y hay una razón técnica que casi nadie sabe.**
> `actions/checkout` corre `git clean -ffdx` **antes de cada checkout**, y la bandera `-x` borra
> **también los archivos ignorados por git**. Si el estado viviera en `infra/terraform.tfstate`
> —justamente porque está en el `.gitignore`, como el TP manda—, **cada corrida lo borraría** y
> verías «+6 to add» para siempre sin entender por qué. Por eso `$HOME/tfstate/miapp.tfstate`, un
> path fijo que comparten tus comandos y el runner (que **es** tu máquina).
> Migrá el estado a ese path una sola vez:
> `terraform init -migrate-state -backend-config="path=$HOME/tfstate/miapp.tfstate"`.
> (Agregar o cambiar un backend **siempre** exige re-init: el error «Backend initialization
> required» es esperable, no un problema.)
>
> ⚠️ **4 — Si un apply queda a medias, no es tragedia**: el próximo `apply` **converge** desde donde
> quedó. El modelo declarativo, trabajando para vos.
>
> 🔴 **5 — Choques con tus entornos que ya corren**: el puerto 8090 tiene que estar libre (cambiá el
> lado `external` si no lo está), y los nombres `-preprod` no pueden repetirse con los de tus
> proyectos de compose. Que preprod conviva **sin pisar** a QA ni a PROD es parte del checkpoint,
> igual que en la nube.

**Idempotencia, drift y ciclo de vida** se hacen igual que en el riel canónico:

```bash
terraform plan  -var-file=local.tfvars   # → "No changes" (§3.2)

docker rm -f miapp-preprod-api           # el drift: alguien borró tu API a mano (§3.3)
terraform plan  -var-file=local.tfvars   # → "docker_container.api will be created"
terraform apply -var-file=local.tfvars   # la realidad vuelve al código

terraform destroy -var-file=local.tfvars # el ciclo de vida completo (§3.5): preprod se va entero
docker ps --filter name=miapp-preprod    # nada — y tus QA y PROD siguen corriendo
terraform apply   -var-file=local.tfvars # …y renace, idéntico
```

> 📌 **Probá también un drift de *modificación*, que el riel de Render no te deja ver**: desconectá
> el contenedor de la red (`docker network disconnect miapp-preprod miapp-preprod-api`) y mirá cómo
> lo describe el plan. Aviso para que no te desconcierte: no esperes un `~` prolijo — vas a ver un
> **replace** (`1 to add, 1 to destroy`) y el motivo aparente van a ser los `ports` con
> `# forces replacement`. Pensá por qué (pista: al desconectar la red, Docker soltó también el port
> mapping). **El drift real y el drift visible no siempre coinciden, y leer eso ES la habilidad.**
>
> 🔴 **Ojo con el `destroy` y los datos**: también se lleva el **volumen**. Hacé el ejercicio del
> §3.5 poniendo `prevent_destroy = true` en `docker_volume.db_data` y comprobá que el destroy se
> niega.

➕ **Opcional, igual que en el riel canónico.** Si lo hacés, **el pipeline** es el del §3.6 con dos cambios, y uno es importante:

- El job `infra-plan` corre en **`runs-on: self-hosted`** (tu runner del TP6), no en `ubuntu-latest`:
  el plan necesita hablar con **tu** Docker local, y un runner hosted no lo ve. El `infra-check`
  (fmt + validate) sigue en `ubuntu-latest`, porque es análisis de código.
  📎 Si en TP6/TP7 fuiste por el riel de Render y nunca registraste el runner: se registra como en
  **TP6 §3.6** (advertencia de seguridad para repos públicos incluida) y tiene que estar
  **actualizado** (≥ 2.327.1 — las actions `@v6` corren Node 24). El runner ya tiene Terraform
  instalado, porque es la máquina donde lo instalaste en §3.1.
- Las variables viajan como `TF_VAR_*`: `TF_VAR_db_password: ${{ secrets.DB_PASSWORD }}` (el mismo
  secret del TP6 §3.6) y `TF_VAR_api_tag`, y el `-backend-config` del `$HOME/tfstate/…` en **todos**
  los `init`, el del runner y los tuyos.

> 💡 **Y una ventaja del plan C que conviene que sepas nombrar.** Como tu runner **es** tu máquina, el
> plan del pipeline lee **tu** estado: ahí sí es un diff de verdad contra lo que existe, y no el
> «crearía todo» que ve el riel canónico (§3.6). Es un accidente feliz de correr todo en un solo
> lugar, y se termina apenas aparece una segunda máquina — que es, otra vez, el argumento del estado
> compartido.
>
> 📌 **Y los dos huecos que no se cierran:** el alumno de plan
> C nunca ve un plan que diga «voy a destruir tu base de datos gestionada» contra un proveedor real,
> y **no tiene la presión de la cuota** (§2.6) que en la nube convierte al `destroy` en una
> necesidad y no en un ejercicio. Son diferencias de escenario, no de concepto. Nombralas.

**✅ Checkpoint (plan C):** los mismos que el riel canónico — preprod declarado y aplicado, conviviendo con QA y PROD sin pisarlos · estado fuera del workspace · «No changes» en la segunda pasada · drift provocado, detectado y reparado · `destroy → apply` que devuelve preprod idéntico.

## 4- Riel alternativo: Azure (Bicep + `what-if`)

| Concepto | Riel canónico (Terraform + Neon/Render) | Azure (Bicep) |
|---|---|---|
| Archivo | `infra/main.tf` (HCL) | `infra/main.bicep` |
| Preview | `terraform plan` | `az deployment group what-if` |
| Aplicar | `terraform apply` | `az deployment group create` |
| Estado | Un archivo en tu máquina (§2.3) | **No hay archivo**: Azure Resource Manager ES el estado |
| Idempotencia | 2do apply → "No changes" | 2do create → sin cambios (deployments incrementales) |
| Drift | `plan` refresca y detecta | `what-if` contra el resource group |
| Lo que se declara | El entorno **`preprod`**: su base de Neon + api y front en Render | Un **resource group `preprod`** nuevo: App Service plan F1 + 2 Web Apps (+ lo que tu app use) |
| Destruir el entorno entero | `terraform destroy` | `az group delete` — el resource group es la unidad de vida y muerte, y es más limpio todavía |
| Credencial del pipeline *(opcional)* | claves de API como secrets del repositorio | una app registrada en Entra ID, con `azure/login` |
| Quién aplica | vos, desde tu terminal | ídem: el `create` lo corrés vos |

**Checkpoints riel Azure:** los mismos que el canónico (archivos + `.gitignore` → el estado explicado, aunque en Azure sea el propio Resource Manager → `what-if` leído → `create` de preprod → 2da pasada sin cambios → drift provocado en el portal y detectado por `what-if` → delete/recreate del resource group de preprod → y, opcional, `what-if` corriendo en cada PR desde el pipeline).

> 📌 El riel Azure requiere subscription (Azure for Students sin tarjeta, si te la dieron). Si a mitad de camino la subscription te falla, el riel canónico o el plan C cubren TODOS los checkpoints del TP — cambiar de riel no es empezar de nuevo: los conceptos son idénticos.
>
> ⚠️ El tier gratuito F1 también tiene su letra chica (60 min de CPU por día, sin SLA): el argumento
> del §2.6 —un entorno que existe cuando se lo necesita— vale igual, con otros números.

---
---

# 📋 Trabajo Práctico 08 – Infraestructura como Código (2026)

## ⚠️ Este es el TP que debés entregar y defender

## 🎯 Objetivo

Que puedas **crear un entorno entero escribiéndolo**: un `preprod` que no existía —su base de datos y sus servicios corriendo tu imagen del TP7— declarado como **código versionado**, que **nace y muere con un comando**.

🔴 **Lo que se mide es una sola cosa: que en la defensa, frente al docente, crees tu preprod desde cero.** Todo lo demás de esta guía está para que ese momento salga bien y para que puedas explicarlo.

Este trabajo se aprueba **solo si podés explicar qué hiciste, por qué lo hiciste y cómo lo resolviste**.

## 🧩 Escenario

Se sumó un integrante nuevo al equipo y tardó **dos días** en armar el entorno "siguiendo el README" — que estaba desactualizado en tres pasos. La semana siguiente, alguien "arregló una cosita" directo en la consola del proveedor y nadie supo qué cambió hasta que QA se comportó raro. Y esta semana el equipo necesita **un entorno más**: uno estable, parecido a producción, donde probar la release antes de promoverla — porque QA está siempre sucio y producción no se usa para probar. La respuesta "hay que clickear todo otra vez, y ojalá salga igual" no alcanza. El equipo decide que la infraestructura reciba el mismo tratamiento que el código: **declarada, versionada, revisada por PR, y capaz de nacer y morir con un comando**.

## 📋 Tareas que debés cumplir

### 1. El entorno `preprod`, declarado y creado desde cero
- Un entorno **nuevo** —base de datos y servicios corriendo **la imagen del TP7**— declarado en
  `infra/`, con los providers **fijados por versión** (Terraform + Neon/Render canónico;
  Bicep/riel Azure; plan C con provider Docker; u otra herramienta IaC con el mismo contrato).
- 🔴 **Tu QA y tu producción no se tocan**: ni se importan, ni se migran, ni se les cambia nada.
  Preprod **convive** con ellos.
- **Ninguna credencial escrita en el HCL**: las claves las leen los providers del entorno.
- **La cadena de conexión y la dirección del backend se DERIVAN, no se copian.**
- `.gitignore` cubriendo estado, `.terraform/` y tfvars **desde antes del primer apply**.
- **`infra/` subida a `main`** por un pull request, con el `.terraform.lock.hcl` y **sin el estado**.
- **Comprobado como entorno, no como recursos**: el endpoint de vida y uno que **toque la base**
  responden, y el front carga contra su propia api.

### 2. El ciclo de vida: nace, muere, y vuelve idéntico
- **`destroy` → `apply`**: preprod desaparece entero y vuelve **idéntico**, con sus URLs
  respondiendo. 🔴 Y **QA y producción siguieron intactos**.
- Es lo que vas a hacer en la defensa, así que hacelo antes más de una vez.

### 3. Release `v8.0.0`
- El cierre del práctico etiquetado con **`v8.0.0`** y publicado como **Release de GitHub con
  notas**, sobre el commit que deja tu `infra/` terminado.
- 📌 El número **no se elige**: lo fija el práctico. Si copiás el comando del TP7 con su número
  viejo, `git tag` falla con *«already exists»*.

```bash
git switch main && git pull
git tag v8.0.0 <sha-del-commit-que-cierra-tu-infra>
git push origin v8.0.0
gh release create v8.0.0 --generate-notes --verify-tag
```

### 🧪 Práctica recomendada (no se entrega nada)

Son los experimentos de la guía. No dejan ningún entregable, pero son de donde salen las preguntas
de la defensa: hacelos.

- **Idempotencia** (§3.2): el segundo `plan` dice «No changes».
- **El estado por dentro** (§3.2): abrilo y encontrá la contraseña.
- **Drift** (§3.3): borrá algo a mano, mirá el `plan` detectarlo y el `apply` repararlo; y creá
  algo que tu código no declara, para ver que el plan no lo ve.
- **`prevent_destroy`** (§3.5): el `destroy` que se niega.

### ➕ Opcional: el plan en cada PR (§3.6)

Un workflow que en cada pull request que toca `infra/` corre fmt, validate y plan, y deja el plan
como artefacto. **No es obligatorio y no suma ni resta para aprobar**; si lo hacés, contalo en la
defensa.

## 📄 Entregables

1. **URL del repositorio público** (formulario de la cátedra). Es lo único que va al formulario.
2. **En el repositorio**: la carpeta `infra/` y la release `v8.0.0`.
3. **En `decisiones.md`** (acumulativo), una sección corta del TP8 con tres cosas:
   - qué riel elegiste y qué versión de cada provider fijaste;
   - qué problemas tuviste y cómo los resolviste;
   - la declaración de uso de IA.

> 📌 **No hay `evidencias.md`, no se piden capturas, y no se pega la salida de ningún comando.**
> La prueba de este práctico no es un texto: es preprod naciendo delante nuestro.

> 🔴 **Preprod NO tiene que estar vivo el día de la defensa — tiene que poder NACER.** Destruirlo
> cuando no lo usás es lo correcto (§2.6: si te pasás de las 750 horas, Render suspende
> producción). Lo que sí tiene que estar vivo, como siempre, son tus **QA y producción**.

> 🔗 **El `infra/` de este TP tiene que seguir sirviendo en marzo**, para el **TP10 Integrador**:
> `terraform apply` tiene que volver a crear preprod aunque ese día no esté corriendo. Y si para
> entonces el provider de Neon se mudó al oficial (§3.2), cambiás el `source`, corrés
> `terraform init -upgrade` y leés el plan que sale.

## 🗣️ Defensa Oral Obligatoria

Se realiza en **P2**, junto con los TPs 5 a 9.

🔴 **Lo primero, en vivo: creá tu preprod.** Con tus claves exportadas, `terraform apply` desde tu
`infra/`; mostrás que el endpoint de vida y el que toca la base responden y que el front carga; y
`terraform destroy`. Traelo ensayado, y **llegá con preprod destruido**: si está vivo, `apply`
contesta «No changes» y no hay nada que mostrar.

Y mientras corre, preguntas como:

- En tu dashboard tenés entornos de dos tipos: los que clickeaste y el que está escrito. ¿En qué
  se diferencian para vos, hoy?
- Leeme el plan que acabás de aplicar: ¿qué se crea? ¿Qué significa `(known after apply)`?
- ¿Qué pasa si aplicás dos veces seguidas?
- ¿De dónde sale la cadena de conexión de tu api? ¿Y la dirección del backend que usa tu front?
- ¿Dónde vive tu estado, qué tiene adentro, y por qué no está en el repositorio?
- ¿Qué se destruye primero y quién decidió ese orden?
- ¿Qué es drift, y qué hace Terraform cuando lo encuentra?
- ¿Cuántos servicios gratuitos tenés ahora, y qué tiene que ver eso con el `destroy`?

## ✅ Evaluación

| Criterio | Peso |
|---|---|
| Configuración técnica (`infra/` en el repositorio, QA y producción intactos, release, `decisiones.md`) | 25% |
| Defensa oral: **preprod creado en vivo**, y comprensión | 75% |

> 📌 **En este TP los pesos son distintos de la regla general del README** (25 % / 25 % / 50 %):
> `decisiones.md` acá son tres renglones y entra en lo técnico, y la defensa —crear preprod en
> vivo— vale 75 %.

| | **Suficiente (4-5)** | **Bien (6-7)** | **Muy bien (8-10)** |
|---|---|---|---|
| **Preprod en vivo** | Lo creás frente al docente, responde, y lo destruís | Además leés el plan mientras corre y decís qué va a pasar antes de que pase | Además explicás de dónde se deriva cada dirección y por qué el orden del destroy es el que es |
| **El código** | Providers fijados, sin credenciales en el HCL, estado fuera del repositorio | La cadena de conexión y la URL del backend se derivan, y lo mostrás | Además explicás por qué se declaró un entorno nuevo y no el que ya tenías |
| **Lo que entendiste** | Contestás qué es el estado, qué es un plan, qué es drift | Lo contestás sobre **tu** repositorio | Sostenés la repregunta: qué sigue siendo artesanal en tu cuenta y qué harías distinto |

> ⚖️ Peso orientativo de este TP en la nota de **P2**: **15%** (la ponderación completa de los 9 TPs está en el reglamento, §5).

## ⚠️ Uso de IA

Podés usar IA (ChatGPT, Copilot, Claude), pero **deberás declarar en `decisiones.md` qué parte fue asistida por IA** y justificar cómo la verificaste. En este TP en particular: si la IA te escribió el HCL, tenés que poder **leer un plan en voz alta** y explicar qué haría cada acción. Si no podés defenderlo, **no se aprueba**.
