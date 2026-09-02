# ⚠️ IMPORTANTE – Guía de Práctica Sugerida

Este documento tiene **dos partes**:

1. **La guía paso a paso** (primera parte): lo que vas a hacer, con los comandos y las capturas.
   🔴 **Y acá, atención: en este práctico lo que la guía hace ES lo que se entrega** — al revés
   del TP2, donde era un ejemplo. Reproducís esto mismo sobre tu repositorio y tu proyecto.
2. **El Trabajo Práctico** (al final): escenario, tareas, entregables y defensa oral, con la
   lista completa de lo que se corrige.

## 📦 Un solo repositorio para todo el semestre

**Todos los prácticos de la materia se hacen sobre el mismo repositorio: el que creaste en el TP1.** No se crea uno nuevo por práctico, y no se arranca de cero en cada uno — cada TP agrega una capa sobre lo que ya está.

Cada TP se cierra con su **tag y su release**, y el número mayor es el número del práctico: **TP1 → `v1.0.0`, TP2 → `v2.0.0`, TP3 → `v3.0.0`**, y así hasta el TP9. Así cada entrega queda con su **estado congelado**: en la defensa se navega el punto exacto en el que cerraste cada una, y podés volver a cualquiera con `git checkout v2.0.0`.

Los archivos `decisiones.md` y `evidencias.md` también son **únicos**: no se rehacen por práctico — se les agrega abajo la sección del TP nuevo.

## Sobre las herramientas en este TP

- La guía usa **GitHub Projects + Issues** (riel canónico).
- El riel **Azure Boards** está cubierto con la tabla de equivalencias del punto 4 — y en este tema en particular, Azure Boards es la herramienta **más completa** de las dos (jerarquía nativa de work items, sprints, queries). Si pensás rendir AZ-400 o querés conocer la herramienta más usada en empresas .NET, este es un buen TP para elegir el riel Azure.
- 📌 **Ninguno de los dos rieles requiere tarjeta de crédito** (GitHub gratis; Azure DevOps gratis hasta 5 usuarios, sin subscription de Azure).

---

# Guía Paso a Paso – Planificación ágil en una plataforma DevOps (Práctica sugerida)

## 1- Objetivos de Aprendizaje

- Entender qué resuelve una **plataforma DevOps/ALM** más allá del repositorio: planificar, ejecutar y trazar el trabajo en un solo lugar.
- Organizar el trabajo con una **jerarquía**: Épica → Historia de usuario → Tarea (+ Bugs).
- Configurar un **sprint** y un **board** para visualizar el flujo.
- Lograr **trazabilidad**: desde el requerimiento hasta el Pull Request que lo implementa.

## 2- Marco teórico

### 2.1 Por qué el plan y el código tienen que vivir juntos

Git guarda el código, pero un proyecto real necesita más: qué hay que hacer (backlog), quién lo está haciendo (board), cuándo (sprints), y cómo se conecta cada cambio de código con el requerimiento que lo originó. Históricamente eso vivía en herramientas separadas — el plan en un Excel o un Trello, el código en el repo — y la conexión entre ambos dependía de la memoria de la gente. Resultado conocido: el plan decía una cosa, el código otra, y nadie podía responder "¿esto ya está hecho?" sin preguntar en el pasillo.

Las plataformas DevOps (GitHub, Azure DevOps, GitLab) integran la gestión del trabajo **alrededor del repo**, y esa integración no es cosmética: es lo que hace posible el enlace automático. Un PR puede **cerrar la historia que implementa** al mergearse; una historia muestra en su historial **qué commits la materializaron**. La diferencia de fondo con un Trello suelto es exactamente esa: acá el plan y el código son **datos enlazados del mismo sistema**, no dos mundos sincronizados a mano.

En el ciclo DevOps de la clase 1, esto es la fase **Plan** — y su conexión con Code. Una precisión sobre métricas, para no pisar lo aprendido: el *lead time for changes* de DORA (el de la clase 1) se mide de **commit a producción** — ese se mide sin backlog. Lo que el plan enlazado habilita es medir la cadena **completa**: desde que el trabajo se decide hasta que está en producción — el tramo que a tu cliente le importa, y que sin enlace plan↔código es inmedible.

### 2.2 La jerarquía de trabajo: del objetivo a la tarea

> 📌 **Una aclaración honesta antes de la tabla**: esta jerarquía de tres niveles es la
> **convención más extendida de la industria** —viene de Jira y de SAFe, y la adoptaron casi todas
> las herramientas—, pero **no es parte de Scrum**: Scrum sólo define el Product Backlog y sus
> ítems, sin niveles. Que sea una convención y no un estándar no la hace menos útil; lo importante
> es que sepas de dónde viene si alguien te lo discute.

| Nivel | Qué representa | Tamaño típico | Ejemplo (de esta materia) |
|---|---|---|---|
| **Épica** | Un objetivo grande, que agrupa valor relacionado | Semanas/meses | "Pipeline DevOps completo para mi app" |
| **Historia de usuario** | Un incremento de valor **observable por alguien**, en lenguaje de usuario | Días | "Como desarrollador quiero que cada PR corra los tests automáticamente para detectar regresiones antes del merge" |
| **Tarea** | Trabajo técnico concreto dentro de una historia | Horas | "Escribir el workflow de build" |
| **Bug** | Comportamiento incorrecto de algo que ya existe | — | "El compose no espera a que la BD esté lista" |

La jerarquía no es burocracia: es **zoom**. El cliente/product owner razona a nivel épica-historia ("¿qué valor entregamos?"); el equipo ejecuta a nivel tarea ("¿qué hago hoy?"). Cada nivel responde una pregunta distinta, y la trazabilidad entre niveles permite navegar del "por qué" al "cómo" y viceversa.

### 2.3 Historias de usuario que sirven (y las que no)

El formato *"Como [rol] quiero [capacidad] para [beneficio]"* no es un mantra: cada parte trabaja. El **rol** te obliga a saber para quién es; la **capacidad** define el qué sin fijar el cómo; el **beneficio** justifica la prioridad (si no podés escribir el "para", quizás no haya que hacerla).

