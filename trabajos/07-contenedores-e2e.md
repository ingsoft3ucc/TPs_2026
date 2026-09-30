# ⚠️ IMPORTANTE – Guía de Práctica Sugerida

Este documento tiene **dos partes**:

1. **Guía de práctica sugerida** (primera parte): paso a paso para aprender haciendo. **NO es lo que se entrega.**
2. **El Trabajo Práctico entregable** (al final): escenario, tareas, entregables y defensa oral. **Eso es lo que se evalúa.**

## 📦 Un solo repositorio para todo el semestre

**Todos los prácticos de la materia se hacen sobre el mismo repositorio: el que creaste en el TP1.** No se crea uno nuevo por práctico, y no se arranca de cero en cada uno — cada TP agrega una capa sobre lo que ya está.

Cada TP se cierra con su **tag y su release**, y el número mayor es el número del práctico: **TP1 → `v1.0.0`, TP2 → `v2.0.0`, TP3 → `v3.0.0`**, y así hasta el TP9. Así cada entrega queda con su **estado congelado**: en la defensa se navega el punto exacto en el que cerraste cada una, y podés volver a cualquiera con `git checkout v2.0.0`.

El archivo `decisiones.md` también es **único**: no se rehace por práctico — se le agrega abajo la sección del TP nuevo.

## Sobre las herramientas en este TP

- La guía usa **ghcr.io** (GitHub Container Registry — almacenamiento y ancho de banda gratis para imágenes públicas) + **Render** (que puede desplegar una imagen ya construida desde un registry público; en §3.2 comprobás primero, con un solo servicio, que tu plan gratis te lo ofrece) + **Playwright** para e2e (gratis, open source; **Cypress** es alternativa equivalente aceptada). Todo sin tarjeta, sobre lo que ya construiste en TP2/TP4/TP6.
- Riel **Azure**: **ACR** (Azure Container Registry — ⚠️ plan Basic ~USD 5/mes, requiere suscripción: es el único componente de la materia con costo; la advertencia ya estaba en el material 2025) + App Service for Containers / ACI. Tabla de equivalencias en §4.
- **Fallback local garantizado**: self-hosted runner + compose consumiendo la imagen del registry — §3.6.

📌 Este TP trabaja **sobre tu app del semestre**, y cambia QUÉ viaja por la cadena del TP6: hasta hoy promovés un commit que el proveedor reconstruye; desde hoy promovés **una imagen inmutable**. Y la pirámide del TP5 suma **los dos escalones que le faltaban**: las **pruebas de integración**, que le hablan a tu api desplegada con su base de verdad, y las **pruebas e2e**, que usan la app como una persona (§2.5 y §2.6).

