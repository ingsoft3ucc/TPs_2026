# ⚠️ IMPORTANTE – Guía de Práctica Sugerida

Este documento tiene **dos partes**:

1. **Guía de práctica sugerida** (primera parte): paso a paso para aprender haciendo. **NO es lo que se entrega.**
2. **El Trabajo Práctico entregable** (al final): escenario, tareas, entregables y defensa oral. **Eso es lo que se evalúa.**

## 📦 Un solo repositorio para todo el semestre

**Todos los prácticos de la materia se hacen sobre el mismo repositorio: el que creaste en el TP1.** No se crea uno nuevo por práctico, y no se arranca de cero en cada uno — cada TP agrega una capa sobre lo que ya está.

Cada TP se cierra con su **tag y su release**, y el número mayor es el número del práctico: **TP1 → `v1.0.0`, TP2 → `v2.0.0`, TP3 → `v3.0.0`**, y así hasta el TP9. Así cada entrega queda con su **estado congelado**: en la defensa se navega el punto exacto en el que cerraste cada una, y podés volver a cualquiera con `git checkout v2.0.0`.

El archivo `decisiones.md` también es **único**: no se rehace por práctico — se le agrega abajo la sección del TP nuevo.

## Sobre las herramientas en este TP

- Todo el riel canónico es **gratis en tu repo público**, y casi todo ya viene incluido: **Dependabot**, **secret scanning con push protection** (always-on en repos públicos) y **CodeQL** son features de GitHub sin costo; el monitoreo externo usa **UptimeRobot** (free: 50 monitores, chequeo cada 5 min, sin tarjeta — su plan gratuito hoy permite cualquier uso, incluso comercial: la restricción no-comercial de fines de 2024 se revirtió en 2026 y rige una *Fair Use Policy*) o **Better Stack** (free: 10 monitores, cada 3 min, sin tarjeta) como alternativa equivalente.
- Para ver errores, demoras y uso se usa **Sentry** (gratis, sin tarjeta, con tu cuenta de GitHub): §3.5.
- Riel **Azure**: los conceptos se cubren con Microsoft Defender for Cloud y Application Insights (availability tests + alertas) — parcialmente de pago; tabla en §4. Para este TP, el riel canónico es también el recomendado para quien cursa el riel Azure (la seguridad del REPO vive en GitHub en ambos rieles).

📌 Este TP trabaja **sobre tu app del semestre y su cadena completa** (TP1→TP8). Es la última pieza del mapa de la clase 1: **operar** lo que construiste — seguridad continua, observabilidad y la cultura de responder a incidentes sin buscar culpables.