Una buena historia además cumple **INVEST**: **I**ndependiente (se puede hacer sola), **N**egociable (es una conversación, no un contrato), **V**aliosa (alguien la quiere), **E**stimable (se entiende lo suficiente para dimensionarla), **S**mall (cabe en días, no semanas), **T**esteable (se puede verificar). Las dos que más se violan en la práctica: *Small* (historias-elefante que nunca terminan) y *Testeable* — que se resuelve con la pieza clave:

**Criterios de aceptación**: la lista concreta y verificable de condiciones que hacen que **esa** historia esté "hecha". *"- [ ] El workflow corre en cada PR a main; - [ ] Un test que falla bloquea el merge"*. Son el acuerdo entre quien pide y quien hace — y en esta materia, el puente directo con el testing (TP5) y con la defensa oral: una historia sin criterios de aceptación no se puede demostrar terminada.

**Definition of Done**: no la confundas con lo anterior, y es una pregunta clásica de entrevista. Los criterios de aceptación son **de cada historia**; la *Definition of Done* es la lista que vale **para todas** — lo que tu equipo considera mínimo para dar cualquier cosa por terminada (por ejemplo: revisada por otro, con tests, mergeada a `main`, desplegada en el ambiente de prueba). En esta materia la Definition of Done se va armando sola, práctico a práctico: lo que hoy es "entró por pull request" en el TP4 suma "con el pipeline en verde", y en el TP5 "con la cobertura sobre el umbral".

Anti-patrones que vas a ver (y evitar): la historia-tarea disfrazada ("Como desarrollador quiero crear la tabla usuarios" — eso es una tarea, nadie 'quiere' una tabla), la historia sin beneficio, y la historia-épica de tres semanas.

### 2.4 Sprints, y por qué limitar el trabajo en progreso

- **Sprint**: una iteración de duración fija —en la industria vas a ver de una a cuatro semanas— con un **objetivo comprometido**
  (el *Sprint Goal*). Ojo con esto, que es lo que más se dice mal: lo que el equipo se compromete
  a cumplir es el OBJETIVO; la lista de items es un **pronóstico** que se renegocia si hace falta.
- **Límite de trabajo en progreso** (*WIP limit*, por *work in progress*): un máximo de tarjetas simultáneas en la columna «en progreso». Viene de **Kanban**, el método de gestión visual de flujo que a su vez lo tomó de la producción industrial japonesa.
  Cuando está llena, no entra nada nuevo hasta que salga algo. Ojo: eso es el **acuerdo del
  equipo**, no un candado de la herramienta — GitHub te avisa poniendo el contador en rojo, pero
  te deja pasar (§3.3). No es una regla arbitraria: es la
  traducción operativa de la idea central de la materia que ya viste en la clase 1 — **empezar
  menos, terminar más**. El trabajo empezado y no terminado no es productividad: es inventario, y
  el inventario tiene costo (más cambio de contexto, más ramas viejas, más conflictos al integrar).

La duración del sprint **la elige el equipo** — acá, vos. Para este TP conviene alinearla con el calendario de entregas de la materia; la elección, con su porqué, va en `decisiones.md`. Lo importante es que puedas explicar por qué elegiste ese número.

### 2.5 Trazabilidad: la conexión requerimiento ↔ código ↔ deploy

La trazabilidad responde dos preguntas espejo: *"¿por qué existe este cambio?"* (del código al requerimiento) y *"¿este requerimiento ya está implementado? ¿en qué commit?"* (del requerimiento al código). Se logra con un mecanismo humilde: **referenciar desde el PR el issue que lo originó** — `Closes #12` en GitHub, `Fixes AB#12` en Azure DevOps (allá se llaman *work items*) (en la descripción del PR — el detalle fino está en §4). Al mergear, la plataforma cierra/transiciona el work item y deja el enlace permanente en ambas direcciones.

Parece un detalle; es infraestructura de auditoría. Con la cadena **historia → PR → commit** armada, podés pararte en una línea de código y navegar hasta el requerimiento que la originó, y viceversa. En industrias reguladas eso es obligatorio; en cualquier equipo serio, es lo que convierte "creo que está en producción" en "está en producción desde el build #142".

### 2.6 GitHub Projects vs Azure Boards: mismo concepto, dos filosofías

- **Azure Boards** es la herramienta *estructurada*: jerarquía nativa de work items (Epic → Feature → User Story → Task), sprints con capacity, queries. Los conceptos de este TP existen como ciudadanos de primera clase. Es la más usada en empresas .NET y la que cubre el examen AZ-400.
- **GitHub Projects** (v2) es la herramienta *flexible*: todo es un issue, y el "proyecto" es una vista enriquecida (tabla/board) con campos custom (Status, Iteration, etc.). La jerarquía se arma con **sub-issues** y labels. Menos estructura impuesta, más cercana al código.
- Para este TP ambas son 100% gratuitas y sin tarjeta. El concepto que te llevás es el mismo: jerarquía + sprint + board + trazabilidad. La herramienta es la instancia.

## 3- Desarrollo de la guía (riel GitHub)

> Trabajamos sobre **el mismo repo de tu app del semestre** (TP2). Y algo importante desde ya: **en este práctico, lo que la guía hace ES tu entrega** — al revés del TP2, donde era un ejemplo. Reproducís lo que ves, sobre tu repositorio y tu proyecto.

### 3.1 Habilitar Projects y crear el proyecto

```bash
# Todos los comandos de esta guía se corren PARADO en el repo de tu app (el del TP2):
cd tu-repo-de-la-app

# Primero, con qué cuenta y con qué permisos está autenticado gh
gh auth status

# gh necesita el scope "project" para manejar Projects.
# Si `gh project list` te contesta un error de permisos, es esto:
gh auth refresh -s project
# Abre el navegador y pide confirmar. Si estás por SSH, en WSL sin navegador o en un
# Codespace, gh imprime en la terminal un código de 8 caracteres y una URL: abrís esa URL
# en el navegador de CUALQUIER equipo (tu teléfono sirve), pegás el código y listo.

# Crear el proyecto (Projects v2, a nivel usuario)
gh project create --owner "@me" --title "IngSoft3 - Mi App DevOps"
gh project list --owner "@me"     # anotá el número que le asignó
```

