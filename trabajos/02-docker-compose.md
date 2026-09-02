# ⚠️ IMPORTANTE – Guía de Práctica Sugerida

Este documento tiene **dos partes**:

1. **Guía de práctica sugerida** (primera parte): paso a paso para aprender haciendo. **NO es lo que se entrega.**
2. **El Trabajo Práctico entregable** (al final): escenario, tareas, entregables y defensa oral. **Eso es lo que se evalúa.**

## 📦 Un solo repositorio para todo el semestre

**Todos los prácticos de la materia se hacen sobre el mismo repositorio: el que creaste en el TP1.** No se crea uno nuevo por práctico, y no se arranca de cero en cada uno — cada TP agrega una capa sobre lo que ya está.

Cada TP se cierra con su **tag y su release**, y el número mayor es el número del práctico: **TP1 → `v1.0.0`, TP2 → `v2.0.0`, TP3 → `v3.0.0`**, y así hasta el TP9. Así cada entrega queda con su **estado congelado**: en la defensa se navega el punto exacto en el que cerraste cada una, y podés volver a cualquiera con `git checkout v2.0.0`.

Los archivos `decisiones.md` y `evidencias.md` también son **únicos**: no se rehacen por práctico — se les agrega abajo la sección del TP nuevo.

## Sobre las herramientas en este TP

📌 **Este TP corre 100% en tu máquina y es gratis: no necesita nube, ni cuenta paga, ni tarjeta de crédito en ningún riel.** La única cuenta externa es la del registry de imágenes (GitHub o Docker Hub, ambas gratuitas).

---

# Guía Paso a Paso – Docker y Compose sobre la app del semestre (Práctica sugerida)

## 1- Objetivos de Aprendizaje

- Entender qué es un contenedor, una imagen y un registry, y por qué son la unidad de despliegue del software moderno.
- Escribir **Dockerfiles multi-stage** para backend y frontend.
- Orquestar un sistema completo (front + back + base de datos) con **docker-compose**.
- Publicar imágenes en un registry.
- **Elegir la aplicación que vas a usar durante todo el semestre** (este TP es el cimiento de los TPs 4 a 9 y del Integrador).

## 2- Marco teórico

### 2.1 El problema que los contenedores resuelven: "funciona en mi máquina"

Toda la historia del deployment de software se resume en una frase que escuchaste (o vas a escuchar) mil veces: *"en mi máquina funciona"*. La app corre perfecta en la notebook del desarrollador… y explota en el servidor, porque el servidor tiene otra versión del runtime, le falta una librería del sistema, o una variable de entorno apunta a otro lado. El software nunca viaja solo: viaja con un **entorno implícito** — y ese entorno, históricamente, se reconstruía a mano en cada máquina, con documentos de instalación que envejecían mal y con servidores que nadie se animaba a tocar porque "andan así" — y que cuando se rompían, no había forma de reconstruirlos.

El contenedor ataca la raíz del problema: **empaqueta la aplicación JUNTO con su entorno** (runtime, librerías, configuración, sistema de archivos) en una unidad ejecutable, inmutable y portable. La misma imagen que corre en tu notebook corre idéntica en el servidor de QA, en la nube, o en la máquina de tu compañero (con el matiz de las imágenes *multi-arch*: mismo artefacto, resuelto por arquitectura). El entorno deja de ser implícito y artesanal: pasa a estar **declarado en un archivo versionado** (el Dockerfile). ¿Te suena? Es la regla de la materia otra vez: *si no está en el repo, no existe*.

Esta idea es la base física de todo lo que sigue en la materia: el pipeline del TP4 va a verificar tu código en cada cambio, el del TP7 va a construir y publicar las imágenes, y los entornos de los TPs 6 y 7 van a desplegarlas. **El contenedor es la unidad de entrega del software moderno.**

### 2.2 Contenedores vs máquinas virtuales

La primera solución al problema del entorno fueron las máquinas virtuales: si el problema es el entorno, virtualicemos la máquina completa. Funciona, pero es pesado. El contenedor logra un aislamiento **suficiente para la mayoría de los casos** —no equivalente: la tabla de abajo lo dice— con otra arquitectura:

| | Máquina Virtual | Contenedor |
|---|---|---|
| Qué virtualiza | Hardware completo (con su propio SO invitado) | Solo el proceso (comparte el kernel Linux — en Mac/Windows, el de una VM) |
| Peso típico | GBs (SO completo adentro) | MBs (solo app + dependencias) |
| Arranque | Minutos | Segundos o menos |
| Densidad | Pocas por host | Decenas/cientos por host |
| Aislamiento | Fuerte (hipervisor) | De proceso (namespaces + cgroups del kernel) |
| Caso de uso hoy | Aislar sistemas operativos / tenants | Empaquetar y desplegar aplicaciones |

El truco técnico: un contenedor **no tiene su propio sistema operativo** — es un proceso al que el **kernel de Linux** le miente elegantemente usando *namespaces* (le muestra su propio árbol de procesos, su propia red, su propio filesystem) y *cgroups* (le limita CPU/memoria). Por eso arranca en milisegundos y pesa megas: no hay nada que "bootear".

> ⚠️ **«El kernel del host» es literal sólo en Linux.** En Mac y en Windows —o sea, en la mayoría de las máquinas de esta materia— no hay kernel de Linux para compartir: Docker Desktop levanta una **máquina virtual con Linux adentro**, y tus contenedores comparten *ese* kernel. Todo lo demás del modelo vale igual, pero explica dos cosas concretas que vas a ver: por qué `/var/lib/docker/volumes/…` no aparece en tu disco (§3.6), y por qué escribir en una carpeta compartida con tu disco es más lento que en un volumen.

Ojo: no son tecnologías enemigas — en la nube conviven: los proveedores corren tus contenedores **adentro** de máquinas virtuales (aislamiento entre clientes por VM, empaquetado de apps por contenedor).

### 2.3 Imagen, contenedor y registry: el modelo mental

La tríada central, con una analogía que ya conocés de programación:

- **Imagen** = la **clase**: el paquete inmutable — tu app + dependencias + filesystem base — construido en capas (*layers*) de solo lectura. Se identifica como `registry/usuario/imagen:tag` (ej: `ghcr.io/ariel/mi-api:v0.1.0`).
- **Contenedor** = la **instancia**: una imagen en ejecución. Podés correr N contenedores de la misma imagen. Cada uno agrega una **capa de escritura efímera** arriba de las capas de la imagen: todo lo que el contenedor escribe vive ahí… y **muere con el contenedor**. (De ahí la necesidad de volúmenes — §2.5.)
- **Registry** = el **repositorio de imágenes**: donde se publican y desde donde se descargan (Docker Hub, ghcr.io, ACR). Es a las imágenes lo que GitHub es al código — y en el TP7 va a ser el puente entre tu pipeline de CI y tus entornos de deploy.

**Las capas importan más de lo que parece.** Las instrucciones que tocan el sistema de archivos —`RUN`, `COPY` y `ADD`— generan una capa cada una; las demás (`WORKDIR`, `EXPOSE`, `ENV`, `ENTRYPOINT`, `CMD`) sólo escriben metadatos en la imagen y no agregan peso. Y las capas se **cachean y se comparten**: si dos imágenes parten de la misma base, la base se almacena una sola vez; si rebuildeás y solo cambió la última capa, las anteriores salen del cache. Entender esto explica por qué el orden de las instrucciones del Dockerfile afecta el tiempo de build (§2.4) y por qué el primer `docker pull` es lento y los siguientes casi instantáneos.

### 2.4 El Dockerfile y los multi-stage builds

El **Dockerfile** es la receta declarativa de la imagen: partí de esta base (`FROM`), copiá estos archivos (`COPY`), ejecutá estos comandos (`RUN`), **documentá** este puerto (`EXPOSE` — no lo publica; eso lo hacen `-p` y `ports:`), y arrancá así (`ENTRYPOINT`/`CMD`). Es código: se versiona, se revisa en PRs, se audita.

Dos reglas de oro que separan un Dockerfile amateur de uno profesional:

1. **Orden de instrucciones = estrategia de cache.** Cuando una capa cambia, Docker invalida esa capa **y todas las posteriores** (todas las instrucciones que siguen en el Dockerfile). Por eso se copia primero el archivo de dependencias (`.csproj`, `package.json`) y se restauran dependencias ANTES de copiar el resto del código: así, cambiar una línea de código no re-descarga todas las dependencias.
2. **Multi-stage: el SDK no viaja a producción.** Compilar necesita el SDK completo (compilador, herramientas); ejecutar solo necesita el runtime. Un build multi-etapa usa una etapa `build` con el SDK y una etapa final que **solo copia los binarios** sobre una imagen de runtime mínima. Resultado: imagen final varias veces más chica, más rápida de transferir, y con **menos superficie de ataque** (no hay compilador ni herramientas que un atacante pueda aprovechar). En esta materia el multi-stage es obligatorio.

**ENTRYPOINT vs CMD (los dos definen "cómo arranca", pero distinto):** `ENTRYPOINT` define el **ejecutable** del contenedor (lo que corre salvo que lo pises con `docker run --entrypoint`); `CMD` define los **argumentos por defecto**, que se pueden reemplazar al invocar `docker run imagen <otros-args>`. Combinados: `ENTRYPOINT ["dotnet", "MiApi.dll"]` hace que el contenedor "sea" tu app; un `CMD ["--verbose"]` agregaría un flag por defecto reemplazable. Si solo hay `CMD`, todo el comando es reemplazable desde `docker run`.

