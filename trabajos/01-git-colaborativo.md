# ⚠️ IMPORTANTE – Guía de Práctica Sugerida

Este documento tiene **dos partes**:

1. **Guía de práctica sugerida** (esta primera parte): un paso a paso para aprender haciendo. Te recomendamos fuertemente completarla, pero **NO es lo que se entrega**.
2. **El Trabajo Práctico entregable** (al final del documento): un escenario con tareas, entregables y defensa oral. **Eso es lo que se evalúa.**

## Sobre las herramientas en esta materia

En esta materia hay dos **rieles** — caminos guiados con soporte de la cátedra — más la opción libre:

- La guía paso a paso usa **GitHub** (riel canónico de la materia: gratis, sin tarjeta de crédito, y es lo que se demuestra en clase).
- Si preferís **Azure DevOps (Azure Repos)**, al final de la guía tenés la **tabla de equivalencias** con los mismos checkpoints.
- Podés usar **cualquier otra plataforma Git** (GitLab, Bitbucket…) siempre que cumplas el contrato de entregables del TP. Para plataformas fuera de GitHub y Azure DevOps el soporte de la cátedra es limitado.

---

# Guía Paso a Paso – Git para equipos (Práctica sugerida)

## 1- Objetivos de Aprendizaje

- Diseñar y justificar una **estrategia de branching** para un equipo.
- Trabajar con **Pull Requests** y code review como mecanismo central de colaboración.
- Configurar **protecciones de rama** que impidan romper `main`.
- Resolver conflictos de merge de forma controlada.
- Versionar entregas con **tags y releases** usando versionado semántico.

## 2- Nivelación previa (si nunca usaste Git)

Esta guía asume que ya sabés hacer `clone`, `add`, `commit`, `push`, `pull` y crear ramas. Si no:

- Guía de nivelación (curso 2025): https://github.com/ingsoft3ucc/TPs_2025/blob/main/trabajos/01-git-basico.md
- Ejercicios interactivos (completar al menos la *Introduction Sequence*): https://learngitbranching.js.org/

## 3- Marco teórico

### 3.1 DevOps: el contexto de toda la materia

> *"DevOps es la unión de personas, procesos y productos para permitir la entrega continua de valor a nuestros usuarios finales."* — Donovan Brown (Microsoft)

Fijate qué **no** dice esa definición: no menciona herramientas, ni nubes, ni pipelines. DevOps es primero una forma de trabajar; las herramientas son el medio. Por eso en esta materia podés elegir la plataforma: los conceptos son los mismos en todas.

**El problema que DevOps resuelve.** Históricamente, desarrollo ("que el software cambie") y operaciones ("que el software no se caiga") eran equipos separados con objetivos en conflicto. El resultado: entregas gigantes cada varios meses, integraciones traumáticas, deploys de viernes a la noche con rollbacks a mano. La respuesta de la industria fue invertir la lógica: **entregar cambios chicos, frecuentes y automatizados**. Un cambio chico es fácil de revisar, fácil de probar, fácil de desplegar y — cuando falla — fácil de encontrar y revertir.

**El ciclo DevOps** (vas a construir cada pieza durante el semestre):

```
  PLAN → CODE → BUILD → TEST → RELEASE → DEPLOY → OPERATE → MONITOR
   ↑                                                            │
   └────────────────────── feedback ────────────────────────────┘
```

| Fase | TP de la materia donde la construís |
|---|---|
| Plan | TP3 (boards, historias, trazabilidad) |
| Code | TP1 (Git, branching, PRs) + TP2 (contenerización de la app) |
| Build | TP4 (CI) |
| Test | TP5 (unit tests, coverage, análisis estático) + TP7 (e2e) |
| Release / Deploy | TP6 (CD, entornos, aprobaciones) + TP7 (contenedores) + TP8 (IaC) |
| Operate / Monitor | TP9 (seguridad, monitoreo, feedback) |

**¿Cómo se mide si un equipo "hace bien DevOps"?** Con las **métricas DORA** (del programa de investigación DevOps Research & Assessment, que estudia miles de equipos desde 2014):

| Métrica | Qué mide | Equipos de élite |
|---|---|---|
| **Deployment frequency** | Cada cuánto se despliega a producción | Varias veces por día |
| **Lead time for changes** | Tiempo desde el commit hasta producción | Menos de 1 día |
| **Change failure rate** | % de deploys que causan fallas | ~5% |
| **Failed deployment recovery time** (ex MTTR) | Cuánto tarda en recuperarse de un deploy fallido | Menos de 1 hora |