> ⚠️ **Si creás el proyecto por comando, el tablero NO se va a llenar solo.** Crear desde la
> web deja configurada la automatización que mete cada issue nuevo (§3.3); `gh project create`
> no elige ningún repositorio, así que ese workflow no queda armado y vas a ver el tablero
> vacío mientras el video muestra que ya está lleno. Se resuelve de dos formas: agregás los
> items con `gh project item-add` (está en §3.3), o entrás una vez al proyecto → menú `⋯` →
> *Workflows* → *Auto-add to project* y elegís tu repositorio.

**Lo mismo, desde la web** (es lo que el video muestra, y probablemente lo que vas a hacer la primera vez):

1. Entrá a tu perfil (`github.com/<tu-usuario>`) → pestaña **Projects** → botón **New project**.
   Los Projects **no viven adentro del repositorio**: viven en tu cuenta. Por eso se buscan acá.
2. ⚠️ Al apretar *New project* **el proyecto ya queda creado**, sin título (`@usuario's untitled project`). Recién después te ofrece de qué partir: en *Start from scratch* elegí **Table**. *Table*, *Board* y *Roadmap* no son proyectos distintos — son **vistas** del mismo, y podés tener varias.
3. Al elegir la plantilla se abre el panel con **Project name**: el nombre se pone **ahí mismo**, antes de confirmar con **Create project**. (Si lo salteaste, se cambia después en `⋯` → *Settings*.)
4. 👀 En ese mismo panel, debajo del nombre, hay una casilla **Import items from repository**
   que viene **tildada** y con un repositorio elegido. Dejala tildada y verificá que el
   repositorio sea el tuyo: **ésa es la casilla que enciende el auto-add** del que habla §3.3,
   o sea la que hace que tus issues aparezcan solos en el tablero. Si la destildás, el tablero
   nace vacío y tenés que agregar todo a mano.

Para uno, la web; para varios, el comando. Ese criterio vale para todo el TP.

> ⚠️ **Visibilidad del Project**: los Projects de usuario nacen **privados**, y tu entregable es
> la URL. Si la entregás así, quien la abra ve un 404 — ni siquiera "no tenés permiso": un "no
> existe".
>
> **Desde la web:** en el proyecto, menú `⋯` → **Settings** → sección **Visibility** → el botón
> muestra el estado actual (*Private*); al abrirlo aparecen las dos opciones con su descripción →
> elegí **Public**. No hay que confirmar nada más. Público es **de lectura**: que cualquiera pueda
> verlo no significa que pueda editarlo.
>
> 🔴 **Ojo con la combinación proyecto público + repositorio privado**: la URL abre, pero los
> items aparecen ocultos para quien no tiene acceso al repo — o sea que el chequeo de la
> ventana de incógnito da "verde" y el que corrige igual no ve nada. Si tu repositorio del
> semestre es privado, el entregable se cubre con capturas completas o dándole acceso de
> lectura a la cátedra.
>
> **Desde la terminal:**
>
> ```bash
> gh project edit <numero> --owner "@me" --visibility PUBLIC
> ```
>
> Este comando es **idempotente**: pedirle `PUBLIC` a un proyecto que ya es público no da error.
> Por eso se puede meter en un script y correrlo las veces que haga falta — una propiedad que vas
> a necesitar en serio a partir del TP4.
>
> Y el control que tenés que hacer **antes de entregar**: abrí tu propia URL en una ventana de
> incógnito. Si abre, listo. **Dejarlo privado no es una opción**: el entregable es la URL, y sin
> visibilidad no hay nada que corregir. Sólo si vas por el riel de Azure —donde no existe un
> Project de GitHub— se cubre con capturas completas o acceso de lectura a la cátedra.

**✅ Checkpoint:** el proyecto aparece en la pestaña *Projects* de tu perfil **y su URL abre sin login** (visibilidad pública).

### 3.2 Crear la épica y las historias como issues

La convención en GitHub: la jerarquía se arma con **issues + sub-issues** (o task-lists), y se etiqueta cada nivel.

Las etiquetas son **del repositorio**, no del proyecto. Desde la web están en
`github.com/<tu-usuario>/<tu-repo>/labels` (o solapa *Issues* → botón *Labels*): ahí ves las que
vienen de fábrica —entre ellas `bug`, que vas a usar y **no** hay que crear— y con **New label**
cargás nombre, descripción y color. Desde la terminal, para las tres de una:

```bash
# Labels para distinguir niveles
gh label create epic --color 6f42c1 --description "Épica"
gh label create story --color 0e8a16 --description "Historia de usuario"
gh label create task --color 1d76db --description "Tarea técnica"

# La épica
gh issue create --title 'EPIC: Pipeline DevOps completo para mi app' \
  --label epic \
  --body 'Objetivo del semestre: CI, calidad automatizada, CD multi-entorno, contenedores, IaC y DevSecOps sobre mi app.'

# Historias (¡en formato historia de usuario!)
gh issue create --title 'CI: build y tests automáticos en cada PR' --label story \
  --body 'Como desarrollador quiero que cada PR ejecute build y tests para detectar regresiones antes del merge.

### Criterios de aceptación
- [ ] El workflow corre en cada PR a main
- [ ] Un test que falla bloquea el merge
- [ ] El reporte de tests queda publicado como artefacto
- [ ] Badge de estado visible en el README'
```

> 📌 **La épica no lleva criterios de aceptación.** No se verifica por sí misma: se da por
> cerrada cuando sus historias están cerradas. Los criterios van donde algo se puede comprobar,
> que es la historia.

