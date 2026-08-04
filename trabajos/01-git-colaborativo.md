# ⚠️ Cómo leer este documento

Tiene **dos partes**, y las dos te sirven:

1. **La guía paso a paso** (lo que sigue): es **el camino para hacer el TP**. Seguila de arriba a abajo y al terminar vas a tener el trabajo hecho. Tiene **checkpoints** ✅ para verificar que vas bien, y cuatro momentos marcados 📸 donde tenés que sacar una captura.
2. **El enunciado del TP** (al final): qué se entrega y qué se evalúa. **La guía no se entrega** — lo que se entrega es tu repositorio, con los archivos `decisiones.md` y `evidencias.md`.

## Sobre las herramientas en esta materia

En esta materia hay dos **rieles** — caminos guiados con soporte de la cátedra — más la opción libre:

- La guía paso a paso usa **GitHub** (riel canónico de la materia: gratis, sin tarjeta de crédito, y es lo que se demuestra en clase).
- Si preferís **Azure DevOps (Azure Repos)**, al final de la guía tenés la **tabla de equivalencias** con los mismos checkpoints.
- Podés usar **cualquier otra plataforma Git** (GitLab, Bitbucket…) siempre que cumplas el contrato de entregables del TP. Para plataformas fuera de GitHub y Azure DevOps el soporte de la cátedra es limitado.

---

# Guía Paso a Paso – Git para equipos (Práctica sugerida)

## 1- Objetivos de Aprendizaje

- Entender cómo un equipo integra su trabajo: **ramas cortas** que entran por Pull Request.
- Trabajar con **Pull Requests**: la unidad con la que un equipo propone, revisa e integra cambios.
- Configurar **protecciones de rama** que impidan romper `main`.
- Resolver conflictos de merge de forma controlada.
- Versionar entregas con **tags y releases** usando versionado semántico.

## 2- Nivelación previa (si nunca usaste Git)

La mayor parte de esta guía se hace **desde la web de GitHub**, así que no necesitás pelearte con la consola para arrancar. Sí vas a usar `clone`, `add`, `commit`, `push` y `pull` en algunos pasos, y la guía te dice exactamente cuándo y para qué. Si nunca los usaste, con estos dos recursos alcanza:

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

No hay uno "correcto": hay que **elegir y justificar** según el valor que le des al historial. En este TP usamos **squash** (te lo damos hecho, igual que la estrategia de branching); en el **TP4** vas a tener que elegir y defender el tuyo, ya con experiencia encima.

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

## 4- Desarrollo de la guía (riel GitHub)

> **Cómo está organizada esta guía.** Cada paso lo vas a hacer donde lo haría un equipo de verdad,
> y eso depende del **rol**, no de la herramienta:
>
> - Lo que hace el **reviewer** y el **administrador del repo** (revisar un PR, mergear, configurar
>   las protecciones, publicar una release) pasa **en la web de GitHub**. Ahí vas a estar la mayor
>   parte del tiempo.
> - Lo que hace el **autor** cuando el cambio deja de ser un archivo suelto (traer el repositorio a
>   tu máquina, versionar archivos que no editás en el navegador, crear el tag) pasa **en tu
>   máquina**: la terminal — o tu editor, ver §4.9. Son unos pocos momentos puntuales, y cada uno
>   está anunciado.
>
> En este TP los tres roles los ocupás vos, pero cada cosa se hace donde va. En varios pasos vas a
> ver un bloque plegable **"Lo mismo por consola"** con los comandos equivalentes, por si algún día
> querés automatizarlo. **No hace falta abrirlos para aprobar el TP.**

📸 **Ojo con las evidencias:** cuatro momentos de esta guía son irrepetibles — una vez que pasan, no
los podés volver a capturar. Están marcados con 📸 **SACÁ LA CAPTURA AHORA**. Son los que van en
`evidencias.md`.

### 4.1 Setup

Lo mínimo indispensable:

1. Una cuenta de **GitHub** (gratis, sin tarjeta): https://github.com/signup
2. **Git** instalado en tu máquina: https://git-scm.com
3. Configurar tu identidad (esto queda grabado en cada commit que hagas):