(Umbrales del reporte DORA 2024 — el último que publicó clusters de performance; desde 2025 DORA describe *perfiles de equipo* en vez de niveles élite/alto/medio/bajo.)

Fijate en la trampa aparente: pareciera que ir más rápido debería romper más. Los datos DORA muestran lo contrario — **los equipos que despliegan más seguido también fallan menos**, porque la velocidad sostenible exige automatización, cambios chicos y buenos tests. Velocidad y estabilidad no son enemigas: son consecuencia de las mismas prácticas. Esta idea es el corazón de la materia.

### 3.2 Control de versiones: la base sobre la que se construye todo

Un sistema de control de versiones (VCS) registra **quién cambió qué, cuándo y por qué**, permite trabajar en paralelo sin pisarse, y volver a cualquier estado anterior. Sin VCS no hay trabajo en equipo serio — pero en DevOps es todavía más: es **la fuente única de verdad**. Todo lo que vas a automatizar en esta materia (los pipelines del TP4, la infraestructura del TP8, la configuración de seguridad del TP9) va a vivir **como archivos dentro del repo**. Regla mental de la materia: *si no está en el repo, no existe*.

**Centralizado vs distribuido:**

| | Centralizado (SVN, TFVC) | Distribuido (Git) |
|---|---|---|
| Dónde está el historial | Solo en el servidor | **Completo en cada clon** |
| Trabajar sin conexión | No (casi nada) | Sí: commit, branch, log, diff — todo local |
| Crear una rama | Costoso (copia en el server) | Instantáneo (es solo un puntero) |
| Modelo de colaboración | Lock / commit directo | Ramas + merge (o PRs) |
| Estado en la industria | Legacy (TFVC está deprecado) | Estándar absoluto |

Git ganó esta pelea hace más de una década — al punto de que "control de versiones" y "Git" son casi sinónimos hoy. Pero entender *por qué* ganó (ramas baratas + historial distribuido) explica todo el flujo de trabajo moderno.

### 3.3 Cómo piensa Git (el modelo mental que evita el 90% de los sustos)

**Git guarda fotos, no diferencias.** Cada commit es un *snapshot* completo del proyecto (optimizado internamente), identificado por un hash (`a1b2c3d…`). Cada commit apunta a su(s) padre(s), formando un **grafo dirigido acíclico (DAG)**:

```
A ← B ← C ← D        ← main
         ↖
           E ← F     ← feature/login
```

**Las tres áreas.** Cuando editás un archivo, el cambio pasa por tres lugares:

```
Working Directory  --git add-->  Staging Area (index)  --git commit-->  Repositorio (.git)
 (tus archivos)                  (lo que va a entrar                    (historial permanente)
                                  en el PRÓXIMO commit)
```

La staging area existe para que un commit sea una **unidad coherente**: podés tener 10 archivos tocados y commitear solo los 3 que forman "un cambio con sentido". Commits chicos y coherentes son los que hacen útil el historial (y el code review).

**Una rama es solo un puntero.** No es una copia de nada: es una etiqueta móvil que apunta a un commit. Al commitear, el puntero de la rama actual avanza. `HEAD` indica en qué rama estás parado. Por eso crear una rama en Git es gratis e instantáneo — y por eso el flujo moderno crea ramas sin miedo.

**Remotes.** Tu clon local y GitHub son repositorios completos e independientes. `origin` es el alias del remoto; `git fetch` trae novedades sin tocar tu trabajo; `git pull` = fetch + merge; `git push` publica tus commits. Entender que **local y remoto pueden divergir** es la clave para no entrar en pánico ante un "rejected: non-fast-forward".

### 3.4 Merges: las cuatro formas de integrar trabajo

**Fast-forward (FF).** Si `main` no avanzó desde que saliste, integrar tu rama es solo mover el puntero — no se crea ningún commit nuevo:

```
Antes:  A ← B ← C(main) ← D ← E (feature)
Después: A ← B ← C ← D ← E (main, feature)
```

**Merge commit (no-FF / 3-way merge).** Si ambas ramas avanzaron (`main` sumó `F` mientras la feature hacía `D` y `E`), el fast-forward es imposible: Git crea un commit con **dos padres** que une las dos líneas, comparando ambas puntas contra el ancestro común:

```
A ← B ← C ← F ←── M (main)     M = commit de merge (2 padres): une F y E,
         ↖       ↗             comparándolas con el ancestro común C
           D ← E (feature)     — de ahí "merge de 3 vías"
```

**Squash.** Todos los commits de la rama se aplastan en **uno solo** sobre `main`. El historial de `main` queda limpio y lineal ("un commit = un feature"), pero se pierde el detalle de cómo se llegó.