- Creá **la historia** y sus **2 tareas** — las mismas del video, sobre tu repositorio.

  > 🔴 **Acá lo que hace la guía SÍ es tu entrega — al revés del TP2.** En el práctico anterior,
  > lo que la guía y el video hacían era un ejemplo y tu entrega era otra cosa.
  > **En este práctico reproducís lo que ves**: la misma épica, la misma historia
  > con sus criterios, sus dos tareas, el bug, la jerarquía, el tablero con su sprint y su
  > límite, y el pull request que cierra la tarea.
  >
  > Lo que **vos** decidís —y es lo que tenés que poder explicar— es la duración de tu
  > sprint y el número de tu límite de trabajo en progreso, más el diagnóstico de la historia mal
  > escrita. **Copiar el procedimiento está bien; no poder explicarlo, no.**

  La historia es la del video: *Como desarrollador quiero que cada PR ejecute build y tests
  para detectar regresiones antes del merge*, con los cuatro criterios de aceptación de arriba.
  Podés copiarlos tal cual, pero en la defensa tenés que poder explicar **por qué cada uno es
  verificable** — un criterio que no se puede comprobar no es un criterio.

  Y sus **dos tareas**, que son el trabajo técnico concreto de esa historia: *escribir el
  workflow de build y tests* y *publicar el reporte de tests como artefacto*.

  > 💡 **La historia mal escrita.** Más abajo (§3.2) la guía crea a propósito una historia mal
  > escrita, para que aprendas a reconocerla. **Ésa no hace falta entregarla** — pero sí se entrega su
  > diagnóstico: en `decisiones.md`, dos renglones diciendo **por qué está mal escrita y cómo la
  > reescribirías**. Es lo único del práctico donde se ve si entendiste o copiaste.
- Vinculá jerarquía: abrí la épica en la web y agregá las historias como **sub-issues** — el botón dice *Create sub-issue*, y en su **dropdown** está *Add existing issue*, que es el que necesitás (tus historias ya existen como issues). En el video esto se hace **por la web**, que es lo más visual. Si preferís la terminal, `gh`
aprendió a hacerlo hace poco — `gh issue edit <numero_de_la_epica> --add-sub-issue <numero_de_la_historia>`,
disponible desde la **versión 2.94** (junio de 2026); con una versión anterior el flag no existe
y te queda la web. Comprobá la tuya con `gh --version`. Lo mismo de cada historia con sus tareas. (Existe la alternativa de task-lists en el cuerpo del issue — `- [ ] #12` — pero es **degradada**: no crea la relación padre-hijo navegable, que es lo que permite subir de la tarea a su historia y de ahí a la épica. Para el requisito de "jerarquía navegable" del TP, usá sub-issues.)
- Creá **el bug** — el mismo del video: *el front carga sin la lista cuando el back todavía no responde* (label `bug`, que ya existe por defecto), con las tres
  cosas que un bug necesita en el cuerpo: **qué pasa, qué esperabas, y cómo reproducirlo**. El
  bug **no cuelga de la jerarquía**: va al costado, entra al tablero como cualquier issue. Si
  preferís cargar uno de tu propia app, mejor todavía — pero no es obligatorio.

> 💡 **¿Por qué al costado, y qué cuenta como bug?** La jerarquía cuenta **lo que
> planificaste construir**: la épica es el objetivo, las historias el valor que vas a
> entregar, las tareas los pasos. Un bug es un defecto de algo que **ya construiste**
> — no era parte del plan, así que no forma parte del árbol. (Colgarlo de la historia
> que lo originó tiene además un efecto feo: esa historia ya está cerrada, y su barra
> de progreso pasaría a mentir.)
>
> Ahora bien, **no todo defecto es un bug**, y la diferencia es *cuándo* aparece:
>
> | Cuándo aparece | Qué es en realidad | Dónde va |
> |---|---|---|
> | Mientras la historia está **en curso**, antes de cerrarla | **No es un bug**: es que la historia todavía no cumple sus criterios de aceptación | Se arregla dentro de la historia. No se crea un issue aparte |
> | Sobre algo **ya entregado** (una versión previa, producción) | Un bug de verdad | Issue propio con label `bug`, al costado, priorizado junto al resto del backlog |
>
> El principio que ordena las dos filas es uno solo: **una historia con defectos no
> está terminada**. Si lo encontraste antes de cerrarla, no descubriste un bug —
> descubriste que te faltaba trabajo. El bug que entregás es del segundo caso.
>
> **Matiz para la defensa**: hay equipos que **sí** registran los defectos encontrados
> dentro del sprint, no para planificar sino para **medir** cuántos se les escapan; en
> ese caso el issue cuelga de la historia, porque es trabajo de esa historia. Y en
> Azure Boards un Bug **puede** ser hijo de una Feature. O sea: lo de "al costado" es
> una **convención de trabajo**, no una regla de la herramienta — y saber cuál usás, y
> por qué, es exactamente lo que se te va a preguntar.

> 🧪 **Ejercicio de dos minutos, y no hace falta entregarlo.** Creá a propósito una historia MAL escrita:
>
> ```bash
> gh issue create --title 'Como desarrollador quiero crear la tabla usuarios' --label story \
>   --body 'Como desarrollador quiero crear la tabla usuarios para guardar los datos.'
> ```
>
> Y
> antes de seguir, diagnosticala: ¿qué tiene de malo? La respuesta está en §2.3. Después
> borrala o dejala en el tablero, como prefieras: **no hace falta entregarla**, en el video queda creada sólo para
> mirarla). Lo que **sí** se entrega es tu diagnóstico, en dos renglones de `decisiones.md`:
> por qué está mal escrita y cómo la reescribirías. Y es exactamente el tipo de cosa que se mira
> en la defensa — mostrarte una y que digas si es historia o es tarea.

```bash
# Para releer una historia sin salir de la terminal:
gh issue view <numero>
```

**✅ Checkpoint:** la épica muestra sus historias como sub-issues con barra de progreso; cada historia tiene sus tareas y criterios de aceptación.

### 3.3 Armar el board y el sprint

Primero, los issues adentro del proyecto. Ojo que es probable que **ya estén todos**, sin que
hayas hecho nada: cuando creaste el proyecto desde la web, la casilla *Import items from
repository* venía **tildada**, y eso deja configurado el workflow *Auto-add to project*
apuntando a tu repo — desde ese momento, cada issue nuevo entra solo (lo ves en el historial
de cada uno, como `github-project-automation`). Es el motivo por el que un proyecto creado
desde la web trae **siete** workflows encendidos y uno creado con `gh project create` trae
seis: ese séptimo es el auto-add, y sin el repo elegido no tendría sobre qué actuar.
Si falta alguno:

```bash
# Agregar un issue al proyecto (o desde la web, con "Add item")
gh project item-add <numero_proyecto> --owner "@me" --url <url_del_issue>
```

En la web del proyecto (tu Project nace con **una sola vista de tabla** — lo demás se crea):
- **Board view**: al lado de la pestaña de la vista, *+ New view* → layout **Board**. Las columnas salen del campo *Status* (`Todo / In Progress / Done`).
- **Campo Iteration**: en la vista de tabla, *+* (agregar campo) → *New field* → nombre (p. ej. *Sprint*) → tipo **Iteration** → fecha de arranque y **duración** (la que hayas elegido y justificado en `decisiones.md`) → *Save*. La herramienta genera las iteraciones hacia adelante. Para asignar historias al sprint: la columna nueva en la tabla, *Sprint 1* en cada una.
- **Workflow automático**: en la **barra del Project**, al lado de *Insights*, está **Workflows**. Los workflows por defecto **ya vienen activados** — comprobá que *"Item closed → Status: Done"* esté en *On* y entendé qué hace: cuando un issue **se cierra**, su tarjeta pasa sola a *Done*. (Si no lo ves en la barra, está en el menú `⋯` de arriba a la derecha; en *Settings* no está — es el error más común.)
  > ⚠️ Ese workflow actúa sobre el **estado propio** de cada item. **No** cierra una historia porque se hayan cerrado sus sub-issues: la barra de progreso llega a `2/2` y la historia sigue abierta hasta que la cerrés vos (a mano, o con el PR que la termine). Lo mismo con la épica.

- **Límite de trabajo en progreso**: en la vista de tablero, el menú `⋯` de la columna
  *In Progress* → *Set limit*. Poné un número chico. Cuando la columna se llena, GitHub te
  avisa — ésa es toda la mecánica: no te lo impide, te lo hace visible (probalo: pasá el
  límite y mirá el contador de la columna ponerse en rojo).
- **Para cuando sean equipo** (no hace falta entregarlo, pero conviene haberlo mirado): el responsable de cada
  item se asigna en **Assignees**, abriendo la tarjeta; y una columna nueva del tablero —por
  ejemplo *Review* entre *In Progress* y *Done*— es un **valor nuevo del campo Status**: el `+`
  al final del board, y se acomoda con *Move left / Move right* desde el menú de la columna.
- **Roadmap**: la tercera vista — el mismo trabajo sobre una línea de tiempo. Nace vacía;
  en el panel *View* → **Dates** le decís qué campo usar (el Sprint) y cada item del sprint
  aparece como una barra en el calendario. Sirve para planificar y para mostrar fechas.
- **Insights** (al lado de *Workflows*): los gráficos del tablero — items por estado y el
  *burn-up* del avance (los gráficos históricos como el burn-up dependen del plan de la
  cuenta; si en la tuya no aparece, no es que hiciste algo mal). **Ojo**: las métricas de
  flujo de la clase 1 (cycle time, lead time, DORA) **no vienen de fábrica** en GitHub — se
  agregan (ver el anexo al final); en Azure Boards son nativas (Analytics).
- **Tres cosas más que existen y no usamos acá**: los **Milestones** (la forma clásica de
  agrupar por entrega; este TP usa el campo Iteration, que vive en el Project); las
  **plantillas de issues** (`.github/ISSUE_TEMPLATE/` — imponen el formato de historia y de
  bug solas, muy recomendable en equipo); y los **tipos de issue nativos** (desde 2025,
  task/bug/feature y tipos custom como Epic — **sólo en cuentas de organización**; en cuentas
  personales, como la del TP, la convención sigue siendo etiquetas).

**✅ Checkpoint:** board con los issues distribuidos, Sprint 1 con historias asignadas, y al cerrar un issue se mueve solo a Done.

### 3.4 Trazabilidad: cerrar historias desde PRs

El ciclo completo, sobre **una de las dos tareas de tu historia**. La del video es *escribir el
workflow de build y tests*, y el cambio que la implementa es el archivo del workflow — el mismo
que vas a extender en el TP4.

> ⚠️ **El PR tiene que hacer lo que el issue dice.** Si cierra un issue que no implementó, la
> trazabilidad es de mentira: el issue queda cerrado y el trabajo, sin hacer. Es lo primero que
> se mira.

**Desde la web** (es lo que el video muestra). Si la tarea dice "escribir el workflow de CI", el cambio es el workflow — no otra cosa:

1. En el repo, **Add file → Create new file**. El nombre lleva la ruta completa:
   `.github/workflows/ci.yml` — la carpeta donde GitHub busca sus workflows.
2. El contenido, lo mínimo (lo que este archivo hace de verdad es el TP4 entero; hoy sólo
   necesita existir):

   ```yaml
   name: CI
   on: [pull_request]
   jobs:
     build:
       runs-on: ubuntu-latest
       steps:
         - uses: actions/checkout@v4
   ```

3. **Commit changes** → GitHub propone crear una **rama nueva** y abrir un pull request (las
   protecciones del TP1 siguen activas: todo entra por PR) → *Propose changes*.
4. En la página del PR, la línea que importa va en la **descripción**:
   `Closes #<numero_de_LA_TAREA>` → **Create pull request**.

   > 🔴 **El número que va ahí es el de la TAREA, no el de la historia.** Un Pull Request
   > implementa una tarea concreta, así que cierra esa tarea. La historia sigue **abierta**
   > hasta que estén hechas las dos, y la cerrás vos. Si ponés el número de la historia, la
   > vas a cerrar con la mitad del trabajo sin hacer — y la trazabilidad queda mintiendo.

**Lo mismo, desde la terminal:**

```bash
git switch -c ci/workflow-de-build-y-tests
mkdir -p .github/workflows
printf 'name: CI\non: [pull_request]\njobs:\n  build:\n    runs-on: ubuntu-latest\n    steps:\n      - uses: actions/checkout@v4\n' > .github/workflows/ci.yml
git add .github && git commit -m 'ci: esqueleto del workflow de build y tests'
git push -u origin ci/workflow-de-build-y-tests

# El PR referencia al issue: al mergearse, el issue se CIERRA solo
gh pr create --title 'CI: esqueleto del workflow' \
             --body 'Agrega .github/workflows/ci.yml con el disparador y el checkout. Closes #<numero_de_LA_TAREA>.'
```