🧪 **Los ejemplos de esta guía están escritos sobre la app de la cátedra** ([`ingsoft3ucc/demo-fullstack`](https://github.com/ingsoft3ucc/demo-fullstack) — .NET 8 + React/Vite + PostgreSQL, la misma de las demos en clase; cómo clonarla y levantarla, base de datos incluida: **TP2 §3.2**). Si un paso no te sale en tu app, probalo primero ahí —donde sabés que funciona— y después llevalo a la tuya. Lo que entregás, siempre, es **tu** app.

---

# Guía Paso a Paso – Contenedores en el pipeline + pruebas e2e (Práctica sugerida)

## 1- Objetivos de Aprendizaje

- Terminar lo que el TP6 dejó a medias a propósito: hacer que QA y PROD **ejecuten la imagen
  publicada** en vez de reconstruir la app — **build once, deploy many**, con el registry de puente
  entre la verificación y los entornos.
- Promover a PROD **exactamente la misma imagen** que QA verificó, sin reconstruir nada.
- Escribir **pruebas de integración** —el escalón del medio— que le hablen a tu API desplegada con su base de datos de verdad, sin navegador y sin ningún doble en el medio: lo que el doble del TP5 no puede ver es justamente qué contesta la dependencia de verdad (§2.5).
- **Leer las dos suites juntas como un diagnóstico**: qué te dice una integración verde con una e2e roja, y qué te dice el caso contrario.
- Escribir **pruebas end-to-end** (Playwright/Cypress) que ejercitan la app desplegada como un usuario real.
- Convertir **las dos suites** en **gate de promoción**: si alguna falla contra QA, PROD no se toca.
- Ponerle **nombre** a lo que está en producción con una release (`v7.0.0`), y saber llegar de ese nombre a la imagen exacta que corre.

## 2- Marco teórico

### 2.1 La imagen como unidad de release: build once, deploy many

En el TP6 tu cadena promueve *commits*: cada entorno recibe el commit verificado… y **lo reconstruye** (Render compila tu repo en cada deploy). Funciona, pero rompe el principio que declaramos en §2.3 de ese TP —y que el propio TP6 te hizo anotar como limitación—: *lo que llega a PROD no es bit a bit lo que probaste en QA*, es una reconstrucción. Distinta hora, posibles dependencias resueltas de nuevo, otro resultado posible. Fijar la versión en el `FROM` achica el problema, no lo cierra: seguís construyendo dos veces, sin poder demostrar que salió lo mismo.

La solución es la pieza que tenés desde el TP2: la **imagen**. El pipeline la construye **una sola vez**, con todo adentro (runtime, dependencias, tu app), la publica en el registry con una identidad única, y los entornos la **ejecutan tal cual** — QA y PROD corren *el mismo binario*, no dos builds "parecidos". Esto es **build once, deploy many**, y es el estándar de la industria para releases:

- **Reproducibilidad**: el deploy deja de depender de "cómo salga el build de hoy".
- **Promoción real**: promover = decirle a PROD "ejecutá la imagen `X`" — la misma `X` que QA verificó.
- **Rollback directo**: volver = ejecutar la imagen anterior, que sigue en el registry, intacta. El catálogo de releases del TP6, ahora con binarios de verdad.

### 2.2 Identidad de una imagen: tags y digests

Una imagen se referencia como `registry/owner/imagen:tag`. Tres cosas sobre esa referencia:

- **`latest` es un puntero, no una versión**: hoy apunta a una cosa, mañana a otra. No dice qué contenido hay detrás: nunca despliegues `latest` a producción, porque después no vas a poder decir qué desplegaste.
- **Los tags son mutables**: cualquiera con permiso de escritura en el paquete puede volver a publicar un tag —`latest`, o también `sha-<commit>`— apuntando a otra imagen. Tu propio pipeline puede hacerlo: volver a correr la corrida de un commit publica otra vez `sha-<ese commit>`, y no necesariamente con el mismo contenido. Por eso una corrida vieja no se vuelve a correr para desplegar.
- **El digest es inmutable**: `imagen@sha256:abc…` identifica el contenido exacto — si cambia un byte, cambia el digest. Es la identidad criptográfica de la imagen. **En este TP no hace falta que lo uses** —se conoce recién después de construir, y habría que pasarlo de un job a otro; §3.5 lo menciona como opción—: tu pipeline promueve por la etiqueta `sha-<commit>`, que se arma sola con el commit de la corrida. Queda acá porque es lo único que, a diferencia de una etiqueta, nadie puede mover.

La convención de la materia: lo que publica el pipeline al integrar (TP6) lleva **el tag del commit** —`sha-<commit>`, trazabilidad exacta: de qué commit salió— y la **promoción entre entornos usa ese tag** (o el digest). Así, "¿qué hay en PROD?" tiene respuesta exacta y verificable. Por eso tu pipeline **no publica `latest`**: en tu registry cada imagen tiene un solo nombre, el de su commit, y ninguna etiqueta que se mueva con cada merge.

### 2.3 El registry como puente: de guardar a SERVIR

El registry (TP2 §2.3 · TP6 §3.0) termina de cambiar de rol: de «lugar donde subí una imagen a mano»
a **pieza central del pipeline** — CI escribe, los entornos leen.

> 🔴 **La parte de escribir ya la construiste en el TP6 (§3.0), y acá no se rehace.** Tu pipeline
> publica las dos imágenes en `ghcr.io`, etiquetadas con el commit, sólo desde `main`, con el
> `GITHUB_TOKEN` del workflow y el permiso `packages: write` del job — ningún token tuyo. Si tu
> publicación del TP6 no quedó andando, arreglala allá antes de seguir: acá se da por hecha.

Lo que cambia esta semana es **quién lee**: en el TP6 la imagen quedaba guardada y Render
reconstruía tu app desde el repositorio; ahora el entorno **ejecuta esa imagen**. El mismo paquete,
con un consumidor nuevo. Por eso importa que los dos paquetes sean públicos, como los dejaste en el
TP6 (un paquete público no vuelve a privado: TP6 §3.0): Render los baja sin credenciales.

### 2.4 Desplegar imágenes: el proveedor deja de buildear

Con la imagen en el registry, el rol del proveedor cambia: ya no compila tu repo — **ejecuta tu imagen**. En Render eso es un servicio *image-backed*: su fuente es **Existing Image**, la URL de tu imagen en el registry. Y el deploy desde el pipeline reutiliza el mecanismo del TP6 con un cambio: el **deploy hook acepta el parámetro `imgURL`**, que le dice QUÉ imagen desplegar:

```
https://api.render.com/deploy/srv-…?key=…&imgURL=<URL-de-la-imagen-URL-encoded>
```

El valor puede ser un **tag o un digest**, y todo lo demás de la URL tiene que coincidir con la imagen configurada en el servicio: si le mandás otra imagen, Render contesta `400` con *«deploy hook cannot change the host, project, or image name. Only the digest or tag may be modified»*, y si el tag no existe, `400` con *«unable to fetch image with provided input»*. Con el `curl -f` de tu job, cualquiera de los dos pone la corrida en rojo. Con eso, tu job de deploy dice *exactamente* qué imagen va a cada entorno — `sha-<commit>` a QA, y a PROD **la misma** —, y «se promueve lo mismo que se verificó» queda escrito como un parámetro de la llamada.

📌 **Lo que esto todavía no cierra, y es el paso siguiente** *(concepto — no se implementa en este TP)*. El hook le dice a Render qué imagen desplegar, pero nada comprueba **desde afuera** que el entorno la haya levantado: si el deploy falla, el servicio sigue sirviendo la versión anterior y contesta igual. Vos lo mirás a mano, en *Events* de cada servicio. Cerrarlo automáticamente tiene dos mitades: que la imagen se **hornee** con el commit que la produjo (un `ARG` en el `Dockerfile`, que el pipeline le pasa con `build-args`, y que la api devuelve en su endpoint de vida), y que el **smoke lo compare** contra el commit de la corrida en vez de preguntar sólo «¿respondés?». Con eso, un deploy que no ocurrió pone la corrida en rojo sin que nadie mire. Acá se estudia; no se implementa ni se evalúa.

### 2.5 El escalón del medio: los tests de integración

En el TP5 dibujamos la pirámide entera y escribimos **la base**: tests unitarios, con la unidad
aislada de todo lo de afuera. Para aislarla **sacaste** la dependencia y pusiste un doble en su lugar
(TP5 §2.3). Hoy subimos un escalón, y la pregunta que lo justifica es ésta:

> Tu test con el doble está en verde: el doble contestó «guardado». **¿Y quién le dijo al doble qué
> contestar?**

Vos. **El doble contesta lo que vos le dijiste; la base de verdad, lo que pasa.** Un ejemplo, en la
app de la cátedra: alguien, en el alta de una tarea, cambia `DateTime.UtcNow` por `DateTime.Now`
—quería la hora de acá—. Compila, y todos los unitarios del TP5 siguen en verde, el del mock
incluido: ninguno le pregunta nada a una base, y un doble no tiene columnas ni tipos, así que acepta
cualquier fecha. La base de verdad, no: la columna de esa fecha guarda un instante exacto, en hora
universal (UTC), y el driver de Postgres —el componente con el que tu app le habla a la base— sólo
acepta escribir ahí una fecha marcada como UTC. `DateTime.Now` trae la hora de tu zona, marcada como
local: al guardar, la rechaza. La api contesta `500`, y en la app ya nadie puede crear una tarea. Con
el doble: verde. Con la base de verdad: rojo. Ningún test que reemplace la base lo puede ver, porque
reemplaza **justo lo que falla**. Una base en memoria (la InMemory de Entity Framework, o SQLite)
tampoco sirve: es otro doble, no pasa por el driver de Postgres, y el cambio de la fecha daría
verde. Eso es **lo que el doble no ve**, y es el mismo límite que el TP5 ya te avisó para el front:
«tu impostor sigue contestando lo de siempre».

Eso es un test de integración: tu API de verdad, con su base de datos de verdad, sin navegador y
**sin ningún doble en el medio**. No corrige al TP5 ni lo reemplaza: allá **SACASTE** la dependencia,
para probar tu lógica sin depender de nada; acá la **VOLVÉS A PONER**, para probar que tu código y la
dependencia de verdad se entienden. Los dos conviven. Corre en segundos —no en milisegundos como los
unitarios, no en minutos como las e2e— y por eso van pocos: donde tu código se encuentra con la base.

**Por qué recién ahora y no en el TP5.** Porque necesita una API andando con una base de verdad
detrás, y en el TP5 no había ninguna: allá los tests corren **adentro de la etapa `test` de tu
Dockerfile**, en un contenedor solo, sin nada al lado. Hoy sí la hay, y la construiste vos en §3.2:
**QA**, ejecutando la imagen exacta que salió de esta corrida. Tu test de integración le habla a esa
API por su dirección pública, sin navegador y sin doble en el medio.

> 📌 **Dos formas de hacer esto, y cuál usamos.** «Test de integración» nombra dos cosas parecidas:
> la **estrecha** —tu API armada en memoria dentro del propio pipeline, contra una base descartable
> que nace y muere con la corrida— y la **amplia**, que es la de este práctico: la API **ya
> desplegada** en un entorno, con su base de verdad. La estrecha llega antes (no necesita deploy) y
> es más aislada; la amplia prueba además que el despliegue quedó bien, y no te obliga a levantar
> una base en el pipeline. Acá usás la amplia porque ya tenés QA: es el mismo concepto, con un
> décimo de la configuración. Si algún día necesitás la estrecha, el camino es
> `WebApplicationFactory` (o su equivalente en tu stack) y un Postgres como `services:` del job.

**Y acá está lo que hace que gane su lugar, aunque la e2e también corra contra QA**: la e2e ve la
pantalla, y cuando se pone roja no sabés **quién** se rompió. La de integración le habla a la API
directamente, así que las dos juntas te dan un diagnóstico, no sólo una alarma:

| integración | e2e | Qué se rompió |
|---|---|---|
| 🟢 | 🔴 | El front: la API está sana, pero la pantalla no la usa bien |
| 🔴 | ⬜ no llega a correr | La API o la base: el front no llega ni a tener la chance |
| 🟢 | 🟢 | Nada de lo que estos dos ven |

La segunda fila no tiene un rojo en la e2e, y no es un error: **la e2e depende de la integración**, así que si la api está rota la e2e queda *salteada*. Es a propósito — no se gasta un browser en confirmar algo que ya sabemos, y es la misma economía que ordena la pirámide: abajo, lo barato y lo que corre primero.

Vas a usar exactamente esa tabla en §3.4, cuando rompas la aplicación a propósito.

📌 **Las tres capas, y qué ve cada una** — tenelo a mano: es el mapa de a quién le habla cada control, y explica por qué una e2e verde no te dice que la api esté bien por dentro:

| | Qué prueba | Qué NO puede ver | Tarda |
|---|---|---|---|
| Unitario (TP5) | Una regla, aislada | Lo que contesta la dependencia de verdad (pusiste un doble) | Milisegundos |
| **Integración de back (acá)** | **Que tu API desplegada y su base de verdad se entienden: endpoint → base, y lo que la base acepta, rechaza y guarda** | **Si la pantalla usa ese endpoint** | **Segundos** |
| End-to-end (acá) | El sistema entero como lo usa una persona | Lo que no recorren tus pocos flujos, y lo que sólo pasa en producción | Minutos |

📌 **¿Y la integración de front?** Existe, y es el escalón del medio **del otro lado**. En vez de
probar un componente solo con dobles (TP5), montás **varios componentes tuyos juntos** —el formulario
y la lista— y comprobás que se entienden entre ellos: que al enviar el formulario, la lista se
actualiza. Se escribe con **vitest + Testing Library** (la biblioteca de pruebas de componentes) y
corre **donde corren tus unitarios**: en el runner, dentro del job `build-frontend`, en milisegundos.
No necesita navegador, ni entorno desplegado, ni base. **En este TP no se pide** —el escalón del
medio lo cubrimos del lado del back—, pero es lo que te falta del lado del front, y conviene que
sepas que existe.

Más arriba, cada prueba ve más y cuesta más —la e2e, además, puede fallar por motivos que no son tu
código—: por eso arriba van pocas.

### 2.6 Las e2e: la punta de la pirámide

La punta quedó pendiente desde el TP5: **pocas** pruebas end-to-end, las más caras y las más valiosas como red final. ¿Por qué recién ahora? Porque una e2e de verdad necesita lo que no teníamos: **un entorno desplegado que ejecute tu imagen**, donde un browser de verdad use la app como un usuario. QA existe desde el TP6, pero allá corría una reconstrucción; desde §3.2 ejecuta la imagen que vas a promover. Hoy lo ejercitamos.

**Playwright** (Microsoft, open source) automatiza un browser real: navega, clickea, escribe, y verifica lo que el usuario vería. Los principios para que tu suite e2e sume en vez de doler:

- **Pocas y críticas**: 3-5 flujos que, si se rompen, tu app no sirve (crear el registro principal, verlo listado, borrarlo). No repitas en e2e lo que los unit tests ya cubren — cada capa de la pirámide verifica lo suyo.
- **Contra el entorno desplegado, no contra localhost**: la e2e del TP corre contra la URL de QA (`baseURL` configurable por variable de entorno). Eso es lo que la hace *end-to-end*: front servido + API + BD reales, con la red y el proveedor en el medio.
- **Esperas explícitas, no `sleep`**: Playwright espera automáticamente a que los elementos aparezcan (*auto-waiting*) — los `sleep(5000)` a mano son la forma más común de fabricar tests *flaky* (¿te suena del TP5? el test que a veces falla entrena al equipo a ignorar el rojo).
- **Tests que limpian lo que crean**: tu e2e corre contra un QA compartido y repetido — si cada corrida deja basura, la décima corrida falla por culpa de la primera. Crear → verificar → borrar.
- **El reporte es primera clase**: Playwright genera un reporte HTML con el screenshot y la traza de cada fallo (con `screenshot: 'only-on-failure'` y `trace: 'on-first-retry'` en la config) — publicado como artefacto (patrón del TP4), es lo que usás para diagnosticar un rojo remoto.

Y la distinción que ata este tema con el TP6, para tenerla explícita: **el smoke test verifica que el sistema RESPONDE; la e2e verifica que FUNCIONA**. Son preguntas distintas — una `BACKEND_URL` mal cargada en el front (nginx manda `/api` a una dirección que no es tu api), un botón muerto o un campo que el front manda con otro nombre pasan el smoke y mueren en la e2e. La primera la atajaría un `curl` más en el smoke, a través del front. Pero el smoke sólo ve lo que se te ocurrió preguntarle; la e2e, lo que le pasa a una persona.

### 2.7 Las pruebas como gate de promoción

El último paso: en la cadena del TP6, los dos jobs de prueba se **encadenan después del deploy a QA y antes de la aprobación a PROD**, y en ese orden — integración primero, porque es más rápida y más barata: si la api está rota, no vale la pena levantar un browser para confirmarlo:

```
CI + imagen → deploy QA (imgURL=sha) → smoke (¿responde?) → integración contra la api de QA
  → e2e contra QA → [aprobación] → deploy PROD (misma imagen) → smoke (¿responde?)
```

Si **cualquiera de las dos** falla, `deploy-prod` ni siquiera llega a pedir aprobación — la promoción se frena en QA, que es donde tiene que frenarse. (Y si la que falla es la integración, la e2e **no llega a correr**: su `needs` no se cumple.) Tu aprobador (TP6) decide con mejor información: ya no aprueba "QA responde", aprueba "la api contestó bien con su base de verdad **y** un browser real usó la app completa". El gate humano decide con la mejor evidencia automática que tiene la cadena.

Nota de realismo para free tiers (¿te suena del TP6?): QA duerme y despierta lento. En tu pipeline lo despertaron el smoke y la integración antes de la e2e, pero por internet un pedido suelto puede demorarse igual: por eso la configuración de tus e2e lleva **tiempos largos** y **un** reintento — otra vez, el contrato del tier hecho código.

## 3- Desarrollo de la guía (riel ghcr.io + Render + Playwright)

> Trabajás sobre tu app del semestre con la cadena del TP6 funcionando. Los ejemplos usan el backend en `./backend` (con su Dockerfile del TP2) y Playwright en `./frontend`; adaptá rutas y nombres. ⚠️ Donde diga `<owner>` va tu usuario de GitHub y donde diga `miapp-…onrender.com` va TU URL.
>
> 📬 **Logística**: el gate humano del TP6 sigue activo (vos como aprobador de PROD) — y esta semana además vas a ver e2e frenando promociones antes de que el gate te llegue a vos.
>
> 📬 **Cómo repartirlo.** El práctico tiene dos mitades que se pueden hacer en días distintos: la
> **cadena** (§3.1–3.2: que los entornos ejecuten la imagen) y las **pruebas** (§3.3–3.4). Hacé la
> cadena primero y comprobá su checkpoint: si QA no está ejecutando la imagen, las pruebas de después
> no prueban lo que este TP quiere probar. Mientras esperás una corrida o un deploy, avanzá con §3.3
> —las dos suites—, que no depende de Render. Y dentro de cada paso, juntá los cambios del YAML en un mismo Pull
> Request (en §3.2, `deploy-qa` y `deploy-prod` juntos) en vez de mandarlos de a uno: cada arreglo de
> una línea cuesta una cadena entera. Lo que no se junta con nada es el cambio de
> fuente en Render: va entre el checkpoint de §3.1 y el Pull Request de §3.2.

🔧 **Si tu app no es .NET + Vite**: todo lo que se evalúa es **lograr** cada cosa, no usar estas
herramientas. Tu fila, para las piezas nuevas de este práctico:

| Lo que tenés que lograr | Los ejemplos de la guía (.NET + Vite) | Cómo se llama en otro lado |
|---|---|---|
| Que el pipeline publique la imagen etiquetada con el commit | `docker/build-push-action` con `tags: …:sha-${{ github.sha }}` | igual en todos: es una acción de CI, no del lenguaje |
| Que el entorno EJECUTE esa imagen | Render *Existing Image* + `imgURL` en el deploy hook | *Deploy from registry* de Railway · App Service con contenedor · `docker compose pull` en un runner propio (§3.6) |
| Pedirle a tu api de verdad, sin navegador | el fixture `request` de Playwright (`npx playwright test e2e/api.spec.js`) | `supertest` · `RestAssured` · `pytest` + `requests` · un script con `curl` — lo que NO vale es un doble en el medio |
| Un navegador de verdad, manejado por código | Playwright (`npx playwright test e2e/tareas.spec.js`) | Cypress · Selenium · Puppeteer — cambia la sintaxis, no el criterio |
| Que el test apunte a un entorno u otro | `E2E_BASE_URL` (el front) y `API_BASE_URL` (la api), leídas en `playwright.config.js` y en el spec | las variables de entorno que tu runner lea; el criterio es que **no estén escritas en el código** |
| Buscar como una persona, no por CSS | `getByLabel` · `getByRole` | `cy.contains` / `cy.findByRole` · `By.xpath` es lo que NO se quiere |
| Reporte publicable, **uno por suite y con nombres distintos** | reporte HTML de Playwright + `upload-artifact` | el reporter que traiga tu herramienta, publicado como artefacto (patrón del TP4). Dos artefactos con el mismo nombre en una corrida la hacen fallar |

> 📌 **¿Tu app tiene un solo Dockerfile?** (por ejemplo, el backend sirve el `dist/` del front).
> Entonces son **dos** servicios image-backed (QA y PROD), no cuatro, y un solo paquete — igual que
> en el TP6, y no se descuenta nada. Decilo en `decisiones.md` en una línea — y ahí mismo, que `API_BASE_URL` y `E2E_BASE_URL` te quedan iguales, porque la api se sirve bajo el mismo host. Lo que se evalúa es que
> **lo que corre sea la imagen que publicaste**; lo que no vale es dejar **una parte** construyéndose
> desde el repo: si tenés dos imágenes, van las dos.

### 3.0 Por dónde arrancar: la cadena primero

Con la cadena del TP6 llegando a producción —que es el requisito—, el orden es éste.
En cada paso conviene saber de antemano **qué vas a ver**, para no asustarte:

| | Qué hacés | Qué vas a ver |
|---|---|---|
| **1** | Primero, el checkpoint de §3.1: anotá el `sha-` de 40 caracteres de tu último merge a `main`, y comprobá con `docker pull` que las dos imágenes son públicas. Después, en Render, **UN** servicio de QA (el front, por ejemplo): le cambiás la fuente a *Existing Image*, con esa imagen. Y lo probás a mano: el `curl` de su deploy hook, con esa misma imagen (§3.2) | Al cambiar la fuente, **nada** — cambiar la fuente no despliega. Con el `curl`: en los *Events* del servicio aparece el deploy de tu imagen, y dice «Triggered via Deploy Hook» |
| **2** | Los otros tres servicios: les cambiás la fuente, con la misma etiqueta (`-backend` en las apis, `-frontend` en los fronts). **Sin** `curl` | Nada todavía: siguen corriendo lo que construyeron. Los despliega la corrida del paso 3 |
| **3** | Un Pull Request: el hook con `imgURL` (§3.2) | Después del merge, los *Events* de los cuatro servicios muestran el deploy de la imagen `sha-<ese commit>`, y esa misma etiqueta está en tus dos paquetes |
| **4** | Las dos suites (§3.3): los pedidos a la api en `e2e/api.spec.js` y los flujos de navegador en `e2e/tareas.spec.js` | Las dos verdes contra tu compose local, y el reporte HTML abriendo |
| **5** | Los dos jobs y el gate (§3.4) | La integración **verde** y la e2e **ROJA** frenando una promoción — la evidencia central del práctico |

El orden no es capricho: primero **un** servicio probado a mano, para que si algo
falla falle en uno y no en producción; después los otros tres; y recién al final el
pipeline, que es lo que deja de nombrar un commit y pasa a nombrar una imagen. Las pruebas van
últimas por la misma razón: contra un QA que todavía reconstruye desde el repo no prueban lo que
este TP quiere probar.

> ⚠️ **Antes del paso 1, mirá *Actions*: ¿te quedó alguna corrida del TP6 esperando aprobación para
> PROD?** Si la aprobás más tarde, cuando tus servicios ya son de imagen, esa corrida despliega por el
> camino viejo —manda `&ref=`, que a un servicio *Existing Image* no le dice nada— y lo que llegue a
> producción no es lo que tu cadena nueva verificó. **Rechazala ahora, con el motivo escrito**: es el
> rechazo con motivo del TP6.

💡 **Mientras esperás una corrida o un deploy** podés avanzar con lo que no depende de
Render: escribir la primera prueba de integración y la primera e2e contra tu compose (§3.3) — las
dos corren contra `localhost` mientras las escribís.

### 3.1 Lo que tu pipeline YA publica, y por qué no se toca

> 🔴 **Acá NO se crea ningún job nuevo, y no se toca el orden de nada.** Desde el TP6 (§3.0), cada uno
> de tus dos jobs de build termina con el paso que **construye y publica su imagen**, etiquetada con
> el commit, y sólo cuando el cambio entra a `main`. Ése es el tercer eslabón de la cadena: si los
> tests fallan, el job muere antes y no se publica nada. Un job aparte que "junte" las imágenes
> **rompe esa garantía**: tendría que volver a construirlas, y entonces lo publicado ya no sería lo
> verificado sino otra construcción — el TP6 §3.0 lo explica con el mismo argumento.

**El TP7 no le agrega nada a este paso.** La línea de `tags` de tus dos pasos de build
**queda como la dejó el TP6**: una sola etiqueta, `sha-${{ github.sha }}` —el commit completo, los 40
caracteres—. Es la que vas a darle a Render como fuente de cada servicio (§3.2) y la que manda cada
deploy. No se agrega `latest` ni ninguna otra etiqueta que se mueva. El paso del backend queda así:

```yaml
# .github/workflows/ci.yml → jobs: → build-backend: → steps:
      - name: Construir y publicar la imagen del backend
        uses: docker/build-push-action@v7
        with:
          context: ./backend
          push: ${{ github.event_name == 'push' && github.ref == 'refs/heads/main' }}
          tags: ghcr.io/${{ github.repository }}-backend:sha-${{ github.sha }}   # igual que en el TP6: no se toca
          cache-from: type=gha,scope=backend
          cache-to: type=gha,mode=max,scope=backend
```

(El paso del frontend es idéntico: sólo cambia `-backend` por `-frontend`.)

> ⚠️ **Minúsculas, como en el TP6 §3.0.** Si tu usuario o tu repositorio tienen mayúsculas,
> `${{ github.repository }}` falla con `repository name must be lowercase`: en todos los bloques de
> esta guía donde aparece, escribí el nombre a mano, en minúsculas, igual que lo resolviste allá.

📌 **¿Y `latest`?** Tu pipeline no lo publica, y es a propósito. §2.2 ya dijo por qué: `latest` no
dice qué contenido hay detrás. Cualquier deploy que lo nombrara ejecutaría «lo último que se
publicó», haya pasado las e2e o no. Con una sola etiqueta por imagen —la de su commit— no hay forma
de desplegar algo sin decir exactamente qué es.

Y una condición que ya cumpliste en el TP6 y **acá pasa a ser requisito de funcionamiento**: los dos
paquetes tienen que estar **públicos**. Render los va a bajar **sin credenciales**; si alguno quedó
privado, no lo puede bajar y el deploy falla (la cátedra no lo pudo medir: sus paquetes nacieron
públicos). Lo que sí ves vos es el `unauthorized` del `docker pull` del checkpoint de abajo. El
práctico los usa públicos porque la alternativa es cargarle a Render una credencial que, según su documentación, es
un token personal tuyo de GitHub, guardado en un servicio de un tercero. Y como una imagen pública
la puede bajar cualquiera, **no lleva secretos**: las contraseñas y las direcciones van en las
variables del servicio (§3.2).

**✅ Checkpoint:** en tu último merge a `main`, los dos paquetes muestran la etiqueta `sha-…` de
ese commit (mirala en la página de cada package, no sólo que la corrida esté verde), y los dos son
**públicos**: desde cualquier máquina, sin sesión, los dos `pull` funcionan, con el commit
**completo**:

```bash
git switch main && git pull             # el merge se hizo en GitHub: traelo
SHA=$(git rev-parse HEAD); echo $SHA    # los 40 caracteres del commit que quedó en main
docker logout ghcr.io                   # para que una sesión vieja del TP2 no haga pasar el pull sin probar nada
docker pull --platform linux/amd64 ghcr.io/<owner>/<tu-repo>-backend:sha-$SHA
docker pull --platform linux/amd64 ghcr.io/<owner>/<tu-repo>-frontend:sha-$SHA
```

🔴 **Anotá ese `sha-…` completo: es el que vas a pegar en Render en §3.2.** Tres cosas sobre él:
- Son los **40 caracteres**. El registry compara la etiqueta letra por letra: no completa el resto
  como hace git. `git show 3f9c2ab` funciona; `docker pull …:sha-3f9c2ab` contesta `manifest unknown`.
  Si lo ves abreviado en una pantalla, no lo copies de ahí: el completo te lo da el `git rev-parse`.
- Es el commit **de `main`**, no el de tu rama: con *squash*, el merge crea un commit nuevo, y los
  commits de un Pull Request no publican imagen (el `push:` del paso sólo es verdadero en `main`). Por
  eso el `git switch main && git pull` va primero.
- Es **el mismo** para las dos imágenes: backend y frontend salen de la misma corrida, así que entre
  una y otra sólo cambia `-backend` por `-frontend`.

Si un `pull` contesta `manifest unknown`, la etiqueta está mal copiada o la corrida de `main` todavía
no terminó sus dos jobs de build.

⚠️ El `--platform linux/amd64` es porque tu pipeline construye para la arquitectura del runner. En
una máquina `arm64` (una Mac con chip Apple, por ejemplo), sin esa opción el `pull` falla diciendo que
no hay una versión para tu plataforma —con el almacén de imágenes clásico de Docker,
`no matching manifest for linux/arm64/v8`; con el de containerd, que Docker Desktop trae en las
instalaciones nuevas, el texto es otro— y no es que la imagen sea privada ni que no exista. Si un
`pull` contesta `unauthorized` o `denied`, eso sí: el paquete quedó privado, y se cambia en su
configuración, como en el TP6.

### 3.2 Los entornos dejan de construir: ahora EJECUTAN tu imagen

Éste es el centro del práctico, y es el arreglo de lo que el TP6 dejó mal **a propósito**: allá
Render **reconstruía** tu aplicación desde el repositorio en cada deploy, así que lo que corría en QA
y en PROD no era la imagen que tus tests aprobaron, sino otra construcción del mismo commit. Acá eso
se termina.

**Antes de tocar Render**: el checkpoint de §3.1 cumplido —los dos `pull` sin sesión— y, en la corrida de
`main` de ese commit, los dos jobs de build en verde (son los que publican). Tené a mano el `sha-…` completo del checkpoint de §3.1: es la
imagen que le vas a dar a Render, y ya comprobaste con el `pull` que existe y que es pública.

> 📌 **Primero con uno solo** —el front de QA, por ejemplo—. Cambiarle la fuente **no despliega
> nada**: el servicio sigue corriendo lo que construyó desde el repo, así que «responde» todavía no
> prueba nada. Por eso, después del paso 1 de abajo, desplegalo **una vez a mano con su hook**, el
> mismo que usa tu pipeline (lo copiás de *Settings → Deploy Hook*):
>
> ```bash
> # en la misma terminal del checkpoint de §3.1. Si abriste otra, parado en main al día, primero:
> #   SHA=$(git rev-parse HEAD)
> # si el servicio que elegiste es la api, va -backend: con el nombre cruzado el hook contesta 400 «deploy hook cannot change the host, project, or image name» (§2.4)
> # 🔴 la URL del hook va ÚLTIMA, y no primero como estás acostumbrado: en un curl, todo lo
> # que empieza con guiones son OPCIONES, y la dirección es el único argumento suelto. Como
> # --data-urlencode se lleva el valor que tiene al lado, la URL tiene que ir después.
> read -rs HOOK && curl -fsS --get --data-urlencode "imgURL=ghcr.io/<owner>/<tu-repo>-frontend:sha-$SHA" "$HOOK"; echo
> ```
>
> Tiene que contestar `{"deploy":{"id":…}}`; en *Events* del servicio tiene que aparecer
> «Deploy live for …» con ese `sha-…`, y la app tiene que responder en su URL. Si contesta `400` con
> «unable to fetch image with provided input», esa etiqueta no existe: casi siempre es el commit
> corto, o el de tu rama. Recién cuando anda seguí con los otros tres, y después el cambio del
> pipeline. Ese deploy a mano es una prueba, no la entrega: lo que corre al final en cada entorno lo
> despliega una corrida.
> *Existing Image* está en el plan gratis (medido por la cátedra el 18-09 en una cuenta Free); si en
> la tuya no aparece, no rehagas nada: consultá con la cátedra y seguí por el fallback del §3.6, que
> cumple todos los checkpoints.

**Los cuatro servicios pasan a *image-backed*** (api y front, en QA y en PROD) — y **no se crean de
nuevo: se les cambia la fuente**, que es lo que conserva la URL, las variables, el deploy hook y el
historial. Por cada uno:

1. En el servicio → *Settings → Build → Source → **Edit*** → **Existing Image** → en *Image URL*, la
   imagen del registry **con la etiqueta del checkpoint de §3.1**:
   `ghcr.io/<owner>/<tu-repo>-backend:sha-<los 40 caracteres>` (o `-frontend:sha-…` para los dos del
   front: la misma etiqueta, cambia sólo el nombre) → *Connect*. Es pública: no hace falta credencial.
   📌 **Esa imagen es sólo el punto de partida.** Render te pide la dirección de una imagen para
   cambiar la fuente, y la de tu último merge es una que ya comprobaste que existe. Lo que corre en
   cada entorno lo nombra después cada deploy de tu pipeline, con su `imgURL`: Render sólo exige que
   coincidan el registry y el nombre de la imagen; la etiqueta (o el digest) puede ser otra (§2.4).
   Los cuatro servicios quedan configurados con el mismo `sha-…`, y no hay que actualizarlo con cada
   merge.
   ⚠️ El diálogo se abre en la pestaña *Git Provider*: pasá a **Existing Image** antes de escribir.
   📌 **Cambiar la fuente no despliega nada**: el servicio sigue corriendo lo último que construyó
   desde el repo hasta que tu pipeline lo despliega con el hook. Y desde ahí, en *Settings*, la
   sección *Build* pasa a llamarse **Image**, y el encabezado del servicio dice **Image** donde antes
   decía *Docker*.
2. **Las variables de entorno no se tocan**: ya están donde tienen que estar (`ConnectionStrings__Default`
   con la base de **su** entorno en los back; `BACKEND_URL` —la api de **su** entorno, sin barra
   final— y `DNS_RESOLVER` en los front). 🔴 Acá se ve por qué el TP6 sacó esas direcciones de
   adentro de la imagen: **es la misma imagen del front la que corre en QA y en PROD**, y lo único
   que las distingue son estas variables.
3. **Los deploy hooks del TP6 siguen siendo los mismos**: no hay que volver a cargar ningún secret.
   Lo único que cambia es lo que el hook lleva pegado: `imgURL` en vez de `ref` (abajo).
4. 📌 **Auto-Deploy**: un servicio image-backed no lo tiene —la opción desaparece de *Settings*, y no
   se redespliega solo cuando cambia una imagen—. El deploy lo dispara tu pipeline, como pide el
   TP6. Y si desplegás a mano desde el panel, **eso no cuenta**: lo que corre en cada entorno tiene
   que haberlo desplegado una corrida.

   🔴 **Y no es inocuo: *Manual Deploy* te puede llevar producción para atrás.** Ese botón
   despliega la imagen que figura en *Settings → Image*, **no la que el servicio está corriendo**.
   Como esa configuración la escribiste una sola vez (la del primer merge) y de ahí en más quien
   cambia la imagen es el hook de tu pipeline —sin tocar *Settings*—, las dos se separan enseguida:
   el servicio corre lo último que desplegó la corrida y *Settings* sigue clavado en la primera.
   Tocar *Manual Deploy* ahí vuelve a la vieja, en verde y sin avisar.
   **Medido por la cátedra el 2026-09-20**: `api-qa` corría `sha-702ef60…`, *Settings* tenía
   `sha-a782701…` (el primer merge), y en *Events* aparecía el deploy de esa imagen.
   Si te pasa, se vuelve desde la corrida —re-disparando el deploy del commit que corresponde—, no
   con otro clic.

> ⚠️ **Si preferís crear servicios nuevos** en vez de cambiar la fuente: mientras el viejo exista, el
> nuevo **no puede quedarse con el mismo nombre** y Render le va a dar OTRA URL. Entonces, antes de
> seguir, tenés que cargar los cuatro deploy hooks nuevos, poner el `BACKEND_URL` de cada front
> apuntando a la **api nueva** de su entorno, cambiar las cuatro URLs de los smoke tests y la
> `E2E_BASE_URL` del workflow, y actualizar las URLs de «Enlaces del TP7». Si no lo
> hacés, tus e2e y el smoke del front pueden seguir dando verde **contra los servicios viejos**, los que
> reconstruyen desde el repo: exactamente lo que este práctico viene a eliminar, y sin que nada se
> ponga rojo. Al recargar las variables, repetí la comprobación del TP6 §3.1 (un dato creado en PROD
> **no** aparece en QA): es el momento en que PROD vuelve a poder quedar apuntando a la base de QA.
> Y borrá los viejos apenas los nuevos estén verdes: las 750 horas del plan gratis son del
> **workspace** (TP6 §2.7), y ocho servicios vivos las consumen el doble de rápido. Anotá en
> `decisiones.md` qué hiciste con los viejos.

**Y en el pipeline cambia UNA idea**: el hook deja de decir *qué commit construir* y pasa a decir
**qué imagen ejecutar**. Donde el TP6 mandaba `&ref=$GITHUB_SHA`, ahora va `imgURL` con la imagen del
commit — y el `&ref=` **se saca**: con una imagen no hay nada que construir, y dejarlo le haría creer
a quien lea tu YAML que el proveedor todavía compila.

```yaml
# .github/workflows/ci.yml → jobs:   (reemplaza al deploy-qa que traías del TP6)
  deploy-qa:
    runs-on: ubuntu-latest
    needs: [build-backend, build-frontend]   # sin CI verde no hay imagen

    if: github.ref == 'refs/heads/main'
    environment: qa
    steps:
      - name: Desplegar en QA la IMAGEN del commit (api + front)
        shell: bash
        env:
          HOOK_API: ${{ secrets.RENDER_HOOK_API_QA }}
          HOOK_FRONT: ${{ secrets.RENDER_HOOK_FRONT_QA }}
        run: |
          # imgURL va como un parámetro más de la URL del hook (que ya trae ?key=…).
          # --get --data-urlencode lo agrega con '&' y le codifica los ':' y '/' de la imagen.
          # La URL del hook va ÚLTIMA: es el único argumento suelto, y --data-urlencode se lleva
          # el valor que tiene al lado (§3.2 lo explica la primera vez que aparece).
          IMG_API="ghcr.io/${{ github.repository }}-backend:sha-${{ github.sha }}"
          IMG_FRONT="ghcr.io/${{ github.repository }}-frontend:sha-${{ github.sha }}"
          curl -fsS --max-time 30 --get --data-urlencode "imgURL=$IMG_API" "$HOOK_API"
          curl -fsS --max-time 30 --get --data-urlencode "imgURL=$IMG_FRONT" "$HOOK_FRONT"

      - name: Smoke test (espera a que QA responda)
        shell: bash
        env:               # ⚠️ TUS URLs de QA
          URL_API: https://miapp-api-qa.onrender.com
          URL_FRONT: https://miapp-front-qa.onrender.com
        run: |
          echo "Esperando a que QA esté vivo..."
          for i in $(seq 1 30); do
            if curl -fsS --max-time 10 "$URL_API/health" > /dev/null \
               && curl -fsS --max-time 10 "$URL_API/api/tareas" > /dev/null \
               && curl -fsS --max-time 10 "$URL_FRONT/" > /dev/null; then
              echo "✅ QA responde (API + BD + front)"; exit 0
            fi
            sleep 20
          done
          echo "❌ QA no respondió a tiempo"; exit 1
```

> ⚠️ `/health` y `/api/tareas` son los de la app de la cátedra: van **los tuyos**, como en el TP6 —
> uno de vida y otro que **toque la base**.

**Y lo mismo en `deploy-prod`**: su paso de deploy y su smoke quedan como en el bloque de §3.4 —la
misma imagen `sha-…`—, con el `needs: deploy-qa` que ya tiene. El
`needs: e2e` y el `concurrency` llegan en §3.4. PROD ya es de imagen desde el paso de Render: si su
deploy siguiera mandando `&ref=`, no diría qué imagen ejecutar. Juntá los dos cambios en este mismo
Pull Request.

> 📌 **Lo que este smoke prueba, y lo que no.** Prueba que el entorno **responde**: la api, la base
> y el front. No prueba que esté ejecutando **tu imagen** — si el deploy falló, el servicio sigue
> sirviendo la versión anterior y el smoke da verde igual. Eso lo mirás vos, en *Events*: el deploy
> de cada servicio tiene que nombrar el `sha-<commit>` de esa corrida y decir «Triggered via Deploy
> Hook». Cerrar ese hueco automáticamente es el concepto de §2.4 —que el endpoint de vida devuelva
> el commit y el smoke lo compare—, y no se implementa en este TP.
>
> ⚠️ **Los DOS contenedores, no uno.** Si dejás el front construyéndose desde el repo, la mitad de
> lo que el usuario ve sigue siendo una reconstrucción. En *Events* del front, el deploy tiene que
> nombrar el `…-frontend:sha-<commit>` de esa corrida.

**✅ Checkpoint:** un merge a `main` termina con los cuatro servicios ejecutando la imagen
`sha-<ese commit>` y el smoke en verde. La evidencia: en *Events* de cada servicio, el deploy dice
«Triggered via Deploy Hook» y nombra esa etiqueta —no un commit construido—, y la misma etiqueta
está en tus dos paquetes.

> 🎬 **Lo que se ve en el video, desde tu terminal** (§3.2):
>
> ```bash
> git switch main && git pull && git switch -c feature/deploy-imagen   # si mergeaste desde la web, el pull lo trae
> git --no-pager diff -U1                      # el paso de deploy-qa con imgURL
> git add .github/workflows/ci.yml             # apartado, para que el próximo diff muestre sólo lo nuevo
> git commit -m "…" && git push -u origin feature/deploy-imagen
> gh pr create --fill
> gh pr checks --watch
> gh pr merge --squash --delete-branch
> ```

### 3.3 Las dos suites con Playwright: pedidos a la api, y navegador

En `./frontend` (reusa el Node que ya tenés):

```bash
cd frontend
npm i -D @playwright/test
npx playwright install chromium     # baja el browser (local)
```

📌 **Una instalación, dos suites.** Playwright hace las dos cosas: maneja un navegador de verdad (los
flujos de `tareas.spec.js`) y también manda pedidos HTTP sin abrir nada (los de `api.spec.js`, con el
fixture `request`). Ése es el ahorro: no hay una segunda herramienta que instalar ni que explicar. El
`install chromium` es sólo para los flujos de navegador — por eso el job `integracion` de §3.4 **no
lo corre**, y por eso tarda segundos.

📌 **El código va en el otro orden, y es a propósito.** En el marco teórico la integración viene
antes que las e2e (§2.5 → §2.6), porque es el escalón de abajo. Acá escribís primero
`tareas.spec.js` (navegador) y después `api.spec.js` (pedidos): las dos usan la misma herramienta, y
los pedidos se entienden en dos líneas una vez que viste a Playwright manejando un navegador. En el
pipeline (§3.4) vuelven al orden de la cadena: integración primero, e2e después.

⚠️ **Commiteá el `package-lock.json`** junto con el `package.json`: el job del §3.4 instala con
`npm ci`, que instala exactamente lo que dice el lockfile. Si no viaja, en el runner no hay
Playwright.

⚠️ **Antes de escribir el primer spec, separá los mundos**: vitest (tus unit tests del TP5) matchea por default TODO archivo `*.spec.js` — incluidos los de Playwright — y explota con un error críptico (`You are calling test() from an async test.describe() block`) que además pondría **rojo tu job `build-frontend`** (y con él, toda la cadena). Cada runner con su carpeta — en `vite.config.js`:

```js
import { configDefaults } from 'vitest/config'   // si tu config importaba sólo de 'vite', sumá esta línea
// … dentro de defineConfig({ test: { … } }):
    exclude: [...configDefaults.exclude, 'e2e/**'],   // vitest NO toca los specs de Playwright (los DOS: el de api y el de navegador viven en e2e/)
```

Y dos líneas de higiene: agregá `playwright-report/` y `test-results/` a tu `.gitignore` (Playwright los genera en cada corrida; no se commitean).

`frontend/playwright.config.js`:

```js
import { defineConfig } from '@playwright/test'

export default defineConfig({
  testDir: './e2e',                // recoge los DOS archivos; cada job de §3.4 corre el suyo por nombre
  timeout: 60_000,                 // tope de CADA test: generoso para las e2e, de sobra para los pedidos a la api
  expect: { timeout: 15_000 },     // tope de CADA aserción: no lo hereda del de arriba (el default es 5 s)
  use: {
    baseURL: process.env.E2E_BASE_URL || 'http://localhost:3000',   // el FRONT. La api tiene la suya: API_BASE_URL, que lee api.spec.js
    trace: 'on-first-retry',         // la traza se graba en el retry de un fallo (los verdes no la generan)
    screenshot: 'only-on-failure',   // screenshot standalone de cada fallo en el reporte
  },
  retries: 1,                      // 1 retry: absorbe una demora suelta; si un test pasa recién ahí, sale «flaky»
  reporter: [['html', { open: 'never' }], ['list']],
})
```

📌 El reintento no esconde un error de verdad —si falla las dos veces, el test queda rojo—, pero sí
puede esconder un test frágil: uno que pasa recién al reintentar aparece en el reporte como
**flaky**, con la corrida en verde. Por eso, abrí el reporte aunque la corrida esté verde. Un flaky
aislado: miralo y seguí, que puede ser una demora de la red. El mismo test flaky dos
corridas seguidas ya no es el plan gratis: es el test, y se arregla, porque un test que a veces falla
es el que entrena al equipo a ignorar el rojo.

📌 El minuto por test alcanza en el pipeline **porque el smoke del job anterior ya despertó QA**. Si
alguna vez corrés las e2e sin ese smoke delante (a mano, contra un QA dormido), subilo a 120 s o
despertá QA con un `curl` antes.

> 🔴 **Los selectores de abajo son de la app de la cátedra, y en la tuya probablemente no existan
> todavía.** `getByLabel` y `getByRole` buscan el **nombre accesible** de cada control —el que lee
> un lector de pantalla—, no su clase ni su placeholder. Antes de escribir el spec, comprobá en tu
> front que estén estas cuatro cosas:
>
> | Lo que el test busca | Qué tiene que existir en tu HTML |
> |---|---|
> | `getByLabel('Título de la nueva tarea')` | un `<label for="…">` asociado al input, o un `aria-label` |
> | `getByRole('button', { name: 'Agregar' })` | el texto visible del botón (o su `aria-label`) |
> | `getByRole('button', { name: 'Borrar X' })` | un `aria-label={'Borrar ' + titulo}` **en cada fila** — sin esto no podés limpiar lo que creaste |
> | `getByRole('alert')` | `role="alert"` en el elemento que muestra el error |
>
> Si tu e2e no encuentra el campo por su label, es porque tu formulario no tiene label: el test te
> está señalando un problema de accesibilidad de tu app, no un límite de Playwright. Agregarlo es
> parte del trabajo de este TP, y es mejor arreglo que esquivarlo con un selector de CSS, que cambia
> con el diseño. Si igual necesitás una salida, `getByTestId` con un `data-testid` explícito es la
> menos frágil —y contá en `decisiones.md` por qué la elegiste—. Para descubrir los nombres que tu
> app **ya** tiene: `npx playwright codegen http://localhost:3000` graba tus clics y te escribe los
> selectores.
>
> 📌 Nada de esto aplica a `api.spec.js`: ahí no hay pantalla, sólo rutas y códigos de estado. Lo que
> tenés que saber de tu api son sus endpoints y qué contesta ante un dato inválido.

#### Los flujos de navegador

`frontend/e2e/tareas.spec.js` — dos flujos críticos para arrancar, que **limpian lo que crean** (el
mínimo del práctico son **tres**: el tercero lo elegís vos — la regla para no quedarte pensando está abajo):

```js
import { expect, test } from '@playwright/test'

test('crear una tarea la muestra en la lista, y borrarla la saca', async ({ page }) => {
  await page.goto('/')
  const titulo = `e2e ${Date.now()}`                    // único: no choca con otra corrida
  await page.getByLabel('Título de la nueva tarea').fill(titulo)
  await page.getByRole('button', { name: 'Agregar' }).click()
  await expect(page.getByText(titulo)).toBeVisible()    // el dato que ESTE test creó, sin sleeps
  await page.getByRole('button', { name: `Borrar ${titulo}` }).click()
  await expect(page.getByText(titulo)).toHaveCount(0)   // y el borrado, comprobado
})

test('un título demasiado largo muestra el error y no crea nada', async ({ page }) => {
  await page.goto('/')
  const titulo = `e2e ${Date.now()} ${'x'.repeat(100)}` // pasa el máximo (100 en la app de la cátedra)
  await page.getByLabel('Título de la nueva tarea').fill(titulo)
  await page.getByRole('button', { name: 'Agregar' }).click()
  await expect(page.getByRole('alert')).toBeVisible()   // el usuario ve el error…
  await expect(page.getByText(titulo)).toHaveCount(0)   // …y en la lista no apareció nada
})
```

Fijate qué tienen en común, porque es lo que la Tarea 3 pide de cada flujo de navegador (y la Tarea 2, de cada prueba de integración): **interactúan** (un test
que sólo navega es un smoke, y el smoke ya lo tenés), **afirman sobre el dato que el propio test
produjo** (su título único, no «la página es visible»), y **comprueban** que lo que crearon ya no
está.

#### Los pedidos a la api

Y el escalón del medio: las mismas ideas, un piso más abajo. Acá no hay pantalla — le hablás a **tu
api de QA directamente**, con su base de datos de verdad detrás, y sin ningún doble en el medio. Es
lo que §2.5 llama *integración amplia*: la api **ya desplegada**, no una copia armada en el pipeline.

Playwright lo hace con el fixture `request`, que **no abre ningún navegador**: es un cliente HTTP.

`frontend/e2e/api.spec.js` — dos pruebas para arrancar (el mínimo del práctico son **tres**: la
tercera la elegís vos, con la misma regla que la de navegador):

```js
import { expect, test } from '@playwright/test'

// La api tiene su propia dirección: NO es la del front. Sin barra al final.
const API = process.env.API_BASE_URL || 'http://localhost:8080'

// una ayuda, para no repetir: pide la lista y la devuelve ya convertida
async function listar (request) {
  const r = await request.get(`${API}/api/tareas`)
  expect(r.status()).toBe(200)
  return await r.json()
}

test('el alta guarda en la base de verdad, y el borrado la saca', async ({ request }) => {
  const titulo = `api ${Date.now()}`                       // único: no choca con otra corrida
  const alta = await request.post(`${API}/api/tareas`, { data: { titulo } })
  expect(alta.status()).toBe(201)                          // la api dice «creada»
  const creada = await alta.json()

  const lista = await listar(request)                      // y la BASE la devuelve: no es un doble
  expect(lista.some(t => t.titulo === titulo)).toBe(true)

  const borrado = await request.delete(`${API}/api/tareas/${creada.id}`)
  expect(borrado.status()).toBe(204)                       // «borrada, no hay nada que devolver»

  const despues = await listar(request)                    // y el borrado, comprobado
  expect(despues.some(t => t.titulo === titulo)).toBe(false)
})

test('un título vacío lo rechaza la api, y no crea nada', async ({ request }) => {
  const antes = await listar(request)

  const alta = await request.post(`${API}/api/tareas`, { data: { titulo: '' } })
  expect(alta.status()).toBe(400)                          // el contrato del error

  const despues = await listar(request)
  expect(despues.length).toBe(antes.length)                // y de verdad no se creó nada
})
```

🔴 **Por qué esto no es un unitario con otro nombre.** El `GET` que sigue al `POST` le pregunta **a
la base**, no a un doble: si tu código le manda una fecha que Postgres rechaza (el caso de §2.5), la
api contesta `500` y esta prueba se pone roja. Un unitario con un doble daría verde, porque el doble
acepta cualquier cosa. Y una base en memoria tampoco sirve: es otro doble, no pasa por el driver de
Postgres.

⚠️ **`API_BASE_URL` es la api, no el front** — la misma distinción que el `BACKEND_URL` del TP6. El
front de QA y la api de QA son dos servicios con dos URLs. Si le pasás la del front, los pedidos van
a dar `404` y vas a pensar que tu api está rota.

📌 **¿Y por qué no armo la api dentro del pipeline, con una base descartable?** Se puede —es la
*integración estrecha* de §2.5— y llega antes, porque no necesita deploy. Pero cuesta bastante más
configuración (`WebApplicationFactory` o su equivalente, más un Postgres como `services:` del job), y
vos ya tenés QA corriendo la imagen exacta de esta corrida. Con un décimo del trabajo, el mismo
concepto. Anotá en `decisiones.md` qué gana y qué pierde esa elección.

```bash
# Probala local contra tu compose (TP2) primero — con tu .env del TP2 (DB_PASSWORD) presente:
cd ..                               # el compose vive en la RAÍZ del repo, no en frontend/
docker compose up -d --build        # --build: que tome lo que le cambiaste al front (los labels, por ejemplo)
# el compose del TP2 publica la api en el 8080 y el front en el 3000: por eso son dos URLs distintas
cd frontend
API_BASE_URL=http://localhost:8080 npx playwright test e2e/api.spec.js       # la api, directo
E2E_BASE_URL=http://localhost:3000 npx playwright test e2e/tareas.spec.js    # el front, con browser
# PowerShell: $env:API_BASE_URL='http://localhost:8080'; npx playwright test e2e/api.spec.js
npx playwright show-report          # el reporte HTML (el de la última corrida)
# ¿Querés VER una traza con los tests en verde? Corré con: npx playwright test --trace on
```

(Con **Cypress** los mismos flujos valen igual; cambia la sintaxis, no el criterio.)

**✅ Checkpoint:** tus **3 pruebas de integración y tus 3 e2e** verdes contra tu compose local (6 en total), el reporte HTML abre, y viste al menos una traza (con `--trace on`, o provocando un fallo momentáneo — la config normal solo la graba cuando algo falla, que es cuando la necesitás).

> 🎬 **Lo que se ve en el video, desde tu terminal** (conceptos: §2.5 y §2.6):
>
> ```bash
> git switch main && git pull && git switch -c feature/e2e
> git --no-pager diff -U1 package.json         # @playwright/test en devDependencies
> git status --short                           # package.json y package-lock.json: viajan los dos
> git add .
> git --no-pager diff --cached -U1 vite.config.js ../.gitignore   # el exclude de vitest y las dos líneas de higiene
> bat --paging=never --style=numbers playwright.config.js   # un archivo nuevo, con número de línea
> bat --paging=never --style=numbers e2e/api.spec.js       # la suite de integración
> bat --paging=never --style=numbers e2e/tareas.spec.js    # la de navegador
> cd ..                                        # el compose vive en la raíz del repo
> docker compose up -d --build
> cd frontend
> API_BASE_URL=http://localhost:8080 npx playwright test e2e/api.spec.js
> E2E_BASE_URL=http://localhost:3000 npx playwright test e2e/tareas.spec.js --trace on   # con traza aunque pasen
> ```

### 3.4 Las dos suites en el pipeline, como gate de promoción

```yaml
# .github/workflows/ci.yml → jobs:   (DOS jobs NUEVOS, entre deploy-qa y deploy-prod)
  integracion:
    runs-on: ubuntu-latest
    needs: deploy-qa                       # QA ya está ejecutando la imagen de ESTA corrida
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@v6

      - name: Setup Node
        uses: actions/setup-node@v6
        with:
          node-version: "22"
          cache: "npm"
          cache-dependency-path: frontend/package-lock.json

      - name: Install (sin browser)
        run: npm ci                        # ← sin `playwright install`: estas pruebas no abren navegador
        working-directory: ./frontend

      - name: Integración contra la api de QA
        env:
          API_BASE_URL: https://miapp-api-qa.onrender.com    # ⚠️ TU api de QA (la api, no el front)
        run: npx playwright test e2e/api.spec.js
        working-directory: ./frontend

      - name: Publicar reporte de integración
        uses: actions/upload-artifact@v6
        if: ${{ !cancelled() }}
        with:
          name: playwright-report-integracion   # distinto del de e2e: dos iguales rompen la corrida
          path: frontend/playwright-report
          retention-days: 7

  e2e:
    runs-on: ubuntu-latest
    needs: integracion                     # si la api está rota, no gastamos un browser en confirmarlo
    timeout-minutes: 20                    # si algo se cuelga, el job muere en vez de esperar horas
    steps:
      - uses: actions/checkout@v6

      - name: Setup Node
        uses: actions/setup-node@v6
        with:
          node-version: "22"
          cache: "npm"
          cache-dependency-path: frontend/package-lock.json

      - name: Install + browser
        run: |
          npm ci
          npx playwright install --with-deps chromium
        working-directory: ./frontend

      - name: e2e contra QA
        env:
          E2E_BASE_URL: https://miapp-front-qa.onrender.com   # ⚠️ TU front de QA
        run: npx playwright test e2e/tareas.spec.js   # ← SU archivo: sin esto vuelve a correr la integración
        working-directory: ./frontend

      - name: Publicar reporte e2e
        uses: actions/upload-artifact@v6
        if: ${{ !cancelled() }}            # el reporte importa MÁS cuando falla (y no cuando cancelás)
        with:
          name: playwright-report-e2e
          path: frontend/playwright-report
          retention-days: 7

  deploy-prod:
    runs-on: ubuntu-latest
    needs: e2e                             # ← EL gate: sin e2e verde, ni siquiera se pide aprobación
    environment: production                # ⏸ el reviewer del TP6 sigue ahí
    concurrency: { group: deploy-prod, cancel-in-progress: false }   # la del TP6 §3.4 — si no la agregaste allá, va ahora
    steps:
      - name: Deploy de la MISMA imagen a PROD
        shell: bash
        env:
          HOOK_API: ${{ secrets.RENDER_HOOK_API_PROD }}
          HOOK_FRONT: ${{ secrets.RENDER_HOOK_FRONT_PROD }}
        run: |
          IMG_API="ghcr.io/${{ github.repository }}-backend:sha-${{ github.sha }}"
          IMG_FRONT="ghcr.io/${{ github.repository }}-frontend:sha-${{ github.sha }}"
          curl -fsS --max-time 30 --get --data-urlencode "imgURL=$IMG_API" "$HOOK_API"
          curl -fsS --max-time 30 --get --data-urlencode "imgURL=$IMG_FRONT" "$HOOK_FRONT"   # ← la MISMA imagen que verificaron QA y las e2e

      - name: Smoke test PROD (espera a que PROD responda)
        shell: bash
        env:               # ⚠️ TUS URLs de PROD
          URL_API: https://miapp-api-prod.onrender.com
          URL_FRONT: https://miapp-front-prod.onrender.com
        run: |
          for i in $(seq 1 30); do
            if curl -fsS --max-time 10 "$URL_API/health" > /dev/null \
               && curl -fsS --max-time 10 "$URL_API/api/tareas" > /dev/null \
               && curl -fsS --max-time 10 "$URL_FRONT/" > /dev/null; then
              echo "✅ PROD responde (API + BD + front)"; exit 0
            fi
            sleep 20
          done
          echo "❌ PROD no respondió a tiempo"; exit 1
```

(El paso de deploy y el smoke de PROD son los que ya pusiste en §3.2: lo nuevo de este bloque son el job `integracion`, el `needs: integracion` del job `e2e`, el `needs: e2e` de `deploy-prod` y el `concurrency`. Fijate el detalle central: `deploy-prod` despliega **exactamente el mismo `sha-…`** que QA verificó y las dos suites ejercitaron, y su smoke lo comprueba igual que el de QA: el log de la corrida muestra el mismo `imgURL` en los dos deploys.)

> 🔴 **El gate tiene que poder decir que no.** Una sola línea lo desarma: `continue-on-error: true`
> en el paso (o el job) de e2e **o de integración**, un `|| true` o un `exit 0` después de
> `npx playwright test`, o un `if: always()` (o `if: ${{ !cancelled() }}`) en `deploy-prod`. Con
> cualquiera de ellas, `deploy-prod` sigue teniendo su `needs: e2e` escrito y pide aprobación igual
> aunque las pruebas hayan fallado; con las dos primeras, además, el job queda en verde y la corrida
> no se ve distinta. Y ahora hay **una trampa más**: un `if: always()` en el job `e2e` lo haría correr
> aunque la integración haya fallado, y entonces el `needs: integracion` deja de frenar nada.
> Los únicos `!cancelled()` de este tramo son los de los **dos** pasos que **publican reporte**. Y tres cosas más que tienen que seguir como están: las e2e viven en **el mismo
> workflow** que `deploy-qa` y `deploy-prod` y corren con el mismo `push` a `main` (un workflow
> aparte o disparado a mano no es un gate, es un test que alguien se acuerda de correr); el
> environment `production` sigue con su **required reviewer** del TP6; y en la corrida se lee, de
> arriba abajo, `build → deploy-qa → integracion → e2e → aprobación → deploy-prod`.

> ⚠️ **Tu QA es uno solo, y lo comparten todas tus corridas** —y ahora, las dos suites—. Con dos
> merges seguidos, la corrida B puede redesplegar QA mientras la integración o las e2e de la A lo
> están usando: QA cambia de imagen con las pruebas de A a medio correr. La ventana es más larga que
> antes, porque son dos jobs seguidos usando QA.
> - **Qué ves**: un rojo en la integración o en la e2e de A que no es de tu código. O algo peor, porque no avisa: un verde
>   que en parte probó la imagen de B.
> - **Cómo lo reconocés**: mirá en *Events* del servicio de QA qué imagen está corriendo. Si es la de B, mirá en *Actions* a qué
>   hora corrió el `deploy-qa` de B: si fue con la e2e de A corriendo, esa e2e no vale. Y si dudás de
>   un rojo, desempata la e2e de B, que trae los dos cambios: si también da rojo, es tu código.
> - **Qué hacés**: la corrida A **se rechaza, con el motivo** (si su e2e quedó roja, ni llega a
>   pedirte aprobación: no hay nada que rechazar). La que vale es la B, que trae los dos cambios y
>   cuya e2e sí probó lo que dice probar. 🔴 **Rechazar la A es tuyo, no lo hace el `concurrency`**
>   (TP6 §3.4): ese mecanismo cancela lo que está *en cola*, y un job esperando tu aprobación no está
>   en cola. No re-corras los jobs `integracion` ni `e2e` de la A: QA ya tiene la
>   imagen de B, así que probarían esa imagen y no la suya. Y la regla: **un merge por vez** mientras la
>   cadena corre (antes de mergear, mirá *Actions*: si la corrida anterior no terminó sus jobs `integracion` y `e2e`,
>   esperá).
> - **Cómo lo resuelve un equipo real**: con un entorno **por corrida**, que nace con ella y se
>   destruye al terminar, así ninguna corrida pisa a otra. Eso queda fuera de este práctico: acá QA
>   es uno solo.
>
> Anotalo en `decisiones.md` como **límite conocido** de tu cadena: qué es, cómo lo reconocerías y
> qué harías.

**La evidencia central del TP**: rompé **la aplicación**, no el test, y rompé **el front**. Cambiá el
texto del botón "Agregar", o hacé que el front mande el título con otro nombre (`title` en vez de
`titulo`): la API lo rechaza, y el smoke sigue verde —le habla a la
API con el nombre correcto: prueba la API, no el front—, y sólo la e2e lo ve. Hacelo
**sin tocar `e2e/`**, mergeá, y mirá la cadena: imagen → QA → smoke **verde** → **integración
VERDE** → **e2e ROJA** → `deploy-prod` ni aparece como pendiente de aprobación. El smoke en verde es
parte del punto: el sistema respondía y aun así estaba roto.

> 🔴 **Y acá cobrás la tabla de §2.5.** La integración quedó **verde** y la e2e **roja**: eso no es
> sólo «algo se rompió», es la cadena diciéndote **quién**. A la api le hablaste directo, con el
> nombre correcto del campo, y su base guardó y borró sin chistar: la api y la base están sanas. El
> que no usa bien la api es el front. Fila 1 de la tabla, sin que nadie abra el código.
>
> Si en cambio hubieras roto **la api**, la integración se habría puesto roja y la e2e **no habría
> corrido** —su `needs` no se cumple—: fila 2, y también es información. Lo que entregás son las
> **dos** cosas: los colores, y lo que leés en ellos.

⚠️ **Rompé el FRONT, no la API.** Lo que la e2e tiene que atrapar es lo que nadie más ve: cómo el
front usa la API. Tampoco sirve una rotura que ataje un unitario del front: si tu test con doble del TP5 afirma el nombre del campo,
elegí otra, como el texto del botón que la e2e busca. Y hay un motivo nuevo: romper el front es lo
que produce el **par verde/rojo** que la evidencia pide. Si rompés la api, la e2e ni corre, y te
quedás sin par.

(Un rojo tarda: cada test espera lo que busca hasta su tope —15 s una aserción, el minuto entero si
no encuentra el botón que tiene que clickear—, y después su retry. En la app de la cátedra, con el
front que manda `title`, la corrida roja de las e2e tardó alrededor de un minuto: no se colgó.)

Descargá el reporte de **esa** corrida y diagnosticá con él, no mirando el código. Desde `frontend/`:

```bash
gh run list --limit 3                                    # el ID de la corrida roja
gh run download <ID> -n playwright-report-e2e -D reporte-rojo          # el reporte ROJO: qué falló
gh run download <ID> -n playwright-report-integracion -D reporte-verde  # el VERDE: la otra mitad del diagnóstico
npx playwright show-report reporte-rojo                  # desde frontend/: lo abre en el navegador
```

(`reporte-rojo` no se commitea: borralo al terminar). En el reporte, el test fallido trae la
captura, y en *Retry #1* la traza, con la pestaña *Network*: ahí ves qué le mandó el navegador a tu
API y qué contestó. Arreglá —desde la raíz del repo: `cd ..` antes del `git switch main`, porque
quedaste en `frontend/`—, mergeá, cadena completa verde, aprobación, PROD.

**✅ Checkpoint:** la secuencia **integración-verde + e2e-roja** frena-promoción capturada —el commit
que rompió la app, la corrida, **los dos** reportes y la traza del fallo—, **el diagnóstico escrito**
(qué te dice ese par), y la corrida completa posterior con PROD recibiendo la misma imagen.

> 🎬 **Lo que se ve en el video, desde tu terminal** (§3.4):
>
> ```bash
> cd ..                                        # de frontend/ a la raíz: el ci.yml vive ahí
> git --no-pager diff -U1                      # el job e2e, en dos tandas, y después deploy-prod
> git add .github/workflows/ci.yml
> git commit -m "…" && git push -u origin feature/e2e
> gh pr create --fill
> gh pr checks --watch                         # en un PR, deploy-qa, integracion, e2e y deploy-prod salen salteados
> gh pr merge --squash --delete-branch
> # la rotura de la app, en su propia rama (y el arreglo, igual):
> git switch main && git pull && git switch -c feature/alta-rota
> git add frontend/src/App.jsx
> # después de bajar el reporte rojo (desde frontend/), el arreglo arranca en la raíz:
> cd ..
> ```

### 3.5 La release que le pone nombre a lo que está en PROD

Cuando PROD se rompe, la pregunta es **a qué volvés, y cómo lo encontrás en dos minutos**. La
respuesta es la release: `v7.0.0` es el **nombre** de una versión que sabés que anduvo en PROD (cada
práctico cierra con su número, como fija el encabezado). Por eso, igual que en el TP6 §3.5, el tag y
la release van sobre **el commit que está en PROD**, no sobre el último commit de `main`. Pueden no
ser el mismo: entre tu última aprobación y este comando pudo entrar otro merge, que rechazaste o que
todavía espera aprobación. Si etiquetás ése, tu `v7.0.0` nombra algo que nunca corrió en PROD, y el
tag nombraría una versión que nadie vio andar ahí.

¿Y cómo sabés qué commit llegó de verdad a PROD? Te lo dice ***Deployments*: una página de tu repo
en GitHub** (el link *Deployments* del sidebar de la home del repo, el mismo del TP6 §3.3) que
registra, por cada environment, qué commit se desplegó y cuándo. El comando de abajo le pregunta
eso mismo, por el environment `production`:

```bash
git switch main && git pull
# ¿qué commit está en PROD? se lo preguntás a Deployments, no al último commit de main:
# ({owner} y {repo} se escriben así, con las llaves: gh las completa con el repo en el que estás)
export SHA_PROD=$(gh api 'repos/{owner}/{repo}/deployments?environment=production' --jq '.[0].sha')
echo $SHA_PROD                              # los 40 caracteres: el tag tiene que ir sobre ESTE commit
git tag v7.0.0 $SHA_PROD
git push origin v7.0.0
gh release create v7.0.0 --generate-notes --verify-tag    # --verify-tag: falla si el tag no existe, en vez de crearlo en el último commit de main
```

(`export` deja la variable disponible para los comandos que siguen, en **esta** terminal: si abrís
otra, volvé a correr el `export`. Y `.[0]` es el último deployment que se **creó**: si después del
último deploy aprobado rechazaste o cancelaste una corrida de PROD, comprobá en *Deployments* del repo
que el commit del `echo` sea el que está activo.)

**Del tag a la imagen hay un solo paso, y ya lo tenés**: tag → commit → imagen. El tag te da el
commit (`git rev-list -n1 v7.0.0`), y tu pipeline publicó las dos imágenes de ese commit con su
nombre, `sha-<commit>`: el backend y el frontend, con la misma etiqueta. Abrí la release en GitHub,
mirá a qué commit apunta, y buscá esa etiqueta en tus dos paquetes: ése es tu catálogo de versiones
(TP6 §2.6), ahora con binarios de verdad. **Volver a una de ellas es mandarle esa imagen al hook**
—el mismo mecanismo del deploy—, y es lo que en una cursada real harías ante un incidente. En este TP
no se practica: el rollback ya lo hiciste en el TP6.

Lo que §2.2 dice de los tags vale también para éste: `sha-<commit>` se puede volver a publicar —por
ejemplo, si volvés a correr la corrida de ese commit—, y el contenido de la imagen ya no sería el que
estuvo en PROD. Lo que no se mueve es el **digest** (`…@sha256:…`), la huella del contenido: si querés
la garantía completa, el `imgURL` del hook acepta el digest en lugar del tag (§2.4).

**✅ Checkpoint:** el tag y la release `v7.0.0` apuntan al commit que estaba en PROD —no al último
de `main`—, y **los dos** paquetes tienen la imagen `sha-` de ese commit. Y podés explicar por qué
el tag va sobre ése, y cómo llegás del tag a la imagen.

### 3.6 Fallback local (mismos conceptos, cero nube)

Tu runner del TP6 + las imágenes del registry: en `compose.qa.yml` **y** en `compose.prod.yml`,
reemplazá **los dos** `build:` por su `image:` —el del backend por
`image: ghcr.io/<owner>/<tu-repo>-backend:${IMAGE_TAG:?falta IMAGE_TAG}` y el del frontend por
`image: ghcr.io/<owner>/<tu-repo>-frontend:${IMAGE_TAG:?falta IMAGE_TAG}`— (el `:?` hace que, si la
variable falta, compose se niegue con ese mensaje en vez de inventar una etiqueta: sin decir QUÉ
imagen no hay deploy, lo mismo que el `imgURL` en Render. Vale también para los comandos a mano:
`ps`, `logs` y `down` te la van a pedir). 🔴 **Los dos, por lo mismo que en §3.2**: si dejás uno con
`build:`, `pull` lo saltea sin decir nada y `up` lo construye local. Si tu runner es una máquina `arm64`, agregale a cada uno de esos servicios
`platform: linux/amd64` — sin eso, el `pull` falla con el mismo `no matching manifest` del §3.1. Y
el job le pasa el tag del commit, sin el `--build` que traías del TP6: ya no hay nada que construir.

```yaml
# .github/workflows/ci.yml → jobs: → deploy-qa: → steps:   (SÓLO en el plan C)
      - name: Deploy QA local (imagen del registry)
        shell: bash
        env:
          IMAGE_TAG: sha-${{ github.sha }}
        run: |
          docker compose -f compose.qa.yml pull
          docker compose -f compose.qa.yml up -d
```

(`deploy-prod` igual, con `compose.prod.yml`.) La integración corre con `API_BASE_URL=http://localhost:8080` y las e2e con `E2E_BASE_URL=http://localhost:3000` —los dos puertos que tu `compose.qa.yml` publica al host— (⚠️ en un runner Windows, los steps con `run:` de este TP llevan `shell: bash`, como en el TP6). El gate, la aprobación y la promoción de la misma imagen funcionan idénticos — el registry es la única nube que este fallback necesita, y es gratis.

**✅ Checkpoint (fallback):** QA local ejecuta las dos imágenes del registry (no un build local) —en
el log del deploy se ve el `pull` bajando el tag `sha-…` de esa corrida— y **la integración y las
e2e** las ejercitan por `localhost`.

## 4- Riel alternativo: Azure

| Concepto | Riel canónico (ghcr + Render) | Azure |
|---|---|---|
| Registry | ghcr.io (gratis, público) | **ACR** — ⚠️ Basic ~USD 5/mes + suscripción (advertencia 2025 vigente) |
| Build+push en CI | `docker/build-push-action@v7` (Actions) | Igual con Actions, o task `Docker@2` (buildAndPush) en Azure Pipelines |
| Login del pipeline | `docker/login-action@v4` + GITHUB_TOKEN | `Docker@2` con service connection al ACR |
| Ejecutar la imagen | Render image-backed + deploy hook `imgURL` | **Web App for Containers** (task `AzureWebAppContainer@1`) o ACI |
| Promoción de la MISMA imagen | `imgURL=sha-<commit>` en QA y PROD | El mismo tag/digest del ACR en ambos Web Apps (slots opcionales) |
| Integración | Playwright en modo pedidos (`request`) contra la URL de la **api** de QA | Igual: es un cliente HTTP, no sabe de nubes |
| e2e | Playwright/Cypress contra la URL del **front** de QA | Igual (los tests no saben de nubes) + `PublishTestResults@2` para el reporte |
| Gate de promoción | `needs: integracion` en `e2e` + `needs: e2e` en `deploy-prod` + environment approval | Stages de integración y de e2e + Approvals del environment de PROD |

**Checkpoints riel Azure:** los mismos (imagen publicada por CI → QA ejecuta la imagen → **integración verde contra la api de QA** → e2e local verde → **las dos** como gate con evidencia de freno → PROD con la misma imagen → tag de release en git).

> 📌 Si no querés el costo del ACR: **ghcr.io funciona como registry también para el riel Azure** (Web App for Containers puede leer de ghcr público) — el riel se define por dónde EJECUTÁS, no por dónde guardás.

---
---

# 📋 Trabajo Práctico 07 – Contenedores en el pipeline + integración y e2e (2026)

## ⚠️ Este es el TP que debés entregar y defender

## 🎯 Objetivo

Que tu unidad de release sea una **imagen inmutable** construida una sola vez por el pipeline, que QA y PROD ejecuten exactamente esa imagen, que una suite de **integración** le hable a su api con la base de verdad y una suite **e2e** la ejercite como un usuario real, las dos contra el entorno desplegado, y que su resultado **decida** si esa misma imagen puede promoverse a PROD.

Este trabajo se aprueba **solo si podés explicar qué hiciste, por qué lo hiciste y cómo lo resolviste**.

## 🧩 Escenario

La cadena del TP6 funciona… hasta que QA y PROD se comportaron distinto con "el mismo código": el rebuild de PROD resolvió una dependencia distinta, y el error tardó horas en explicarse. Además, el smoke test dice "responde", pero nadie verifica que la app **funcione** de punta a punta antes de promover. Y apareció un tercer problema: un cambio de una palabra que compilaba y pasaba todos los unitarios, y que la base rechazó: desde ese deploy, nadie pudo crear un registro hasta que alguien avisó. El equipo decide **tres** cambios: la release pasa a ser **una imagen** (lo que se prueba es lo que se despliega, bit a bit); la promoción pasa a exigir que **un browser real haya usado la app completa** sin romperse; y, para el tercer problema, que **una prueba le hable a la api con su base de verdad** antes de que el browser entre — así, cuando algo se rompe, la cadena no sólo avisa: dice **quién** (§2.5).

## 📋 Tareas que debés cumplir

Son **cinco**, y siguen el orden de la cadena: cerrar la cadena (1), la suite de integración (2), la suite e2e (3), el gate (4) y la release (5).

### 1. Cerrar la cadena: que los entornos ejecuten LA IMAGEN, no una reconstrucción
Del TP6 llega **media** cadena, y ésa es la novedad de este práctico. Allá tu pipeline **publica** la
imagen etiquetada con el commit… pero el proveedor **reconstruye** tu app desde el repositorio para
desplegarla, así que lo que corre no **es** la imagen que publicaste: es otra construcción del mismo
código. Este TP cierra ese hueco.

- Hacé que los **cuatro** servicios —api y front, en QA y en PROD— ejecuten las imágenes del
  registry en vez de reconstruir, y que el pipeline les diga **qué imagen** con `imgURL=…:sha-<commit>`.
  (Con un solo Dockerfile, son dos servicios: §3 🔧.)
- **Que se pueda comprobar desde afuera**: en los *Events* de cada servicio, el deploy dice
  «Triggered via Deploy Hook» y nombra la imagen `sha-<commit>` de esa corrida — y esa misma
  etiqueta está en tu package. Ésa es la evidencia de que el entorno **ejecuta** tu imagen y no
  una reconstrucción.

- El deploy lo dispara **siempre** tu pipeline (la única excepción es el deploy a mano de §3.2, que es una
  prueba y no la entrega): un deploy hecho desde el panel de Render
  no cuenta, y se nota: en *Events* ese deploy no dice «Triggered via Deploy Hook», y no hay corrida
  que lo respalde.
- `docker pull` de las dos imágenes **por su etiqueta `sha-<commit>`**, desde cualquier máquina, sin
  credenciales.

> 📌 **Por qué importa cerrarlo**: si lo que QA ejecuta no es exactamente lo que
> se verificó, las pruebas de punta a punta de este TP no prueban nada sobre lo que va a ir a
> producción. Es el cimiento del práctico, no un trámite.

### 2. Suite de integración contra la API de QA
- **Mínimo 3 pruebas** en `frontend/e2e/api.spec.js`, con el fixture `request` de Playwright
  (**sin navegador**), contra la **api de QA desplegada**, por su dirección pública (`API_BASE_URL`).
  Nada de `WebApplicationFactory` ni de un Postgres levantado en el job: la base de verdad es la de QA.
- **Las tres**: (a) **alta + verificación + borrado** —`POST`, `GET` que encuentra el dato creado,
  `DELETE`, `GET` que ya no lo encuentra—; (b) **alta inválida**: título vacío → la api contesta
  `400` **y no se creó nada**; (c) la tercera **la elegís vos**, con la regla de la Tarea 3.
- **Las que crean datos se fabrican un nombre que no se repite** —con la hora adentro, para que ninguna otra corrida use el mismo— **y borran lo que crearon**, comprobándolo — la
  misma regla que las e2e, y por el mismo motivo: QA es uno solo y lo comparten todas tus corridas.
  (La del título vacío no crea nada: lo que comprueba es justamente eso.)
- 🔴 **Sin dobles y sin navegador**: nada de mockear la api, ni de levantar una base en memoria (una
  InMemory o un SQLite es otro doble: no pasa por el driver de Postgres — §2.5). Nada de `test.skip`
  ni de pruebas que se saltean solas si QA no responde.
- **En el pipeline**: job `integracion` con `needs: deploy-qa`, en el **mismo** workflow, y su reporte
  publicado como artefacto propio, con nombre distinto del de e2e.
- **El par que da diagnóstico**: en la corrida roja de la Tarea 4, la integración queda **verde** y la
  e2e **roja**, y sabés leer qué significa eso (§2.5).

### 3. Suite e2e real
- **Mínimo 3 flujos críticos** (Playwright o Cypress) corriendo **contra QA desplegado**, y cada uno,
  leyendo el spec: **interactúa** (llena, clickea: un test que sólo navega es un smoke), **afirma
  sobre el dato que el propio test produjo** (su título único, el mensaje de error, el elemento que
  desapareció — `expect(page.locator('body'))` o un test sin `expect` no cuentan), y **limpia lo que
  crea, comprobándolo**. Entre los tres, al menos **uno de creación** (crear → verlo → borrarlo → ya
  no está) y **uno de error/validación** (dato inválido → el usuario ve el error → no se creó nada);
  el tercero, elegilo vos, y **así se elige**: el flujo que tu usuario hace **todos los días** —el
  que, si mañana no anda, te escriben—. En la app de la cátedra es que la tarea siga ahí al recargar;
  en la tuya puede ser el login, la búsqueda, o el alta del dato principal. Si dudás entre dos, elegí
  el que toque la base.
- 🔴 **Contra el sistema de verdad, sin red de contención**: nada de `page.route` con `fulfill`,
  `routeFromHAR`, `cy.intercept` con una respuesta inventada, MSW ni ningún mock del backend —si el navegador no le pega a tu
  API, no es end-to-end, y no se aprueba—; nada de `test.skip` ni de tests que se saltean solos si el
  entorno no responde (el cold start se resuelve con **timeouts**, no salteando).
- **Reporte publicado** como artefacto en cada corrida (también cuando falla), y en el resumen de la
  corrida se tiene que leer cuántos tests pasaron: `N passed` con N ≥ 3, **cero** salteados.

> 📌 **Las e2e contra `localhost` valen sólo en el fallback del §3.6**, y con tres condiciones que se
> leen en tu repo: el job corre en tu runner `self-hosted`; tus `compose.qa.yml` y `compose.prod.yml`
> usan `image:` del registry —si dicen `build:`, es un build local y no cuenta—; y el log del deploy
> muestra el `pull` del tag `sha-…` de esa corrida. En el riel de Render, `E2E_BASE_URL` es la URL
> pública de tu QA.

### 4. Las pruebas como gate de promoción
- **La cadena de `needs:` completa**: `integracion` con `needs: deploy-qa`, `e2e` con
  `needs: integracion`, y `deploy-prod` con `needs: e2e`. Sin la del medio, la integración es un
  adorno: corre, se pone roja, y la promoción sigue igual.
- **El gate tiene que poder decir que no**: `deploy-prod` con `needs: e2e`, y en ese tramo nada que
  lo desarme — ni `continue-on-error`, ni `|| true` o `exit 0` después de Playwright, ni `if: always()`
  o `!cancelled()` en `deploy-prod` (§3.4). Con cualquiera de esas, la Tarea 4 no se aprueba aunque
  la corrida roja esté.
- **Una sola corrida**: las dos suites viven en el mismo workflow que `deploy-qa` y `deploy-prod`,
  encadenadas —`integracion` con `needs: deploy-qa`, `e2e` con `needs: integracion`— y disparadas por
  el `push` a `main`. Un workflow aparte o a mano no es un gate.
- **Lo del TP6 sigue vigente**: el environment `production` tiene su required reviewer, y en la
  corrida completa que entregás se ve la pausa de revisión y quién aprobó.
- **Evidencia de una e2e roja frenando la promoción**: el **commit que rompió la app**, la corrida
  con el smoke en verde, la e2e en rojo y `deploy-prod` sin arrancar, y el reporte con
  screenshot/traza del fallo → fix → cadena completa verde → aprobación → PROD.
- 🔴 **El rojo se provoca rompiendo la APLICACIÓN, no el test.** Cambiá algo real del front —el texto
  del botón que la e2e busca, el endpoint que el front llama, el nombre del campo que el front manda
  (`title` en vez de `titulo`)— y mostrá que la e2e lo detecta. Tampoco sirve una rotura que ataje un unitario del front:
  si tu test con doble del TP5 afirma el nombre del campo, elegí otra, como el texto del botón que
  la e2e busca.
  El diff del commit que rompe toca código de la app, no `e2e/`: un test editado para que falle no
  prueba nada, y en la defensa se pregunta qué cambiaste.
- PROD recibe **la misma imagen** (`sha-<commit>`) que QA verificó: se lee en el log (el mismo
  `imgURL` en los dos deploys, el mismo commit en los dos smokes). Un deploy sin `imgURL`, o con una
  etiqueta que no sea el `sha-<commit>` de esa corrida, no es una promoción.

### 5. La release que le pone nombre a lo que está en PROD
- Al cerrar el práctico, **`v7.0.0`**: el tag y la release de GitHub sobre **el commit que está en
  PROD** —el que dice *Deployments*, no el último de `main`— (`--verify-tag`). Su imagen es la
  `sha-<commit>` de ese commit, que tu pipeline ya publicó (§3.1).
- **Del tag a la imagen, en un paso**: `git rev-list -n1 v7.0.0` te da el commit, y esa etiqueta
  está en tus dos paquetes. Es tu catálogo de versiones, ahora con binarios de verdad.

## 📄 Entregables

1. **URL del repositorio público** (formulario de la cátedra). **Es lo único que va al formulario**: las URLs de QA y PROD van en «Enlaces del TP7» (abajo).
2. **En `decisiones.md`, arriba de todo, una sección «Enlaces del TP7»** (cómo convive con la del
   TP6 lo decidís vos: lo que importa es que se encuentre) con: (a) los **dos paquetes públicos** del registry, con sus etiquetas
   `sha-…`; (b) de la corrida donde **la integración quedó VERDE y la e2e ROJA**: el commit que rompió la app, la corrida,
   y **los dos** artefactos de reporte de esa corrida (`playwright-report-integracion`, en verde, y `playwright-report-e2e`, en rojo); (c) la **corrida completa en verde** posterior,
   hasta PROD; (d) las URLs de QA y PROD, **vivas hasta la defensa**, porque en P2 se navegan en vivo. Los paquetes y las URLs viven afuera del repo, y las corridas son unas pocas entre
   muchas: si no están ahí, la corrección no las busca.
3. **`decisiones.md`** (acumulativo) explicando:
   - Build once, deploy many: qué problema del TP6 resuelve la imagen como unidad (con TU ejemplo del escenario o uno propio).
   - Tu estrategia de etiquetas (`sha-<commit>` en el registry, `v7.0.0` en git): qué garantiza cada una, por qué tu pipeline **no** publica `latest`, con qué imagen configuraste los servicios de Render y por qué no es ésa la que corre, y —si creaste servicios **nuevos** en vez de cambiarle la fuente a los viejos (§3.2)— qué hiciste con los viejos y por qué.
   - Cómo llegás de la release a la imagen: del tag `v7.0.0` al `sha-<commit>` que hay que desplegar, en un paso.
   - Cómo se comprueba, desde afuera, que un entorno **ejecuta** tu imagen y no una reconstrucción —y qué **no** alcanza a probar el smoke, que sólo pregunta si responde (concepto: §2.4).
   - Qué flujos elegiste para e2e y **qué pruebas para integración** (la tercera de cada suite la elegiste vos: ¿por qué ésas? ¿quién te escribe si se rompen?); qué NO pusiste **en cada suite** y por qué (pirámide).
   - **Qué probás en integración y qué en e2e, y por qué no es lo mismo**: el par verde/rojo que diagnosticó tu rotura, leído contra la tabla de §2.5.
   - Por qué tu integración es la **amplia** —contra la api de QA ya desplegada— y no la estrecha (§2.5 📌): qué gana y qué pierde esa elección.
   - Cómo manejás el cold start del free tier en las e2e (timeouts/retries) y qué es un test flaky.
   - Cómo lograste que **la misma imagen del front** sirva en QA y en PROD (qué se configura por variable y qué queda adentro de la imagen).
      - Problemas encontrados y cómo los resolviste. Declaración de uso de IA.
> 📌 **En este TP tampoco hay `evidencias.md`**, igual que desde el TP3 y por el mismo motivo: tu
> repositorio es público, así que quien corrige abre *Actions* y ve la corrida que construye y publica, **los dos reportes** y **la integración verde con la e2e roja frenando la promoción**; abre el *package* y ve sus tags;
> abre la release `v7.0.0` y ve su commit; abre *Deployments* y ve qué commit está desplegado en cada
> entorno —y, por el tag `sha-<commit>`, qué imagen le corresponde a cada uno—. Sacar
> capturas de eso es duplicar lo que ya está a la vista.
>
> ⚠️ **Lo único que no es público son los *Events* de Render**: viven adentro de tu panel, y no se
> pueden linkear. Ésa es la evidencia que mostrás **vos, en vivo, en la defensa**: abrís cada
> servicio y se lee «Triggered via Deploy Hook» con el `sha-<commit>` de esa corrida. Tenelos a mano.

## 🗣️ Defensa Oral Obligatoria

Se realiza en **P2**, junto con los TPs 5 a 9. Vas a mostrar tu trabajo y responder preguntas como:
- ¿Qué significa "build once, deploy many" y qué problema concreto resuelve respecto de tu TP6?
- `sha-<commit>` y el tag de git `v7.0.0`: ¿qué garantiza cada uno? ¿Cuál usás para promover y por qué? ¿Por qué tu pipeline no publica `latest`?
- En Render configuraste cada servicio con la imagen de un merge viejo: ¿es ésa la que está corriendo en QA ahora? ¿Cómo lo sabés sin entrar al panel?
- ¿Quién puede escribir en tu registry y con qué credencial? ¿Por qué no hay ningún token tuyo en el YAML?
- Mostrame cómo tu pipeline le dice a QA QUÉ imagen ejecutar. ¿Qué contestaría el hook si el `imgURL` llevara el commit corto? ¿Y cómo te enterarías si alguien desplegara a mano desde el panel?
- ¿Qué verifica tu e2e que el smoke test no puede verificar? ¿Y qué verifican tus unit tests que la e2e no debería?
- Tenés tres capas de prueba corriendo. ¿Qué ve la de **integración** que no ve ni el unitario ni la e2e? Dame un cambio de una línea que sólo ella atraparía.
- En tu corrida roja, la integración quedó verde y la e2e roja. ¿Qué te dice eso, y por qué no hizo falta leer el código para saberlo? ¿Y qué habrías visto si hubieras roto la api?
- Tu integración le habla a la api de QA ya desplegada. ¿Qué otra forma había de escribirla, qué gana y qué pierde cada una, y por qué elegiste ésta?
- ¿Por qué tus e2e corren contra QA y no contra un localhost en el runner? ¿Qué perderías?
- Mostrame la e2e roja que frenó una promoción: ¿qué se rompió, cómo lo diagnosticaste con **los dos** reportes, y qué NO pasó gracias al gate?
- ¿Qué es un test flaky, por qué es peor que no tener test, y qué hace tu configuración para no fabricarlos?
- Tus **dos suites** comparten la base de QA, y entre las dos la usan más tiempo: ¿cómo evitás que una corrida ensucie a la siguiente, y cómo te darías cuenta si pasara?
- Tu release `v7.0.0` es un tag de git: ¿sobre qué commit lo pusiste, y por qué ése y no el último de `main`? ¿Cómo llegás desde el tag a la imagen de ese commit?
- Si mañana cambiás Render por otro proveedor de contenedores, ¿qué sobrevive de este TP? (pista: casi todo — ¿por qué?)

## ✅ Evaluación

| Criterio | Peso |
|---|---|
| Configuración técnica (imagen por pipeline, entornos que EJECUTAN la imagen, **integración**, e2e, gate, release) | 25% |
| Claridad y justificación en `decisiones.md` | 25% |
| Defensa oral: comprensión y argumentación | 50% |

**Cómo se distingue un 4 de un 8**, para que sepas contra qué se mira. El 4 es lo que piden las
Tareas: está o no está. Lo que mueve la nota de ahí para arriba es el criterio:

| | 4 — mínimo (las Tareas) | 6 — correcto | 8 o más |
|---|---|---|---|
| La imagen | los cuatro servicios la **ejecutan** (dos, si tenés un solo Dockerfile — §3 🔧): el deploy de cada uno nombra la etiqueta `sha-` de la corrida | explicás por qué una reconstrucción no daría lo mismo aunque el código sea idéntico | explicás qué le pasaría a tu cadena si alguien moviera un tag a mano, y por qué tu pipeline no publica `latest` |
| e2e | 3 flujos contra QA que interactúan, afirman sobre su propio dato y limpian | aguantan el cold start sin sleeps, y explicás cómo | explicás qué NO pusiste en e2e y por qué (pirámide) |
| Integración | 3 pruebas contra la api de QA (sin navegador, sin dobles) que crean un dato propio, con un nombre que no se repite, y lo borran, y el job corre antes de la e2e | explicás por qué un unitario con doble no habría visto lo que ésta ve (§2.5) | leés el par verde/rojo de tu corrida roja y decís QUIÉN se rompió, sin abrir el código |
| El gate | una e2e roja —con la integración verde—, provocada rompiendo la **app**, frenó una promoción, y el reviewer del TP6 sigue | diagnosticaste el rojo con el reporte (traza o screenshot), no mirando el código | explicás qué clase de bug tu gate **no** ataja |
| La release | `v7.0.0` (tag y release) sobre el commit de PROD, no sobre el último de `main` | explicás por qué ése y no el último | llegás del tag a la imagen de ese commit y mostrás la etiqueta en tu package |

> ⚖️ Peso orientativo de este TP en la nota de **P2**: **20%** (la ponderación completa de los 9 TPs está en el reglamento, §5).

## ⚠️ Uso de IA

Podés usar IA (ChatGPT, Copilot, Claude), pero **deberás declarar en `decisiones.md` qué parte fue asistida por IA** y justificar cómo la verificaste. En este TP en particular: si la IA te escribió las e2e, tenés que poder explicar **qué verifica cada test, contra qué entorno corre y qué pasa si falla**. Si no podés defenderlo, **no se aprueba**.