**Rebase.** Tus commits se **reaplican uno a uno** arriba de la punta de `main`, como si hubieras empezado hoy. Historial lineal sin commit de merge — pero los commits reescritos son *nuevos* (cambia el hash). De ahí la regla de oro: **nunca rebasear commits que ya compartiste** con otros.

| Estrategia | El historial queda… | Ganás | Perdés |
|---|---|---|---|
| Merge commit | Fiel, con "diamantes" | Trazabilidad completa | Ruido visual con muchos PRs |
| Squash | Lineal, 1 commit por PR | Legibilidad, revert fácil | El paso a paso interno del PR |
| Rebase | Lineal, commits originales | Ambas cosas… | …a costa de reescribir historia (riesgoso compartido) |

No hay uno "correcto": hay que **elegir y justificar** según el valor que le des al historial. (En este TP lo vas a decidir y defender.)

Ojo con el conteo: el **fast-forward no es una opción que elijas** en el botón de merge de un PR — es el caso automático cuando `main` no avanzó. Las tres estrategias de la tabla son las opciones reales del botón.

**Conflictos: por qué existen y por qué no son un error.** Git fusiona automáticamente cuando los cambios tocan partes distintas. Cuando dos ramas modifican **la misma línea**, Git no puede decidir por vos — no sabe cuál versión es "la correcta" — y te lo delega marcando el archivo:

```
<<<<<<< HEAD
Proyecto IngSoft3 - versión A
=======
Proyecto IngSoft3 - versión B
>>>>>>> feature/titulo-b
```

Resolver un conflicto es tomar una decisión de **contenido**, no ejecutar un comando: elegís qué queda (una versión, la otra, o una síntesis), borrás los marcadores, y commiteás. El conflicto es la consecuencia natural del trabajo en paralelo; lo que sí es evitable es el *conflicto gigante*: ramas cortas e integración frecuente hacen conflictos chicos y triviales. Ramas de semanas producen el famoso *merge hell*.

### 3.5 Estrategias de branching: las reglas del juego del equipo

Una estrategia de branching responde: ¿qué ramas existen, cuánto viven, y cómo llega el código a `main`? Es una decisión de **equipo**, no individual — y tiene consecuencias directas sobre CI/CD. Estas estrategias se apoyan en el **Pull Request (PR)**: la propuesta formal de integrar una rama, con revisión de por medio — lo definimos en detalle en §3.6.

| Estrategia | Cómo funciona | Cuándo conviene |
|---|---|---|
| **Trunk-based** | Ramas de vida muy corta (horas/días) que se integran a `main` constantemente. `main` siempre deployable. | Equipos maduros con CI fuerte y buenos tests. Es la práctica que DORA correlaciona con alto rendimiento. |
| **GitHub Flow** | Una rama por feature → PR → review → merge a `main` → deploy. Sin ramas permanentes extra. | La mayoría de los proyectos con deploy continuo. **Recomendada para esta materia.** (Es trunk-based con la formalidad del PR.) |
| **GitFlow** | Ramas permanentes `main` + `develop`, más `feature/`, `release/`, `hotfix/`. | Productos con releases versionadas y varias versiones en soporte simultáneo (ej: software instalado en clientes). Overkill para deploy continuo. |

```
GitHub Flow:                          GitFlow (simplificado):

main ●───●───────●─────●──→           main    ●───────────●────────●──→  (solo releases)
      \         /     /                        \          ↑ release/1.0
       ●───●───●     /                develop   ●───●───●───●───●──→
     feature/a      /                            \     ↖ feature/a
             ●───●─●                              ●───●
           feature/b
```

La conexión con DevOps: cuanto más largas viven las ramas, más tarde se integra, más grandes los conflictos y más lento el *lead time*. La investigación DORA es explícita: los equipos de élite integran a trunk **al menos una vez por día**. GitFlow no es "malo" — es la herramienta correcta para otro problema (versiones múltiples en soporte).

### 3.6 Pull Requests y code review: calidad como proceso, no como heroísmo

Un **Pull Request** es la propuesta formal de integrar una rama: empaqueta los commits, el diff, la conversación de revisión y las verificaciones automáticas en un solo lugar. Es el **punto de control de calidad** del flujo — y a partir del TP4, también el lugar donde correrán tus pipelines.