- Revisás el diff del PR (las protecciones del TP1 siguen activas: todo entra por PR), mergeás, y verificás: el issue quedó cerrado, movido a Done en el board, y en su historial aparece el PR que lo cerró.

> ⚠️ `Closes #N` solo cierra el issue si el PR apunta a la **rama por defecto** (`main`). Si tu PR apunta a otra rama, el merge no cierra nada — y no avisa. La palabra vale en la **descripción del PR** o en un **mensaje de commit**; en un comentario posterior **no** funciona. Y para este TP conviene la descripción del PR: por mensaje de commit el issue igual se cierra, pero **el issue no queda enlazado al PR** — y ese enlace es justamente lo que se mira al corregir.

**✅ Checkpoint:** 1 PR mergeado que cierra su issue automáticamente vía `Closes #N`, visible en el historial del issue.

---

## 4- Riel alternativo: Azure Boards

Mismos conceptos y checkpoints; en Azure la jerarquía es **nativa** (no hace falta simularla con labels):

| Concepto | GitHub (riel canónico) | Azure DevOps (Boards) |
|---|---|---|
| Proyecto/espacio | Projects v2 (`gh project create`) | Organización + Proyecto (proceso **Agile** o Scrum) |
| Épica | Issue con label `epic` + sub-issues | Work item **Epic** (nativo) — ⚠️ ver nota de jerarquía abajo |
| Historia | Issue con label `story` + criterios en el body | Work item **User Story** (campo Acceptance Criteria propio) |
| Tarea / Bug | Issue con label / label `bug` | Work items **Task** / **Bug** (nativos) |
| Jerarquía | Sub-issues / task-lists | Parent/child links nativos (vista *Backlogs* con niveles) |
| Sprint | Campo *Iteration* del Project | **Iterations** del proyecto + vista *Sprints* (con capacity) |
| Board | Board view del Project (Status) | Tablero por nivel de backlog |
| Trazabilidad PR→work item | `Closes #N` en el PR | **`Fixes AB#N`** en la **descripción** del PR (con GitHub↔Boards; `AB#N` a secas solo linkea, NO transiciona) o *linked work items* (con Azure Repos) — y policy que **exige** work item linkeado en cada PR |
| Automatización | Workflows del Project | Estado del work item se actualiza al completar el PR |
| CLI | `gh issue` / `gh project` | `az boards work-item create` / `az boards iteration` |
| Costo | Gratis | Gratis hasta 5 usuarios, **sin** subscription de Azure ni tarjeta |

> ⚠️ **Nota de jerarquía en Azure (proceso Agile)**: la jerarquía completa es **Epic → Feature → User Story → Task** (Feature es un nivel intermedio opcional que este TP saltea). El link Epic → User Story directo es **válido**, pero la vista *Backlogs* por niveles agrupa Stories bajo Features — si querés esa vista niveleada limpia, materializá tu "épica del semestre" como **Feature** (o creá Epic + Feature 1:1).
>
> 🔴 **Paso previo obligatorio si tu código está en GitHub y tu tablero en Azure Boards**: hay
> que **conectar las dos plataformas**, si no `AB#12` es texto plano y no pasa absolutamente
> nada. En el **Marketplace de GitHub**, instalá la app **Azure Boards** en tu repositorio
> (*Install* → elegís el repo) y autorizala contra tu organización y proyecto de Azure DevOps.
> Recién después de eso el mecanismo de abajo funciona. Es el paso que más gente saltea, y el
> síntoma es siempre el mismo: se mergea el PR y el work item se queda como estaba.
>
> ⚠️ **Transicionar work items desde el PR (GitHub ↔ Azure Boards)**: `AB#12` a secas **solo crea el link — no cambia el estado**. Para el equivalente de `Closes`, escribí **`Fixes AB#12`** (lleva la Story a *Resolved*) o **`Closed AB#12`** (a *Closed*) — **en la descripción del PR** (en el título no funciona), y solo si el PR mergea a la rama por defecto. Si usás Azure Repos para el código, el enlace se hace con *linked work items* del PR (o exigido por policy).

**Checkpoints riel Azure:** los mismos 4 (proyecto creado → épica/feature con la historia y sus 2 tareas en jerarquía nativa + 1 bug → sprint configurado con board → 1 PR que transiciona el estado con `Fixes AB#N`/`Closed AB#N` en la descripción, o work item link).

---
---

# 📋 Trabajo Práctico 03 – Planificación y trazabilidad (2026)

## ⚠️ Este es el TP que debés entregar y defender

## 🎯 Objetivo

Montar la gestión del proyecto sobre tu repositorio en una plataforma DevOps: jerarquía de trabajo, sprint, tablero y **trazabilidad demostrable** entre requerimientos y Pull Requests.

Este trabajo se aprueba **solo si podés explicar qué hiciste, por qué lo hiciste y cómo lo resolviste**.

## 🧩 Escenario

Ya tenés la app contenerizada (TP2) y un flujo de Git ordenado (TP1). Ahora el cliente pide visibilidad: quiere saber **qué se está haciendo, qué falta y cómo se conecta cada cambio de código con lo pedido**. Como líder técnico, tenés que montar la gestión del proyecto en una plataforma DevOps: la jerarquía de trabajo, el tablero con su sprint, y la trazabilidad entre lo que se pide y el código que lo implementa.

## 📋 Tareas que debés cumplir

### 1. Estructura de trabajo
- 1 **épica**: la del video — *Pipeline DevOps completo para mi app*. La épica **no lleva criterios de aceptación**: no se verifica por sí misma, se da por cerrada cuando sus historias están cerradas. Los criterios van donde algo se puede comprobar, que es la historia.
- **1 historia de usuario** (formato *Como… quiero… para…*, con sus criterios de aceptación) vinculada a la épica: la del video — *CI: build y tests automáticos en cada PR*.
- **2 tareas** vinculadas a esa historia.
- **1 bug** (el del video, o uno de tu app si preferís).
- La jerarquía debe ser **navegable** (sub-issues en GitHub; parent-child en Azure — las task-lists NO cumplen este requisito, ver §3.2).