🧪 **Los ejemplos de esta guía están escritos sobre la app de la cátedra** ([`ingsoft3ucc/demo-fullstack`](https://github.com/ingsoft3ucc/demo-fullstack) — .NET 8 + React/Vite + PostgreSQL, la misma de las demos en clase; cómo clonarla y levantarla, base de datos incluida: **TP2 §3.2**). Si un paso no te sale en tu app, probalo primero ahí —donde sabés que funciona— y después llevalo a la tuya. Lo que entregás, siempre, es **tu** app.

---

# Guía Paso a Paso – DevSecOps + Observabilidad + Continuous Feedback (Práctica sugerida)

## 1- Objetivos de Aprendizaje

- Entender **shift-left security**: la seguridad como verificación continua del pipeline, no como auditoría final.
- Activar las **tres capas** de seguridad automatizada: secretos (push protection), dependencias (Dependabot/SCA) y código propio (CodeQL/SAST).
- Instrumentar **observabilidad** mínima: health checks, monitoreo externo y **alertas accionables**.
- Ver **cómo se comporta tu aplicación en producción**: qué falla, qué tarda y qué se usa — y llevar
  un dato de ésos a tu tablero, como trabajo para la próxima versión.
- *(Práctica recomendada)* Medir tu equipo con las **métricas DORA** y escribir un postmortem **blameless** (sin culpables) de un incidente simulado.

## 2- Marco teórico

### 2.1 Shift-left: la seguridad se corre a la izquierda

El modelo clásico: el equipo construye durante meses y, al final, "pasa seguridad" — una auditoría tardía, cara, y enemiga de todos. El problema es estructural: cuanto más tarde se encuentra un problema, más caro es arreglarlo (¿te suena? es el MISMO argumento de la integración continua del TP4, aplicado a seguridad).

**DevSecOps / shift-left** mueve la seguridad "a la izquierda" del diagrama — a cada commit, a cada PR, al pipeline: verificaciones automáticas, continuas y baratas, que atrapan los problemas cuando nacen. La seguridad deja de ser un equipo que dice "no" al final y pasa a ser **una verificación más del pipeline** — igual que los tests, igual que el quality gate del TP5.

Vos ya venís haciendo shift-left sin llamarlo así — mirá el **arco de los secretos** de la materia: TP2, los secretos salen del código (`.env` gitignoreado); TP4, suben cifrados a la plataforma (secrets + enmascarado); TP6, se acotan por entorno (environment secrets); TP8, las claves de tus proveedores viajan por el entorno y **nunca se escriben en el código** de tu infraestructura. Hoy cae la última pieza: el escáner que **bloquea el push** de un secreto antes de que llegue al repo. Cinco capas de defensa, construidas de a una por TP.

### 2.2 Las tres capas: secretos, dependencias y código propio

**Capa 1 — Secretos: secret scanning + push protection.** GitHub escanea tu repo (y cada push) buscando **patrones de credenciales** de cientos de proveedores — tokens de GitHub, claves de AWS, de Stripe… En repos públicos está **activo y gratis siempre**. Y la parte central es **push protection**: el push que contiene un secreto reconocido **se bloquea antes de entrar**: no es una alerta que llega después. (Detalle honesto: detecta patrones *conocidos*; tu password inventado "hunter2" no matchea ningún patrón — por eso las capas anteriores del arco siguen importando.)

**Capa 2 — Dependencias: SCA y Dependabot.** Un dato de tu app: la mayor parte del código que ejecuta **no lo escribiste vos** — son tus dependencias (y las dependencias de tus dependencias). Esa es tu mayor superficie de ataque, y tiene nombre de industria: *software supply chain*. El **Software Composition Analysis (SCA)** compara tu árbol de dependencias contra bases de vulnerabilidades conocidas (**CVEs** — *Common Vulnerabilities and Exposures*, el catálogo público donde cada vulnerabilidad recibe su identificador). En GitHub eso es **Dependabot**: (a) **alertas** cuando una dependencia tuya tiene una vulnerabilidad publicada, y (b) **PRs automáticos** que proponen la actualización — que tu CI verifica como a cualquier PR. Pensalo un segundo: todo tu semestre (tests + coverage + gate + e2e) es lo que te permite mergear un update de dependencia **con confianza** en minutos. Sin esa red, los updates dan miedo y se postergan — y las apps quedan vulnerables. (Detalle honesto #2: las vulnerabilidades en dependencias **transitivas** se alertan cuando aparecen en el grafo de dependencias — con npm, siempre: el lockfile las lista todas —, pero el PR automático es para las directas, con la excepción de npm, donde a veces puede resolver una transitiva actualizando a la dependencia madre.)

**Capa 3 — Tu código: SAST.** El **CodeQL** que la Clase 5 te dejó anunciado como el cuarto guardián: análisis estático orientado a **seguridad** — inyección SQL, XSS, secretos hardcodeados, path traversal — tratando tu código como una base de datos consultable. Corre en el pipeline (default setup, gratis en públicos) y publica en la pestaña **Security**. Para ponerle vocabulario al "qué busca": el **OWASP Top 10** — la lista canónica de las categorías de vulnerabilidades web más críticas (inyección, fallas de autenticación, fallas en la cadena de suministro de software — la ex "componentes vulnerables", rebautizada en la edición 2025 con el mismo nombre del problema que acabás de leer…) — es el mapa mental mínimo que un ingeniero debe reconocer. Y una cuarta mini-capa que a veces muerde: **licencias OSS** — cada dependencia trae una licencia, y no todas permiten cualquier uso (¿te suena BSL, del TP8?).

### 2.3 Observabilidad: ver lo que pasa adentro (desde afuera)

Tu app está en producción. Pregunta: **¿cómo sabés que está viva — y si no lo está, por qué?** Que ningún usuario se haya quejado no prueba que esté viva.

- **Monitoring** responde preguntas *conocidas*: ¿responde el health? ¿cuánta CPU usa? Son tus **preguntas predefinidas** con umbrales.
- **Observabilidad** es la propiedad de un sistema de dejarte responder preguntas **que no habías anticipado** ("¿por qué los usuarios de este proveedor ven errores desde ayer?") a partir de las señales que emite. Los **tres pilares** de esas señales:
  1. **Logs** — eventos discretos con contexto ("falló el pago X con error Y"). Los tenés desde siempre (`docker compose logs`, los logs de Render).
  2. **Métricas** — números agregados en el tiempo (requests/seg, % de errores, latencia). Baratas de almacenar, ideales para alertar.
  3. **Trazas** — el recorrido de UNA request a través de los servicios (front → API → BD: dónde se gastó el tiempo). Es el más útil cuando hay muchos servicios.

En este TP instrumentamos lo **mínimo**: el **health check** (tu `/health` — el que tu smoke test consume desde el TP6 — hoy suma un segundo consumidor: es el contrato público de "estoy viva" para cualquiera que necesite preguntarlo, y ahora también lo consume un monitor externo) + **monitoreo externo con alertas** (un servicio ajeno a tu infra le pregunta a tu PROD "¿estás viva?" cada N minutos y te avisa cuando no — externo, porque si tu infra se cae, tu propio monitoreo se cae con ella).

Sobre las **alertas**, la regla: una alerta debe ser **accionable** — despertar a alguien solo si ese alguien puede y debe hacer algo YA. Alertas por todo = fatiga de alertas = nadie mira nada = la alerta importante pasa sin que nadie la lea (¿te suena la lógica? es el "main rojo tolerado" del TP4, versión operaciones).

### 2.4 El círculo se cierra: DORA como tablero + blameless

**Las métricas DORA** de la clase 1 vuelven, pero ahora al revés: aquel día eran datos de la industria; hoy son **tu tablero**. Con lo que construiste podés medirte de verdad: *deployment frequency* (tus releases), *lead time* (de commit a PROD — tu cadena lo hace medible), *change failure rate* (¿cuántos deploys salieron mal?) y *failed deployment recovery time* (tu rollback del TP6, cronometrado). Medirse no es burocracia: es el feedback loop del equipo sobre sí mismo — la fase MONITOR del ciclo alimentando el próximo PLAN, como prometía el diagrama de la clase 1.

**Y la pieza cultural final: el postmortem blameless.** Cuando algo se rompe en producción (va a pasar), el equipo profesional escribe un **postmortem**: qué pasó (timeline), qué impacto tuvo, cuál fue la **causa raíz**, qué acciones lo previenen, y qué se aprendió. La palabra clave es **blameless** — sin culpables — y no es buenismo: es **diseño de incentivos**. Si el postmortem busca culpables, la próxima vez la gente esconde información, y la organización pierde exactamente lo que más necesita: la verdad sobre cómo falla su sistema. La pregunta madura nunca es "¿quién se equivocó?" sino "**¿qué hizo posible que este error llegara a producción?**" — porque a la persona la podés reemplazar y el sistema va a volver a fallar igual. (¿Te suena la lógica de fondo? Es la de toda la materia: los procesos importantes no dependen de la memoria ni de la buena voluntad — se cablean en el sistema.)

## 3- Desarrollo de la guía (riel GitHub — todo gratis en tu repo público)

> Trabajás sobre tu repo del semestre con la cadena completa viva. La mayoría de los pasos son configuración corta + **análisis con criterio** — este TP se gana leyendo y decidiendo, más que tipeando.

### 3.1 Dependabot: tus dependencias, vigiladas

Creá `.github/dependabot.yml`:

```yaml
version: 2
updates:
  - package-ecosystem: "nuget"            # las dependencias del backend
    directory: "/backend"
    schedule:
      interval: "weekly"

  - package-ecosystem: "npm"              # las del frontend
    directory: "/frontend"
    schedule:
      interval: "weekly"

  - package-ecosystem: "github-actions"   # ¡tus actions también son dependencias!
    directory: "/"
    schedule:
      interval: "weekly"
```

**Commiteá y pusheá el archivo a `main`** (por PR, como todo — Dependabot lee la configuración desde la default branch del remoto: un `dependabot.yml` local no hace nada). Entrá también a *Settings → Advanced Security* y prendé dos cosas: **Dependabot alerts** y **Dependabot security updates**. Con la segunda, cuando se publica una falla en una dependencia tuya, Dependabot abre solo el pull request que la arregla.

> 💡 Detalle .NET: el `directory` debe contener el `.sln` (o los `.csproj`) **directamente** — el escaneo no es recursivo, aunque el `.sln` sí arrastra a sus proyectos. Si tu solución vive en una subcarpeta, apuntá el `directory` ahí.

A los pocos minutos —a veces tarda horas— vas a tener PRs de actualización (medido el 2026-10-08: cinco, a los minutos del merge). Ésos los abre el `dependabot.yml`: proponen las versiones nuevas y llegan cada semana, aunque no haya ninguna falla. El que vas a mergear seguramente sea uno de ésos. Elegí **uno**, y analizalo antes de mergear: ¿qué actualiza y de qué versión a cuál (SemVer del TP6: ¿patch, minor, major?)? ¿El changelog menciona breaking changes? ¿Tu CI está verde sobre ese PR? — ese verde es TODO tu semestre (tests, coverage, gate, e2e) diciéndote "podés". Mergealo, y anotá en `decisiones.md` qué actualizaba y por qué te animaste a mergearlo.

> 🔧 **¿El botón de merge sale gris y dice que la rama está desactualizada?** Pasa cuando mergeaste
> otra cosa después de que Dependabot abrió el suyo (medido el 2026-10-08). Escribí un comentario en
> ese pull request que diga `@dependabot rebase`: Dependabot lo pone al día, tu pipeline vuelve a
> correr, y el botón se habilita.

**✅ Checkpoint:** Dependabot activo en los 3 ecosistemas, con las alertas y las actualizaciones de seguridad prendidas, y al menos 1 PR de actualización **analizado y mergeado**.

### 3.2 Push protection: el secreto que no llegó al repo

En tu repo público, secret scanning y push protection vienen **activos por default** — verificalo en *Settings → Advanced Security* (sección *Secret Protection*).

> ➕ **Lo que sigue es práctica recomendada — no se entrega.** De acá salen preguntas de la defensa:
> hacela igual. Lo que se entrega es la protección verificada activa (tarea 3).

Ahora, la demo controlada: vas a intentar filtrar un secreto de verdad y ver el bloqueo.

1. Generá un **PAT de GitHub SIN NINGÚN scope** (Settings → Developer settings → Personal access tokens → classic, sin tildar nada, expiración mínima). Sin scopes no sirve para nada — es un secreto "real" pero inofensivo, perfecto para la demo.
2. Pegalo en un archivo (`config-prueba.txt`) — ⚠️ **TU token real**, no un `ghp_` inventado: los tokens de GitHub llevan un checksum en sus últimos caracteres, y un string con checksum invento **no matchea** el patrón (así el escáner elimina falsos positivos — dato que te sirve para la reflexión (b) de abajo):

```bash
git checkout -b demo-secreto
echo "token = <pegá acá TU PAT sin scopes>" > config-prueba.txt
# Windows PowerShell 5.1: el echo escribe UTF-16 y el patrón puede NO detectarse —
#   usá Git Bash, el editor, o: Set-Content config-prueba.txt "token = <TU PAT>" -Encoding utf8
git add config-prueba.txt && git commit -m "Config de prueba (demo push protection)"
git push origin demo-secreto
# → ¡RECHAZADO! GitHub bloquea el push (GH013): "Push cannot contain secrets"
#   con el tipo de secreto, archivo y línea. El secreto NUNCA llegó al repo.
```

3. Mirá el bloqueo, y después limpiá **de verdad**: `git checkout main && git branch -D demo-secreto` — el commit descartable muere con su rama. (No intentes "borrar el archivo y amendear": el amend usa el index y el secreto **queda adentro del commit** — el próximo push rebota igual y no vas a entender por qué.) Y **revocá el PAT** aunque nunca haya viajado: higiene.

Dos preguntas para pensar, que vuelven en la defensa: (a) GitHub te ofrece un link para *bypassear* el bloqueo ("es un falso positivo / lo arreglo después") — ¿por qué usarlo "para ir rápido" destruye el valor del guardián? (b) ¿qué NO detecta este escáner? (tu password inventado, un secreto en un formato no reconocido) — ¿qué capas del arco cubren ese hueco?

**✅ Checkpoint:** push protection verificada activa; y, como práctica, viste el push **bloqueado** y el PAT de prueba quedó revocado.

### 3.3 CodeQL: el análisis de tu código

> 🎯 **Éste es el cuarto guardián de `main`, el que la Clase 5 te dejó anunciado.** Allá quedaron
> tres —el Pull Request obligatorio (TP1), el build verde (TP4) y tu umbral de cobertura (TP5)— y se
> dijo que faltaba el que **lee tu código sin ejecutarlo**, para encontrar lo que a nadie se le
> ocurrió testear. Es éste. Llega recién acá, y no en el TP5, porque un hallazgo de análisis
> estático necesita la teoría de seguridad de este práctico. Sin ella se queda en *«la herramienta
> dijo algo»*. Lo que más se aprende no es activarlo: es **leer lo que encuentra y decidir si es
> explotable en TU app**.

*Settings → Advanced Security → Code Security → CodeQL analysis → Set up → Default* (dos clicks; detecta C#, JS/TS y tus workflows de GitHub Actions, y corre en cada PR y push). Dale unos minutos a su primer análisis y andá a **Security → Code scanning**.

> ➕ **Lo que sigue es práctica recomendada — no se entrega.** Lo que se entrega es CodeQL activo con
> su primer análisis terminado (tarea 2).

Para practicar:

- Si hay findings: elegí **al menos 2** y analizalos en serio: ¿qué detectó? ¿en qué categoría OWASP cae? ¿es explotable en TU app o es un falso positivo razonado? ¿qué hiciste (fix / descarte justificado)?
- Si tu app está limpia (pasa — es chica y ya la venís puliendo): **provocá un finding didáctico** en una rama — p. ej. construí una respuesta concatenando input del usuario sin sanitizar, o un secreto hardcodeado de mentira — mirá a CodeQL cazarlo en el PR, analizalo… y cerrá la rama sin mergear (el finding y su análisis son la evidencia; el código malo no entra).

> 📌 **Un hallazgo que vas a ver casi seguro: «Workflow does not contain permissions».** Medido el
> 2026-10-08 sobre la aplicación de la cátedra: un trabajo del pipeline no limita lo que puede hacer
> la corrida, así que corre con los permisos que el repositorio da por defecto. En un repositorio
> nuevo de tu cuenta personal esos permisos son de sólo lectura; en uno viejo, en uno de una
> organización, o si alguien los cambió, pueden incluir escribir. El arreglo es declarar, en ese
> trabajo, sólo lo que usa. Si sólo baja el código, alcanza con leer:
>
> ```yaml
>     permissions:
>       contents: read
> ```
>
> Ojo: lo que no nombrás queda prohibido. Si ese trabajo crea una release o sube una imagen,
> agregale lo que necesita (`contents: write`, `packages: write`) y mirá que la corrida siga en verde.

**✅ Checkpoint:** CodeQL corriendo en cada PR; y, como práctica, ≥2 findings (reales o provocados) analizados con categoría OWASP y decisión.

### 3.4 Observabilidad mínima: monitor externo + alerta real

1. Creá tu cuenta en **uptimerobot.com** (free, sin tarjeta) — o Better Stack, equivalente.
2. **Monitor HTTP(s)** apuntando al **`/health` de tu PROD** (`https://miapp-api-prod.onrender.com/health`), intervalo 5 minutos, alerta por email (tu contacto por default).

   > 🔴 **Antes de crearlo, comprobá que tu endpoint le conteste al monitor.** Medido el 2026-10-08:
   > el plan gratuito de UptimeRobot pregunta con un pedido de tipo **HEAD** (un GET que pide sólo la
   > respuesta, sin el cuerpo; cambiarlo es de pago), y hay frameworks cuyo endpoint sólo contesta
   > GET. La aplicación de la cátedra era uno: el monitor marcó **Down** con producción viva.
   >
   > ```bash
   > curl -I https://<tu-api-de-prod>/health      # tiene que decir 200, no 405
   > ```
   >
   > Si dice 405, hacé que tu endpoint acepte también HEAD y llevalo a producción por tu pipeline. En
   > ASP.NET es cambiar el `MapGet` por:
   >
   > ```csharp
   > app.MapMethods("/health", new[] { "GET", "HEAD" }, () => Results.Ok(new { status = "ok" }));
   > ```
   >
   > En Express, Spring o Django suele andar sin tocar nada; en FastAPI no: agregá `"HEAD"` a los
   > métodos de la ruta. Comprobalo igual.

3. **La letra chica, que acá tiene consecuencia sobre tu producción.** Tu servicio gratuito de
   Render se duerme a los 15 minutos sin tráfico, y un monitor que le pega cada 5 minutos **lo
   mantiene despierto para siempre**. Un servicio despierto todo el mes consume entre 720 y 744 de
   las **750 horas** que comparten **todos** tus servicios gratuitos (TP8 §2.6). Con eso solo ya casi
   no queda margen para los demás, y si te pasás **Render suspende todos, producción incluida**.
   🔴 **La regla: el monitor se prende para hacer el ejercicio y se PAUSA al terminar** (*Pause* en
   el panel de UptimeRobot; el historial no se pierde). Monitoreá **sólo PROD**, y prendelo de
   nuevo el día de la defensa si querés mostrarlo corriendo.
4. **La alerta de verdad**: provocá una caída controlada — en Render, *Suspend* del servicio de PROD (o `docker stop` en el fallback local) — esperá la alerta **DOWN** en tu email, reanudá el servicio, y esperá la de **UP**. Tené paciencia con la segunda: medido el 2026-10-08, el DOWN llegó a los 5 minutos de suspender, y el UP puede tardar **más de 20 minutos** después de reanudar: Render deja el servicio dormido y le contesta 503 al monitor mientras arranca. Para acelerarlo, abrí vos el `/health` en el navegador después de reanudar: eso lo despierta. Las dos quedan en el historial del monitor, con su hora: acabás de medir tu "tiempo de detección" — ¿cuánto tardaste en enterarte por el monitor vs cuánto habrías tardado si te avisaba un usuario?

**✅ Checkpoint:** monitor sobre el `/health` de PROD, el par DOWN/UP en su historial, y el monitor **pausado** al terminar.

### 3.5 Cómo se comporta tu aplicación: errores, demoras y uso

El monitor de §3.4 contesta una sola pregunta: ¿está viva? Que conteste no quiere decir que ande
bien. Para saber **qué falla, qué tarda y qué se usa**, la aplicación tiene que contarlo ella misma.
Eso se hace con una librería que se agrega al código y manda esos datos a un servicio. Acá usamos
**Sentry**.

> 📌 **Lo gratis, verificado en su página el 2026-10-08:** un usuario, **5.000 errores** y **5
> millones de mediciones de tiempo** por mes, con 30 días de historial. Sin tarjeta. Para una
> aplicación de cursada sobra.

**1 · La cuenta y los dos proyectos.** Entrá a `sentry.io` con tu cuenta de GitHub. Creá **dos
proyectos**: uno para tu back, eligiendo la plataforma de tu stack (en el ejemplo, *ASP.NET Core*), y
otro para tu front (*React*). Cada proyecto te da una dirección propia, el **DSN**: es adónde tu
aplicación manda los datos.

**2 · El back.** Una librería y un bloque de configuración:

```bash
cd backend
dotnet add DemoApi/DemoApi.csproj package Sentry.AspNetCore
```

```csharp
// Program.cs, antes de builder.Build()
builder.WebHost.UseSentry(o =>
{
    o.Dsn = builder.Configuration["SENTRY_DSN"] ?? "";   // llega por el entorno
    o.TracesSampleRate = 1.0;                            // medir todos los pedidos
});
```

El DSN del back **no se escribe en el código**: va como variable de entorno. En Render, en tu
servicio de **producción**: *Environment → Add Environment Variable* → `SENTRY_DSN`. Medido: si la
variable no está —en tu máquina, en tus tests, en QA— la librería queda apagada y la aplicación anda
igual.

**3 · El front.** Lo mismo, del otro lado:

```bash
cd frontend
npm install @sentry/react
```

```js
// src/main.jsx, antes de renderizar
import * as Sentry from '@sentry/react'

Sentry.init({
  dsn: '<el DSN de TU proyecto de front>',
  environment: window.location.hostname,     // de qué sitio vino: QA, producción o localhost
  integrations: [Sentry.browserTracingIntegration()],
  tracesSampleRate: 1.0,
})
```

> 🔴 **Acá el DSN sí va escrito en el código, y no es un descuido.** El DSN de un front no es un
> secreto: sólo sirve para **enviar** datos, no para leerlos, y cualquier aplicación web lo expone
> porque viaja en el JavaScript que baja el navegador. Es la diferencia con una clave de API, que sí
> permite leer o modificar. Tu push protection no lo va a frenar, y está bien que no lo frene.

**4 · A producción, como siempre.** Rama, pull request, tu pipeline en verde, merge, y la
promoción a producción con su aprobación. Es un cambio de código más.

**5 · Usala, y rompela una vez.** Abrí tu aplicación de producción y usala unos minutos: listá,
agregá, borrá. Y para ver un error llegar, agregá **temporalmente** un endpoint que falle:

```csharp
app.MapGet("/api/falla", string () => throw new InvalidOperationException("Falla de prueba del TP9"));
```

Llamalo un par de veces desde el navegador. **Sacalo cuando termines el práctico.**

**6 · Mirá las tres cosas.** En Sentry:

| Lo que querés saber | Dónde | Qué vas a ver |
|---|---|---|
| **Qué falla** | *Issues* | cada error distinto, con el archivo y la línea, cuántas veces pasó y a cuántos usuarios |
| **Qué tarda** | *Dashboards → Backend Overview*, tabla *Transactions*, columnas *P50* y *P95* | cuánto tarda cada operación de tu api; y en *Queries by Time Spent*, tus consultas a la base |
| **Qué se usa** | la misma tabla, columna *TPM* (pedidos por minuto); y *Frontend Overview* para las visitas | qué operaciones se piden mucho y cuáles casi nunca |

*P50* es lo que tarda la mitad de los pedidos; *P95*, lo que tardan casi todos: sólo el 5 % más
lento lo supera. Para «tu operación más lenta», mirá el P95.

Esto es lo que midió la cátedra el 2026-10-08 sobre su aplicación, después de usarla un minuto:

```
GET  /api/tareas          14 pedidos   p95 1,4 s   ← la más usada, y la más lenta
POST /api/tareas           4 pedidos   p95 1,1 s
DELETE /api/tareas/{id}    1 pedido    0,3 s
GET  /api/falla            2 pedidos   → 1 error en Issues, visto por 2 usuarios
```

El 1,4 segundo no es un error de medición: es el primer pedido después de que la base gratuita se
durmió. Es justo la clase de cosa que nadie ve hasta que la mide.

**7 · Y ahora cerrá el círculo.** Elegí **un** dato de los que viste —tu operación más lenta, tu
error, o la parte que nadie usa— y abrí con él una tarjeta en tu tablero del TP3. Con el número
adentro. Eso es lo que la clase 1 llamaba feedback: lo que pasa en producción decide qué se hace
después.

> 🔧 **¿Tu stack es otro?** Sentry tiene librería para Node, Python, Java, Go, PHP y los frameworks
> de front habituales. Al crear el proyecto elegís la plataforma y te muestra el bloque exacto. Lo
> que se evalúa es que veas las tres cosas, no con qué librería.

**✅ Checkpoint:** tu aplicación de producción manda datos a Sentry; viste un error, tu operación
más lenta y tu operación más usada; y hay una tarjeta en tu tablero nacida de uno de esos datos.

### 3.6 DORA: medite a vos mismo

> ➕ **Práctica recomendada — no se entrega.** De acá salen preguntas de la defensa: hacela igual.

Con tu repo y tu historial de deploys, calculá **tus** cuatro métricas del semestre (aproximadas, a mano — el punto es el ejercicio, no la precisión):

- **Deployment frequency**: ¿cuántos deploys a PROD hiciste? (tu historial de Deployments / releases)
- **Lead time for changes**: elegí 3 cambios y medí commit → PROD (los timestamps están todos en tu repo).
- **Change failure rate**: ¿cuántos deploys causaron un problema? (contá el gate frenando, el rollback del TP6…)
- **Failed deployment recovery time**: tu rollback, ¿cuánto tardó de punta a punta? En el TP6 lo hiciste reconstruyendo; ahora, con el TP7 cerrado, la vuelta es bajar una imagen que ya existe. (¿Nunca lo ejecutaste? Hacelo ahora — el `imgURL` de la versión anterior — y cronometralo; o usá la resolución de tu incidente simulado de §3.7 como dato.)

Anotate los números y pensalos: ¿dónde estás respecto de los clusters de la clase 1? ¿cuál métrica mejorarías primero y QUÉ pieza de tu cadena la mueve?

**✅ Checkpoint (práctica):** tus 4 números con su método de cálculo y la reflexión.

### 3.7 El postmortem blameless (incidente simulado)

> ➕ **Práctica recomendada — no se entrega.** De acá salen preguntas de la defensa: hacela igual.

Simulá un incidente **realista** end-to-end — ideas: el Suspend de §3.4 "fue un compañero probando permisos"; o desplegá a PROD una versión con la connection string rota (saltándote QA con un deploy manual "de emergencia", un error común); o "borraron" el contenedor de la BD local (drift del TP8). Vivilo de verdad: detección (¿la alerta? ¿un usuario?), diagnóstico (logs, historial de deploys), resolución (rollback / repair / apply).

Después escribí **`postmortem.md`** (raíz del repo) con el formato profesional:

```markdown
# Postmortem: [título del incidente] — [fecha]

## Resumen        (2-3 líneas: qué pasó y cuánto duró)
## Impacto        (qué/quiénes se vieron afectados, por cuánto tiempo)
## Timeline       (hora a hora: detección, diagnóstico, mitigación, resolución)
## Causa raíz     (el QUÉ del sistema — no el QUIÉN: qué hizo posible el fallo)
## Acciones       (concretas, con dueño: qué cambia para que no se repita)
## Aprendizajes   (qué sabemos ahora que no sabíamos antes)
```

La vara del blameless: si tu causa raíz nombra a una persona ("Juan se equivocó"), no llegaste a la causa raíz — seguí preguntando **por qué el sistema dejó que ese error llegara** (¿faltó un gate? ¿una protección? ¿una alerta?). Las acciones deben ser **cambios al sistema**, no promesas de "tener más cuidado".

**✅ Checkpoint (práctica):** `postmortem.md` completo, con causa raíz de sistema y acciones concretas — al menos una implementable con lo que ya sabés de la materia.

## 4- Riel alternativo: Azure

| Concepto | Riel canónico (GitHub — gratis) | Azure |
|---|---|---|
| Secretos | Secret scanning + push protection (always-on en públicos) | GitHub igual (el repo vive ahí) · GH Advanced Security for Azure DevOps si el repo está en ADO (de pago) |
| Dependencias | Dependabot (alertas + PRs) | Igual en GitHub · Defender for Cloud (recomendaciones, de pago) |
| SAST | CodeQL default setup | Igual en GitHub · Defender/ADO: extensiones de pago |
| Monitoreo externo | UptimeRobot / Better Stack → /health | **Application Insights**: availability tests (los *standard tests* — los URL ping clásicos se retiran el 30/09/2026 — se cobran por ejecución) + alertas |
| Métricas/logs | Logs del proveedor + monitor externo | App Insights: telemetría completa (requests, dependencias, trazas) — el punto fuerte del riel |
| Errores, demoras y uso | Sentry (§3.5) | Application Insights: errores, tiempos por operación y uso |
| Alertas | Email del monitor | Azure Monitor alerts (action groups: email/SMS) |
| Postmortem/DORA | Manual (este TP) | Igual (no depende del proveedor) |

> 📌 Si usás Application Insights en lugar de Sentry, en la defensa mostrás lo mismo ahí: un error, tu
> operación más lenta y la más usada.

**Checkpoints riel Azure:** los mismos — con la aclaración de que las 3 capas de seguridad del repo se hacen en GitHub igual (tu repo vive ahí), el monitoreo puede ser App Insights availability test + alerta en lugar de UptimeRobot, y los errores, demoras y uso pueden verse en App Insights en lugar de Sentry.

> 📌 La cultura (blameless, DORA, alertas accionables) es idéntica en todos los rieles, y no depende del proveedor que uses.

---
---

# 📋 Trabajo Práctico 09 – DevSecOps + Observabilidad + Continuous Feedback (2026)

## ⚠️ Este es el TP que debés entregar y defender

## 🎯 Objetivo

Endurecer y observar tu sistema: las tres protecciones automáticas de GitHub activas en tu
repositorio, un monitor externo que te avisa cuando producción se cae, y tu aplicación contando
qué falla, qué tarda y qué se usa.

🔴 **Lo que se mide es un recorrido en vivo:** en la defensa abrís tu repositorio, tu monitor y tu Sentry,
mostrás cada pieza funcionando, y explicás qué mira y qué no.

Este trabajo se aprueba **solo si podés explicar qué hiciste, por qué lo hiciste y cómo lo resolviste**.

## 🧩 Escenario

Tu app funciona y se entrega sola — pero dos sustos de la misma semana cambiaron la conversación del equipo: un compañero casi pushea un token real ("lo salvó el ojo de otro en la review… de casualidad"), y PROD estuvo caída **cuarenta minutos un sábado** sin que nadie lo supiera hasta que un usuario avisó por Instagram. La conclusión del equipo: construir y entregar ya lo saben hacer; ahora falta **operar**. Que los secretos no dependan de ojos atentos, que las dependencias no queden viejas sin que nadie lo note, y que la caída la anuncie una alerta y no un usuario.

## 📋 Tareas que debés cumplir

### 1. Dependencias vigiladas
- **Dependabot activo** en los ecosistemas de tu app (backend + frontend + actions): el
  `dependabot.yml` en `main`, y las **alertas** y las **actualizaciones de seguridad** de Dependabot
  prendidas en *Settings → Advanced Security* (§3.1).
- **Al menos un PR de actualización mergeado**, con tu pipeline en verde.

### 2. Tu código, escaneado
- **CodeQL activo** (configuración por defecto), con su primer análisis terminado y visible en
  *Security → Code scanning*.

### 3. Los secretos, con guardia
- **Push protection verificada activa** en *Settings → Advanced Security*.

### 4. Producción observada
- **Monitor externo** sobre el `/health` de tu producción.
- En su historial, **una caída provocada por vos**: el aviso de DOWN y el de UP.
- 🔴 **El monitor pausado al terminar** (§3.4): despierto todo el mes se come tu cuota de Render.

### 5. Tu aplicación, observada
- **Sentry en el back y en el front** de tu producción (§3.5), subido por un pull request. En el
  riel Azure puede ser Application Insights (§4).
- En **Sentry**: **al menos un error**, y la tabla con **tu uso real**.
- **Una tarjeta en tu tablero** del TP3 nacida de un dato que viste: tu operación más lenta, tu
  error, o lo que nadie usa.

### 6. Release `v9.0.0`
- El cierre del práctico etiquetado con **`v9.0.0`** y publicado como Release de GitHub, igual que
  en los prácticos anteriores.

### 🧪 Práctica recomendada (no se entrega nada)

No dejan ningún entregable, pero de ahí salen las preguntas de la defensa: hacelas.

- **El secreto bloqueado** (§3.2): intentá pushear un token de prueba y mirá cómo GitHub lo frena.
- **Dos hallazgos de CodeQL, analizados** (§3.3): qué detectó, y si es explotable en tu app.
- **Tus cuatro métricas DORA** (§3.6).
- **El postmortem sin culpables** (§3.7), sobre un incidente simulado: por ejemplo, la caída de la tarea 4.

## 📄 Entregables

1. **URL del repositorio público** (formulario de la cátedra). Es lo único que va al formulario.
2. **En el repositorio**: el `dependabot.yml`, el PR de Dependabot mergeado, Sentry en tu código, la
   tarjeta en tu tablero y la release `v9.0.0`.
3. **En `decisiones.md`** (acumulativo), una sección corta del TP9 con tres cosas:
   - qué actualizaba el PR de Dependabot que mergeaste, y por qué te animaste a mergearlo;
   - cuánto tardó tu monitor en avisarte de la caída, y cuál es tu operación más usada y cuál la
     más lenta;
   - qué problemas tuviste, y la declaración de uso de IA.

> ⏳ **Lo que se muestra en la defensa vence a los 30 días.** Sentry gratuito guarda 30 días: el
> error, las demoras y el uso. Y el panel gratuito de UptimeRobot muestra los últimos 30 días. Si
> pasó más de un mes, antes de la defensa usá tu aplicación de producción unos minutos y provocá un
> error de nuevo. Para eso, volvé a subir el endpoint de prueba de §3.5 por un pull request, y
> sacalo después. Lo mismo con la caída del monitor: prendelo, repetí la caída y volvé a pausarlo.

> 📌 **No hay `evidencias.md` ni capturas.** Lo que se evalúa se ve navegando: tu pestaña *Security*,
> tus pull requests, y el historial de tu monitor.

## 🗣️ Defensa Oral Obligatoria

Se realiza en **P2**, junto con los TPs 5 a 8.

🔴 **Lo primero, en vivo: el recorrido.** Con tu repositorio, tu monitor y tu Sentry abiertos, mostrás:

1. en *Security*: las alertas de Dependabot, el análisis de CodeQL y la protección de secretos, activos;
2. tu **PR de Dependabot mergeado**, con su corrida en verde;
3. tu **monitor**, con el DOWN y el UP en su historial;
4. tu **Sentry** (o tu Application Insights, en el riel Azure): un error, tu operación más lenta y
   la más usada, y la tarjeta que abriste.

Y de cada pieza, preguntas como:

- ¿Qué es shift-left, y a qué práctico anterior se parece el argumento?
- ¿Por qué tus dependencias son la mayor parte del código que ejecutás? ¿Qué hace Dependabot y qué no?
- ¿Por qué pudiste mergear ese PR con confianza? ¿Qué pieza del semestre te la dio?
- ¿Qué pasa cuando alguien intenta pushear un secreto? ¿Qué secreto **no** detectaría el escáner?
- ¿Qué lee CodeQL, y en qué se diferencia de tus tests?
- ¿Por qué el monitor es **externo**? ¿Cuánto tardó en avisarte, y cómo te habrías enterado sin él?
- ¿Qué hace que una alerta sirva? ¿Por qué pausás el monitor al terminar?
- Tu monitor dice que estás viva. ¿Qué te dice Sentry que el monitor no puede decirte?
- ¿Cuál es tu operación más lenta, y por qué? ¿Qué harías con ese dato?
- El DSN de tu front está escrito en el código de un repositorio público. ¿Por qué no es un problema?
- Si algo se rompe en producción, ¿qué pregunta se hace un equipo en vez de «quién fue»?

## ✅ Evaluación

| Criterio | Peso |
|---|---|
| Configuración técnica (las protecciones activas, el PR mergeado, el monitor, Sentry, la release, `decisiones.md`) | 25% |
| Defensa oral: **el recorrido en vivo**, y comprensión | 75% |

> 📌 **En este TP los pesos son distintos de la regla general del README** (25 % / 25 % / 50 %), igual
> que en el TP8: `decisiones.md` son tres renglones y entra en lo técnico.

| | **Suficiente (4-5)** | **Bien (6-7)** | **Muy bien (8-10)** |
|---|---|---|---|
| **El recorrido** | Mostrás las cinco piezas funcionando (Dependabot, CodeQL, push protection, el monitor y Sentry) | Además decís qué mira cada una | Además decís qué **no** mira cada una, y qué otra pieza cubre ese hueco |
| **Lo que entendiste** | Contestás qué es shift-left, qué es un monitor, qué es blameless | Lo contestás sobre **tu** repositorio | Sostenés la repregunta: qué harías distinto en un proyecto de verdad |

> ⚖️ Peso orientativo de este TP en la nota de **P2**: **15%** (la ponderación completa de los 9 TPs está en el reglamento, §5).

## ⚠️ Uso de IA

Podés usar IA (ChatGPT, Copilot, Claude), pero **deberás declarar en `decisiones.md` qué parte fue asistida por IA** y justificar cómo la verificaste. En este TP en particular: si la IA te configuró algo, tenés que poder mostrarlo funcionando y explicar qué mira. Si no podés defenderlo, **no se aprueba**.