**¿Por qué revisar código si "ya funciona"?** Porque el code review no busca solo bugs:
- **Detección temprana**: un defecto encontrado en review cuesta minutos; en producción, horas o días (y usuarios).
- **Difusión de conocimiento**: el reviewer aprende esa parte del código; mañana puede mantenerla. Se elimina el "solo Juan entiende ese módulo".
- **Propiedad colectiva**: el código deja de ser "mío" y pasa a ser "nuestro" — el estándar de calidad se vuelve del equipo.

**Qué mira un buen reviewer:** ¿el cambio hace lo que dice? ¿es legible para el próximo que lo toque? ¿tiene tests? ¿introduce riesgos (seguridad, performance)? ¿es coherente con el diseño existente? **Qué NO debería mirar:** estilo y formato — eso se automatiza (linters), no se discute entre humanos.

**La regla práctica más importante:** PRs chicos. Un PR de 100 líneas recibe review de verdad; un PR de 3.000 líneas recibe un "LGTM 👍" sin leer. Si el PR es grande, partilo.

**Protecciones de rama: la política hecha configuración.** Las branch protections convierten el acuerdo del equipo ("todo entra por PR revisado") en una **regla que la plataforma hace cumplir**. Esto es un patrón que vas a ver toda la materia — *policy as code / as configuration*: los procesos importantes no dependen de la memoria ni de la buena voluntad, están codificados y son auditables. Es la versión social de lo mismo que hace un pipeline con los tests.

### 3.7 Versionado semántico, tags y releases

Un **tag** marca un commit con un nombre inmutable ("acá estaba el código de la v1.0.0"). Una **release** le agrega comunicación: notas de qué cambió, binarios adjuntos si aplica.

**SemVer** (`MAJOR.MINOR.PATCH`, ej. `2.4.1`) es el contrato de versionado estándar:
- **MAJOR**: cambios incompatibles ("si actualizás, algo tuyo puede romperse").
- **MINOR**: funcionalidad nueva compatible.
- **PATCH**: corrección de bugs, sin cambios de comportamiento esperado.
- Sufijos: `1.0.0-beta.1` (prerelease), `+build.5` (metadata).

El número de versión no es decorativo: es **información para el que consume** tu software. En TP2 vas a taguear imágenes de contenedor con semver, y en TP6 los deploys van a nacer de tags — la disciplina empieza acá.

## 4- Desarrollo de la guía (riel GitHub, CLI-first)

> Vamos a usar la **CLI oficial de GitHub (`gh`)** además de `git`. Todo lo que hagas por CLI también se puede hacer por la web — pero la CLI es reproducible, se puede documentar en un script, y es el mismo enfoque que vas a usar en pipelines.

### 4.1 Setup