### 2. Sprint y board
- Un sprint configurado —con la duración que hayas elegido, justificada en `decisiones.md`—, con la historia y sus dos tareas asignadas.
- Board con columnas de flujo y automatización mínima (cerrar → Done).
- Un **límite de trabajo en progreso** configurado en la columna *In Progress*. El número lo
  elegís vos, y la regla de arranque es **la cantidad de personas más uno** — trabajando solo, **dos**. El «más uno» es la válvula para cuando algo queda esperando (una revisión, una respuesta) y necesitás avanzar en otra cosa; pasarse de ahí hace que el límite deje de limitar. Señal para ajustarlo: **si nunca lo alcanzás, está demasiado alto**. El número va **justificado en `decisiones.md`**.

### 3. Decisiones de planificación
- En `decisiones.md` van **cinco cosas**: la duración de tu sprint y su porqué · el número de tu límite de trabajo en progreso y su porqué · el **diagnóstico de la historia mal escrita** (por qué está mal y cómo la reescribirías) · los problemas que encontraste y cómo los resolviste · la declaración de uso de IA.

### 4. Trazabilidad
- **1 PR mergeado** (con las protecciones del TP1 activas) que **cierre/transicione el work item automáticamente** (`Closes #N` en GitHub / `Fixes AB#N` en la descripción con Azure Boards): el que implementa una de las dos tareas de tu historia (§3.4).
- Desde una **tarea cerrada** debe poder navegarse hasta el PR y los commits que la implementaron — y de ahí subir a su historia y a la épica. Ésa es la vuelta completa (es lo que hace el video).

## 📄 Entregables

> ## ✅ Lo que se corrige — la lista completa
>
> - **URL del repositorio** + **URL del proyecto** (público)
> - La **épica**, la **historia** con sus criterios y sus **2 tareas** — las del video—,
>   colgadas en **jerarquía navegable**
> - **1 bug** — **al costado** de la jerarquía, no colgando de nadie
> - **Sprint** configurado, con la duración que elegiste, y la historia y sus tareas asignadas
> - **Tablero** con la automatización mínima y el **límite de trabajo en progreso**
> - **1 pull request** mergeado que implementa una de las dos tareas y **cierra su issue solo**
> - **`decisiones.md`** (`evidencias.md` no hace falta en este TP: el proyecto es público)
>
> Los dos archivos van **en el mismo repositorio del TP1 y el TP2**, agregando el título del
> TP3 debajo de lo que ya escribiste.
>
> ✅ **Todo lo que la guía y los videos hacen ES tu entrega** — la épica, la historia con sus
> criterios, sus dos tareas, la jerarquía, las tres etiquetas (`epic`, `story`, `task`), el
> tablero con su sprint y su límite, y el pull request. **Reproducilo sobre tu repositorio.**
> El **bug** también: el del video, o uno de tu app si preferís.
>
> ⚠️ **Las cinco cosas que son ejercicio y no hace falta entregar** (si te quedan en el tablero, no molestan): la **historia mal escrita**
> (pero sí su diagnóstico, en `decisiones.md`), la **columna de revisión** (se crea y se borra),
> el **Roadmap** y la **asignación de responsable**, más el **reporte de métricas** del anexo.
>
> 🔴 **Y lo que se defiende no es haber copiado**: es poder explicar por qué elegiste esa
> duración de sprint, ese límite de trabajo en progreso, y por qué cada criterio de aceptación es
> verificable. Copiar el procedimiento está bien; no poder explicarlo, no.

1. **URL del repositorio público + URL del proyecto/board** (se cargan en el formulario de la cátedra). El Project de GitHub **debe ser público** (§3.1) — no es opcional. Sólo si usás Azure Boards cubrí el entregable con capturas completas o acceso de lectura a la cátedra.
2. **`decisiones.md`** explicando **cinco cosas**:
   - La **duración que le diste al sprint** y su porqué.
   - El **número del límite de trabajo en progreso** y su porqué.
   - El **diagnóstico de la historia mal escrita** (§3.2): por qué está mal escrita y cómo la reescribirías. Dos renglones.
   - Problemas encontrados y cómo los resolviste.
   - La declaración de uso de IA (§ más abajo): qué parte fue asistida y cómo la verificaste.
> 💡 **En este TP no hay `evidencias.md`.** Tu proyecto es **público**: quien corrige abre su URL
> y ve en vivo la jerarquía, el sprint, el límite y el issue cerrado por el pull request. Sacar
> capturas de lo mismo sería duplicar lo que ya se ve. **Con dos excepciones**: si vas por **Azure Boards**,
> sí necesitás cubrirlo — con capturas completas de esas tres cosas o dándole acceso de lectura
> a la cátedra.
>
> *(En los otros prácticos `evidencias.md` sigue existiendo: acá se saltea porque el entregable
> es un proyecto público y se ve solo.)*

> 📅 **Fecha de entrega y formulario**: los publica la cátedra en el aula virtual. La duración
> del sprint que elijas conviene alinearla con ese calendario — es justamente el tipo de
> justificación que se espera en `decisiones.md`.

> ❓ **La duda que aparece siempre**: *¿la historia se entrega abierta o cerrada?* **Abierta**.
> Lo único que va cerrado es la tarea que cerró tu pull request; la otra tarea y la historia
> quedan abiertas, porque el trabajo sigue en el TP4.

## 🗣️ Defensa Oral Obligatoria