```bash
git config --global user.name "Tu Nombre"
git config --global user.email "tu@mail.com"
```

4. **Dejar a Git autenticado contra GitHub.** Esto no es opcional: sin esto, tu primer `git push`
   va a fallar. Desde 2021 GitHub **no acepta tu contraseña** desde la terminal, así que necesitás
   una de estas dos:

   - **Camino recomendado — instalar `gh`** (https://cli.github.com) y correr `gh auth login`:
     elegí *GitHub.com* → *HTTPS* → *Login with a web browser*. Además de autenticarte, deja a Git
     configurado para siempre. Verificá con `gh auth status`.
   - **Sin instalar nada** — creá un *Personal Access Token* en
     https://github.com/settings/tokens (*Generate new token (classic)* → alcance **repo**), y
     cuando `git push` te pida usuario y contraseña, poné tu usuario y **pegá el token como
     contraseña**. Guardalo en algún lado: no se vuelve a mostrar.

   > 💥 Si al pushear ves *"Authentication failed"*, *"Support for password authentication was
   > removed"* o *"could not read Username"*, es esto — no es un problema de permisos del
   > repositorio. En Windows, Git suele abrirte una ventana del navegador y resolverlo solo.

> 💡 Más allá de la autenticación, `gh` sirve para hacer desde la terminal cosas que normalmente
> harías en la web: es lo que usan los bloques "Lo mismo por consola" de esta guía. Podés ignorarlos
> por completo: **el TP se aprueba haciendo todo por la web.**

**✅ Checkpoint:** `git config --global user.name` te devuelve tu nombre, entrás a tu cuenta de GitHub
desde el browser, y tenés resuelto cómo te vas a autenticar al pushear (token o `gh auth login`).

### 4.2 Crear el repositorio (en la web)

Crear un repositorio es un acto de administrador — se hace en la web y se ve todo lo que estás
decidiendo:

1. Entrá a **https://github.com/new**
2. **Repository name**: `ingsoft3-tp01`
3. **Visibility**: **Public** — es requisito de la materia (§4 del reglamento explica por qué).
4. **Add README**: activá el interruptor. Así el repositorio nace con un archivo y una rama `main`,
   en vez de nacer vacío.
5. **Create repository**.

Fijate en la URL que te quedó: `github.com/<tu_usuario>/ingsoft3-tp01`. **Esa es la URL que vas a
entregar.**

<details>
<summary>💻 Lo mismo por consola</summary>

```bash
gh repo create ingsoft3-tp01 --public --add-readme
```
</details>

**✅ Checkpoint:** el repositorio existe, es público, y tiene un `README.md` en la rama `main`.

### 4.3 Traerlo a tu máquina y hacer tu primer commit

Esto sí es trabajo de autor, y por eso va en tu máquina. Son los tres comandos que vas a repetir toda
la materia: `add`, `commit`, `push`.

```bash
git clone https://github.com/<tu_usuario>/ingsoft3-tp01.git
cd ingsoft3-tp01
```

`clone` te baja una **copia completa** del repositorio, con todo su historial — no una foto del
último estado (§3.2).

Ahora **creá** un archivo llamado `.gitignore` en la raíz del proyecto (con tu editor, o con
`touch .gitignore` y después abriéndolo). Es el archivo que le dice a Git qué cosas **no** se
versionan: binarios, dependencias, secretos.

Todavía no elegiste la app del semestre —eso pasa en el TP2—, así que por ahora alcanza con un
`.gitignore` mínimo que sirve para cualquier proyecto. Copiá esto adentro:

```gitignore
# dependencias y artefactos de build
node_modules/
bin/
obj/
dist/
build/

# secretos: NUNCA se versionan
.env
*.local

# basura del sistema operativo y del editor
.DS_Store
Thumbs.db
.vscode/
.idea/
```

> 💡 Cuando en el TP2 elijas tu app y sepas con qué está hecha, vas a ampliarlo con el archivo que
> corresponda a esa tecnología — hay uno por stack en https://github.com/github/gitignore. Ese
> repositorio es la referencia de la industria para esto.

Después:

```bash
git status                     # ves el archivo nuevo, sin seguimiento
git add .gitignore             # lo pasás al área de staging (§3.3)
git commit -m "chore: agrega .gitignore base del proyecto"
git push
```

Mensajes descriptivos y en tiempo imperativo: el historial es documentación.

**✅ Checkpoint:** el `.gitignore` se ve en la página del repositorio, y `git log --oneline` te
muestra tu commit.

### 4.4 Proteger `main` (en la web)

Esta es la configuración más importante del TP: hacer que **nadie pueda pushear directo a `main`** —
ni siquiera vos. Todo cambio va a tener que entrar por un Pull Request.

En la web son tres decisiones:

1. En tu repositorio: **Settings → Branches**, y ahí el botón para agregar la regla.
   ⚠️ GitHub le viene cambiando el nombre: hoy dice **Add rule**, y según la cuenta puede
   aparecer como *Add branch protection rule* o *Add classic branch protection rule*. Los tres
   llevan al mismo formulario. Lo que NO tenés que usar es *Go to rulesets* (ver el aviso de §4.4).
2. **Branch name pattern**: `main`
3. Activá **Require a pull request before merging**.
   ⚠️ Al activarlo, GitHub **tilda solo** la casilla *Require approvals* que aparece debajo, con el
   valor 1. **Destildala**: tiene que quedar en **cero aprobaciones obligatorias**. Si te la olvidás
   tildada, en el paso §4.5 no vas a poder mergear tu propio PR y el mensaje no te va a decir que el
   problema está acá.
4. Bajá hasta el final y activá **Do not allow bypassing the above settings**.
5. **Create** / **Save changes**.

Por qué cada una:

- **Require a pull request**: es la regla del juego — nada entra a `main` sin pasar por un PR.
- **Cero aprobaciones**: este TP es **individual**, y en GitHub **el autor de un PR nunca puede
  aprobar su propio PR** (no es configurable: la opción aparece deshabilitada, y por API devuelve
  `422 — Can not approve your own pull request`). Si pidieras una aprobación, no podrías mergear
  nunca. En un equipo real, acá iría 1 o más.
- **Do not allow bypassing**: la regla te alcanza **también a vos**, que sos el dueño del repo. Sin
  esto, GitHub te dejaría saltearla — y una protección que el dueño puede saltear es de adorno.

> ⚠️ **Si en vez de *Branches* usaste *Rules → Rulesets*** (una feature distinta y más nueva de
> GitHub que hace lo mismo): el equivalente de *Do not allow bypassing* es dejar la **bypass list
> vacía**. No mezcles los dos mecanismos — elegí uno y verificá con la prueba de acá abajo.

#### La prueba de fuego: que te rechace a vos

Una protección que nunca rechazó nada no se sabe si funciona. Probala:

```bash
echo "test" >> README.md
git commit -am "test: intento de push directo"
git push          # ← esto TIENE que fallar
```

Vas a ver un error que dice `protected branch hook declined` o similar.

📸 **SACÁ LA CAPTURA AHORA** — ese rechazo es la evidencia número uno del TP.

Después deshacé el commit local, que ya no sirve:

```bash
git reset --hard HEAD~1
```

<details>
<summary>💻 Lo mismo por consola (opcional, avanzado)</summary>

La misma regla se puede configurar contra la API de GitHub. Queda scripteada y reproducible, que es
la ventaja — pero es bastante más densa que las tres tildes de arriba:

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

`enforce_admins: true` es el equivalente de *Do not allow bypassing*.

</details>

**✅ Checkpoint:** el push directo a `main` es rechazado por GitHub, y tenés la captura.

### 4.5 Tu primer Pull Request (entero en la web)

Este es **el ciclo que vas a repetir toda la materia**. La primera vez lo hacemos entero en la web,
porque ahí el Pull Request se ve como lo que es: un objeto con su conversación, sus commits y su
diff.

**Paso 1 — Editar el archivo.** En la página del repositorio, abrí `README.md` y tocá el **lápiz**
(*Edit this file*). Agregale al final una sección de instalación:

```markdown
## Instalación

git clone <url-del-repo>
```

**Paso 2 — Intentar guardar.** Botón **Commit changes…**. Y acá pasa lo importante: como `main` está
protegida, GitHub **no te deja commitear directo** y te ofrece otra cosa —
***Create a new branch for this commit and start a pull request***. Elegila.

El nombre de la rama seguí la convención de la materia: `feature/<descripcion>` (ver §4.9).
Ejemplo: `feature/seccion-instalacion`. Después, **Propose changes**.

> 🧠 Pará un segundo acá: **la protección de `main` te empujó sola al camino correcto.** Eso es lo
> que hace una regla bien puesta — no te grita, te desvía.

**Paso 3 — Crear el Pull Request.** GitHub te lleva a la pantalla de comparación. Verificá que diga
`base: main ← compare: feature/seccion-instalacion`, poné un título claro y, en la descripción,
**qué cambia y por qué**. Botón **Create pull request**.

El cuerpo del PR no es decorativo: es lo que leería un reviewer — y lo que vas a agradecer vos mismo
dentro de tres meses.

**Paso 4 — Mirar el PR.** Esta página va a ser tu casa toda la materia. Recorré las pestañas:

- **Conversation**: la discusión y el historial de lo que fue pasando.
- **Commits**: qué commits trae la rama.
- **Files changed**: el **diff** — en verde lo que entra, en rojo lo que sale.

Leé el diff entero con las preguntas de §3.6: ¿hace lo que dice el título? ¿se entiende? En un
equipo, este es el momento en que otra persona te deja comentarios; trabajando solo, es tu última
oportunidad de mirar el cambio con ojo crítico antes de que entre.

**Paso 5 — Mergear.** Volvé a *Conversation* y apretá **Merge pull request**. La flechita `▾` al lado
del botón te deja elegir el tipo de merge — usá **Squash and merge** (§3.4: un commit por PR, el
historial de `main` queda legible). Confirmá, y después tocá **Delete branch**: la rama ya cumplió su
función, y las ramas de feature son descartables, no mascotas.

**Paso 6 — Traer el cambio a tu máquina.** El merge ocurrió **en GitHub**; tu copia local todavía no
lo tiene:

```bash
git switch main
git pull
git log --oneline -3      # ahí está tu cambio
```

Este paso se olvida siempre, y es la causa número uno del próximo conflicto tonto.

<details>
<summary>💻 Lo mismo por consola</summary>

```bash
git switch main && git pull
git switch -c feature/seccion-instalacion   # -c = crear la rama y pararse en ella
# ... editar el README ...
git add README.md
git commit -m "docs: agrega sección de instalación al README"
git push -u origin feature/seccion-instalacion   # -u solo la primera vez de cada rama
gh pr create --title "Agrega sección de instalación" --body "Qué cambia y por qué."
gh pr diff <numero>                              # el diff, en la terminal
gh pr merge <numero> --squash --delete-branch
git switch main && git pull
```
</details>

> 💥 **Errores típicos de la primera vez (ninguno es grave):**
> - **El PR muestra "0 files changed"** → el commit quedó en otra rama, o no llegaste a commitear.
> - **`git push` rechazado con "protected branch"** → estás pusheando a `main` directo. Bien: la
>   protección funciona. Trabajá en una rama.
> - **Hiciste un commit parado en `main` sin querer** → mientras no puedas pushearlo no pasa nada:
>   `git switch -c feature/lo-que-sea` se lleva ese commit a una rama nueva, y después
>   `git switch main && git reset --hard origin/main` te deja `main` limpio.
> - **Cerraste el PR en vez de mergearlo** (*Close* ≠ *Merge*) → reabrilo con *Reopen*, no se perdió
>   nada.

**✅ Checkpoint:** 1 PR mergeado a `main`, la rama borrada, y tu `main` local actualizado.

### 4.6 Provocar y resolver un conflicto

Un conflicto no es un error: es lo que pasa cuando dos personas tocan la misma línea (§3.4). Como acá
estás solo, vas a hacer de las dos personas — y lo fabricamos a propósito, porque es mucho mejor que
tu primer conflicto sea en un entorno controlado y no en tu primer trabajo.

**La receta: dos ramas que nacen de `main` y cambian la MISMA línea.**

**Rama A** — desde la web: editá el `README.md`, cambiá **la primera línea** por
`# Proyecto IngSoft3 - versión A`, y **creá el PR** (rama `feature/titulo-a`) igual que en §4.5.
🛑 **Pará ahí: NO lo mergees todavía.** Primero tenés que crear la rama B.

**Rama B** — igual, pero **partiendo otra vez de `main`**, sin enterarse de lo que hizo A: cambiá esa
**misma** primera línea por `# Proyecto IngSoft3 - versión B` y creá el PR (`feature/titulo-b`).

> ⚠️ Este es el punto donde se arruina el ejercicio: si la rama B nace de la A, no hay conflicto.
> Antes de crear B, asegurate de estar parado en `main`.

**Mergeá el PR de A.** Entró limpio.

**Ahora mirá el PR de B**: GitHub te avisa que **no se puede mergear automáticamente** porque hay
conflictos.

📸 **SACÁ LA CAPTURA AHORA** — el aviso de conflicto en el PR.

#### Resolverlo desde la web

En el PR de B, tocá **Resolve conflicts**. Se abre un editor con el archivo y **los marcadores**:

```
<<<<<<< feature/titulo-b
# Proyecto IngSoft3 - versión B
=======
# Proyecto IngSoft3 - versión A
>>>>>>> main
```

Leelos con calma, porque es el corazón del ejercicio: arriba está **tu** versión, abajo la que ya
está en `main`, y los símbolos `<<<<<<<`, `=======` y `>>>>>>>` son las fronteras.

📸 **SACÁ LA CAPTURA AHORA** — los marcadores. Ojo: en el paso siguiente los vas a borrar y ya no
vas a poder volver a capturarlos.

**Resolver es decidir el contenido, no ejecutar un comando.** Elegí qué queda —una versión, la otra,
o una combinación—, **borrá las tres líneas de marcadores** para que el archivo quede como si el
conflicto nunca hubiera existido, y tocá **Resolve** — el botón está deshabilitado hasta que no quede ningún marcador, así que
si no te deja, es que todavía queda alguno. Después, **Commit merge**. (En algunas cuentas ese
primer botón aparece como *Mark as resolved*.)

Ahora el PR de B se puede mergear. Hacelo.

> 💡 Si GitHub no te deja mergear **inmediatamente** después de resolver, esperá unos segundos: está
> recalculando el estado. No es que tu resolución esté mal.

<details>
<summary>💻 Lo mismo por consola</summary>

GitHub resuelve en la web los conflictos simples como este. Cuando son grandes, el editor web no
alcanza y se resuelve en tu máquina — así se hace:

```bash
git fetch origin                    # primero traés las ramas que creaste en la web
git switch feature/titulo-b         # ahora sí existe localmente
git merge origin/main               # ← acá aparece el conflicto

# abrí el README.md en tu editor: vas a ver los mismos marcadores.
# Editalo, dejá el contenido final y borrá los marcadores. Después:

git add README.md
git commit -m "fix: resuelve conflicto de título tomando la versión B"
git push
```

⚠️ Si hacés `git commit` **sin** `-m`, Git te abre el editor de texto de la terminal (normalmente
`vim`) para que escribas el mensaje. Si te pasa y no sabés cómo salir: apretá `Esc`, escribí `:wq` y
Enter.

Un detalle del que se aprende: los nombres al lado de los marcadores dependen de **dónde estés
parado**. `HEAD` siempre es tu rama actual.
</details>

**✅ Checkpoint:** el PR de B mergeado después de resolver el conflicto, con las dos capturas (el
aviso y los marcadores).

### 4.7 Versionar la entrega: tag y release

Un **tag** marca un commit con un nombre inmutable; una **release** le agrega comunicación (§3.7).

El tag se crea desde tu máquina:

```bash
git switch main && git pull
git tag -a v1.0.0 -m "Primera versión estable del TP"
git push origin v1.0.0
```

Y la release se publica desde la web, que es donde se ve para qué sirve:

1. En el repositorio: **Releases → Draft a new release** (o *Create a new release*).
2. **Choose a tag**: elegí el `v1.0.0` que acabás de subir.
3. **Release title**: `v1.0.0`
4. **Describe this release**: qué incluye esta versión — escrito para humanos, no para máquinas.
5. **Publish release**.

📸 **SACÁ LA CAPTURA AHORA** — la release publicada.

**Semver en 30 segundos:** `MAJOR.MINOR.PATCH` → rompés compatibilidad = MAJOR; agregás
funcionalidad compatible = MINOR; arreglás un bug = PATCH.

<details>
<summary>💻 Lo mismo por consola</summary>

```bash
gh release create v1.0.0 --title "v1.0.0" \
  --notes "Primera release: protecciones de rama, el flujo de Pull Requests funcionando y un conflicto resuelto."
```
</details>

**✅ Checkpoint:** la release `v1.0.0` se ve en la página del repositorio.

### 4.8 Los dos archivos que se entregan

El TP no se entrega como un zip: **se entrega tu repositorio**. Y adentro tienen que estar estos dos
archivos, en la raíz.

**`evidencias.md`** — las cuatro capturas marcadas 📸 en esta guía, cada una con una línea que diga
qué se está viendo:

```markdown
# Evidencias — TP1

## 1. Push directo a main rechazado
![push rechazado](img/push-rechazado.png)
GitHub rechaza el push porque main está protegida y la regla alcanza también al dueño del repo.

## 2. El PR de la rama B no se puede mergear: conflicto
...
```

**`decisiones.md`** — tres cosas, cortas y honestas:

1. **Por qué Git no pudo resolver el conflicto solo** — y qué habría tenido que pasar para que
   nunca apareciera.
2. **Qué problemas encontraste** y cómo los solucionaste. Los tropiezos bien contados valen más que
   un camino perfecto: son los que demuestran que entendiste.
3. **Declaración de uso de IA**: qué partes hiciste con ayuda de inteligencia artificial y cómo
   verificaste lo que te devolvió (§ *Uso de IA* del enunciado).

> 💡 Estos dos archivos también son cambios al repositorio — así que entran por **Pull Request**,
> como todo lo demás. A esta altura el ciclo ya te tiene que salir sin mirar la guía: ese es,
> justamente, el punto del TP.

**✅ Checkpoint:** `decisiones.md` y `evidencias.md` están en `main`, y llegaron ahí por un PR.

### 4.9 Dos cosas más que conviene saber

**La convención de nombres de rama.** En esta materia usamos `feature/<descripcion>` para
funcionalidad nueva y `fix/<descripcion>` para correcciones. No es la única convención posible: es la
que vamos a usar para que todos los repos se lean igual. En el TP4, cuando ya tengas el flujo
funcionando, vas a poder discutir y justificar la tuya.

**Esto mismo se hace desde tu editor.** Visual Studio Code —y cualquier editor moderno— trae
integración con Git y con GitHub: podés crear ramas, ver el diff, commitear, pushear, resolver
conflictos con una vista lado a lado, y hasta abrir Pull Requests, sin salir del editor. Es como
trabaja la mayoría de la industria. Lo dejamos para el final a propósito: **primero conviene entender
qué está haciendo el botón**, y después usar el que te ahorre tiempo. Si querés probarlo, mirá la
extensión *GitHub Pull Requests* de VS Code.

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

**Checkpoints riel Azure:** los mismos de la guía (repo creado → primer commit → push directo rechazado → 2 PRs → conflicto resuelto → tag + changelog).

> 📌 Este TP **no necesita nube ni tarjeta en ningún riel**: tanto GitHub como Azure DevOps son gratis para esto.

---
---

# 📋 Trabajo Práctico 01 – Git colaborativo (2026)

## ⚠️ Este es el TP que debés entregar y defender

## 🎯 Objetivo

Poner a funcionar —y **defender**— el flujo de trabajo con el que un equipo integra código: ramas,
Pull Requests, revisión, protecciones sobre `main` y versionado de la entrega.

Este trabajo se aprueba **solo si podés explicar qué hiciste, por qué pasa lo que pasa, y cómo lo
resolviste**.

## 🧩 Escenario

Arrancás un proyecto nuevo y, antes de escribir la primera línea de código de producción, dejás
funcionando el flujo de trabajo: cómo entra el código a `main`, qué se revisa antes de integrarlo, y
cómo se versiona lo que se entrega. El TP es individual: los tres roles de un equipo —el que escribe,
el que revisa y el que administra el repositorio— los vas a ocupar vos.

## 📋 Tareas que debés cumplir

### 1. Repositorio
- Repositorio **público** en la plataforma elegida (GitHub, Azure Repos u otra).
- `.gitignore` en la raíz (el base alcanza: la app del semestre se elige en el TP2).

### 2. Protecciones
- `main` protegida: **imposible pushear directo** — ni siquiera para vos, que sos el administrador
  (sin bypass). Todo cambio entra por Pull Request.
- **Evidencia del rechazo**: la salida del intento de push directo a `main`.

### 3. Pull Requests
Al menos **2 PRs mergeados**, y uno de ellos tiene que haber requerido **resolver un conflicto de
merge** (lo fabricás a propósito, §4.6).

Todo cambio al repositorio entra por PR — incluidos los dos archivos de la entrega. Así que los PRs
te van a salir solos: lo que se evalúa es que el historial muestre el flujo funcionando.

### 4. Versionado
- Tag `v1.0.0` (semver) sobre `main`, con su **release** publicada y notas de qué incluye.

> 📌 La estrategia de branching (GitHub Flow), la convención de nombres de rama y el tipo de merge
> **te los damos** en esta guía: son las reglas de la materia, y todavía no tenés la experiencia para
> elegirlas con criterio. En el **TP4**, con el flujo ya funcionando y un pipeline encima, vas a
> tener que elegir y justificar la tuya.

## 📄 Entregables

1. **URL del repositorio público** con todo el historial: ramas, PRs con sus conversaciones,
   conflicto resuelto, tag y release. Se carga en el formulario de la cátedra (el link
   está en este repositorio y se comparte en clase).
2. **`decisiones.md`** en la raíz del repositorio, con tres cosas:
   - **Por qué Git no pudo resolver el conflicto solo** y qué habría tenido que pasar para que
     nunca apareciera.
   - **Qué problemas encontraste y cómo los solucionaste** (los errores contados con honestidad valen
     más que un camino perfecto).
   - **Declaración de uso de IA**: qué partes hiciste con ayuda de inteligencia artificial y cómo
     verificaste lo que te devolvió.
3. **`evidencias.md`** (también en la raíz) con las cuatro capturas marcadas 📸 en la guía: el push
   directo rechazado, el aviso de conflicto en el PR, los marcadores del conflicto, y la release
   publicada.

## 🗣️ Defensa Oral Obligatoria

Se realiza en **P1 (clase 5)**, junto con la defensa de los TPs 2 a 4. Vas a mostrar tu repositorio
navegando en vivo y responder preguntas como:

- ¿Para qué sirve proteger `main` si en el equipo "se tienen confianza"? ¿Qué pasó cuando intentaste
  pushear directo?
- ¿Qué es una rama *realmente* para Git? ¿Y por qué el merge que hiciste en GitHub no aparecía en tu
  máquina hasta que hiciste `pull`?
- ¿Qué pasa cuando dos personas modifican la misma línea? Mostrame tu conflicto y explicame por qué
  Git no pudo resolverlo solo.
- El code review es la práctica central de un equipo (§3.6) y en este TP trabajaste solo:
  ¿qué buscarías vos en el Pull Request de un compañero, y qué NO discutirías nunca en una revisión?
- ¿Qué significa el número de versión que le pusiste al tag?

## ✅ Evaluación

| Criterio | Peso |
|---|---|
| Configuración técnica (protecciones, PRs, historial, release) | 25% |
| Claridad y justificación en `decisiones.md` + `evidencias.md` | 25% |
| Defensa oral: comprensión y argumentación | 50% |

> ⚖️ Peso orientativo de este TP en la nota de **P1**: **5%** (la ponderación completa de los 9 TPs está en el reglamento, §5).

## ⚠️ Uso de IA

Podés usar IA (ChatGPT, Copilot, Claude), pero **deberás declarar en `decisiones.md` qué parte fue asistida por IA** y justificar cómo la verificaste. Si no podés defenderlo, **no se aprueba**.