- Instalar `git` (https://git-scm.com) y `gh` (https://cli.github.com).
- Configurar identidad y autenticarse:

```bash
git config --global user.name "Tu Nombre"
git config --global user.email "tu@mail.com"

gh auth login        # seguir el wizard (GitHub.com, HTTPS, login por browser)
gh auth status       # verificar
```

**✅ Checkpoint:** `gh auth status` te muestra logueado con tu cuenta.

### 4.2 Crear el repositorio del proyecto

```bash
# Crea un repo PÚBLICO en tu cuenta, con README, y lo clona localmente
gh repo create ingsoft3-tp01 --public --add-readme --clone
cd ingsoft3-tp01
```

- Agregar un `.gitignore` acorde al stack que vas a usar en la materia (por ejemplo el de `dotnet`, `node`, etc. — podés partir de https://github.com/github/gitignore).
- Primer commit con mensaje descriptivo:

```bash
git add .gitignore
git commit -m "chore: agrega .gitignore para el stack del proyecto"
git push
```

**✅ Checkpoint:** el repo existe en `github.com/<tu_usuario>/ingsoft3-tp01`, es público, y tiene README + .gitignore en `main`.

### 4.3 Elegir y documentar la estrategia de branching

- Elegí una de las 3 estrategias (para esta materia recomendamos **GitHub Flow**).
- Creá el archivo `decisiones.md` y documentá: cuál elegiste, por qué, y qué convención de nombres de rama van a usar (ej: `feature/<descripcion>`, `fix/<descripcion>`).
- Subilo a `main` (por ahora directo — en el próximo paso eso deja de ser posible).

**✅ Checkpoint:** `decisiones.md` está en `main` con la estrategia elegida y la convención de nombres de ramas.

### 4.4 Proteger `main`

Ahora hacemos que **nadie pueda pushear directo a `main`** (ni siquiera vos): todo cambio entra por Pull Request.

Por CLI (la API de branch protection):

```bash
gh api --method PUT "repos/{owner}/{repo}/branches/main/protection" \
  --input - <<'EOF'
{
  "required_pull_request_reviews": { "required_approving_review_count": 0 },
  "required_status_checks": null,
  "enforce_admins": true,
  "restrictions": null
}
EOF
```

Qué configura cada línea:

- `required_pull_request_reviews` con `required_approving_review_count: 0`: **todo cambio tiene que entrar por PR**, pero sin approvals obligatorias de otra persona — este TP es individual, así que vas a poder mergear tus propios PRs. En un equipo real acá iría `1` o más (lo viste en §3.6, y puede caer en la defensa: ¿qué número pondrías en un equipo de 4 y por qué?).
- `enforce_admins: true`: la regla te aplica **también a vos**, que sos admin del repo. Sin esto, GitHub te dejaría saltear la protección — y una protección con bypass habilitado es de adorno.

(Equivalente por web: *Settings → Branches → Add branch protection rule* — es la misma feature que configura el comando, y ahí vas a ver reflejada la regla. **Ojo:** los *Rulesets* (Settings → Rules) son una feature distinta y más nueva de GitHub; si preferís usarlos, el análogo de `enforce_admins` es dejar la *bypass list* vacía. No mezcles las dos: elegí un mecanismo y verificá el checkpoint.)

- Verificá la protección intentando un push directo:

```bash
echo "test" >> README.md
git commit -am "test: intento de push directo"
git push    # ← debe FALLAR con "protected branch"
git reset --hard HEAD~1   # deshacemos el commit local
```

**✅ Checkpoint:** el push directo a `main` es rechazado por GitHub. Guardá la captura/salida — es evidencia para el TP.

### 4.5 Tu primer Pull Request, paso a paso (a prueba de errores)

Este es **el ciclo que vas a repetir toda la materia**: rama → cambio → PR → merge. La primera vez lo hacemos con lupa, paso por paso, diciendo qué hace cada comando y qué tenés que ver en pantalla. Si en algún paso lo que ves no coincide, frená ahí y revisá — no sigas arrastrando el error.

> 🧠 **Antes de arrancar, un concepto que puede caer en la defensa:** en GitHub **el autor de un PR no puede aprobar su propio PR** — no es configurable: en la web las opciones *Approve* y *Request changes* aparecen deshabilitadas sobre un PR propio, y por API el intento devuelve `422 — Can not approve your own pull request`. Por eso en §4.4 configuramos la protección con **0 approvals obligatorias**: exige que todo entre por PR, pero te deja mergear los tuyos — el flujo completo, trabajando solo. En un equipo real ese número sería 1 o más. (En Azure DevOps, en cambio, la auto-aprobación **sí es configurable** — ver la tabla del punto 5. Mismo concepto, decisiones de plataforma distintas.)

**Paso 0 — Pararte en `main` actualizado.** Toda rama nueva nace de `main` al día:

```bash
git switch main    # te parás en main
git pull           # traés lo último del remoto
git status         # debe decir: "nothing to commit, working tree clean"
```

**Paso 1 — Crear la rama de la feature.** El nombre sigue la convención que documentaste en `decisiones.md` (ej: `feature/<descripcion>`):

```bash
git switch -c feature/seccion-instalacion
```

`-c` = *create*: crea la rama Y te para en ella. Verificá con `git branch` — la rama con `*` es donde estás.

**Paso 2 — Hacer el cambio y commitearlo.** Editá `README.md` agregándole una sección de instalación (o el cambio que toque). Después:

```bash
git status                     # ves el archivo en rojo (modificado, sin stagear)
git add README.md              # lo pasás a staging (ahora en verde)
git commit -m "docs: agrega sección de instalación al README"
```

Mensajes descriptivos, en tiempo imperativo: el historial es documentación.

**Paso 3 — Publicar la rama en GitHub:**

```bash
git push -u origin feature/seccion-instalacion
```

`-u` vincula tu rama local con la remota — solo hace falta la **primera vez** que pusheás cada rama; después alcanza con `git push`. Si te olvidás del `-u`, Git te muestra el error `no upstream branch` **junto con el comando exacto para arreglarlo**: copialo y listo.

**Paso 4 — Abrir el Pull Request.** Por CLI:

```bash
gh pr create --title "Agrega sección de instalación" \
             --body "Qué cambia y por qué."
```

El comando te imprime la **URL del PR** — abrila en el browser. (Equivalente por web: apenas pusheaste, la página del repo te muestra un banner amarillo *"feature/seccion-instalacion had recent pushes — Compare & pull request"*: ese botón abre el formulario del PR. Verificá que diga `base: main ← compare: feature/...`, completá título y descripción, y *Create pull request*.)

El body del PR no es decorativo: **qué cambia y por qué** es lo que leería un reviewer — y lo que vas a agradecer vos mismo en 3 meses.

**Paso 5 — Mirar el PR como lo miraría un reviewer.** En la página del PR, pestaña **Files changed**: ahí está el diff — verde lo que entra, rojo lo que sale. Leelo entero preguntándote lo de §3.6: ¿hace lo que dice el título? ¿es legible? Si ves algo mejorable, no lo arregles "en silencio": dejalo escrito como comentario (posá el mouse sobre la línea → botón azul `+` → escribí → *Add single comment*). En §4.6 esta auto-revisión se vuelve obligatoria.

**Paso 6 — Mergear.** Por CLI (el número de PR te lo dijo `gh pr create`; también lo ves con `gh pr list`):

```bash
gh pr merge <numero> --squash --delete-branch
```

- `--squash` es **el tipo de merge**: usá el que documentaste en `decisiones.md` (`--merge` / `--squash` / `--rebase`).
- `--delete-branch` borra la rama (remota y local) después del merge: la rama ya cumplió su función — las ramas de feature son descartables, no mascotas.

(Equivalente por web: botón verde **Merge pull request** — la flechita `▾` al lado elige el tipo de merge — → *Confirm* → botón *Delete branch*.)

**Paso 7 — Cerrar el ciclo localmente.** El merge ocurrió **en GitHub**; tu `main` local todavía no lo tiene:

```bash
git switch main
git pull                       # ahora sí: tu cambio está en main
git log --oneline -3           # verificalo — ahí está el commit del merge
```

> 💥 **Errores típicos de la primera vez (ninguno es grave):**
> - **`gh pr create` dice "No commits between main and tu-rama"** → te olvidaste de commitear (paso 2) o estás parado en `main`. `git status` y `git branch` te dicen cuál de los dos.
> - **El PR muestra "0 files changed"** → mismo caso: el commit no está en la rama que pusheaste.
> - **`git push` rechazado con "protected branch"** → estás intentando pushear a `main` directo. Bien: **la protección funciona**. Volvé al paso 1 y trabajá en una rama.
> - **Hiciste un commit parado en `main` sin querer** → no pasa nada mientras no puedas pushearlo: `git switch -c feature/lo-que-sea` se lleva ese commit a una rama nueva, y después `git switch main && git reset --hard origin/main` deja tu `main` limpio.
> - **Cerraste el PR en vez de mergearlo** (*Close* ≠ *Merge*) → reabrilo desde la página del PR (*Reopen*) — no se perdió nada.

**✅ Checkpoint:** 1 PR mergeado a `main` por el ciclo completo, y tu `main` local actualizado con ese cambio.

### 4.6 Segunda feature: el ciclo con self-review de verdad

Repetí el ciclo completo de §4.5 para una **segunda feature** (otra sección del README, un archivo nuevo — lo que sume). Esta vez, el paso 5 es obligatorio y con evidencia:

1. Abrí **Files changed** y encontrá algo mejorable **de verdad** en tu propio diff: un typo, una frase confusa, algo que falta. Siempre hay algo — la consigna es encontrarlo antes de mergear, que es exactamente lo que hace un buen reviewer.
2. Dejá **un comentario sobre esa línea** del PR diciendo qué cambiarías y por qué — como se lo escribirías a otra persona.
3. **Resolvelo con un commit adicional** en la misma rama (`git add` + `git commit` + `git push` — el PR se actualiza solo).
4. Marcá la conversación como **Resolved** en el PR, y recién ahí mergeá.

Eso que acabás de hacer — comentario, corrección, resolución — es exactamente la ronda de code review de un equipo real, con vos en los dos roles. El PR queda como evidencia: la conversación y el commit de corrección son parte de lo que se evalúa.

**✅ Checkpoint:** 2 PRs mergeados a `main`, al menos uno con un comentario de review sobre una línea + commit de corrección + conversación resuelta.

### 4.7 Provocar y resolver un conflicto

Los conflictos no son un error: son la consecuencia normal de trabajo en paralelo. Acá vas a simular vos solo a las dos personas: dos ramas que tocan la misma línea. Vamos a fabricarlo a propósito:

```bash
# Rama A: modificar la línea 1 del README
git switch main && git pull
git switch -c feature/titulo-a
# editar la línea 1 del README → "Proyecto IngSoft3 - versión A"
git commit -am "docs: cambia título (versión A)" && git push -u origin feature/titulo-a
gh pr create --fill

# Rama B (SIN partir de A): modificar la MISMA línea
git switch main
git switch -c feature/titulo-b
# editar la línea 1 del README → "Proyecto IngSoft3 - versión B"
git commit -am "docs: cambia título (versión B)" && git push -u origin feature/titulo-b
gh pr create --fill
```

- Mergeá el PR de la rama A. El PR de la rama B ahora tiene conflicto.
- Resolvelo localmente:

```bash
git switch feature/titulo-b
git fetch origin
git merge origin/main        # ← acá aparece el conflicto
# abrir el archivo, resolver los marcadores <<<<<<< ======= >>>>>>>
git add README.md
git commit                   # commit de merge
git push
```

- Ahora el PR de B se puede mergear.

> 💡 **Dos gotchas verificados en la práctica:**
> 1. Los nombres junto a los marcadores dependen de **dónde estés parado**: `HEAD` es siempre TU rama actual. Como acá resolvés parado en `feature/titulo-b`, vas a ver tu versión bajo `HEAD` y la de `main` bajo `origin/main`.
> 2. Si intentás mergear el PR **inmediatamente** después de pushear la resolución, GitHub puede rechazarlo porque todavía no recalculó el estado del conflicto. Esperá unos segundos y reintentá — no es que tu resolución esté mal.

**✅ Checkpoint:** captura del conflicto (los marcadores en el archivo) y del PR de B mergeado después de resolverlo. Explicá en `decisiones.md` cómo lo resolviste y con qué criterio elegiste qué versión quedaba.

### 4.8 Tags y releases (versionado semántico)

```bash
git switch main && git pull
git tag -a v1.0.0 -m "Primera versión estable del TP"
git push origin v1.0.0

gh release create v1.0.0 --title "v1.0.0" \
  --notes "Primera release: estrategia de branching, protecciones, 4 PRs mergeados."
```

**Semver en 30 segundos:** `MAJOR.MINOR.PATCH` → rompés compatibilidad = MAJOR; agregás funcionalidad compatible = MINOR; arreglás un bug = PATCH.

**✅ Checkpoint:** la release `v1.0.0` es visible en la página del repo.

---

## 5- Riel alternativo: Azure DevOps (Azure Repos)

Si elegís el riel Azure, los **conceptos y checkpoints son idénticos** — cambia dónde vive cada cosa:

| Concepto | GitHub (riel canónico) | Azure DevOps |
|---|---|---|
| Repositorio | Repo en tu cuenta (`gh repo create`) | Proyecto → Azure Repos (`az repos create`) |
| Protección de `main` | Branch ruleset / branch protection | **Branch policies** sobre `main` (Repos → Branches → ⋮ → Branch policies) |
| PR obligatorio sin approvals (trabajo individual) | `required_approving_review_count: 0` (paso 4.4) | Policy "Minimum number of reviewers = 1" + checkbox *"Allow requestors to approve their own changes"* **activado** (Azure no tiene "0 reviewers": el mínimo es 1, pero con ese checkbox podés aprobarte y mergear solo) |
| Prohibir push directo | Efecto de la protección | Efecto automático de tener cualquier policy activa |
| Pull Request | `gh pr create / review / merge` | `az repos pr create` o web; tipos de merge equivalentes (merge/squash/rebase/semi-linear) |
| Aprobar tu propio PR | **Prohibido siempre, no configurable** (UI deshabilitada; API devuelve `422 Can not approve your own pull request`) — por eso este TP usa 0 approvals | **Configurable**: el checkbox *"Allow requestors to approve their own changes"* de la policy de reviewers — acá lo activás para trabajar solo; en un equipo real va desactivado |
| Saltearse la protección siendo admin | Posible salvo que actives `enforce_admins` / ruleset sin bypass — **activalo** (paso 4.4) | Posible con permiso "Bypass policies" — no lo uses en este TP |
| Release | `gh release create` (releases de GitHub) | Tags de Git (`git tag`); no existe el concepto "release page" — documentá las notas en un `CHANGELOG.md` |
| CLI | `gh` | `az devops` / `az repos` (extensión Azure DevOps de Azure CLI) |
| Cuenta necesaria | GitHub gratis, sin tarjeta | Azure DevOps gratis hasta 5 usuarios, sin tarjeta (no requiere subscription de Azure) |

**Checkpoints riel Azure:** los mismos de la guía (auth ok → repo creado → estrategia documentada → push directo rechazado → 2 PRs por el ciclo completo, uno con self-review → conflicto resuelto → tag + changelog).

> 📌 Este TP **no necesita nube ni tarjeta en ningún riel**: tanto GitHub como Azure DevOps son gratis para esto.

---
---

# 📋 Trabajo Práctico 01 – Git colaborativo (2026)

## ⚠️ Este es el TP que debés entregar y defender

## 🎯 Objetivo

Diseñar, implementar y **defender** el flujo de trabajo Git de un equipo de desarrollo: estrategia de branching, code review por Pull Requests, protecciones de rama y versionado de entregas.

Este trabajo se aprueba **solo si podés explicar qué hiciste, por qué lo hiciste y cómo lo resolviste**.

## 🧩 Escenario

Sos el líder técnico de un equipo que arranca un proyecto nuevo. Antes de escribir la primera línea de código de producción, tenés que dejar definido y funcionando el flujo de trabajo: cómo se nombra y se ramifica, cómo entra el código a `main`, qué se revisa antes de mergear, y cómo se versionan las entregas. El TP es individual: vos mismo vas a operar bajo esas reglas — y demostrar que el flujo funciona de punta a punta.

## 📋 Tareas que debés cumplir

### 1. Repositorio y estrategia
- Repositorio **público** en la plataforma elegida (GitHub, Azure Repos u otra).
- Estrategia de branching elegida y **justificada** en `decisiones.md` (incluí: por qué esa y no las otras dos, y la convención de nombres de ramas).
- `.gitignore` acorde al stack.

### 2. Protecciones y code review
- `main` protegida: **imposible pushear directo** (incluso siendo admin — sin bypass); todo cambio entra por Pull Request.
- Al menos **3 PRs mergeados**, de los cuales:
  - al menos 1 con una **ronda de self-review** (comentario de revisión sobre una línea de tu propio PR + commit de corrección + conversación resuelta antes del merge),
  - al menos 1 que haya requerido **resolver un conflicto de merge**.
- Tipo de merge (merge/squash/rebase) elegido y justificado.

### 3. Versionado
- Tag `v1.0.0` (semver) sobre `main` con release/changelog con notas de qué incluye.

## 📄 Entregables

1. **URL del repositorio público** con todo el historial: ramas, PRs con sus conversaciones, conflicto resuelto, tag y release. Se carga en el formulario de la cátedra (el link está en el aula virtual y se comparte en clase).
2. **`decisiones.md`** en la raíz del repo explicando:
   - Estrategia de branching elegida y justificación.
   - Tipo de merge elegido y justificación.
   - Cómo resolviste el conflicto y con qué criterio.
   - Problemas encontrados y cómo los solucionaste.
3. **`evidencias.md`** (también en la raíz del repo) con capturas/links de: push directo rechazado, PR con la ronda de self-review (comentario + commit de corrección), conflicto (marcadores visibles) y su resolución, release publicada.

## 🗣️ Defensa Oral Obligatoria

Se realiza en **P1 (clase 5)**, junto con la defensa de los TPs 2 a 4. Vas a mostrar tu trabajo y responder preguntas como:
- ¿Por qué elegiste esa estrategia de branching? ¿Cuándo NO la usarías?
- ¿Qué relación hay entre el tamaño/duración de las ramas y las métricas DORA? ¿Por qué "ir más rápido" no implica "romper más"?
- ¿Qué es la staging area y por qué existe? ¿Qué es una rama *realmente* para Git?
- ¿Qué diferencia hay entre merge, squash y rebase? ¿Qué perdés y qué ganás con cada uno?
- ¿Qué pasa si dos personas modifican la misma línea? Mostrame cómo lo resolviste.
- ¿Para qué sirve proteger `main` si el equipo "se tiene confianza"?
- ¿Podés aprobar tu propio PR en GitHub? ¿Y en Azure DevOps? ¿Por qué tu protección pide 0 approvals, y qué número pondrías en un equipo real?
- ¿Podés mergear sin cumplir la protección siendo admin? ¿Cómo lo evitaste en tu repo?
- En tu PR con self-review: ¿qué comentaste, por qué, y cómo lo resolviste?
- ¿Qué significa el número de versión que elegiste para tu tag?

## ✅ Evaluación

| Criterio | Peso |
|---|---|
| Configuración técnica (protecciones, PRs, historial, release) | 25% |
| Claridad y justificación en `decisiones.md` + `evidencias.md` | 25% |
| Defensa oral: comprensión y argumentación | 50% |

> ⚖️ Peso orientativo de este TP en la nota de **P1**: **20%** (la ponderación completa de los 9 TPs está en el reglamento, §5).

## ⚠️ Uso de IA

Podés usar IA (ChatGPT, Copilot, Claude), pero **deberás declarar en `decisiones.md` qué parte fue asistida por IA** y justificar cómo la verificaste. Si no podés defenderlo, **no se aprueba**.