Vas a mostrar tu trabajo y responder preguntas como:
- ¿Qué diferencia hay entre épica, historia y tarea? Mostrame las tres en tu proyecto.
- Tomá uno de tus criterios de aceptación: ¿cómo lo verificás? ¿Y por qué ése sirve y «que el CI funcione bien» no?
- Te muestro esta historia — ¿es una historia o es una tarea disfrazada? ¿Por qué? *(Es el diagnóstico que escribiste en `decisiones.md`.)*
- Mostrame el camino desde tu tarea cerrada hasta el commit, y de ahí para arriba hasta la épica.
- ¿Por qué elegiste esa duración de sprint y ese límite de trabajo en progreso? ¿Qué pasa si lo subo a diez?
- ¿Por qué el bug no cuelga de la historia? ¿Y cómo sabés que algo es un bug y no trabajo que te faltó?
- ¿Qué te da GitHub Projects / Azure Boards que un Trello no te da?
- Tu límite de trabajo en progreso es un número que elegiste vos: ¿qué te haría subirlo, y qué
  señal te diría que quedó demasiado alto?

## ✅ Evaluación

| Criterio | Peso |
|---|---|
| Configuración técnica (jerarquía, sprint, board, trazabilidad) | 25% |
| Claridad y justificación en `decisiones.md` | 25% |
| Defensa oral: comprensión y argumentación | 50% |

> ⚖️ Peso orientativo de este TP en la nota de **P1**: **10%** (la ponderación completa de los 9 TPs está en el reglamento, §5).

## ⚠️ Uso de IA

Podés usar IA (ChatGPT, Copilot, Claude), pero **deberás declarar en `decisiones.md` qué parte fue asistida por IA** y justificar cómo la verificaste. Si no podés defenderlo, **no se aprueba**.

## Anexo (opcional) — medir el lead time con una Action gratuita

> Es lo que se ve al final del video de la parte 2. **No es entregable** de este TP — es para
> que veas, con datos de TU repo, las métricas que en la clase 1 miramos en abstracto.

### Qué estamos midiendo, y por qué importa

De las cuatro métricas DORA de la clase 1 (frecuencia de deploy, lead time for changes, tasa
de fallos, tiempo de recuperación), la que ya podés medir con lo que tenés es el **lead time
for changes**: cuánto tarda un cambio desde que se escribe hasta que está integrado y
verificado. Es el termómetro de tu flujo: un lead time corto significa PRs chicos, revisión
ágil y CI rápida; uno largo delata PRs gigantes, ramas viejas o revisiones que duermen.

GitHub no trae esta métrica de fábrica (lo nativo es *Insights*: items por estado y burn-up).
Pero el ecosistema la resuelve — y la forma más simple y gratuita es una **GitHub Action** del
marketplace que corre adentro de tu propio repo, sin registrarte en ningún servicio:
[`DeveloperMetrics/lead-time-for-changes`](https://github.com/marketplace/actions/dora-lead-time-for-changes).
Tiene una hermana, [`deployment-frequency`](https://github.com/DeveloperMetrics/deployment-frequency),
que mide la primera métrica DORA con la misma mecánica.

### Cómo se instala

Es un workflow más. Creá `.github/workflows/dora.yml` — por la web (*Add file → Create new
file*, como el ci.yml del video) o por terminal. Con las protecciones del TP1 activas, entra
**por pull request**, como todo:

```yaml
name: DORA - lead time
on: [workflow_dispatch]
jobs:
  lead-time:
    runs-on: ubuntu-latest
    steps:
      - uses: DeveloperMetrics/lead-time-for-changes@main
        with:
          workflows: 'CI'
```

Dos detalles del yml: `on: [workflow_dispatch]` significa que **se corre a mano** (no en cada
push — es un reporte, no un control); y `workflows: 'CI'` le dice qué workflow tuyo contar
como «verificación» del cambio (el del video se llama CI; si el tuyo se llama distinto, va tu
nombre). En el repo **público** del TP no hace falta nada más. Si el tuyo fuera **privado**,
la Action necesita un token para leer tus PRs: `actions-token: ${{ secrets.GITHUB_TOKEN }}`
—el del propio workflow— y, si aun así no alcanza, `pat-token` con un token personal que
tenga permiso de lectura sobre *actions* y *metadata*. Los dos parámetros están documentados
en el README de la Action.

### Cómo se corre y cómo se lee

Pestaña **Actions** → *DORA - lead time* → botón **Run workflow** → esperás unos segundos →
entrás al run y mirás el **Summary**. El reporte dice, por ejemplo:

> *Lead time for changes is 10.73 minutes with an Elite rating, over the last 30 days.*

Cómo lo calcula: toma tus **PRs mergeados de los últimos 30 días** y mide cuánto estuvo
abierto cada uno, le suma la duración del workflow de verificación, y promedia. La
calificación (elite / high / medium / low) sale de los benchmarks del reporte DORA — *elite*
es menos de un día.

### Las letras chicas (leelas antes de presumir del número)

- **Es una aproximación.** El lead time DORA «de libro» se mide de commit **a producción**.
  Tu repo todavía no despliega, así que la Action mide hasta el merge+CI — la parte del viaje
  que ya existe. Cuando en el TP6 tengas deploy, la medición se completa de verdad.
- **Con pocos PRs, el número se sesga.** La Action mira hasta 100 PRs de los últimos 30 días;
  si tenés dos, el promedio es una anécdota. Sirve igual: la idea es ver DÓNDE vive la
  métrica y qué la mueve, no auditarte con dos datos.
- **Un lead time «elite» en un TP no dice nada de tu proceso real** — tus PRs son chicos y
  los mergeás vos. Lo interesante empieza cuando el número CRECE: ahí hay una conversación
  sobre PRs más chicos o revisiones más rápidas.

### El puente con Azure y con lo que viene

En Azure DevOps hay algo parecido sin instalar nada: **Analytics** trae widgets nativos de
cycle time y lead time, y dashboards armables — parte de la ventaja «viene puesto» del riel
Azure. Pero ojo con una diferencia que conviene tener clara, porque es exactamente el tipo de
precisión que se pregunta: esos widgets miden el recorrido del **work item** (de creado —o de
empezado— a cerrado), mientras que el lead time de **DORA** mide el recorrido del **cambio**:
del commit a producción. Se parecen, cuentan cosas distintas, y ninguno reemplaza al otro. En GitHub, como viste, se agrega en cinco minutos. Y en los TP4-6, cuando tu pipeline
haga build, tests y deploy de verdad, estas mismas métricas van a medir TU proceso completo —
este anexo es la semilla de eso.