### 2.5 Persistencia: los volúmenes

Consecuencia directa del modelo de capas: **los contenedores son efímeros por diseño**. Borrás el contenedor, su capa de escritura muere. Eso es una *feature* para la app (contenedores descartables y reemplazables = deploys y rollbacks triviales) y una catástrofe para los datos (¿la base de datos pierde todo con cada recreación?).

La solución es la separación explícita: **el estado vive en volúmenes** — almacenamiento gestionado por Docker que se monta dentro del contenedor y **sobrevive** a su destrucción. El contenedor de PostgreSQL puede morir y recrearse mil veces; sus datos, montados en `db_data:/var/lib/postgresql/data`, persisten. Esta separación contenedor-efímero / estado-persistente es un principio arquitectónico que te vas a cruzar toda la carrera (y es la pregunta de defensa más popular de este TP).

### 2.6 Redes: los servicios se encuentran por nombre

Docker crea redes virtuales internas donde cada contenedor es alcanzable **por su nombre** vía un DNS embebido. En la práctica: tu backend no se conecta a `localhost:5432` ni a una IP — se conecta a `db:5432`, donde `db` es el nombre del servicio en el compose. Esto desacopla la topología: no importa en qué IP cayó el contenedor de la base, siempre es `db`.

**Tres ámbitos** que confunden al principio (y explican el 90% de los problemas de red del TP):

1. **Dentro de la red de compose** (contenedor → contenedor): por nombre de servicio y puerto interno. Es el caso `backend → db` (`Host=db`).
2. **Desde tu máquina (host)**: solo llegás a los puertos explícitamente publicados (`ports: "8080:8080"`).
3. **⚠️ El caso trampa — la SPA**: si tu frontend es una SPA (React/Angular/Vue servida por nginx), el JavaScript **se ejecuta en el BROWSER**, que vive en el host — ¡no dentro de la red de compose! Por eso tu front NO puede pedirle nada a `http://backend:8080`: ese nombre no resuelve en tu browser. El nombre de servicio solo aplica a tráfico que **nace dentro de un contenedor** (back→db, o nginx→back).

   Hay dos formas de resolverlo, y **la de este TP es la primera**:

   **(a) Ruta relativa + proxy en el front (lo que usa el sample y lo que recomendamos).** Tu SPA llama a `/api/...`, sin host ni puerto escritos en ninguna parte. Quien traduce esa ruta es el servidor de desarrollo (Vite/CRA) cuando trabajás local, y **nginx** cuando corre en contenedor. El frontend no sabe dónde vive el backend, así que la misma imagen sirve en cualquier entorno — que es exactamente lo que el TP7 va a necesitar. Y como para el browser todo sale del mismo origen, **no hay CORS que configurar**. La configuración completa de nginx está en §3.5.

   **(b) URL absoluta al puerto publicado + CORS.** Tu SPA llama a `http://localhost:8080`, el puerto que el compose publica del backend. Funciona, pero tiene dos costos: `localhost:3000` llamando a `localhost:8080` es *cross-origin*, así que tu backend necesita **CORS** habilitado para ese origen (si en dev usabas el proxy de Vite, ese proxy desaparece al servir con nginx y el browser empieza a bloquear los fetch: ves el sistema "levantado" pero muerto); y la URL queda **escrita en el código del front**, así que cambiar de entorno obliga a recompilar la imagen.

   Si tu app ya está escrita con URLs absolutas, no hace falta que la reescribas para este TP — pero anotá en `decisiones.md` cuál de los dos caminos usaste y por qué.

### 2.7 Compose: el sistema completo, declarado

Tu sistema no es UN contenedor: son varios (front, back, BD) con red común, orden de arranque, variables y volúmenes. `docker-compose.yml` **declara** todo eso en un archivo versionado, y `docker compose up` lo materializa. Tres ideas para llevarte:

- **Declarativo, no imperativo**: el archivo describe el estado deseado ("existen estos 3 servicios, conectados así"), no los pasos para llegar. Es tu primera experiencia de infraestructura declarada — la semilla conceptual del TP8 (IaC).
- **`depends_on` no es suficiente**: garantiza el orden de *arranque*, no de *readiness*. Que el contenedor de la BD haya arrancado no significa que acepte conexiones — para eso está el `healthcheck` + `condition: service_healthy`. La diferencia "arrancó" vs "está listo" reaparece en cada sistema distribuido que toques.
- **Los secretos no se declaran en el YAML**: van en un `.env` que NO se commitea (con un `.env.example` que sí). En el TP4 esos secretos migran a la plataforma de CI — la disciplina arranca acá.

## 3- Desarrollo de la guía

> 🔴 **Esta guía se hace DOS veces, y el orden importa.**
>
> **Primera pasada — sobre la app de la cátedra (§3.2), para practicar.** Todos trabajan sobre la
> misma aplicación, así que podés comparar lo que te sale contra lo que tiene que salir, y cuando
> algo no da, el problema está acotado a Docker y no a tu proyecto. **Esta pasada no se entrega.**
>
> **Segunda pasada — sobre TU app, cuando la tengas elegida (§3.8).** Es la que se entrega y la que
> vas a defender. Para entonces ya no vas a estar aprendiendo Docker: vas a estar resolviendo lo
> propio de tu proyecto, que es donde de verdad se te va el tiempo.
>
> Saltearse la primera pasada parece un atajo y es lo contrario: es aprender Docker y pelear con tu
> stack al mismo tiempo, sin saber cuál de los dos te está fallando.

> 🖥️ **Sobre el sistema operativo.** Los comandos de esta guía —y el video— están escritos para una terminal **macOS/Linux** (bash o zsh). **Los de Docker son idénticos en los tres sistemas**: `docker build`, `docker run`, `docker compose up`, `docker push` se escriben igual en Windows, Mac y Linux. Lo que cambia son los comandos **del sistema operativo** que usamos alrededor (listar archivos, copiar, matar un proceso). Si estás en Windows, esta tabla es tu traductor:
>
> | Para qué | macOS / Linux (bash, zsh) | Windows (PowerShell) |
> |---|---|---|
> | Listar archivos | `ls` | `ls` o `dir` |
> | Listar **incluyendo los ocultos** (los que empiezan con punto) | `ls -a` | `ls -Force` |
> | Ver un archivo | `cat archivo` | `cat archivo` |
> | Ver las últimas líneas de un log | `tail -3 archivo` | `Get-Content archivo -Tail 3` |
> | Copiar un archivo | `cp origen destino` | `cp origen destino` |
> | Definir una variable de entorno | `export CLAVE=valor` | `$env:CLAVE = "valor"` |
> | Mandar un proceso a segundo plano con su log a un archivo | `comando > /tmp/back.log 2>&1 &` | `Start-Process comando -RedirectStandardOutput $env:TEMP\back.log` |
> | **Ver quién ocupa el puerto 8080** | `lsof -ti:8080 -sTCP:LISTEN` | `Get-NetTCPConnection -LocalPort 8080 -State Listen \| Select OwningProcess` |
> | **Matar ese proceso** | `kill $(lsof -ti:8080 -sTCP:LISTEN)` | `Stop-Process -Id <pid> -Force` |
> | Pedirle algo a una URL | `curl -s <url>` | `curl.exe -s <url>` ⚠️ con `.exe` |
> | Filtrar líneas de una salida | `grep <texto>` | `Select-String <texto>` |
> | Carpeta temporal | `/tmp` | `$env:TEMP` |
> | **Partir un comando en varias líneas** | `\` al final de la línea | `` ` `` (acento grave) al final |
> | **Encadenar dos comandos** | `uno && otro` | `uno; otro` (en PowerShell 5.1, el que trae Windows por defecto) |
>
> ⚠️ **La barra invertida al final de línea NO existe en PowerShell.** En esta guía los comandos
> largos están escritos **en una sola línea** justamente por eso: se copian y se pegan igual en los
> tres sistemas. Si en algún ejemplo ves un `\` al final de un renglón y estás en PowerShell,
> reemplazalo por un acento grave `` ` `` — y cuidado, no puede quedar **ningún espacio después**.
> Si te olvidás, el error no menciona el problema: vas a ver cuatro mensajes seguidos del estilo
> `docker: invalid reference format` y `-e : The term '-e' is not recognized…`.
>
> ⚠️ **La trampa de PowerShell con `curl`**: ahí `curl` es un alias de `Invoke-WebRequest`, que **no** acepta las mismas opciones. Escribí `curl.exe` (con la extensión) para usar el curl de verdad, o `Invoke-RestMethod <url>` que además te muestra el JSON ya formateado (y te ahorra el `jq`).
>
> 💡 **Alternativa que evita todo esto**: si tenés WSL (Windows Subsystem for Linux) o Git Bash, ahí los comandos de la columna de la izquierda funcionan tal cual. Docker Desktop en Windows se integra con WSL sin configuración extra, y es lo que recomendamos.

### 3.1 Instalar Docker

- Docker Desktop (Windows/Mac) o Docker Engine (Linux): https://docs.docker.com/get-docker/
- **Verificá que quedó bien instalado** antes de seguir:

```bash
docker --version           # si dice "command not found": no está instalado
docker compose version     # sin guion en el medio: compose viene adentro de Docker
docker ps                  # ⚠️ éste es el que prueba que el MOTOR está prendido
```

