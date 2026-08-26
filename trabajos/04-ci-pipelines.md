# ⚠️ IMPORTANTE – Guía de Práctica Sugerida

Este documento tiene **dos partes**:

1. **Guía de práctica sugerida** (primera parte): paso a paso para aprender haciendo. **NO es lo que se entrega.**
2. **El Trabajo Práctico entregable** (al final): escenario, tareas, entregables y defensa oral. **Eso es lo que se evalúa.**

## Sobre las herramientas en este TP

En esta materia hay dos **rieles** — caminos guiados con soporte de la cátedra — más la opción libre:

- La guía paso a paso usa **GitHub Actions** (riel canónico: gratis e ilimitado en repos públicos, sin tarjeta de crédito, y es lo que se demuestra en clase).
- Si preferís **Azure Pipelines** (Azure DevOps), al final tenés la **tabla de equivalencias** con los mismos checkpoints. **Estado 2026 de ese riel** ([verificado contra la documentación de Microsoft](https://learn.microsoft.com/en-us/azure/devops/organizations/projects/public-projects-retirement?view=azure-devops)): desde **abril 2026** Azure DevOps **no permite crear proyectos públicos nuevos** (los existentes pasan a privados en 2027), y el grant gratuito de minutos Microsoft-hosted (1800 min/mes) hoy exige **vincular la organización a una suscripción de Azure** (que normalmente pide tarjeta; la suscripción *Azure for Students* no la pide y debería servir — verificalo ANTES de apostar tu TP a eso). Alternativa sin tarjeta en ese riel: un **agente self-hosted** (tu máquina; 1 job paralelo gratis). En cualquier caso, el **código puede seguir viviendo en GitHub** (Azure Pipelines buildea repos de GitHub) — así el entregable "repo público" no se pierde.
- Podés usar cualquier otro CI (GitLab CI, CircleCI…) cumpliendo el contrato de entregables, con soporte limitado de la cátedra. **Ojo**: GitLab pide tarjeta para validar sus runners compartidos — no es camino garantizado.

📌 Este TP trabaja **sobre tu app del semestre** (la del TP2), en el repo con las protecciones del TP1.

🧪 **Los ejemplos de esta guía están escritos sobre la app de la cátedra** ([`ingsoft3ucc/demo-fullstack`](https://github.com/ingsoft3ucc/demo-fullstack) — .NET 8 + React/Vite + PostgreSQL, la misma de las demos en clase; cómo clonarla y levantarla, base de datos incluida: **TP2 §3.2**). Si un paso no te sale en tu app, probalo primero ahí —donde sabés que funciona— y después llevalo a la tuya. Lo que entregás, siempre, es **tu** app.

---

# Guía Paso a Paso – CI: Pipelines as Code (Práctica sugerida)

## 1- Objetivos de Aprendizaje

- Entender la **integración continua** como práctica (no como herramienta).
- Escribir **pipelines como código** (YAML versionado en el repo).
- Automatizar la **construcción** de tu app en cada push y Pull Request, con tu propio Dockerfile.
- Entender el **cache** de capas: para qué sirve y por qué el pipeline no puede depender de él.
- Convertir el pipeline en un **gate obligatorio** del Pull Request.

## 2- Marco teórico

### 2.1 Integración continua: la práctica detrás de la herramienta

**CI (Continuous Integration)** no es "tener un pipeline": es la práctica de **integrar el trabajo de todos, con frecuencia, verificando cada integración automáticamente**. La herramienta es el medio; la práctica es integrar seguido y enterarse *ya* cuando algo se rompe.

¿Qué problema ataca? El que viste en la clase 1 con DORA: integrar tarde es integrar caro. Si cada desarrollador trabaja semanas en su rama, el día de la integración es un evento traumático ("integration hell"). Si todos integran a diario y **cada integración dispara build + tests automáticos**, los problemas se detectan en minutos, cuando el cambio que los causó es chico y fresco en la memoria de su autor.

> 📌 **¿Qué quiere decir «verificar»?** Dos cosas, y las dos son parte de la definición: que el
> proyecto **se construya** (que compile, que la imagen se arme) y que **los tests pasen**. Un
> pipeline que sólo construye está haciendo la mitad del trabajo. En **este** TP vas a cablear la
> primera mitad; la segunda —los tests adentro del mismo pipeline— llega en el **TP5**, que es la
> clase de testing, y va a ser una etapa más de tu Dockerfile. No es que la CI no los incluya: es
> que todavía no vimos cómo se escriben.

Dos reglas culturales definen a un equipo que hace CI de verdad:
1. **El build roto es la prioridad número uno.** Si `main` está en rojo, nadie apila trabajo encima; se arregla o se revierte AHORA. Un `main` rojo tolerado durante días convierte el pipeline en decoración.
2. **Si no pasó por el pipeline, no existe.** Nada llega a `main` sin que las máquinas lo hayan verificado. (¿Te suena? Es la protección de rama del TP1 — allá exigiste que todo entrara por Pull Request; acá le sumás la verificación automática. La revisión humana sigue siendo tuya: leés tu propio diff antes de mergear, aunque la plataforma no pueda exigírtelo — ver §3.4, paso 4.)

### 2.2 Pipeline as Code: por qué el pipeline vive en el repo

El CI clásico se configuraba **clickeando en una interfaz web**: el pipeline vivía en el servidor, fuera del control de versiones. Problemas conocidos: no se podía revisar en un PR, no se sabía quién cambió qué, no se podía reproducir, y migrar de servidor era arqueología.

> 📌 **El contraste es entre formas de configurar, no entre herramientas.** Así se trabajaba con Hudson —que en 2011 pasó a llamarse Jenkins— antes de 2016. Desde entonces Jenkins también hace pipeline as code, con su `Jenkinsfile` versionado en el repo igual que este YAML. Lo que quedó atrás es el pipeline a clicks, no un producto.

**Pipeline as Code** invierte eso: el pipeline es un archivo YAML **dentro del repo** (`.github/workflows/ci.yml`). Consecuencias:
- Se **versiona y se revisa** igual que el código (un cambio al pipeline es un PR más).
- **Viaja con el código**: clonás el repo, tenés el pipeline. Cada rama puede ajustar su pipeline.
- Es **la cuarta aparición del patrón de la materia** — todo lo importante se **declara explícitamente, en vez de hacerse a mano**: TP1 (las protecciones, declaradas en un JSON que se aplica por API), TP2 (el entorno, en Dockerfile y compose), TP3 (el plan, enlazado al código), y ahora el proceso de build — declarado en tu Dockerfile y disparado desde el YAML del pipeline. El TP8 (IaC) va a ser la quinta. Fijate **dónde vive cada una**: el entorno, el proceso de build y la infraestructura viajan como **archivos dentro del repo**; las protecciones y el plan viven en la **configuración de la plataforma**. Declarativas las cinco; versionadas en tu repositorio, tres.

### 2.3 Anatomía de un pipeline (GitHub Actions ↔ Azure Pipelines)

Los conceptos son universales; cada plataforma les pone nombre propio:

| Concepto | GitHub Actions | Azure Pipelines |
|---|---|---|
| El proceso completo | **Workflow** (un YAML en `.github/workflows/`) | **Pipeline** (`azure-pipelines.yml`) |
| Qué lo dispara | **Event** (`on: push`, `pull_request`, …) | **Trigger** |
| Unidad que corre en una máquina | **Job** | **Job** (agrupados en **Stages**) |
| Paso individual | **Step** (comando `run:` o `uses:` una action) | **Step** (script o **Task**) |
| Bloque reutilizable de terceros | **Action** (Marketplace) | **Task** (catálogo + Marketplace) |
| La máquina que ejecuta | **Runner** (hosted o self-hosted) | **Agent** (Microsoft-hosted o self-hosted) |

La estructura mental: *un evento dispara un workflow; el workflow tiene jobs (que corren en paralelo por defecto, cada uno en un runner limpio); cada job es una secuencia de steps.* Los jobs **no comparten filesystem** — si el job B necesita algo que produjo el job A, viaja como artefacto (§2.5) o se declara `needs: A` para el orden.

**Runners hosted vs self-hosted**: los hosted (máquinas de GitHub/Microsoft) son cero mantenimiento, efímeros y limpios en cada corrida — el default correcto. Los self-hosted (tu propia máquina registrada como runner) sirven cuando necesitás hardware/red propios… y son la pieza que hace posible el **fallback local** de esta materia en TP6/7 (desplegar a tu propia máquina desde el pipeline).

> ⚠️ **Un self-hosted runner ejecuta en TU máquina lo que diga el workflow.** Por eso GitHub
> recomienda usarlos **solo con repos privados**: en uno público, cualquiera puede abrir un Pull
> Request desde un fork con un workflow que corra código arbitrario en tu equipo. Tu repo de la
> materia es público, así que si en TP6/TP7 usamos el fallback local va con la **aprobación manual
> de workflows de forks** activada (*Settings → Actions → Fork pull request workflows*) o sobre un
> repo privado aparte. No lo registres "para probar" sin eso.

### 2.4 Triggers: cuándo corre el pipeline

Los cuatro que vas a usar:
- **`push`** (a `main`): deja constancia de cómo quedó `main` después del merge — es la corrida que lee el badge (§3.5) y la que deja el cache que después reutiliza cualquier PR (§3.2). Con el gate puesto rara vez sorprende: lo que iba a romper ya lo frenó la corrida del PR.
- **`pull_request`**: verifica lo que *quiere* integrarse — **el más importante**: corre ANTES del merge, sobre el resultado propuesto, y alimenta al gate del PR (§2.7).
- **`workflow_dispatch`**: disparo manual del workflow. (Ojo: para volver a correr una corrida que ya existe **no** hace falta — eso es el botón *Re-run* de la propia corrida. Aparece en el anexo opcional de métricas del TP3.)
- **`schedule`** (cron): corridas periódicas (builds nocturnos, chequeos de dependencias — lo vas a ver en TP9).

### 2.5 Artefactos y cache: qué se guarda entre corridas (y qué no)

Cada job arranca en una máquina **limpia**: todo lo que quede adentro del runner se pierde cuando el job termina. Eso vale también para lo que tu pipeline PRODUCE — las dos imágenes que construye nacen y mueren ahí. Guardarlas para poder usarlas después tiene nombre, se llama **registry**, y ya lo usaste: es lo que hiciste a mano en el TP2 §3.7, cuando publicaste tus imágenes. Lo que todavía no hicimos es que las publique **el pipeline**; a lo que se guarda de una corrida se lo llama **artefacto**.

Y sí: las plataformas ofrecen dónde dejarlo (en GitHub, `actions/upload-artifact`; en Azure DevOps,
`PublishPipelineArtifact`), y lo vas a usar en el TP5 para el reporte de tests. **En este TP tu
pipeline no guarda nada a propósito**, por dos razones: el lugar de una imagen es un registry, no el
almacén de artefactos; y conservar lo que produce una corrida recién tiene sentido cuando el pipeline
verifica algo más que la compilación. La salida de tu pipeline esta semana es otra, y no es menor:
**el check en verde que habilita el merge**.

Lo que sí necesitás hoy es el otro mecanismo:
- **Cache**: una **optimización** para no rehacer en cada corrida lo que no cambió. Como tu build es un `docker build`, lo que se cachea son las **capas de la imagen**: si la capa que instala dependencias no cambió, se reutiliza en vez de rehacerse. Se guarda y se recupera con `cache-to`/`cache-from` (§3.2), y puede desaparecer en cualquier momento — el pipeline debe funcionar igual sin él, solo más lento.

Regla mnemotécnica: *artefacto = lo que el build produce y querés conservar; cache = lo que el build ya hizo una vez y no hace falta rehacer.*

### 2.6 Secrets: la configuración que no va en el YAML

Un pipeline que despliega o publica necesita credenciales (tokens, passwords). **Jamás en el YAML**
— tu repo es público y cualquiera lo lee. Para eso las plataformas ofrecen **secrets**: valores
cifrados que se guardan en la configuración del repositorio y se inyectan en runtime como variables
de entorno. En GitHub tienen 3 alcances: repositorio, environment (TP6) y organización.

Dos cosas para tener presentes desde ahora, aunque el pipeline de este TP todavía no necesite
ninguna credencial:

- **El enmascarado cubre el valor literal, y nada más.** La plataforma reemplaza el secret por
  `***` en los logs, pero cualquier transformación —invertirlo, partirlo en dos, codificarlo en
  Base64— lo filtra igual; la documentación de GitHub lo dice explícitamente. El enmascarado es una
  red contra accidentes, no una autorización para loguear secrets.
- **Los logs de un repo público son públicos y permanentes.** Si una credencial real aparece en un
  log, borrar el step no la borra: lo único que la desactiva es **revocarla**.

📌 Tu pipeline de esta semana no necesita ninguna credencial. La primera de verdad aparece cuando el
pipeline empiece a **desplegar** tu app en algún servicio — y ahí la vas a usar como corresponde:
pasándosela al comando que la necesita, sin imprimirla nunca.

### 2.7 El pipeline como gate: cerrando el círculo con el TP1

En el TP1 protegiste `main`: nada entra sin pasar por un PR. Este TP le agrega al PR su verificación automática: **required status checks** — el PR no se puede mergear si el pipeline no está en verde. El PR obligatorio del TP1 garantiza que ningún cambio entre sin pasar por esa puerta; el pipeline garantiza que lo que entra se construye sin errores. La puerta sin verificación no alcanza, y la verificación sin puerta tampoco.

Con esto, el flujo de tu repo queda: rama → PR → **pipeline corre automáticamente** → revisás el diff → merge (solo con el pipeline en ✓). Eso ES integración continua operativa — y es parte de lo que vas a tener que demostrar en vivo en P1, y más adelante en el **trabajo integrador de fin de cursada** (el que habilita a rendir el final).

### 2.8 El badge: el semáforo en la puerta

El **status badge** en el README muestra el estado del último build de `main` en tiempo real. Parece cosmético; es cultura: hace visible el estado del proyecto a cualquiera que pase, y un badge en rojo, a la vista de cualquiera que entre al repo, es una presión sana para arreglarlo.

## 3- Desarrollo de la guía (riel GitHub Actions)

> Trabajás sobre **tu app del semestre** (TP2), en tu repo con las protecciones del TP1. Los ejemplos usan .NET para el back y Node para el front — **adaptá comandos e imágenes a tu stack**; la estructura es lo transferible.
>
> 📬 **Logística**: son **4 PRs** si hacés cada sección por separado (§3.1, §3.2, §3.4 y §3.5 — el gate de §3.3 es configuración del repo, no un PR). Tip válido: apilar §3.1 y §3.2 en un solo PR (el workflow completo de una) los baja a **3**.

### 3.1 Tu primer workflow: que el pipeline construya tu imagen

Tu app ya se construye de una manera: el **Dockerfile del TP2**. El pipeline no va a inventar otra
— va a usar ése. Es la decisión de diseño de este TP y conviene que la puedas defender: si el
pipeline compilara por su cuenta con `dotnet` y `npm`, tendrías **dos definiciones de build** que
tarde o temprano divergen, y estarías verificando una compilación distinta de la que después
desplegás.

> 📌 **Ese archivo ya existe**: lo creaste en el TP3 (§3.4, el PR que cierra una tarea) con un job `build` que sólo hacía el
> checkout, para tener algo que enlazar a una tarea. Lo que sigue **lo reemplaza entero**: el job
> `build` desaparece y en su lugar van `build-backend` y `build-frontend`. Anotalo, porque esos dos
> nombres son los que vas a poner en el gate (§3.3) — y el `build` viejo ya no va a existir.

Mirá de dónde partís y abrí la rama:

```bash
ls backend/Dockerfile frontend/Dockerfile   # los dos del TP2
ls .github/workflows                        # ci.yml, el del TP3
bat --paging=never .github/workflows/ci.yml # siete líneas: sólo hace checkout
git checkout -b feature/ci-backend
```

> 💡 **`bat` en vez de `cat`**, como en el TP2: numera las líneas y colorea el YAML, y en este
> práctico eso importa más que nunca — el desglose de abajo va **por número de línea**, y cuando
> algo falle, GitHub te va a decir en qué línea. Si no lo tenés: `brew install bat` (macOS),
> `apt install bat` (Ubuntu, ahí el comando queda `batcat`), `winget install sharkdp.bat` (Windows).
> Con `cat` funciona igual, sin números ni colores.

Editá `.github/workflows/ci.yml` **en esa rama** (los pipelines también entran por PR):

```yaml
name: CI

on:
  pull_request:
    branches: [main]      # ← el que hace el trabajo: verifica ANTES del merge
  push:
    branches: [main]      # ← sólo para que main tenga su corrida (la que lee el badge, §3.5)

jobs:
  build-backend:
    runs-on: ubuntu-latest
    steps:
      - name: Qué estamos verificando
        env:
          RAMA: ${{ github.head_ref || github.ref_name }}
        run: echo "Rama $RAMA · commit $GITHUB_SHA"

      - name: Bajar el código al runner
        uses: actions/checkout@v6

      - name: Construir la imagen del backend
        uses: docker/build-push-action@v7
        with:
          context: ./backend          # ajustá a donde esté TU Dockerfile
          push: false                 # todavía no publicamos la imagen en ningún lado
          tags: backend:ci            # el nombre de la imagen (hoy sólo vive dentro del runner)
```

Son **cinco claves** y ninguna es de relleno: qué lo dispara, qué job, en qué máquina, qué pasos, y
con qué contexto se construye. El cache llega en §3.2 — primero conviene ver el pipeline andando.

#### El archivo, línea por línea

| Línea | Qué es | Qué pasa si la tocás |
|---|---|---|
| `name: CI` | El nombre del **workflow**: lo que ves en la pestaña *Actions* y en el listado de corridas | Cambiarlo no rompe nada. El badge NO usa este nombre: usa el del **archivo** (`ci.yml`) — si renombrás el archivo, ahí sí se rompe |
| `on:` | **Cuándo** corre. Todo lo que va abajo son disparadores | Sin `on:` el workflow no corre nunca |
| `pull_request:` | Dispara al abrir un PR, al pushear un commit nuevo a su rama, y al reabrirlo | Sin esto no hay check sobre el PR, y **el gate de §3.3 se queda sin nada que exigir** |
| ` branches: [main]` | Sólo para PRs que apuntan a `main` | Un PR hacia otra rama no dispara nada: sus checks required quedan pendientes para siempre |
| `push:` + `branches: [main]` | Dispara cuando algo entra a `main` (o sea, después de cada merge) | Sin esto el badge no tiene una corrida de `main` de la cual leer el estado |
| `jobs:` | Abre la lista de **unidades de trabajo**. Cada una corre en su propia máquina | — |
| `build-backend:` | El **id del job**, elegido por vos. Es el nombre del **check**, y es literal el que vas a escribir en el gate | Si lo renombrás después de cablear el gate, el gate espera un check que ya no existe y bloquea todo |
| `runs-on: ubuntu-latest` | La **máquina** que ejecuta: una Ubuntu limpia que GitHub presta y destruye al terminar | `windows-latest` y `macos-latest` existen y consumen más minutos; para construir imágenes Docker va Linux |
| `steps:` | La secuencia de pasos del job, **en orden**. Si uno falla, los que siguen no corren | — |
| `- name: …` | El **nombre del paso**: el rótulo que vas a leer en la lista de pasos de la corrida. Ojo, no es el `name:` de la línea 1, que nombra el workflow entero | Sin él, GitHub muestra el comando o la action como título (`Run actions/checkout@v6`) |
| `run: echo …` | Un paso **tuyo**: un comando de shell. Los steps son de dos clases, `uses:` (una action del catálogo, hecha por otros) y `run:` (lo que escribís vos) | Este en particular sólo imprime en el log qué rama y qué commit se están verificando |
| `$RAMA`, `$GITHUB_SHA` | El contexto de la corrida. 🔴 Ojo con la rama: en un PR, `GITHUB_REF_NAME` **no** es tu rama, vale `<numero>/merge` — porque GitHub construye una mezcla de tu rama con `main`. Por eso se arma `RAMA` con `github.head_ref` | Si usás `GITHUB_REF_NAME` a secas, el log dice `7/merge` y nadie entiende qué rama es |
| `- uses: actions/checkout@v6` | Trae **tu código** al runner. Sin esto, la máquina está vacía y no hay nada que construir | El build falla diciendo que no encuentra archivos |
| `@v6` | La **versión** de la action. Se fija a propósito | Sin versión (`@main`) tu pipeline cambia solo el día que sus autores publiquen algo |
| `- uses: docker/build-push-action@v7` | La action que **construye la imagen** con tu Dockerfile. El nombre dice *build* **y** *push*, pero hoy usamos sólo la primera mitad: `push: false` apaga la otra | — |
| `with:` | Abre los **parámetros** de la action de arriba (como los argumentos de una función) | — |
| ` context: ./backend` | La carpeta que se le manda a Docker: ahí tiene que estar tu `Dockerfile` | Si apunta mal, el error es `failed to read dockerfile` |
| ` push: false` | **No publicar** la imagen en ningún registry: hoy sólo se verifica que se construya | En `true` pediría credenciales y fallaría |
| ` tags: backend:ci` | El **nombre** que lleva la imagen construida. Hoy vive y muere en el runner | — |

> 🔑 **Lo que NO está y conviene notar**: no hay una sola línea de .NET, de Node ni de Python. El
> workflow no sabe cómo se compila tu app — eso lo sabe **tu Dockerfile**. Por eso este mismo archivo
> le sirve a cualquiera de tus compañeros, sea cual sea su stack.

> 🔍 **El `name:` está en dos niveles y confunde.** El de la línea 1 nombra el **workflow** (lo que
> ves en *Actions*). Los de cada step nombran ese **paso** (lo que ves en la lista de pasos de la
> corrida). Ninguno de los dos cambia el nombre del **check**: ése sale del **id del job**, así que
> el check se va a llamar `build-backend` — que es literal lo que vas a pedir en el gate de §3.3.

Commiteá, pusheá y abrí el PR:

```bash
git add .github/workflows/ci.yml
git commit -m 'ci: build del backend en cada PR'
git push -u origin feature/ci-backend
gh pr create --fill
```

> `--fill` evita el cuestionario interactivo de `gh pr create`: toma el **título y la descripción del
> mensaje del commit**. Por eso el mensaje del commit importa — es el título del PR.

- Andá a la pestaña **Actions**: el workflow ya está corriendo — el evento `pull_request` lo
  disparó solo.
- Explorá la corrida: jobs, steps, y el log del build (vas a ver las etapas de tu Dockerfile).

> 📌 **¿Y si mi app tiene un solo Dockerfile?** Si tu proyecto no está partido en backend y frontend,
> hacés **un** job en vez de dos, el gate exige ese único check, y lo explicás en `decisiones.md`. No
> perdés nada del TP: el paralelismo se evalúa por lo que decidiste y podés justificar, no por
> llegar a dos jobs a la fuerza.

> 📌 **¿Y si mi app no es .NET?** Da igual: el pipeline no sabe qué hay adentro del Dockerfile.
> Esta es la ventaja de construir así — el workflow es el mismo para cualquier stack, y lo que
> cambia es tu Dockerfile, que ya escribiste en el TP2.

> 🔧 **¿Anda en tu máquina y falla en el runner?** Es lo más frustrante de esta semana, y casi
> siempre es una de tres. (1) **Archivos que no viajaron**: el runner clona tu repo, así que lo que
> está en `.gitignore` o nunca commiteaste no existe ahí — mirá el log del `COPY` que falla.
> (2) **Mayúsculas**: macOS y Windows no distinguen `Models/User.cs` de `models/user.cs`; el runner
> es Linux y sí. (3) **Arquitectura**: si tenés una Mac con chip M, tu imagen local es ARM y la del
> runner es x86 — si tu Dockerfile fija una imagen base o un binario de una arquitectura, ahí falla.
> El log te dice el step y la línea del Dockerfile: empezá por ahí, no por el workflow.

**✅ Checkpoint:** el workflow corre automáticamente en tu PR y la imagen se construye en verde.

### 3.2 El frontend en un job paralelo, y el cache de capas

El frontend tiene su propio Dockerfile (TP2), así que es el mismo patrón en un **job aparte** — que
corre en paralelo, en otra máquina:

```yaml
  build-frontend:
    runs-on: ubuntu-latest
    steps:
      - name: Qué estamos verificando
        env:
          RAMA: ${{ github.head_ref || github.ref_name }}
        run: echo "Rama $RAMA · commit $GITHUB_SHA"

      - name: Bajar el código al runner
        uses: actions/checkout@v6

      - name: Preparar el constructor que sabe cachear
        uses: docker/setup-buildx-action@v4

      - name: Construir la imagen del frontend
        uses: docker/build-push-action@v7
        with:
          context: ./frontend
          push: false
          tags: frontend:ci
          cache-from: type=gha,scope=frontend
          cache-to: type=gha,mode=max,scope=frontend
```

> 💡 **Nada de tests todavía**: ni acá ni en el backend. Testing es el TP5 — hoy los dos jobs sólo
> construyen. Desde el TP5, tu Dockerfile va a tener además una etapa de tests y el pipeline va a
> fallar también cuando un test se ponga en rojo.

**El cache, que es lo nuevo acá.** Docker construye tu imagen en **capas**: cada instrucción que toca el sistema de
archivos (`RUN`, `COPY`, `ADD`) deja una — las demás sólo escriben metadatos, como viste en el
TP2 §2.3. Si una capa no cambió, se puede reutilizar en vez de rehacerla — por eso el
Dockerfile del TP2 copia primero los archivos de dependencias y recién después el código: así, si
sólo cambiaste código, la capa que instala dependencias se reutiliza.

El problema es que cada corrida del pipeline arranca sin ninguna capa de tu build. Eso es lo que
resuelven las dos líneas del ejemplo, que tenés que agregar **en los dos jobs** (el del backend de
§3.1 también) junto con el paso de `setup-buildx-action`:

| Línea nueva | Qué es | Qué pasa si falta |
|---|---|---|
| `- uses: docker/setup-buildx-action@v4` | Prepara un **constructor aparte** de Docker, el único que sabe exportar capas a un almacén externo | Sin esto las dos líneas de abajo no hacen nada: nunca vas a ver `CACHED` |
| `cache-from: type=gha` | Al empezar, **trae** las capas guardadas | Sin esto construye todo de cero siempre |
| `cache-to: type=gha,mode=max` | Al terminar, **guarda** las capas | Sin esto nunca hay nada que traer |
| `type=gha` | El almacén: el **cache de GitHub Actions**. No es el Docker de tu máquina ni el del runner | Otros valores (`type=registry`, `type=local`) existen y se usan en otros escenarios |
| `mode=max` | Guarda **todas** las capas, también las intermedias | Con el default (`min`) sólo guarda las de la imagen final: reutiliza mucho menos |
| `scope=backend` / `scope=frontend` | El **estante** donde cada job guarda lo suyo | 🔴 Sin esto los dos jobs comparten estante y **se pisan**: el último en terminar deja su cache y borra el del otro. Resultado: un job muestra `CACHED` y el otro no, y cuál cambia en cada corrida |

Ojo con qué se guarda y dónde, porque se presta a confusión: son **las capas de tu imagen**, y viajan
al **cache de GitHub** (`type=gha`) — no al Docker de tu máquina ni al del runner, que nace vacío en
cada corrida.

> 🔴 **El `scope` no es opcional cuando hay dos jobs, y su ausencia no da error.** Si los dejás sin
> `scope`, los dos usan el mismo por default (`buildkit`) y **se pisan**: la documentación de Docker
> lo dice con todas las letras — *"si construís varias imágenes, cada build sobreescribe el cache de
> la anterior, y queda sólo el último"*. Lo que ves entonces es desconcertante y parece azar: un job
> muestra `CACHED` y el otro no, y **cuál** cambia de una corrida a la otra, según cuál terminó
> último. No está roto tu Dockerfile: están compartiendo estante. Con un `scope` distinto por job,
> los dos reutilizan siempre.
> ([doc](https://docs.docker.com/build/cache/backends/gha/))

**Y por eso hace falta `setup-buildx-action`.** Las capas las guarda el **constructor**: el programa
de Docker que arma la imagen. El que viene de fábrica —el que usaste en el TP2 sin pensarlo— las
guarda en el disco de la máquina y no sabe sacarlas de ahí. Como esa máquina se destruye al terminar
la corrida, guardarlas ahí no sirve de nada. Ese paso pone **otro constructor**, que corre en su
propio contenedor y sí sabe mandar las capas al almacén de GitHub y traerlas de vuelta. Él no
construye nada: deja el constructor listo para los steps que siguen.
>
> 🔴 **Si te lo olvidás, el build FALLA** — no queda callado. El job se pone en rojo en el paso de
> build, con un error que dice que el constructor de fábrica no puede exportar cache. Hoy, textual:
> `Cache export is not supported for the docker driver. Switch to a different driver, or turn on
> the containerd image store, and try again.` Es de los pocos errores que dicen exactamente qué
> falta — y el mensaje puede cambiar de redacción entre versiones, lo que no cambia es el motivo.
>
> *(Detalle fino, por si alguien lo pregunta: el driver de fábrica **sí** puede exportar cache si
> el demonio tiene habilitado el* containerd image store*. En los runners de GitHub hoy no lo está,
> así que el paso hace falta.* [Docker — Cache
> backends](https://docs.docker.com/build/cache/backends/)*)*

Corré el workflow **dos veces**: en la segunda vas a ver `CACHED` en las capas que no cambiaron.

> ⏱️ **No mires el cronómetro, mirá el log.** Es tentador esperar que la segunda corrida tarde
> menos, y con una app del tamaño de la de la materia **no va a pasar**: puede tardar lo mismo o
> más. Guardar el cache también cuesta —al terminar, la corrida sube todas las capas al almacén— y
> cada corrida cae en una máquina distinta. Medido en el repo de la demo: primera corrida 44s el
> backend, segunda 75s, con nueve capas reutilizadas. El cache paga cuando construir es caro de
> verdad (instalar cientos de dependencias, compilar algo grande); en un proyecto de la materia la
> ganancia es chica. **La evidencia que se pide es la palabra `CACHED`**, no el tiempo.

El commit y el push van más abajo, cuando esté el archivo completo. Lo que importa acá es **cómo**
se ven las dos corridas, porque tienen que ser **una después de la otra, no las dos juntas**:

```bash
# 1) el cambio de esta sección, a la misma rama del PR
git push

# 2) ESPERÁ a que esa corrida TERMINE (el cache se sube recién al final).
#    Recién entonces, un commit vacío para disparar la segunda:
git commit --allow-empty -m 'ci: segunda corrida para ver el cache'
git push
```

> 🔴 **Si pusheás los dos seguidos no vas a ver `CACHED`, y no es un error tuyo**: las corridas se
> solapan y cuando la segunda empieza a construir, la primera todavía no terminó de subir su cache.
> `--allow-empty` es lo que deja hacer un commit sin ningún cambio: sirve justamente para disparar
> una corrida nueva sin tocar el código.

> 🔍 **Cada job va a reutilizar una cantidad distinta de capas, y puede que alguno no reutilice
> ninguna.** Es normal: una capa se reutiliza sólo si nada de lo que ella depende cambió, y eso lo
> decide **cómo está escrito tu Dockerfile** (qué se copia primero, qué después) junto con qué
> cambiaste desde la corrida anterior. Si un job no muestra `CACHED` no está roto — es lo que el
> orden de ese Dockerfile permite.

> ⚠️ **Las dos corridas tienen que ser del MISMO PR** (pusheá otro commit a la misma rama), y el
> motivo es el **alcance** del cache. Una corrida puede traer capas guardadas por **su propia rama**
> y por la **rama base** del PR (normalmente `main`); lo que **no** ve es lo que guardaron otras
> ramas ni otros PRs. Y lo que guarda un PR queda atado a ese PR: sólo lo reutilizan las corridas
> siguientes **de ese mismo PR**.
>
> Hoy tu `main` todavía no guardó nada, así que la única forma de ver `CACHED` es que las dos
> corridas sean del mismo PR. Cuando mergees, la corrida del `push` a `main` **sí** deja el cache
> ahí — y a partir de entonces cualquier PR nuevo lo aprovecha ya en su primera corrida. Por eso el
> workflow también corre en `main`: además de darle estado al badge, es la corrida que deja el cache
> para todos.
>
> ([Fuente: docs de GitHub, *Dependency caching reference* — «Workflow runs can restore caches
> created in either the current branch or the default branch… cannot restore caches created for
> child branches or sibling branches».](https://docs.github.com/en/actions/reference/dependency-caching-reference))

> 🔑 **La propiedad que hay que entender** (está entre las preguntas de ejemplo del enunciado): el cache **puede
> desaparecer** — la plataforma lo desaloja cuando quiere, y tiene límite de tamaño. Tu pipeline
> tiene que funcionar exactamente igual sin él, sólo que más lento. Si **falla** sin cache, no
> tenías un cache: tenías una dependencia escondida, y eso es un bug.

**El archivo completo**, como te queda al terminar esta sección. Copiá de acá si te perdiste en el
camino, y fijate que buildx y las dos líneas del cache están en **los dos** jobs:

```yaml
name: CI

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  build-backend:
    runs-on: ubuntu-latest
    steps:
      - name: Qué estamos verificando
        env:
          RAMA: ${{ github.head_ref || github.ref_name }}
        run: echo "Rama $RAMA · commit $GITHUB_SHA"

      - name: Bajar el código al runner
        uses: actions/checkout@v6

      - name: Preparar el constructor que sabe cachear
        uses: docker/setup-buildx-action@v4

      - name: Construir la imagen del backend
        uses: docker/build-push-action@v7
        with:
          context: ./backend
          push: false
          tags: backend:ci
          cache-from: type=gha,scope=backend
          cache-to: type=gha,mode=max,scope=backend

  build-frontend:
    runs-on: ubuntu-latest
    steps:
      - name: Qué estamos verificando
        env:
          RAMA: ${{ github.head_ref || github.ref_name }}
        run: echo "Rama $RAMA · commit $GITHUB_SHA"

      - name: Bajar el código al runner
        uses: actions/checkout@v6

      - name: Preparar el constructor que sabe cachear
        uses: docker/setup-buildx-action@v4

      - name: Construir la imagen del frontend
        uses: docker/build-push-action@v7
        with:
          context: ./frontend
          push: false
          tags: frontend:ci
          cache-from: type=gha,scope=frontend
          cache-to: type=gha,mode=max,scope=frontend
```

> 💡 **Para releerlo, cincuenta líneas no entran en una pantalla.** `bat` sabe mostrar un tramo
> conservando la numeración real, que es lo que hace falta cuando alguien te dice «mirá la línea 41»:
>
> ```bash
> bat --paging=never --line-range 33:54 .github/workflows/ci.yml   # el job del frontend
> bat --paging=never --line-range 1:31  .github/workflows/ci.yml   # el del backend
> ```

Ahora sí, commiteá y pusheá (es **el** commit de esta sección — el de arriba era el mismo).
**Si venís del PR de §3.1 y todavía no lo mergeaste**, alcanza con pushear a la misma rama: el PR se
actualiza solo y el pipeline vuelve a correr.

```bash
git add .github/workflows/ci.yml
git commit -m 'ci: frontend en paralelo y cache de capas'
git push
```

**Si ya mergeaste el de §3.1**, esto va en un PR nuevo:

```bash
git checkout main && git pull
git checkout -b feature/ci-frontend
git add .github/workflows/ci.yml
git commit -m 'ci: frontend en paralelo y cache de capas'
git push -u origin feature/ci-frontend
gh pr create --fill
```

> 💡 **Para ver el cache funcionando hacen falta dos corridas.** La primera construye todo desde
> cero y guarda las capas; la segunda es la que las reutiliza. **Esperá a que la primera termine**
> —el cache se sube al final— y recién ahí pusheá cualquier cambio chico a la misma rama: un renglón
> en el README, o `git commit --allow-empty -m 'segunda corrida'`. Si pusheás los dos seguidos, las
> corridas se solapan y la segunda no reutiliza nada.
>
> No hace falta abrir otro Pull Request: **el PR sigue la rama, no el commit**. Todo lo que pushees
> mientras esté abierto entra solo, y el pipeline vuelve a correr.

> 🔧 **¿Un job dice `CACHED` y el otro no?** Y peor: en la corrida siguiente se dan vuelta. No es
> tu Dockerfile — es el `scope` compartido, que está explicado arriba. Revisá que cada job tenga el
> suyo, distinto del otro.

**✅ Checkpoint:** dos jobs en paralelo, ambos verdes, y la segunda corrida reutilizando capas.

> 🔧 **Si el workflow no aparece en la pestaña Actions**, no está fallando: GitHub no lo está
> leyendo. Casi siempre es una de estas dos, en este orden:
> 1. **La ruta**: tiene que ser exactamente `.github/workflows/ci.yml` — con el punto adelante, en
>    la **raíz** del repo, y `workflows` en plural.
> 2. **La rama**: el workflow tiene que estar en la rama del Pull Request. Si lo escribiste y no lo
>    pusheaste, no existe para GitHub.
>
> 🔧 **Si aparece pero falla al instante, sin llegar a construir nada**, es el YAML: un espacio de
> más o de menos lo rompe. GitHub **no** lo ignora — crea una corrida fallida por cada commit y te
> muestra el error del archivo. Abrí la corrida y leelo, o pegá el archivo en el editor web de
> GitHub, que te marca la línea.

### 3.3 El pipeline como gate del PR

Convertí el pipeline en **requisito de merge** (se suma al PR obligatorio del TP1). Hay dos caminos
para lo mismo, y conviene leer los avisos antes de elegir.

> 💡 **El camino recomendado es la web**, el mismo que usaste en el TP1: *Settings → Branches →*
> editar la regla de `main` → tildar *Require status checks to pass before merging*, buscar
> `build-backend` y `build-frontend`, y tildar también *Require branches to be up to date* (eso es
> `strict`). Toca sólo lo que tocás, y no te puede borrar nada del TP1. El comando de más abajo es
> el camino rápido para el que ya se siente cómodo.

> 🔴 **El buscador sólo ofrece checks que corrieron en los últimos 7 días** (lo dice la propia
> pantalla: *"Search for status checks in the last week for this repository"*). Si intentás configurar
> el freno antes de la primera corrida, `build-backend` y `build-frontend` no aparecen y parece que
> algo está mal. No lo está: corré el workflow una vez (§3.1) y volvé. Es el orden de esta guía.

> 🔴 **Ojo con el número de approvals.** Va en **0**, igual que en el TP1: como el trabajo es
> individual y GitHub **nunca** te deja aprobar tu propio Pull Request, si ponés 1 no vas a poder
> mergear nunca más. Lo que bloquea el merge en este TP no es una aprobación humana: es el
> **pipeline en verde** (los `required_status_checks`).

Si vas por el comando, **leé esto antes de correrlo**:

> 🔴 **El PUT REESCRIBE la protección entera, no le agrega una línea.** Todo campo omitido vuelve a
> su default, así que todo lo que configuraste en el TP1 y no esté en este JSON **se pierde** — por
> eso el cuerpo de abajo también re-declara lo del TP1 (cero approvals + `enforce_admins`). Mirá
> primero cómo está la tuya (`gh api "repos/{owner}/{repo}/branches/main/protection"`) y si tiene
> algo más, agregalo al JSON o hacelo por la web.

```bash
gh api --method PUT "repos/{owner}/{repo}/branches/main/protection" \
  --input - <<'EOF'
{
  "required_pull_request_reviews": { "required_approving_review_count": 0 },
  "required_status_checks": {
    "strict": true,
    "contexts": ["build-backend", "build-frontend"]
  },
  "enforce_admins": true,
  "restrictions": null
}
EOF
```

Y para leerla sin ahogarte en JSON, antes y después:

```bash
gh api "repos/{owner}/{repo}/branches/main/protection" \
  --jq '{pr: .required_pull_request_reviews, admins: .enforce_admins.enabled}'
gh api "repos/{owner}/{repo}/branches/main/protection" --jq '.required_status_checks'
```

> ⚠️ **Qué dicen esas dos claves.** `strict: true` exige además que la rama esté actualizada con
> `main` antes de mergear. Los `contexts` son el **nombre visible del job**: el id (`build-backend`)
> sólo vale mientras el job no defina `name:` — si después le ponés `name: Build Backend`, el gate
> queda esperando un check que ya no existe y bloquea todo. (Nota fina: la API marca `contexts` como
> legacy a favor de `checks: [{"context": "…"}]` — ambos funcionan hoy; usamos `contexts` por
> simplicidad.)

> ⚠️ **Un check obligatorio que nunca corre bloquea el PR para siempre.** Tu workflow se dispara con
> `pull_request: branches: [main]`, así que un PR que apunte a **otra rama** no lo dispara — y los dos
> checks quedan en *Expected*, esperando una corrida que no va a llegar, sin mensaje de error útil. En
> este TP todos los PRs van contra `main`, así que no te va a pasar; pero cuando lo veas, ya sabés qué
> es. (Lo mismo aplica si más adelante le agregás filtros de `paths:` al workflow.)

> 🔧 **Si te trabaste**: activaste el gate con un job que todavía no existe (por ejemplo, exigiste
> `build-frontend` antes de escribirlo) y ahora ningún PR se puede mergear. La salida es volver a
> correr la configuración **sin ese context** (o destildarlo en la web) hasta que el job exista. No
> hace falta borrar la protección entera.

**✅ Checkpoint:** en un PR nuevo, la sección de checks muestra los dos jobs como **Required**, y el botón de merge queda bloqueado hasta que estén en verde.

### 3.4 Romper el build (a propósito) y ver el gate actuar

> 🔴 **Antes de empezar: mergeá los PRs anteriores.** El pipeline que corre sobre un PR es el que
> queda al combinar tu rama con `main`. Si el PR del workflow sigue abierto, en `main` todavía está
> el `ci.yml` esqueleto del TP3 — y la rama que vas a romper sale de ahí: **corre el workflow viejo,
> el que sólo hace checkout, y da verde**. Vas a ver un tilde verde sobre código que no compila y no
> vas a entender por qué. (Nos pasó grabando el video, y costó un rato darse cuenta.)

La verificación central del TP — el pipeline sirviendo para algo:

1. En una rama, **rompé el build a propósito**. Tiene que ser un error que frene la construcción de
   **tu imagen**, y cuál sirve depende de tu stack:

   | Tu app | Cómo romperla |
   |---|---|
   | **Compila** (C#, Java, Go…) | Importá algo que no existe: `using NoExiste;` en un `.cs`. También sirve borrar un punto y coma |
   | **Se empaqueta** (React/Vite, Angular, Vue…) | Un import a un archivo inexistente: `import x from './no-existe'` en un archivo que sí se usa. El bundler lo resuelve **durante el build** y falla |
   | **Ni compila ni empaqueta** (Python, Express sin build, PHP, Ruby…) | Tu Dockerfile nunca mira tu código, así que romperlo no cambia nada. Rompé **las dependencias**: agregá un paquete que no existe a `requirements.txt` o `package.json`, y el `pip install` / `npm ci` del Dockerfile falla |

   > 🔴 **Si tu app no compila ni empaqueta, leé esto antes de perder una tarde.** Podés escribir
   > `import estonoexiste` en cualquier archivo y el `docker build` **va a dar verde igual**: durante
   > la construcción nadie ejecuta tu código. No configuraste nada mal — es cómo funciona tu stack.
   > Por eso ahí se rompe la dependencia, no el código.

   Abrí el PR.
   ```bash
   git checkout main && git pull && git checkout -b feature/demo-gate
   echo '// TODO: endpoint de salud' >> backend/DemoApi/Program.cs   # un cambio de verdad
   echo 'using NoExiste;' >> backend/DemoApi/Program.cs             # y la rotura a propósito
   tail -3 backend/DemoApi/Program.cs
   docker build ./backend        # comprobá que falla también en tu máquina (comando del TP2)
   git commit -am 'demo: rompe el build a propósito'
   git push -u origin feature/demo-gate
   gh pr create --fill
   ```

2. Mirá el pipeline fallar y el merge bloqueado: **ese PR es la evidencia central del TP.** Alcanza
   con romper **uno** de los dos jobs: un solo check en rojo ya bloquea el merge. Fijate
   dónde falla: en el step que construye la imagen, y el log te dice exactamente qué línea.
3. Arreglá el error con otro commit; el pipeline re-corre solo; el PR se destraba.

   ```bash
   grep -v 'using NoExiste;' backend/DemoApi/Program.cs > /tmp/p.cs && mv /tmp/p.cs backend/DemoApi/Program.cs
   git commit -am 'fix: saca el using que no existe'
   git push
   ```

   > **Ese primer comando es sólo una forma de borrar una línea sin abrir un editor** — `grep -v`
   > deja pasar todas las líneas *menos* la que le nombrás, y el resultado reemplaza al archivo. Está
   > escrito así porque en el video no se puede filmar un editor de texto. **Abrí el archivo y borrá
   > la línea a mano: es exactamente lo mismo.** Igual con el `echo … >>` que la agregó: eso sólo
   > agrega texto al final del archivo.

4. **Antes de mergear**, dejá abierto un segundo Pull Request (el bloque de acá abajo). Después sí:
   abrí *Files changed*, leé tu propio diff entero, y **recién ahí** mergeás. El gate verifica que
   compile; que el cambio tenga sentido lo verificás vos (regla cultural 2 de §2.1).

> 📌 **Sí, este PR se mergea** — mergear no borra nada: el PR queda en el historial con sus corridas,
> sus checks en rojo y en verde y sus commits, que es exactamente la evidencia. Lo que no hay que
> hacer es **cerrarlo sin mergear** ni borrar la rama antes de que quede el registro.

**Antes de mergear, dejá abierto un segundo Pull Request.** Cualquiera sirve; es de relleno:

```bash
gh pr list                          # mirá los que ya tenés abiertos
git checkout main && git pull
git checkout -b docs/muestra-del-freno
echo '' >> README.md
git commit -am 'docs: muestra del freno'
git push -u origin docs/muestra-del-freno
gh pr create --fill
git checkout feature/demo-gate     # volvés al PR de la demostración
```

> 🔑 **Para qué**: cuando mergees el PR de la rotura, volvé a este otro. Va a decir que está
> desactualizado y va a aparecer un botón **Update branch** — su verde quedó viejo, porque se sacó
> contra un `main` que ya no existe. Eso es *Require branches to be up to date* (§3.3), y **con un
> solo Pull Request abierto no se puede ver**: hacen falta dos al mismo tiempo. Sacale una captura.
> Después apretá *Update branch*, mirá correr el pipeline sobre la mezcla, y mergealo o cerralo.

> 💡 **¿Y por qué no romper un test?** Porque testing es el TP5, y todavía no lo dimos. Desde el
> TP5 tu pipeline va a fallar también por un test en rojo — el mecanismo del gate es el mismo, sólo
> cambia qué lo hace fallar.

**✅ Checkpoint:** evidencia de la secuencia completa rojo → bloqueado → fix → verde → merge.

### 3.5 El badge en el README

```markdown
[![CI](https://github.com/<owner>/<repo>/actions/workflows/ci.yml/badge.svg)](https://github.com/<owner>/<repo>/actions/workflows/ci.yml)
```

> 🔴 **Son dos direcciones, no una, y por eso los corchetes de afuera.** La de
> adentro es la **imagen** (`…/badge.svg`); la de afuera es **adónde te lleva al
> hacerle clic** (el historial de ese workflow). Si escribís sólo la imagen
> —`![CI](…badge.svg)`— el badge se ve igual, pero al clickearlo abrís el SVG
> suelto: una página en blanco. Es el error más fácil de cometer acá, porque no
> se nota mirando el README.
>
> GitHub te da la línea armada, ya con tu usuario y tu repo adentro: *Actions → tu workflow → ⋮ →
> Create status badge*. El botón verde la copia al portapapeles.
>
> ⚠️ **Ojo con el orden.** A esta altura `main` ya tiene el gate puesto, así que la línea **no entra
> con un commit directo**: editás el README, commiteás **en una rama nueva**, abrís el PR y esperás
> los dos checks en verde. El badge también pasa por el mismo camino que todo lo demás.
>
> 📌 **Se escribe UNA vez.** Después no se toca nunca más: la imagen se actualiza sola con cada
> corrida de `main`. Automatizar la escritura de esa línea sería peor que hacerla a mano — te
> llenaría el historial de commits para conseguir lo que ya tenés.

**✅ Checkpoint:** el badge se ve en el README, muestra el estado real del último build de `main`, y
**al hacerle clic te lleva al historial de corridas**.

## 4- Riel alternativo: Azure Pipelines

Mismos conceptos y checkpoints; mapa de equivalencias:

| Concepto | GitHub Actions (canónico) | Azure Pipelines |
|---|---|---|
| Archivo | `.github/workflows/ci.yml` | `azure-pipelines.yml` (raíz del repo) |
| Estructura | workflow → jobs → steps | pipeline → stages → jobs → steps |
| Trigger de PR | `on: pull_request` | `pr:` — ⚠️ **si tu código está en Azure Repos, `pr:` se ignora**: la validación se configura en las *branch policies* de la rama destino |
| Construir la imagen | `docker/build-push-action@v7` | task `Docker@2` (o `DockerCompose@0`) |
| Runner | `runs-on: ubuntu-latest` | `pool: vmImage: ubuntu-latest` |
| Artefactos *(desde el TP5)* | `actions/upload-artifact@v6` | `PublishPipelineArtifact@1` |
| Cache de capas | `cache-from`/`cache-to: type=gha` | `Cache@2` + `--cache-from` en la task Docker |
| Secrets *(más adelante)* | Settings → Secrets / `gh secret set` | Library → Variable groups (lock) o variables secretas del pipeline |
| Reporte de tests *(TP5)* | Artefacto .trx | `PublishTestResults@2` (pestaña Tests nativa — punto fuerte de ADO) |
| Gate del PR | Required status checks (branch protection) | **Build validation** en Branch policies |
| Badge | `…/badge.svg` | Status badge del pipeline (⋮ → Status badge) |
| Minutos gratis | Ilimitado en repos públicos, sin tarjeta | 1800 min/mes — el grant exige **org vinculada a una suscripción Azure** (tarjeta; *Azure for Students* debería servir — verificar). Sin tarjeta: agente **self-hosted** (1 job paralelo gratis). Desde abr-2026 no se crean proyectos públicos nuevos |

**Checkpoints riel Azure:** los mismos 5 (pipeline corre en PR → los dos builds en paralelo con cache → build validation activa → rojo-bloqueado-fix-verde → badge). Recomendación 2026 para este riel: **código en GitHub (repo público) + Azure Pipelines como CI** — conservás el entregable de repo público. (Nota del combo híbrido: en ese caso el gate del PR es el de **GitHub** — required status checks sobre el check que reporta Azure Pipelines — no la Build validation policy, que aplica solo con Azure Repos.)

> 📌 **El riel garantizado sin tarjeta es GitHub Actions** (ilimitado en repos públicos). En Azure Pipelines, los minutos hosted hoy requieren suscripción vinculada (tarjeta) — la alternativa sin tarjeta ahí es el agente self-hosted.

---
---

# 📋 Trabajo Práctico 04 – CI: Pipelines as Code (2026)

## ⚠️ Este es el TP que debés entregar y defender

## 🎯 Objetivo

Automatizar la verificación de tu app del semestre: cada push y cada PR construyen tus imágenes, con el pipeline como **requisito obligatorio de merge**.

Este trabajo se aprueba **solo si podés explicar qué hiciste, por qué lo hiciste y cómo lo resolviste**.

## 🧩 Escenario

Ya tenés la app contenerizada (TP2), pero la verificación sigue siendo artesanal: cada uno construye y prueba en su máquina "cuando se acuerda", y ya se coló un cambio que rompió `main` un viernes. Como líder técnico, decidís que **ninguna línea vuelve a entrar a `main` sin que las máquinas la verifiquen primero**. Tu trabajo: dejar la integración continua operativa y blindada.

## 📋 Tareas que debés cumplir

### 1. Workflow como código
- Workflow en el repo (entró por PR, como todo) que corre en **cada PR y cada push a `main`**.

### 2. Build de las dos imágenes, en paralelo
- Build del **backend y del frontend** **con tus Dockerfiles del TP2** (jobs separados, en paralelo). El pipeline no compila por su cuenta: usa la misma definición de build que después se despliega.

> 📌 **¿Tu app tiene un solo Dockerfile?** Entonces se entrega con **un solo job**, y no se
> descuenta nada. Lo que se evalúa es que el pipeline construya **lo que tu app tiene**, no que
> haya dos jobs porque sí. Decilo en `decisiones.md` en una línea y listo. Lo que **no** vale es
> inventar un segundo job vacío para llegar al número.

> 📌 **En este TP el pipeline no corre tests**: testing es el TP5, y todavía no se dio. Acá el
> pipeline verifica que tu app **compila** en una máquina limpia — que ya es más de lo que tenías.

### 3. Cache de capas
- **Cache de capas** funcionando: una segunda corrida que reutilice lo que no cambió — se ve `CACHED` en el log del build.

### 4. El pipeline como gate
- **Required status checks** activos sobre `main`: sin pipeline verde no hay merge (el PR obligatorio del TP1 sigue vigente).
- **Demostración del gate actuando**: secuencia documentada de compilación rota → PR bloqueado → fix → pipeline verde → merge.

### 5. Visibilidad
- **Status badge** del workflow en el README.

## 📄 Entregables


> 🏷️ **Cerrá el práctico con su tag y su release**, como hiciste en el TP1. La numeración sigue el
> número del práctico: **TP4 → `v4.0.0`**.
>
> ```bash
> git tag -a v4.0.0 -m "TP4 cerrado"
> git push origin v4.0.0
> ```
>
> Y la release desde la web —*Releases → Draft a new release*—, eligiendo ese tag, con el título
> `v4.0.0` y una descripción de qué incluye. Cada práctico queda así con su **estado congelado**:
> podés volver a él si rompés algo, y en la defensa se navega el punto exacto en el que cerraste.
1. **URL del repositorio público** (se carga en el formulario de la cátedra — el link está en el README del repositorio de la materia) con el workflow, el historial de corridas y los PRs.
2. **`decisiones.md`** (en la raíz, se acumula sobre el de TPs anteriores) explicando:
   - Estructura elegida del pipeline (¿por qué esos jobs? ¿por qué en paralelo?).
   - Qué cachea tu pipeline (capas: cuáles se reutilizan y cuáles no) y qué pasa si el cache desaparece.
   - Por qué el pipeline construye con tu Dockerfile en vez de compilar por su cuenta.
   - Problemas encontrados y cómo los resolviste.
   - Declaración de uso de IA.

> 📌 **En este TP no hay `evidencias.md`.** Tu repositorio es público: quien corrige abre la pestaña
> *Actions* y ve las corridas y el cache reutilizado; abre el PR y ve
> los checks *Required* bloqueando y la secuencia de rojo a verde; y ve el badge al entrar. Sacar
> capturas de lo mismo sería duplicar lo que ya se ve — igual que en el TP3.

## 🗣️ Defensa Oral Obligatoria

Se realiza en **P1 (clase 5)**, junto con los TPs 1 a 3. Vas a mostrar tu trabajo y responder preguntas como:
- ¿Qué es integración continua? ¿Puede haber CI sin pipeline? ¿Y pipeline sin CI?
- ¿Qué dispara tu workflow y por qué elegiste esos triggers? ¿Qué diferencia hay entre correr en `push` y en `pull_request`?
- ¿Por qué tus jobs corren en paralelo? ¿Qué NO comparten dos jobs?
- ¿Qué produce tu pipeline y dónde queda? ¿Qué es el cache y qué pasa si desaparece?
- ¿Por qué tu pipeline construye con el Dockerfile en vez de compilar por su cuenta? ¿Qué problema evita eso?
- Mostrame el PR donde el gate bloqueó un merge. ¿Qué dos condiciones exige tu `main` hoy para aceptar un merge?
- ¿Qué significa `strict: true` en tus status checks?
- Si mañana migrás a Azure Pipelines, ¿qué conceptos de tu workflow sobreviven y qué cambia?

## ✅ Evaluación

| Criterio | Peso |
|---|---|
| Configuración técnica (workflow, gate, cache — reproducible en tu repo) | 25% |
| Claridad y justificación en `decisiones.md` | 25% |
| Defensa oral: comprensión y argumentación | 50% |

> ⚖️ Peso orientativo de este TP en la nota de **P1**: **45%** (la ponderación completa de los 9 TPs está en el reglamento, §5).

## ⚠️ Uso de IA

Podés usar IA (ChatGPT, Copilot, Claude), pero **deberás declarar en `decisiones.md` qué parte fue asistida por IA** y justificar cómo la verificaste. Si no podés defenderlo, **no se aprueba**.