Ojo con la diferencia, porque confunde: los dos primeros contestan igual **aunque Docker Desktop esté cerrado** — sólo imprimen la versión del programa que tenés instalado, sin hablar con nada. El que necesita el motor de fondo es el tercero. Si `docker ps` dice `Cannot connect to the Docker daemon`, está instalado pero **apagado**: abrí Docker Desktop y esperá a que arranque. Y si copiaste un tutorial viejo que usa `docker-compose` con guion, ése era el programa aparte de hace unos años — hoy es un subcomando.

> 🪟 **Si estás en Windows**, Docker Desktop pide cuatro cosas que conviene saber antes de empezar,
> porque en una notebook prestada o corporativa pueden ser un muro: **WSL 2** (el instalador lo
> ofrece; en algunas versiones hay que instalar el kernel aparte), **virtualización habilitada en la
> BIOS/UEFI**, **permisos de administrador** para instalar, y **un reinicio**. Docker Desktop es
> gratis para uso personal y educativo. Dejate un rato para esto: no es un `next-next-finish`.

> 💡 Si querés la prueba completa de que el motor levanta contenedores: `docker run hello-world`.
> Baja una imagen mínima, la corre e imprime un mensaje. No hace falta para seguir la guía.

**✅ Checkpoint:** los dos comandos de arriba responden con su número de versión.

### 3.2 Descargar la app con la que vas a hacer esta guía

La guía necesita una app concreta sobre la cual practicar. Para que puedas arrancar **hoy** —sin esperar a decidir cuál va a ser tu app del semestre (§3.3)— la cátedra publica un sample chico y ya probado: [`ingsoft3ucc/demo-fullstack`](https://github.com/ingsoft3ucc/demo-fullstack) — backend **.NET 8** (minimal API), frontend **React + Vite**, base **PostgreSQL**. Es la misma app de las demos en clase.

> ⚠️ Es la app de **práctica**, no tu entrega. El TP se entrega sobre **tu** app del semestre: clonar el sample y entregarlo no cuenta. Si ya tenés elegida tu app y preferís hacer la guía directamente sobre ella, podés: saltá a §3.3.

**Prerequisitos para correrla local:** `git` (ya lo usaste en el TP1), [.NET SDK 8](https://dotnet.microsoft.com/download/dotnet/8.0), [Node 20 o superior](https://nodejs.org) (el Dockerfile del sample usa la 22) y Docker (§3.1). Verificá los dos primeros con `dotnet --version` y `node --version` antes de seguir.

**Paso 1 — clonar la rama de arranque.** `demo-c2-inicio` es el repo *sin* Dockerfiles ni compose: justamente lo que vas a escribir vos en §3.4–§3.7.

```bash
git clone --branch demo-c2-inicio https://github.com/ingsoft3ucc/demo-fullstack.git practica-tp2
cd practica-tp2
```

> 💡 La rama `main` del mismo repo tiene la **solución completa** (Dockerfiles + compose ya escritos). Miralá **después** de intentarlo por tu cuenta, o si te trabás: `git switch main`.

**Paso 2 — la base de datos (tu primer contenedor útil).** El backend espera un PostgreSQL en `localhost:5432` con la base `app` ya creada. En vez de instalar PostgreSQL en tu máquina, levantalo como contenedor —para eso instalaste Docker:

```bash
docker run -d --name pg-tp2 -e POSTGRES_PASSWORD=postgres -e POSTGRES_DB=app -p 5432:5432 postgres:16-alpine

docker ps    # el contenedor pg-tp2 debe figurar corriendo
```

Eso coincide con la connection string que ya trae `backend/DemoApi/appsettings.json` (`Host=localhost;Database=app;Username=postgres;Password=postgres`). ¿Ya tenés un PostgreSQL propio ocupando el 5432? Publicá otro puerto (`-p 5433:5432`) y ajustá la cadena a `Host=localhost;Port=5433;…`. ⚠️ **Y acordate de agregarle `Port=5433;` también a la variable `ConnectionStrings__Default` de §3.4**: allá el host cambia a `host.docker.internal`, pero el puerto sigue siendo el que publicaste. Si te lo olvidás, el contenedor va a buscar el 5432 —que es tu PostgreSQL de siempre, el que no tiene la base `app`— y se va a morir con un error que no tiene nada que ver con Docker.

**Las tablas las crea la app al arrancar** (`db.Database.EnsureCreated()` en `Program.cs`): no hay script de schema que correr a mano. Guardate el dato — en §3.6 vuelve a hacer falta, porque el PostgreSQL del compose también nace vacío.

**Paso 3 — levantar back y front:**

```bash
# terminal 1 → API en http://localhost:8080
cd backend && dotnet run --project DemoApi

# terminal 2 → SPA en http://localhost:5173 (Vite proxea /api al backend)
cd frontend && npm install && npm run dev
```

> 🔴 **Estos comandos son de ESTA tecnología, no del universo.** El sample es .NET + Node; el tuyo puede ser cualquier cosa. Lo que no cambia es el par de pasos —**instalar dependencias** y **arrancar**—, y es el punto de partida de lo que después vas a escribir en tu Dockerfile (⚠️ **no se copian tal cual**: adentro del Dockerfile van sus equivalentes de producción, ver el recuadro de abajo). Anotá los tuyos ahora:
>
> | Stack | Instalar dependencias | Arrancar (desarrollo) |
> |---|---|---|
> | .NET | `dotnet restore` (implícito en `run`) | `dotnet run --project <Proyecto>` |
> | Node / Express | `npm install` | `npm start` o `node index.js` |
> | React / Vite | `npm install` | `npm run dev` |
> | Angular | `npm install` | `ng serve` |
> | Java / Maven | `mvn install` | `mvn spring-boot:run` |
> | Java / Gradle | `./gradlew build` | `./gradlew bootRun` |
> | Python / Django | `pip install -r requirements.txt` | `python manage.py runserver` |
> | Python / FastAPI | `pip install -r requirements.txt` | `uvicorn main:app --reload` |
> | Go | `go mod download` | `go run .` |
>
> 🔴 **Ojo con la diferencia entre desarrollo y producción**, porque es la confusión más común al escribir el Dockerfile. `dotnet run` (o `npm run dev`) es **de desarrollo**: hace todo junto —restaurar, compilar, ejecutar— y encima se queda escuchando cambios. En el Dockerfile eso va **separado en tres pasos**, y el último es la versión de producción:
>
> | | Desarrollo (tu máquina) | En el Dockerfile |
> |---|---|---|
> | .NET | `dotnet run --project X` | `dotnet restore` → `dotnet publish -c Release` → `ENTRYPOINT ["dotnet","X.dll"]` |
> | Node/Express | `npm run dev` | `npm ci` → (`npm run build`) → `CMD ["node","dist/index.js"]` |
> | React/Vite | `npm run dev` | `npm ci` → `npm run build` → lo sirve nginx |
> | Python | `python manage.py runserver` | `pip install -r requirements.txt` → `CMD ["gunicorn",...]` |
> | Java | `mvn spring-boot:run` | `mvn package` → `ENTRYPOINT ["java","-jar","app.jar"]` |
>
> Anotá **los dos juegos** de comandos de tu proyecto: los vas a necesitar a los dos.

> 💡 **¿Preferís una sola terminal?** Mandalos a segundo plano **con el log a un archivo**, o no vas a poder ni escribir: los dos procesos siguen imprimiendo encima de lo que tipeás.
> ```bash
> (cd backend && dotnet run --project DemoApi > /tmp/back.log 2>&1 &)
> (cd frontend && npm install --silent && npm run dev > /tmp/front.log 2>&1 &)
> ```
> 🪟 En Windows esto no tiene traducción cómoda a PowerShell (`Start-Process` necesita separar
> ejecutable y argumentos, y no acepta el `cd … &&`): **usá dos terminales**, que es la forma
> recomendada igual.
>
> Para ver qué pasó: `tail -3 /tmp/back.log`, que muestra las últimas tres líneas — es lo que vas a ver en el video. Si querés seguirlo en vivo, `tail -f /tmp/back.log` y `Ctrl-C` para salir del seguimiento (no mata el backend).

```bash
curl -s http://localhost:8080/health | jq     # {"status": "ok"}
```

> 🪟 **Windows, dos avisos que valen para TODOS los `curl` de esta guía.** (1) En PowerShell `curl` es
> un alias de otra cosa: escribí **`curl.exe -s ...`**, o usá `Invoke-RestMethod <url>`, que además te
> muestra el JSON ya formateado. (2) **`jq` no viene instalado en Windows**: instalalo con
> `winget install jqlang.jq`, o sacá el `| jq` y listo — la respuesta se lee peor, pero funciona
> igual. En macOS y en la mayoría de las distros de Linux ya está.

**✅ Checkpoint:** en `http://localhost:5173` creás una tarea, aparece en la lista, y **sigue ahí después de refrescar** (está en la BD, no en memoria). Recién ahora tiene sentido contenerizar: ya sabés cómo se ve "funcionando" antes de meter Docker en el medio.

> 🧹 Cuando llegues al compose (§3.6), apagá esta BD de práctica para no quedarte con dos PostgreSQL dando vueltas: `docker rm -f pg-tp2`. Y apagá también los contenedores sueltos del back y del front que hayas dejado corriendo (§3.4 y §3.5): compose quiere esos mismos puertos.

### 3.3 Elegir la app del semestre ⭐ (la decisión más importante del TP)

Esta app te acompaña hasta el final: en TP4 le armás CI, en TP5 tests y análisis estático, en TP6 la desplegás con aprobaciones, en TP7 viaja como imagen, en TP8 su infra se define como código, en TP9 la asegurás y monitoreás, y es la base del **Integrador**.

**Requisitos mínimos:**
- Un servicio de **backend** con API (el lenguaje lo elegís vos: .NET, Node, Java, Python, Go…).
- Un servicio de **frontend** (SPA o SSR: React, Angular, Vue…).
- Interacción con una **base de datos** (relacional o no).

**Puede ser:** un desarrollo propio tuyo, o un proyecto open-source de GitHub que entiendas y puedas modificar.

> 🔴 **La app es INDIVIDUAL: cada uno tiene que traer una aplicación DISTINTA.** Dos personas no pueden entregar el mismo sistema. Este trabajo práctico y todos los que siguen —hasta el integrador— se hacen y se defienden de a uno, sobre tu propia aplicación. Por eso **no sirve la app de un trabajo grupal de otra materia**: si tres compañeros eligen la misma, las tres entregas son la misma y la defensa individual deja de tener sentido. Si el proyecto que querés usar es de un grupo, tomá **tu** parte y armá algo tuyo con eso, o elegí otra cosa.

**Criterios de elección (documentalos en `decisiones.md`):**
- ¿Buildea y corre localmente hoy, sin magia? (probalo ANTES de comprometerte)
- ¿Tiene (o podés escribirle) tests? Lo vas a necesitar en TP5.
- ¿Entendés el código lo suficiente como para modificarlo? (el Integrador exige hacer cambios en vivo)
- Tamaño: alcanza con CRUD + 2-3 pantallas. **Más grande no suma nota, solo suma fricción.**

**¿En qué repo vive?** En el **mismo repo del TP1** — el de las protecciones y el `decisiones.md`: el código de la app entra ahí (por PR, claro), y así tu **historial del semestre es uno solo** — que es exactamente lo que el Integrador evalúa como evidencia. ¿Preferís arrancar un repo nuevo (típico si adaptás un proyecto de GitHub)? Podés: migrá `decisiones.md`/`evidencias.md`, **recreá las protecciones del TP1** (su §4.4) en el repo nuevo, avisá a la cátedra, y ese repo pasa a ser TU repo del semestre — el historial vale desde ahí.

**✅ Checkpoint:** app elegida, corre localmente (la BD puede ser un contenedor, como en §3.2 — no hace falta instalar el motor en tu máquina), vive en tu repo del semestre (con protecciones), y anotaste en `decisiones.md` por qué esa.

**Política de cambio de app** (para que la decisión no te paralice): podés cambiar de app **hasta el TP4 inclusive**, rehaciendo sobre la nueva lo ya entregado (TP2/TP3 — que al ser chicos, se rehacen rápido). Después del TP4 el costo de cambio crece mucho: consultá a la cátedra antes. Elegir con cuidado hoy sigue siendo lo barato.

### 3.4 Dockerfile del backend (multi-stage)

> 📋 **Los archivos que vas a escribir en esta guía.** Son la entrega. (El `.env.example` es la
> excepción: ése sí viene con el clon del sample, para que el `cp` de §3.6 funcione — pero en **tu**
> app lo escribís vos.)
> Tenelos a mano como checklist.
>
> | Archivo | Dónde va | Sección |
> |---|---|---|
> | `backend/Dockerfile` | al lado de tu `.sln` / `package.json` del back | §3.4 |
> | `backend/.dockerignore` | misma carpeta que el Dockerfile del back | §3.4 |
> | `frontend/Dockerfile` | al lado del `package.json` del front | §3.5 |
> | `frontend/.dockerignore` | misma carpeta que el Dockerfile del front | §3.5 |
> | `frontend/nginx.conf` | misma carpeta que el Dockerfile del front | §3.5 |
> | `docker-compose.yml` | **raíz** del proyecto | §3.6 |
> | `.env.example` + `.env` | **raíz**, al lado del compose | §3.6 |
> | `docker-compose.registry.yml` | **raíz** | §3.7 |
>
> ✍️ **Creá estos archivos con VS Code o cualquier editor de código, no con el Bloc de notas**: Notepad
> guarda `Dockerfile.txt` sin avisarte, y Windows oculta la extensión, así que en el explorador vas a
> ver `Dockerfile` y Docker te va a decir `failed to read dockerfile: no such file or directory`
> hablando de un archivo que estás viendo en pantalla.

**`backend/Dockerfile`** — éste es el del sample, el que se construye y se recorre en el video. Si tu
app es otra, **cambian los nombres; la estructura de dos etapas no**:

```dockerfile
# ---- Etapa 1: build (imagen con SDK) ----
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src

# Solución multi-proyecto: se copian el .sln y cada .csproj (respetando rutas)
# antes del restore — capa cacheable: solo se rehace si cambian los .csproj.
COPY Backend.sln .
COPY DemoApi/DemoApi.csproj DemoApi/
COPY DemoApi.Tests/DemoApi.Tests.csproj DemoApi.Tests/
RUN dotnet restore Backend.sln

COPY . .
RUN dotnet publish DemoApi/DemoApi.csproj -c Release -o /app/publish

# ---- Etapa 2: runtime (imagen mínima, sin SDK) ----
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS final
WORKDIR /app
COPY --from=build /app/publish .
EXPOSE 8080
ENTRYPOINT ["dotnet", "DemoApi.dll"]
```

> 💡 **Si tu solución .NET tiene un solo proyecto** (sin `.sln`), las tres líneas de `COPY` de arriba
> se reducen a una: `COPY MiApi.csproj .` seguida de `RUN dotnet restore`. El principio es el mismo —
> **los archivos de proyecto primero, el código después**—, cambia cuántos archivos hay que copiar.
> Y acordate de que el `ENTRYPOINT` lleva **el nombre de tu `.dll`**, no el del ejemplo.
>
> 💡 **Puertos según versión de .NET**: la imagen `aspnet:8.0` escucha en **8080** por default; en .NET 6/7 el default era **80**. Si tu app es ≤7: mapeá `-p 8080:80` (y en compose `"8080:80"`) o seteá `ASPNETCORE_URLS=http://+:8080` (la variable `ASPNETCORE_HTTP_PORTS` existe recién desde .NET 8). Si el contenedor "anda pero no responde", empezá por acá.

**`backend/.dockerignore`** — va **al lado del Dockerfile**, no en la raíz del repo: Docker lo busca
en la carpeta que le pasás como contexto (`./backend`). Por eso son **dos** archivos, uno por
contexto, con contenidos distintos:

```
**/bin/
**/obj/
**/TestResults/
.git/
```

> 🔴 **Los asteriscos importan.** `bin/` solo excluye un `bin` que esté en la raíz del contexto; tu
> solución los tiene adentro de cada proyecto (`DemoApi/bin`, `DemoApi.Tests/obj`), y `**/bin/` los
> agarra en cualquier nivel. Si no los excluís, el `COPY . .` mete adentro del contenedor el
> `obj/project.assets.json` que generó tu `dotnet run` de §3.2 —con rutas de **tu** máquina— pisando
> el que acaba de generar el `restore`. El error que vas a ver habla de un paquete que sí existe:
> `error NETSDK1064: Package … was not found`. Imposible de diagnosticar sin saber esto.
>
> Ojo: el nombre empieza con punto, así que en un `ls` común no aparece; para verlo, `ls -a`.

> 💡 Para **leer** estos archivos en la terminal alcanza con `cat`. Si querés colores y numeración de línea —útil cuando alguien te dice "mirá la línea 7"— instalá [`bat`](https://github.com/sharkdp/bat) (`brew install bat` en macOS, `apt install bat` en Ubuntu —ojo, ahí el comando queda `batcat`, no `bat`—, `winget install sharkdp.bat` en Windows) y usá `bat --paging=never <archivo>` (sin ese flag abre un paginador y hay que salir con `q`). Es lo que vas a ver en el video.
- Build y prueba:

> 🧹 **Antes del `docker run`: apagá el backend que dejaste corriendo en §3.2.** Los dos quieren el puerto 8080 y Docker corta con `port is already allocated`. Si lo levantaste en primer plano, `Ctrl-C` en esa terminal; si lo mandaste a segundo plano con la receta de arriba —la del paréntesis—, `kill %1` **no** te va a servir (el proceso no queda en la tabla de trabajos de tu terminal): usá (o, sin depender del número de trabajo, `kill $(lsof -ti:8080 -sTCP:LISTEN)`). La BD (`pg-tp2`) dejala corriendo: la va a usar el contenedor.

```bash
ls backend                 # todavía no hay Dockerfile: lo escribís vos
bat --paging=never backend/Dockerfile        # (o `cat`) — leelo entero antes de construir
bat --paging=never backend/.dockerignore     # el otro archivo que escribiste acá
docker build -t mi-backend:dev ./backend
docker run --rm -d --name mi-backend-run -p 8080:8080 -e ConnectionStrings__Default='Host=host.docker.internal;Database=app;Username=postgres;Password=postgres' mi-backend:dev

bat --paging=never backend/DemoApi/appsettings.json   # el archivo que la variable viene a pisar
curl -s localhost:8080/health | jq                   # tiene que contestar {"status":"ok"}

# Y ahora compará los tamaños — esto va a `evidencias.md`:
docker images mcr.microsoft.com/dotnet/sdk:8.0       # la imagen que COMPILA
docker images mcr.microsoft.com/dotnet/aspnet:8.0    # la que solo EJECUTA
docker images mi-backend:dev                         # la tuya
```

> 🐧 **Si estás en Linux**: `host.docker.internal` **no existe** en Docker Engine (es una comodidad de Docker Desktop, que está en Mac y Windows). Agregale el flag que lo crea:
> ```bash
> docker run --rm -d --name mi-backend-run --add-host=host.docker.internal:host-gateway -p 8080:8080 -e ConnectionStrings__Default='Host=host.docker.internal;Database=app;Username=postgres;Password=postgres' mi-backend:dev
> ```
> Si te da `Name or service not known`, es esto.
>
> 🔴 **¿Por qué esa variable de entorno, si la connection string ya está en `appsettings.json`?** Porque ahí dice `Host=localhost`, y **adentro del contenedor `localhost` es el contenedor mismo**, no tu máquina. La app arranca, no encuentra ninguna base, y el contenedor **se muere en el arranque** (`Failed to connect to 127.0.0.1:5432`). `host.docker.internal` es el nombre con el que un contenedor llega a la máquina que lo hospeda.
>
> Y fijate **cómo** se arregla: no editando el código ni rehaciendo la imagen, sino pasándole una variable al arrancar. Ése es el principio que sostiene todo lo que viene: **la misma imagen, distinta configuración según dónde corra**. En §3.6 la misma variable va a apuntar a `db` —el nombre del servicio— y en el TP6, a la base de producción. Si tu app no lee su configuración del entorno, este es el momento de arreglarla: en el TP2 es un rato, en el TP6 es un problema.
>
> *(¿Tu app lee la conexión con otro nombre? Buscá el nombre exacto en tu código. En .NET, `ConnectionStrings__Default` corresponde a `ConnectionStrings:Default` — el doble guión bajo es cómo se anidan las claves. En Node suele ser `DATABASE_URL`; en Spring, `SPRING_DATASOURCE_URL`.)*

**✅ Checkpoint:** la API responde corriendo en contenedor. Comparar `docker images`: ¿cuánto pesa tu imagen final vs la imagen del SDK?

### 3.5 Dockerfile del frontend (multi-stage: build + servidor estático)

Acá son **tres** archivos, no uno: el Dockerfile, su `.dockerignore`, y la configuración del servidor
web. El tercero es el que más se olvida y el que hace que el sistema "levante" pero no funcione.

**`frontend/Dockerfile`**:

```dockerfile
# ---- Etapa 1: build ----
FROM node:22-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci                 # ci, no install: respeta el lockfile
COPY . .
RUN npm run build

# ---- Etapa 2: nginx sirve los estáticos ----
FROM nginx:alpine
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 80
```

> 💡 **Ajustá la ruta de salida del build a tu framework**: Vite emite `dist/`, Create React App `build/`, Angular `dist/<proyecto>/browser/`. Si el `COPY --from` falla, el 99% de las veces es esta ruta.
> **¿Tu front es SSR (Next.js, Nuxt)?** El patrón nginx-estáticos no aplica: la etapa final necesita el runtime de Node (`FROM node:22-alpine` + `CMD ["node", "server.js"]` o el standalone output de Next). Consultá la doc de deploy con Docker de tu framework.

**`frontend/.dockerignore`** — mismo principio que el del backend, contenido distinto:

```
node_modules/
dist/
.git/
```

> 🔴 **Por qué `node_modules/` no es opcional acá.** Si no lo excluís, el `COPY . .` copia los
> `node_modules` **de tu máquina** encima de los que `npm ci` acaba de instalar para Linux. Vite usa
> binarios compilados por plataforma, así que el build muere con
> `You installed esbuild for another platform than the one you're currently using` o
> `Cannot find module @rollup/rollup-linux-x64-musl`. Nada en el mensaje sugiere que el problema sea
> un archivo que no escribiste.

**`frontend/nginx.conf`** — 🔴 **éste es el archivo que hace que la app funcione de verdad.** Tu SPA
llama a `/api/...` con una ruta **relativa** (sin host ni puerto — así se escribe un frontend
portable). En desarrollo, quien traduce esa ruta hacia el backend es el servidor de Vite; en el
contenedor, Vite ya no está: quien tiene que traducirla es nginx, y no lo hace solo. Hay que
decírselo acá:

```nginx
server {
    listen 80;

    # SPA: estáticos, con fallback a index.html para las rutas del router
    location / {
        root /usr/share/nginx/html;
        index index.html;
        try_files $uri $uri/ /index.html;
    }

    # Todo lo que llegue a /api se reenvía al backend por la red interna de compose.
    # Acá SÍ vale el nombre de servicio: nginx corre DENTRO de la red — y al ser
    # same-origin para el browser, además evita CORS (§2.6).
    #
    # 🔴 El nombre va en una VARIABLE, no escrito directo en el proxy_pass. Con el
    # nombre directo, nginx lo resuelve AL ARRANCAR y, si el backend todavía no
    # existe, se niega a levantar: «host not found in upstream». O sea que el
    # contenedor del frontend no podría correr solo — que es justo lo que hacemos
    # dos líneas más abajo. Con variable, resuelve recién cuando llega un pedido.
    resolver 127.0.0.11 valid=10s ipv6=off;
    set $backend_api http://backend:8080;

    location /api/ {
        proxy_pass $backend_api;
    }
}
```

> 🔴 **Dos detalles que parecen de más y no lo son.**
> 1. **El `proxy_pass` va SIN barra al final.** Con `proxy_pass http://backend:8080/;` nginx reescribe
>    el prefijo y manda `/api/tareas` a `/tareas` — tu API devuelve 404 en todas las llamadas.
> 2. **Un solo `resolver`, el de Docker (`127.0.0.11`).** Agregar un DNS público "por las dudas"
>    (`8.8.8.8`) hace que nginx reparta las consultas entre los dos, y el público no sabe qué es
>    `backend`: el resultado es un **502 intermitente** con todo levantado y sano, que no se arregla
>    esperando. Nos pasó grabando el video, tres veces.

- Build y prueba:

```bash
ls frontend                        # todavía no hay Dockerfile: lo escribís vos
bat --paging=never frontend/Dockerfile        # (o `cat`) — leelo entero, cambia bastante del backend
bat --paging=never frontend/nginx.conf        # el que hace que la app hable con el backend
bat --paging=never frontend/.dockerignore     # el tercero: sin él, el build muere
docker build -t mi-frontend:dev ./frontend
docker run --rm -d --name mi-frontend-run -p 3000:80 mi-frontend:dev
```

**✅ Checkpoint:** en `http://localhost:3000` se ve la interfaz servida desde el contenedor.

> ⚠️ **Y acá la lista de tareas NO va a cargar — está bien.** El contenedor del frontend existe y el
> del backend existe, pero son **dos contenedores sueltos que no se conocen entre sí**: no comparten
> red, así que el nombre `backend` del `nginx.conf` todavía no resuelve. Vas a ver la interfaz con un
> cartel de error. Eso es exactamente el problema que resuelve §3.6, y por eso el checkpoint de acá
> dice "se ve la interfaz" y no "la app funciona".

### 3.6 Compose: el sistema completo con un comando

```yaml
services:
  db:
    image: postgres:16-alpine          # o la BD que use tu app
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: app                 # ⚠️ crea la BD que tu connection string espera (sin esto: "database app does not exist")
    volumes:
      - db_data:/var/lib/postgresql/data   # los datos SOBREVIVEN al contenedor
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      retries: 10

  backend:
    build: ./backend
    environment:
      ConnectionStrings__Default: "Host=db;Database=app;Username=postgres;Password=${DB_PASSWORD}"
    ports:
      - "8080:8080"                    # para curl/Postman: el browser entra por el 3000 (§2.6)
    depends_on:
      db:
        condition: service_healthy     # espera a que la BD esté LISTA, no solo iniciada

  frontend:
    build: ./frontend
    ports:
      - "3000:80"
    depends_on:
      - backend

volumes:
  db_data:
```

> 💡 **¿Quién crea las TABLAS?** El Postgres del contenedor nace con la BD `app` vacía (no es tu Postgres local, ni el `pg-tp2` de §3.2). Asegurate de que tu app aplique el schema al arrancar (EF Core: `Database.Migrate()` o `EnsureCreated()` en el startup — así lo hace el sample de la cátedra) o usá scripts de inicialización en `/docker-entrypoint-initdb.d`. Si la prueba de persistencia "no crea datos", empezá por acá.
>
> 💡 **¿Tu app no usa PostgreSQL?** Cambian cuatro cosas del servicio `db` — imagen, variables, ruta del volumen y healthcheck. **El resto del compose es idéntico**:
>
> | Motor | Imagen | Variables mínimas | Ruta de datos (volumen) | Healthcheck |
> |---|---|---|---|---|
> | PostgreSQL | `postgres:16-alpine` | `POSTGRES_PASSWORD`, `POSTGRES_DB` | `/var/lib/postgresql/data` | `pg_isready -U postgres` |
> | MySQL / MariaDB | `mysql:8` | `MYSQL_ROOT_PASSWORD`, `MYSQL_DATABASE` | `/var/lib/mysql` | `mysqladmin ping -h 127.0.0.1` |
> | SQL Server | `mcr.microsoft.com/mssql/server:2022-latest` | `ACCEPT_EULA=Y`, `MSSQL_SA_PASSWORD` (≥8 chars, compleja) | `/var/opt/mssql` | `/opt/mssql-tools18/bin/sqlcmd -C -U sa -P "$MSSQL_SA_PASSWORD" -Q "SELECT 1"` |
> | MongoDB | `mongo:7` | `MONGO_INITDB_ROOT_USERNAME`, `MONGO_INITDB_ROOT_PASSWORD` | `/data/db` | `mongosh --eval "db.adminCommand('ping')"` |
>
> **SQLite es el caso raro**: no es un servicio, es un **archivo** dentro del contenedor del backend — así que no lleva servicio `db` ni `depends_on`, pero **sí necesita volumen** (montá la carpeta del `.db`, ej. `app_data:/app/data`) o los datos se borran con el contenedor y el TP pierde su prueba de persistencia.
>
> ⚠️ **No uses una BD en la nube para este TP** (Neon, Atlas, Azure SQL): el punto es que el sistema completo levante offline con un comando. Las bases gestionadas aparecen en el TP6, para los entornos QA/PROD.
>
> 🔴 **Editá el `.env` después de copiarlo**: el valor que trae la plantilla (`super-secreto-local`) es de ejemplo — poné la contraseña que quieras para tu base.
>
> Y un gotcha que cuesta encontrar: **si después cambiás esa contraseña, la BD no la toma**. PostgreSQL fija la contraseña la primera vez que inicializa su volumen y luego ignora la variable. Para que el cambio tenga efecto hay que borrar el volumen (`docker compose down -v`) y dejar que la BD se cree de nuevo — con la pérdida de datos que eso implica.
>
> 💡 **Formato del `.env`** (y de su `.env.example` commiteado): una variable por línea, `CLAVE=valor`:
> ```
> DB_PASSWORD=super-secreto-local
> ```
>
> 🔴 **El `.env` es lo primero, no lo último.** Como no se commitea, cuando clonás el repo **no está** — y compose no falla: reemplaza la variable que falta por **vacío** y sigue. Lo único que ves es un `warning: The "DB_PASSWORD" variable is not set`, y después la base se niega a arrancar (`Database is uninitialized and superuser password is not specified`). Antes del primer `up`, siempre: `cp .env.example .env`.

Puntos que tenés que poder explicar de tu propio compose:
- Los servicios se hablan **por nombre** (`Host=db`): compose crea una red interna con DNS.
- El **volumen** hace que los datos de la BD sobrevivan a `docker compose down` (sin `-v`).
- `depends_on` + `healthcheck`: la diferencia entre "el contenedor arrancó" y "el servicio está listo".
- Los secretos van en un **`.env` que NO se commitea** (agregalo al `.gitignore`; commiteá un `.env.example`).

**Primero, la limpieza.** Compose va a querer los puertos 8080 y 3000, y vos tenés ahí los dos
contenedores sueltos de §3.4 y §3.5. La base de práctica (`pg-tp2`) no molesta —el `db` del compose
ni siquiera publica puerto— pero ya no la usa nadie, así que también se va:

```bash
docker ps                                              # mirá qué hay dando vueltas
docker rm -f pg-tp2 mi-backend-run mi-frontend-run     # -f: los para Y los borra
docker ps                                              # y ahora no queda nada
```

**Después, el archivo del secreto.** Éste lo escribís vos y **se commitea** (el de verdad, no):

```bash
ls -a                   # mirá dónde va: en la raíz, al lado del docker-compose.yml
cp .env.example .env    # ⚠️ PRIMERO esto, o el compose arranca sin contraseña
                        # y después EDITALO: poné la contraseña que quieras
cat .env                # una línea: DB_PASSWORD=<lo que hayas puesto>
```

> ✍️ **En tu propia app, la plantilla la creás vos**, con este contenido (tres líneas, sin ningún
> secreto adentro — es lo que le dice al que clone qué variables hacen falta):
> ```
> # Copiá este archivo a .env (git lo ignora) y poné un valor real:
> #   cp .env.example .env
> DB_PASSWORD=super-secreto-local
> ```
> Y verificá que el archivo real esté listado en tu `.gitignore`.

**Y ahora sí, levantar todo:**

```bash
bat --paging=never docker-compose.yml   # leelo entero antes de levantarlo: cada línea importa
docker compose up -d --build
docker compose ps          # esperá a ver db "healthy" y backend/frontend "running"
docker compose logs backend

# El volumen, de verdad: dónde está y quién lo administra
docker volume ls                                    # el nombre lleva adelante el de la CARPETA
docker volume inspect practica-tp2_db_data          # driver "local" y su Mountpoint
```

**Prueba de persistencia.** Creá un par de registros desde la app en `localhost:3000` y después
verificalos por la API, que es más rápido que ir y volver al navegador (usá el endpoint de listado de
**tu** app):

```bash
curl -s localhost:8080/health              # esperá a que conteste antes de seguir
curl -s localhost:8080/api/tareas | jq     # ahí están

docker compose down && docker compose up -d
curl -s localhost:8080/health              # ⏳ de nuevo: el backend tarda unos segundos en volver
curl -s localhost:8080/api/tareas | jq     # SIGUEN: el volumen sobrevivió al contenedor

docker compose down -v && docker compose up -d
curl -s localhost:8080/health              # ⏳ y una vez más
curl -s localhost:8080/api/tareas | jq     # vacío: -v borró también los volúmenes
```

> ⏳ **Si un `curl` te contesta `Recv failure: Connection reset by peer` o `Empty reply from server`,
> NO es que se borraron los datos**: es que el backend todavía está arrancando. `docker compose up -d`
> devuelve el control apenas los contenedores arrancan, pero antes de servir hay que esperar el
> chequeo de salud de PostgreSQL y el arranque de la API. Por eso el `curl` a `/health` va primero:
> cuando ése contesta, el otro anda.

> 🔴 **Volumen nombrado vs. bind mount** (se escriben casi igual y no son lo mismo):
>
> | | Cómo se escribe | Dónde viven los datos | Cuándo usarlo |
> |---|---|---|---|
> | **Volumen nombrado** | `db_data:/var/lib/postgresql/data` | En un área que administra Docker (`/var/lib/docker/volumes/…`; en Mac/Windows, **dentro de la VM de Docker**, no en tu disco) | **Datos de bases de datos** — es lo que usa este TP |
> | **Bind mount** | `./datos:/var/lib/postgresql/data` | En **tu** carpeta, la ves y la abrís | Meter código o configuración a un contenedor, ver cambios en vivo |
>
> La diferencia visual es un carácter: si lo de la izquierda empieza con `./` o `/`, es una ruta tuya; si es una palabra suelta, es un volumen de Docker (y hay que declararlo en `volumes:`).
>
> Las dos formas, una al lado de la otra:
>
> ```yaml
> # ═══ A · VOLUMEN NOMBRADO — lo que usa este TP ═══════════════════════
> services:
>   db:
>     volumes:
>       - db_data:/var/lib/postgresql/data   # ← un NOMBRE: lo administra Docker
>
> volumes:            # ← y por eso hay que declararlo acá abajo
>   db_data:
>
>
> # ═══ B · BIND MOUNT — una carpeta tuya ═══════════════════════════════
> services:
>   db:
>     volumes:
>       - ./datos:/var/lib/postgresql/data   # ← una RUTA: ./ = la del proyecto
>
> # (acá NO va ninguna sección `volumes:`: la carpeta ya existe en tu disco,
> #  la vas a ver aparecer al lado del docker-compose.yml)
> ```
>
> ⚠️ Para la **BD**, usá volumen nombrado. Un bind mount del directorio de datos de PostgreSQL en Mac/Windows es notablemente más lento (hay una VM en el medio) y suele dar problemas de permisos.
>
> 💡 **¿Qué es ese `| jq`?** `jq` formatea e indenta el JSON que devuelve la API, que si no sale todo apelmazado en una línea (y sin salto al final, así que el prompt siguiente se te pega). Viene instalado en macOS y en la mayoría de las distros; si no lo tenés, sacá el `| jq` y funciona igual, se lee peor.
>
> 💡 `down` apaga; `down -v`, además, **olvida**. Es la diferencia que más se pregunta en la defensa.

**✅ Checkpoint:** `docker compose up -d` levanta todo el sistema funcionando end-to-end (front habla con back, back con BD), y la prueba de persistencia se comporta como esperabas.

### 3.7 Publicar las imágenes en un registry

Riel canónico — **GitHub Container Registry (ghcr.io)**, gratis para imágenes públicas.

**Paso 1 — generá el token.** Para publicar en ghcr necesitás un *Personal Access Token* **clásico** con el permiso `write:packages`. Se crea así, una sola vez:

> GitHub → tu foto (arriba a la derecha) → **Settings** → abajo de todo, **Developer settings** → **Personal access tokens** → **Tokens (classic)** → *Generate new token (classic)*.
> Ponele un nombre (`TP2 ghcr`), una expiración, y tildá **`write:packages`** (al tildarlo se marca solo `read:packages` y `repo`). *Generate token* → **copiá el token ahora**, porque no se vuelve a mostrar.

🔴 Tiene que ser **classic**, no *fine-grained*: los fine-grained **no funcionan con ghcr** — el `docker login` te dice `Succeeded` y después el `push` falla con `denied: permission_denied`. Es el error más confuso de este paso.

**Paso 2 — logueate al registry.** Docker te va a pedir una contraseña: **pegá el token**, no tu contraseña de GitHub (no la vas a ver mientras la pegás, es normal).

```bash
docker login ghcr.io -u <tu_usuario>
# Password: ← pegás el token y Enter
# → Login Succeeded
```

> 💡 ¿Preferís no tipearlo cada vez? Guardalo en una variable y pasáselo por entrada estándar — así el token no queda en el historial del shell:
> ```bash
> export CR_PAT=ghp_xxxxxxxx
> echo $CR_PAT | docker login ghcr.io -u <tu_usuario> --password-stdin
> ```
> Si tenés el GitHub CLI y preferís usar su token: `gh auth refresh -h github.com -s write:packages` y después `gh auth token | docker login ghcr.io -u <tu_usuario> --password-stdin`. Ojo que el `refresh` no siempre toma el permiso nuevo; si el push sigue dando `denied`, volvé al token clásico de arriba.

**Paso 3 — tag + push.** ⚠️ **`<tu_usuario>` va todo en MINÚSCULAS**, aunque tu usuario de GitHub
tenga mayúsculas: Docker no las acepta en el nombre de una imagen y corta con
`invalid reference format: repository name must be lowercase`. (El `docker login` sí las acepta, lo
que hace el error todavía más desconcertante.)

```bash
docker tag mi-backend:dev ghcr.io/<tu_usuario>/mi-backend:v0.1.0
docker tag mi-frontend:dev ghcr.io/<tu_usuario>/mi-frontend:v0.1.0

docker images | grep -E 'ghcr|mi-backend|mi-frontend'    # (PowerShell: | Select-String 'ghcr|mi-')

docker push ghcr.io/<tu_usuario>/mi-backend:v0.1.0
docker push ghcr.io/<tu_usuario>/mi-frontend:v0.1.0
```

> 💡 Mirá el **identificador** de las imágenes en ese último listado y compará con `mi-backend:dev`:
> es el mismo. `docker tag` **no copia nada** — le agrega un nombre más a una imagen que ya existe.
> Ese detalle vuelve en el Paso 5.

**Paso 4 — hacelas públicas.** 🔴 **Los packages de ghcr nacen privados**, y mientras lo estén
**nadie puede hacer `docker pull` de tu imagen**: ni la cátedra, ni otra máquina, ni un pipeline, ni
un compose que la referencie. Es el tropiezo más común de esta sección, y hay que hacerlo **para las
dos** imágenes:

> Tu perfil de GitHub → pestaña **Packages** → clic en el package → **Package settings** (abajo a la
> derecha) → **Change visibility** → *Public* → confirmar escribiendo el nombre.

> 💡 La imagen recién pusheada **no aparece dentro del repositorio**, así que buscala en tu perfil.
> Si querés que quede linkeada al repo (útil para el TP7), agregale al Dockerfile la línea
> `LABEL org.opencontainers.image.source=https://github.com/<tu_usuario>/<tu_repo>`.

**Paso 5 — comprobá que quedó público de verdad.** Decir que está público es fácil; la prueba es
bajarla sin credenciales:

```bash
docker compose down                                     # si tenés algo corriendo con esa imagen
docker logout ghcr.io                                   # dejás de estar autenticado
docker rmi ghcr.io/<tu_usuario>/mi-backend:v0.1.0 mi-backend:dev
docker pull ghcr.io/<tu_usuario>/mi-backend:v0.1.0      # …y la bajás de cero
```

> 🔴 **Fijate que el `rmi` lleva LOS DOS nombres.** Como `docker tag` no copió nada, esa imagen tiene
> dos nombres y un solo cuerpo: si borrás sólo el del registro, Docker te contesta `Untagged` y no
> borra nada.
>
> ⚠️ **Y aun así el `pull` va a decir `Already exists` en todas las capas. Está bien, no fallaste.**
> Docker no guarda imágenes: guarda **capas**, identificadas por su contenido. Tu `docker compose up
> -d --build` de §3.6 construyó su propia imagen del backend (se llama `<tu-carpeta>-backend`) a
> partir del mismo código, así que sus capas son **las mismas** y siguen en tu disco mientras esa
> imagen exista. Borrar los dos nombres no borra el cuerpo si alguien más lo está usando.
>
> Lo que este paso prueba —y es lo que importa— es que **pudiste pedir la imagen sin credenciales**,
> estando deslogueado. La descarga de verdad la vas a ver en el Paso 6, donde también se van las
> imágenes del compose.

Si el `pull` funciona sin sesión, cualquiera puede correr tu imagen — **eso** es el checkpoint, no que la página diga *Public*.

> ⚠️ **Una advertencia honesta sobre arquitecturas.** La imagen que publicaste sirve para máquinas con
> el **mismo tipo de procesador** que la tuya: si la construiste en una Mac moderna es para ARM, y si
> la construiste en una PC común es para Intel/AMD. Alguien con la otra arquitectura recibe
> `no matching manifest for linux/amd64 in the manifest list entries` — y los runners de CI del TP7
> son Intel. Para este TP alcanza con saberlo (anotalo en `decisiones.md` y decí en qué máquina la
> construiste); en el **TP7** lo vamos a resolver con `docker buildx`, que construye para las dos a la vez.

**Paso 6 — ¿y para qué publicaste?** Para que el sistema se pueda levantar **sin tu código**. El `docker-compose.yml` que escribiste usa `build: ./backend`, así que necesita el repositorio. Escribí una variante que use `image:` con el nombre completo, y probala:

**`docker-compose.registry.yml`**, en la raíz, al lado del otro. Es el **mismo** compose: mismos
servicios, mismo volumen, mismo healthcheck, mismo `depends_on`. Lo único que cambia son las dos
líneas donde decía `build:`:

```yaml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: app
    volumes:
      - db_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      retries: 10

  backend:
    image: ghcr.io/<tu_usuario>/mi-backend:v0.1.0     # ← antes: build: ./backend
    environment:
      ConnectionStrings__Default: "Host=db;Database=app;Username=postgres;Password=${DB_PASSWORD}"
    ports:
      - "8080:8080"
    depends_on:
      db:
        condition: service_healthy

  frontend:
    image: ghcr.io/<tu_usuario>/mi-frontend:v0.1.0    # ← antes: build: ./frontend
    ports:
      - "3000:80"
    depends_on:
      - backend

volumes:
  db_data:
```

```bash
bat --paging=never docker-compose.registry.yml   # compará con el otro: cambian dos líneas
docker compose down --rmi local     # ← el flag se lleva TAMBIÉN las imágenes que compose construyó
docker rmi ghcr.io/<tu_usuario>/mi-backend:v0.1.0 ghcr.io/<tu_usuario>/mi-frontend:v0.1.0 mi-frontend:dev
docker builder prune -af            # ← y esto vacía el CACHE DE CONSTRUCCIÓN, que también las guarda
docker compose -f docker-compose.registry.yml up -d    # NO construye: descarga, y AHORA sí se ve
```

> 🔴 **Las capas se esconden en TRES lugares, y hay que sacarlas de los tres.** Docker no guarda
> imágenes: guarda **capas**, identificadas por su contenido, y sólo las libera cuando ya nadie las
> referencia.
> 1. La imagen que construyó el compose de §3.6 (`<tu-carpeta>-backend`) → la saca el `--rmi local`.
> 2. Los nombres que le pusiste vos (`mi-backend:dev`, el del registro) → los saca el `rmi`.
> 3. **El cache de construcción** → lo saca el `builder prune`. Éste es el que nadie ve venir: puede
>    tener más de un giga de capas guardadas para acelerar tus próximos builds.
>
> Si te salteás cualquiera de los tres, el `up` te contesta `Already exists` en todas las capas y no
> baja nada — el mismo malentendido del Paso 5. Con los tres, la descarga se ve capa por capa.
>
> ⚠️ Después del `builder prune`, tu próximo `docker build` va a tardar como el primero: se quedó sin
> cache. Es el precio de hacer la prueba en serio, y la hacés una sola vez.
> (`mi-backend:dev` no está en la lista del `rmi` porque ya lo borraste en el Paso 5; si lo ponés,
> Docker te contesta `No such image`.)

> ⚠️ **Este compose también necesita el `.env`.** Usa `${DB_PASSWORD}` igual que el otro, así que si
> se lo pasás a alguien "para levantar sin el código", tiene que ir acompañado de la plantilla —o de
> la contraseña por otro canal—. Son **dos** archivos, no uno. Eso no es un defecto: es la disciplina
> del secreto que no viaja por el repositorio.

El sample tiene este archivo resuelto en su rama `main`, para que compares. **Es el punto de partida del TP7**: allá el pipeline construye y publica, y el entorno consume exactamente así.

> ⚠️ **Un gotcha más de ghcr**: el `docker login` da OK con cualquier token válido, tenga o no el permiso de packages… y recién falla el `push`, con `denied` — el login exitoso **no** garantiza el permiso.

**¿Por qué ghcr y no Docker Hub?** Los dos sirven y **aceptamos cualquiera de los dos**. Elegimos ghcr para la cátedra porque: (1) ya tenés la cuenta —es la de GitHub del TP1—; (2) las imágenes quedan junto al código, así tu entrega es un solo lugar; y (3) en el **TP7**, cuando publique el pipeline en vez de vos, GitHub Actions puede autenticarse contra ghcr **sin secretos**: usa el `GITHUB_TOKEN` del propio workflow, declarándole `permissions: packages: write`. Docker Hub también se automatiza sin problema — solo que ahí hay que crear un token y guardarlo como secreto del repo.

**Si preferís Docker Hub**: `docker login` (sin servidor, es el default) + `docker tag mi-backend:dev <usuario>/mi-backend:v0.1.0` + `docker push <usuario>/mi-backend:v0.1.0`. Ventaja: los repos públicos nacen públicos, así que te ahorrás el paso de cambiar visibilidad. Desventajas menores: las cuentas gratuitas tienen límite de **descargas** por hora (afecta al `pull`, no al `push`) y solo permiten **un** repositorio privado.

**✅ Checkpoint:** las imágenes son visibles en tu perfil (pestaña *Packages* en GitHub), con visibilidad **pública**, y cualquiera puede hacer `docker pull` de ellas.

### 3.8 Y ahora, sobre TU app — esto es lo que se entrega

🔴 **Todo lo que hiciste hasta acá fue sobre el sample de la cátedra. Eso es la práctica, no la
entrega.** Es el error más caro de este trabajo práctico y el más fácil de cometer: los checkpoints
dan verde, el repo `practica-tp2` hace exactamente lo que el TP pide… y no es tu aplicación.

Volvé a tu repositorio del semestre (el que elegiste en §3.3) y rehacé el recorrido. Son los mismos
ocho archivos, con los nombres y comandos de **tu** stack:

- [ ] `backend/Dockerfile` multi-stage + `backend/.dockerignore`
- [ ] `frontend/Dockerfile` multi-stage + `frontend/.dockerignore` + `frontend/nginx.conf`
- [ ] `docker-compose.yml` en la raíz, con volumen, `depends_on` + `healthcheck`, y la contraseña por variable
- [ ] `.env.example` commiteado (y el `.env` real ignorado)
- [ ] `docker-compose.registry.yml` apuntando a **tus** imágenes publicadas
- [ ] Un `README.md` con los pasos de arranque desde cero (ver el escenario, más abajo)

Lo que **no** se transfiere tal cual: los nombres de proyecto, los comandos de build de tu lenguaje
(la tabla de §3.2 los tiene por stack), la ruta de salida del build del front, y el puerto en el que
escucha tu API. Lo que **sí** se transfiere entero: la estructura de dos etapas, el orden de las
instrucciones para aprovechar el cache, la disciplina del secreto por variable, y la idea de que la
configuración entra por el entorno y no por el código.

> 💡 Si te queda tiempo, hacelo al revés: escribí primero los archivos de tu app y usá el sample sólo
> para desatascarte. Se aprende bastante más.

---

## 4- Riel alternativo: Azure

En este TP no hay nada que requiera Azure — todo es local + registry gratuito. La equivalencia relevante llega en el **TP7**, cuando el pipeline construya y publique las imágenes:

| Concepto | Riel canónico | Riel Azure |
|---|---|---|
| Registry | ghcr.io (gratis, público) / Docker Hub | **Azure Container Registry (ACR)** — ~USD 5/mes plan Basic, requiere subscription. Se ve en TP7 con advertencia de costos |
| Autenticación al registry | `gh auth token` / PAT | `az acr login` |
| Dónde corre esto | Tu máquina | Tu máquina (igual: Docker es Docker en todos lados — ese es justamente el punto) |

---
---

# 📋 Trabajo Práctico 02 – Contenedores: la app del semestre (2026)

## ⚠️ Este es el TP que debés entregar y defender

## 🎯 Objetivo

Contenerizar la aplicación que vas a usar durante **todo el semestre**: Dockerfiles multi-stage para back y front, orquestación completa con compose, e imágenes publicadas en un registry.

Este trabajo se aprueba **solo si podés explicar qué hiciste, por qué lo hiciste y cómo lo resolviste**.

## 🧩 Escenario

Tu proyecto adopta contenedores como unidad de despliegue. Como responsable técnico, tenés que dejarlo en un estado donde **cualquier persona que clone el repo levante el sistema completo con un solo comando**, y donde las imágenes estén publicadas para que otros entornos (QA, PROD — los vas a construir en los próximos TPs) las consuman.

## 📋 Tareas que debés cumplir

### 1. Elección de la app del semestre
- App con backend + frontend + base de datos (propia o de GitHub), corriendo localmente.
- Justificación de la elección en `decisiones.md` según los criterios de la guía (§3.3). **Esta app es la que vas a usar en TP4–TP9 y en el Integrador — elegila con eso en mente.**

### 2. Dockerfiles
- Dockerfile **multi-stage** para el backend y para el frontend, cada uno con **su** `.dockerignore` (son dos, uno por carpeta de build).
- Si tu frontend es una SPA servida por nginx, su archivo de configuración (`nginx.conf`) con el proxy hacia el backend.

### 3. Compose
- `docker-compose.yml` que levanta el sistema completo (front + back + BD) con `docker compose up -d`, incluyendo:
  - volumen para persistencia de la BD,
  - comunicación entre servicios por nombre (red de compose),
  - `depends_on` con `healthcheck` de la BD,
  - secretos vía `.env` no commiteado (con `.env.example` commiteado).

### 4. Publicación
- Imágenes de back y front publicadas en un registry gratuito (ghcr.io o Docker Hub) con tag semver (`v0.1.0`) y visibilidad **pública** (las dos).
- `docker-compose.registry.yml`: la variante que **baja** las imágenes en vez de construirlas, probada de verdad.

### 5. Arranque documentado
- Un `README.md` con los pasos exactos para levantar tu sistema en una máquina limpia, empezando por el `cp .env.example .env`. Es lo que se te va a pedir en la defensa.

## 📄 Entregables


> 🏷️ **Cerrá el práctico con su tag y su release**, como hiciste en el TP1. La numeración sigue el
> número del práctico: **TP2 → `v2.0.0`**.
>
> ```bash
> git tag -a v2.0.0 -m "TP2 cerrado"
> git push origin v2.0.0
> ```
>
> Y la release desde la web —*Releases → Draft a new release*—, eligiendo ese tag, con el título
> `v2.0.0` y una descripción de qué incluye. Cada práctico queda así con su **estado congelado**:
> podés volver a él si rompés algo, y en la defensa se navega el punto exacto en el que cerraste.
1. **URL del repositorio, público** (se carga en el formulario de la cátedra) con la app, los dos Dockerfiles y sus `.dockerignore`, el `nginx.conf` del frontend si corresponde, el `docker-compose.yml`, el `.env.example`, el `docker-compose.registry.yml` y el `README.md` de arranque. Si tu repo del TP1 quedó privado, cambiale la visibilidad: *Settings → General → abajo de todo, Danger Zone → Change visibility*.
2. **`decisiones.md`** explicando:
   - Qué app elegiste y por qué (contra los criterios de la guía).
   - Decisiones de contenerización: imágenes base elegidas, estructura multi-stage, qué persiste y qué no.
   - Problemas encontrados y cómo los resolviste.
3. **`evidencias.md`** con capturas/salidas de:

   - `docker compose up -d` desde cero y el sistema funcionando end-to-end,
   - la prueba de persistencia (`down` / `up` conserva datos; `down -v` los limpia),
   - comparación de tamaño imagen final vs imagen de SDK,
   - las imágenes publicadas en el registry.

> 📁 **Dónde van esos dos archivos**: en la **raíz** del repositorio, al lado del `README.md` — igual
> que en el TP1. Si ya tenés un `decisiones.md` de allá, **seguí escribiendo abajo** con un título
> `## TP2 — Contenedores`: el historial del semestre es uno solo.
>
> 🔀 **¿Por PR o directo a `main`?** Por **PR**, como en el TP1: es lo que las protecciones de aquel
> trabajo dejaron configurado y lo que el Integrador evalúa como evidencia de proceso. Un PR por
> tanda de trabajo alcanza; no hace falta uno por archivo.
>
> 📅 **Fecha y entrega**: la fecha límite y el formulario los publica la cátedra en el aula virtual.
> Lo que se carga ahí es **una sola cosa**: la URL de tu repositorio.

## 🗣️ Defensa Oral Obligatoria

Vas a mostrar tu trabajo y responder preguntas como:
- ¿Qué diferencia hay entre imagen y contenedor? ¿Y entre `CMD` y `ENTRYPOINT`?
- ¿Por qué tu Dockerfile es multi-stage? ¿Qué pasaría si no lo fuera?
- ¿Qué pasa con los datos si borro el contenedor de la BD? ¿Y con `docker compose down -v`?
- ¿Cómo se encuentra el backend con la BD? ¿Qué es `db` en tu connection string?
- ¿Por qué `depends_on` solo no alcanza y hace falta el healthcheck?
- ¿Por qué el `.env` no está en el repo? ¿Dónde van a vivir esos secretos cuando esto corra en un pipeline? *(spoiler: TP4)*
- En vivo: cloná tu repo en una carpeta limpia y levantalo siguiendo **tu propio `README`**.

> 💡 Van a ser **dos** comandos, no uno: `cp .env.example .env` y después `docker compose up -d`. Eso
> no es un defecto de tu entrega — es el punto. El secreto es lo único que no puede viajar en el
> repositorio, y por eso el arranque necesita un paso manual. Tenelo pensado para no dudar delante
> del profe.

## ✅ Evaluación

| Criterio | Peso |
|---|---|
| Configuración técnica (Dockerfiles, compose, registry, reproducibilidad) | 25% |
| Claridad y justificación en `decisiones.md` + `evidencias.md` | 25% |
| Defensa oral: comprensión y argumentación | 50% |

> ⚖️ Peso orientativo de este TP en la nota de **P1**: **40%** (la ponderación completa de los 9 TPs está en el reglamento, §5).

## ⚠️ Uso de IA

Podés usar IA (ChatGPT, Copilot, Claude), pero **deberás declarar en `decisiones.md` qué parte fue asistida por IA** y justificar cómo la verificaste. Si no podés defenderlo, **no se aprueba**.
