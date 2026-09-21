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

- La guía paso a paso usa **GitHub Actions + environments** con destinos gratuitos **verificados sin tarjeta** (jul-2026): **Render** para la app (web services gratis, que corren tus contenedores; ojo: su Postgres gratuito expira a los 30 días, por eso NO lo usamos) y **Neon** para la base de datos (Postgres serverless, plan gratuito permanente, sin tarjeta). Las *environment protection rules* (aprobaciones) son **gratis en repos públicos** en cualquier plan de GitHub — otra vez, tu repo público pagando dividendos.
- **Alternativas conocidas y su estado 2026**: Koyeb dejó de ser garantizado (puede exigir tarjeta para verificación anti-fraude, con una pre-autorización de USD 29 que luego cancela); Railway es trial; Fly.io pide tarjeta. Si un free tier muta a mitad de semestre (pasa), está el **fallback definitivo de la materia**: `self-hosted runner + docker-compose local` como entornos QA/PROD — documentado completo en §3.6 y **válido como entrega**.
- Riel **Azure**: Azure Pipelines environments + 2 Web Apps **F1** (tier gratuito vigente: 60 min de CPU/día, sin SLA). Tabla de equivalencias en §4.

📌 Este TP trabaja **sobre tu app del semestre**, extendiendo el pipeline de TP4/TP5: hasta hoy tu pipeline **verifica**; hoy empieza a **desplegar**.

🧪 **Los ejemplos de esta guía están escritos sobre la app de la cátedra** ([`ingsoft3ucc/demo-fullstack`](https://github.com/ingsoft3ucc/demo-fullstack) — .NET 8 + React/Vite + PostgreSQL, la misma de las demos en clase; cómo clonarla y levantarla, base de datos incluida: **TP2 §3.2**). Si un paso no te sale en tu app, probalo primero ahí —donde sabés que funciona— y después llevalo a la tuya. Lo que entregás, siempre, es **tu** app.

---

# Guía Paso a Paso – CD: environments, aprobaciones y deployment patterns (Práctica sugerida)

## 1- Objetivos de Aprendizaje

- Distinguir **CI, Continuous Delivery y Continuous Deployment** (y decidir cuál corresponde a tu contexto).
- Cerrar el CI publicando un **artefacto identificable**: la imagen etiquetada con el commit que la
  produjo, y sólo cuando la verificación pasó.
- Sacar la **configuración afuera de la imagen** (la dirección del backend, la cadena de conexión):
  la misma imagen corre en QA y en PROD, y lo que cambia son las variables del entorno.
- Modelar **entornos** (QA, PROD) con **environments** de GitHub: secrets con alcance, historial de deploys y **reglas de protección**.
- Construir la **promoción**: CI verde → deploy automático a QA → **aprobación manual** → PROD.
- Conocer las **estrategias de deployment** de la industria (rolling, blue-green, canary, feature flags) y el rollback.
- Publicar la entrega como **release de GitHub** con notas, etiquetada con el tag del práctico.

## 2- Marco teórico

### 2.0 El artefacto: qué entrega el pipeline cuando termina de verificar

Hasta acá tu pipeline **verifica**: construye, corre los tests, mide la cobertura y bloquea el merge si no llega. Preguntá ahora
qué **queda** cuando terminó. Un tilde verde y un reporte de coverage. La imagen que construyó nació
y murió adentro del runner, que se destruye al terminar el job.

Y ahí está la lección de esta semana:

> **Un pipeline que no produce artefacto no es integración continua: es compilación.**

Integrar continuamente significa que el trabajo de todos converge en algo **que existe** y que se
puede tomar y llevar a un entorno. Si cada corrida verifica y no deja nada, el despliegue —que es
de lo que se trata este práctico— tiene que volver a construir; y lo que se despliega ya no es,
literalmente, lo que se verificó. Es la misma trampa que el TP4 evitó cuando decidió que el
pipeline construyera con **tu** Dockerfile en vez de compilar por su cuenta: dos definiciones de
build divergen. Dos construcciones del mismo código, también.

Tres cosas que hacen que ese artefacto signifique algo:

**Nace de un build verde o no nace.** No es un detalle de cableado: es la condición que le da
sentido al registry. Si se publicara igual cuando los tests fallan, estar publicado dejaría de
querer decir *«esto pasó la verificación»* y el registry pasaría a ser un depósito de cosas que
alguien construyó alguna vez.

**Su lugar es un registry, no el almacén de artefactos.** El TP4 ya lo dejó dicho (§2.5) y en el TP2
(§3.7) lo hiciste **a mano**: publicaste tus imágenes vos. Lo que cambia hoy es quién lo hace y
cuándo: lo hace el pipeline, y sólo cuando corresponde.

**Se etiqueta con el commit que la produjo.** Una imagen etiquetada con su commit se puede comparar
con el código que la originó: es lo que te va a permitir contestar, más adelante en este mismo
práctico, la pregunta que gobierna todo despliegue serio — *«¿lo que está corriendo es lo que se
verificó?»*. Las etiquetas que se mueven de una imagen a otra (`latest` y parientes) son harina de
otro costal: se ven junto con la promoción (§2.3), que es lo único que les da significado.

#### La cadena que hace que el registry sea confiable

Fijate que no hace falta ningún control nuevo para garantizar que en el registry sólo haya cosas
verificadas. Sale de encadenar **tres** cosas que ya tenés:

1. **Nada entra a `main` sin el pipeline en verde** — el gate del TP4 más el umbral de cobertura del TP5.
2. **Sólo lo que entra a `main` se publica** — el `if` de rama del §3.0.
3. 🔴 **Y el paso que publica es el ÚLTIMO del mismo job que corrió los tests.** Si algo falla antes,
   el job muere y ese paso **nunca llega a correr**. Sin esto los otros dos no alcanzan: un pipeline
   que publica primero y testea después cumple 1 y 2, y el registry se le llena de imágenes rotas.

De donde: **nada llega al registry por accidente sin haber pasado la verificación completa.** Y eso
no se demuestra montando un caso: se demuestra leyendo la configuración. Es la diferencia entre un
principio que se sostiene por disciplina y uno que se sostiene solo.

> 📌 **«Por accidente» no es de más.** Vos podés subir una imagen a mano con `docker push` desde tu
> notebook, y nadie te lo impide: la cadena garantiza lo que **el pipeline** publica, no lo que
> físicamente puede entrar. Decirlo así es lo que la defensa te va a pedir.

> 📌 **Esto vivía en el TP5 hasta 2026.** Se mudó acá porque es donde *sirve*: el artefacto no es un
> tema de testing, es **lo que se promueve** entre entornos (§2.3). Sin una imagen inmutable y
> etiquetada por commit, «se despliega a PROD lo mismo que se probó en QA» es una promesa que no se
> puede cumplir.

### 2.1 La escalera: CI → Continuous Delivery → Continuous Deployment

Tres términos que la industria mezcla y tu defensa no puede mezclar:

- **Continuous Integration** (TP4): cada cambio se integra y se **verifica** automáticamente. Termina en un artefacto verificado… que no va a ningún lado solo.
- **Continuous Delivery**: cada cambio verificado queda **listo para desplegarse con un click** — el pipeline llega hasta producción, pero el último paso lo autoriza un humano. El despliegue es una **decisión de negocio**, no un evento técnico traumático.
- **Continuous Deployment**: se saca al humano — todo cambio que pasa las verificaciones llega **solo** a producción. Requiere una madurez de tests y monitoreo altísima (si tu red de seguridad es floja, automatizaste la propagación de errores).

La pregunta de diseño no es "¿cuál es mejor?" sino "¿cuánta confianza automatizada tenemos?". En esta materia construimos **Continuous Delivery**: QA automático, PROD con aprobación. Es el punto donde está la mayoría de la industria — y donde el gate humano todavía enseña algo.

**Deploy ≠ release**: *desplegar* es poner binarios nuevos a correr; *release* es exponer la funcionalidad a los usuarios. Con **feature flags** (§2.5) podés desplegar código apagado y prenderlo después — desacoplando el riesgo técnico del riesgo de producto.

### 2.2 Environments: los entornos como objetos de primera clase

Un **environment** en GitHub es la representación formal de un entorno de despliegue (QA, staging, producción). No es una carpeta ni una convención: es un objeto de la plataforma con tres superpoderes:

1. **Secrets y variables con alcance**: el secret `RENDER_HOOK_API_QA` del environment `qa` NO es visible para un job que despliega a `production` — cada entorno ve solo sus credenciales. Es el tercer alcance de secrets que anunciamos en el TP4 (repo → environment → organización).
2. **Reglas de protección** (*deployment protection rules*): **required reviewers** (hasta 6 personas/equipos; el job que apunta a ese environment queda **pausado** hasta que alguien apruebe) y **wait timer** (espera fija antes de desplegar). En repos **públicos** están disponibles gratis en todos los planes (en privados requieren plan Enterprise — tu repo público, de nuevo, es la razón por la que esta clase es gratis).
3. **Historial de deployments**: qué commit está en qué entorno y cuándo llegó — el link **Deployments** en el sidebar de la home del repo, filtrable por environment (quién aprobó cada uno queda registrado en el run del workflow). Trazabilidad de despliegues sin herramientas extra.

En el YAML, un job declara su entorno con `environment: qa` — y con eso hereda secrets, protecciones e historial. La mecánica de la aprobación es elegante: no hay "botón de deploy"; hay un job que **no arranca** hasta que las reglas del environment lo permitan.

### 2.3 La promoción: el mismo cambio sube de entorno

El pipeline de CD tiene forma de **cadena con compuertas**:

```
CI verde (build+tests+calidad)  →  deploy QA (automático)  →  smoke test  →  [aprobación]  →  deploy PROD
```

Dos principios de diseño que tenés que poder defender:

- **Se promueve lo MISMO que se verificó.** Lo que llega a PROD no se "rebuildea con fe": es el mismo commit que pasó por QA. Si rebuildeás entre entornos, lo que probaste y lo que desplegaste son cosas distintas — y el bug que se cuele va a ser inexplicable. 🔴 **Y acá va una honestidad que te conviene tener presente, porque es el tema del TP7**: en este práctico el proveedor (Render) **reconstruye** tu app desde el repositorio, así que promovés el mismo *commit*, no la misma *imagen*. La imagen que tu pipeline publica en el §3.0 queda ahí, lista y etiquetada, pero todavía no es la que corre. Convertirla en lo que efectivamente se despliega es exactamente lo que hace el TP7.
- **Cada deploy se verifica a sí mismo**: después de desplegar, el pipeline hace un **smoke test** — un chequeo mínimo de vida (¿responde `/health`? ¿la API lista datos?) — antes de declarar éxito. Deploy sin smoke test es deploy con los ojos cerrados: "terminó el script" no es "el sistema está vivo".

En GitHub Actions la cadena se arma con dos piezas que ya conocés de vista del TP4, ahora usadas en serio: `needs:` (que allá mencionamos para ordenar jobs, acá encadena la promoción) e `if: github.ref == 'refs/heads/main'` (el `if` que usaste con `!cancelled()` en el TP5, ahora condicionando por rama: los PRs verifican pero NO despliegan — solo lo integrado despliega).

### 2.4 La aprobación manual: qué compra el gate humano

¿Por qué frenar la máquina justo antes de PROD? Porque la aprobación compra cosas que la automatización (todavía) no da:

- **Timing de negocio**: no desplegar en el pico de ventas, esperar el anuncio, coordinar con soporte.
- **Un último contexto humano**: el aprobador mira la evidencia — ¿QA verde? ¿smoke test pasó? ¿qué cambia este deploy? — y decide con información, no con fe.
- **Responsabilidad explícita**: queda registrado quién aprobó cada deploy a producción (en el run del workflow). "¿Quién autorizó esto?" deja de ser una pregunta retórica.

La contracara: una aprobación **sin criterios** es teatro — un humano clickeando "Approve" por reflejo agrega latencia sin agregar seguridad. Por eso el TP te pide definir (y defender) **qué mira tu aprobador antes de aprobar**, y también te pide la evidencia de un **rechazo**: un gate que nunca rechaza no está funcionando, está decorando.

### 2.5 Deployment patterns: cómo despliega la industria sin romper

Con un solo servidor, desplegar es "apagar lo viejo, prender lo nuevo" (*recreate*) — con su ventana de downtime. La industria desarrolló estrategias para desplegar **sin** (o casi sin) interrupciones y con riesgo controlado:

| Estrategia | Cómo funciona | Downtime | Costo | Rollback | Cuándo usarla |
|---|---|---|---|---|---|
| **Recreate** | Baja todo lo viejo, sube lo nuevo | Sí (ventana) | 1× infra | Redesplegar lo anterior | Entornos internos, QA, apps chicas |
| **Rolling** | Reemplaza instancias de a tandas | No (capacidad reducida) | 1× (+margen) | Gradual (lento) | Default de Kubernetes y fleets |
| **Blue-green** | Dos entornos completos; el router cambia de uno al otro | No (switch instantáneo) | **2×** infra | **Instantáneo** (volver el switch) | Cuando el rollback rápido lo vale |
| **Canary** | La versión nueva recibe primero un % chico de tráfico real, se observa, y se amplía | No | 1× (+canario) | Cortar el canario | Cambios riesgosos con buen monitoreo |
| **Feature flags** | El código llega apagado; se prende por config, por usuario o por % | No | Complejidad en código | **Apagar el flag** (segundos) | Desacoplar deploy de release; probar en prod |

**Cómo se prende un flag «para el 10 %».** El flag no es un interruptor: es una pregunta que el código hace por cada usuario —¿está prendido para ESTE usuario?— y la contesta una regla que vive **fuera del código**: un servicio de flags (LaunchDarkly, Unleash) o una configuración que la app lee mientras corre. Una variable de entorno no alcanza: se lee al arrancar, y para cambiarla hay que reiniciar o redesplegar. Por eso cambiar la regla no es un deploy. La regla se abre por etapas: primero el equipo (mira quién es el usuario), después un porcentaje (del identificador sale un número de 0 a 99, siempre el mismo; si da menos de 10, la ve — y al subir a 20 los primeros siguen adentro), al final todos, y la condición se borra del código. **No es canary**: en canary corren DOS versiones a la vez y reparte el balanceador, desde afuera de la app; con un flag corre UNA sola versión y reparte esa condición, desde adentro.

Cuatro observaciones para la defensa: (1) **canary y flags requieren observabilidad** — sin métricas que digan "el canario sangra", el patrón es ruleta (por eso Monitor/TP9 cierra el ciclo); (2) los patrones se **combinan**: blue-green para la infraestructura + flags para las features es un combo común; (3) en **rolling** conviven las dos versiones durante el reemplazo — cuidado con los cambios de esquema de BD o de contratos que la versión vieja no entiende; (4) los **flags viejos que nadie limpia son deuda técnica** — el patrón exige la disciplina de retirarlos cuando la feature se consolida.

**Rollback**: el plan B es una feature, no una vergüenza. La pregunta profesional no es "¿va a fallar?" sino "**cuando** falle, ¿cuánto tardamos en volver?" (¿te suena? es la 4ta métrica DORA de la clase 1). Regla práctica: el rollback más confiable es **redesplegar la versión anterior conocida** — otro motivo para promover artefactos inmutables versionados y no "lo último de la rama".

### 2.6 Versionado: SemVer y releases

> 🔴 **Dos numeraciones, y no son la misma — acá se confunden todos.** Esta materia usa el tag
> `vN.0.0` para **cerrar cada práctico** (TP6 → `v6.0.0`): es el punto que se navega en la defensa,
> y el número no lo elegís vos, lo fija el número del TP. **SemVer es otra cosa**: es cómo la
> industria numera *releases de producto* según lo que cambió (¿agregaste funcionalidad? MINOR.
> ¿sólo arreglos? PATCH). Acá abajo está para que sepas distinguirlas; **este TP no te pide entregar
> nada sobre SemVer**, sólo el tag `v6.0.0`.

Cuando los deploys se vuelven frecuentes, "¿qué versión está en PROD?" necesita mejor respuesta que un hash. **SemVer** (`MAJOR.MINOR.PATCH`): MAJOR rompe compatibilidad, MINOR agrega sin romper, PATCH arregla. El tag (`v1.2.0`) marca el commit exacto, y la **Release** de GitHub le agrega notas legibles (`gh release create v1.2.0 --generate-notes` las arma desde los PRs mergeados — los buenos mensajes que venís escribiendo desde el TP1, que con squash se vuelven el título del PR, acá cobran). Cada llegada a PROD de este TP se etiqueta y se publica como release: es tu changelog público y tu catálogo de puntos de rollback.

### 2.7 La letra chica de los free tiers (y el plan C)

> 🔴 **Las 750 horas son del WORKSPACE, no de cada servicio.** Las consumen los *web services*
> —tus cuatro servicios: back y front, en QA y en PROD—. Un servicio despierto las 24 h se come 720
> él solo. No te va a pasar
> —duermen a los 15 minutos sin tráfico— pero sí te puede pasar si dejás algo haciendo ping para
> que no duerma. Si el workspace se queda sin horas, Render **suspende los servicios hasta fin de
> mes**, y tus URLs tienen que seguir vivas hasta la defensa. Presupuestalo en `decisiones.md`.

> 🔴 **Y los minutos de build: 500 por mes en el workspace gratuito.** Cada deploy de este TP
> reconstruye tu app en Render (hasta cuatro construcciones por promoción: back y front, QA y PROD).
> Si se acaban, Render deja de construir hasta fin de mes: el hook responde igual, tu smoke da verde
> contra la versión vieja y no te enterás. Mirá el consumo en *Workspace → Billing* y presupuestalo
> en `decisiones.md`.

🕐 **Los números que ordenan tu semana**, medidos en la corrida de la cátedra: crear las cuentas y
los cuatro servicios, ~40 min; **cada build de Render, 2 a 5 min** por servicio (y son dos por
deploy); el smoke arranca después y puede tardar otro minuto largo si el servicio estaba dormido;
cada ciclo *Pull Request → checks → merge → deploy → smoke* te lleva **entre 8 y 12 minutos**, y el
práctico tiene **cinco** de esos ciclos encadenados. Con eso: el práctico entero son **entre 9 y 12
horas**, de las cuales ~3 son de espera pura. Entra en una semana; no entra en una tarde.

Desplegar gratis existe, con **contrato**: el free tier de Render (750 horas de instancia por mes, **por workspace**) duerme el servicio tras ~15 min sin tráfico y el primer request lo despierta con un **cold start** de hasta ~1 minuto (tu smoke test necesita **reintentos con espera**, no un curl seco); Neon suspende el cómputo a los ~5 min idle (se despierta solo, mucho más rápido) y limita almacenamiento (0.5 GB) y horas de cómputo mensuales. Diseñar sabiendo el contrato del tier ES una habilidad de la materia — los timeouts y reintentos que escribas acá son los mismos que vas a escribir contra cualquier infra real.

Y como siempre: si el free tier muta, el **fallback local** (self-hosted runner del TP4 + docker-compose del TP2 como entornos QA/PROD) cumple **todos** los checkpoints de este TP. Desplegar a infraestructura propia es exactamente así: es lo que hace cualquier empresa que corre sus servicios en sus propios servidores.

## 3- Desarrollo de la guía (riel GitHub Actions + Render + Neon)

> Trabajás sobre **tu app del semestre** con el pipeline de TP4/TP5. Los ejemplos asumen `./backend` (.NET con Dockerfile) y `./frontend` (Vite). Si un paso de proveedor difiere para tu stack, el concepto (deploy disparado por pipeline + environment + aprobación) es lo que no cambia.
>
> 🔴 **Render y Neon son UNA opción, no la consigna.** Podés usar cualquier proveedor —Railway,
> Fly.io, Koyeb, Supabase, Azure, AWS, tu propio servidor— o el fallback local del §3.6. Lo que se
> corrige es el **contrato**, y es el mismo en todos lados: (1) **dos entornos separados**, QA y
> producción, cada uno con su URL pública; (2) **una base de datos por entorno**, no una compartida;
> (3) el front y el back **corriendo como contenedores**, a partir de tus Dockerfiles; (4) el deploy
> lo **dispara tu pipeline** con el commit que verificó —el auto-deploy del proveedor, apagado—; y
> (5) producción **detrás de la aprobación** del environment de GitHub. Si tu proveedor cumple eso,
> vale igual: lo que cambia son los nombres de los campos, y eso lo resolvés con su documentación.
> Contá en `decisiones.md` cuál usaste y cómo cumplís los cinco puntos.

> 📬 **Logística**: esta semana aparece el gate humano — y lo vas a operar vos mismo: sos el **required reviewer de producción** de tu propio repo, y vas a aprobar (y rechazar) tus deploys. A diferencia de los PRs, GitHub **sí permite aprobar tus propios deployments** — por eso el flujo funciona trabajando solo.

### 🔧 Tu stack y tu nube, de un vistazo

🔧 **Toda esta guía está escrita sobre la app de la cátedra (.NET + Vite) con GitHub Actions,
ghcr.io, Render y Neon. Si lo tuyo es otra cosa, esta tabla es tu índice: buscá tu fila y seguí.**
Nada de lo que se evalúa depende de estas herramientas — se evalúa que **logres** cada cosa.

| Lo que tenés que lograr | Los ejemplos de la guía | Cómo se llama en otro lado |
|---|---|---|
| **Publicar la imagen desde el pipeline** | `docker/build-push-action` → `ghcr.io` | Docker Hub · ECR · Artifact Registry · el registry de GitLab |
| **Que sólo la rama principal publique** | `if: github.ref == 'refs/heads/main'` | `rules:` de GitLab · `trigger.branches` de Azure |
| **Etiquetar con el commit** | `:sha-${{ github.sha }}` | igual en todos: la variable del commit del CI |
| **Un entorno con su propia configuración** | environments `qa` / `production` de GitHub | *Environments* de GitLab · *Environments* de Azure Pipelines |
| **Que el deploy lo dispare el pipeline** | deploy hook de Render **con `&ref=<sha>`** | `flyctl deploy --image` · `railway up` · `az webapp deploy` · `kubectl set image` |
| 🔴 **Desplegar EL commit verificado, no la punta de la rama** | el `&ref=` de arriba | en los que despliegan **por imagen** sale gratis: el tag ya identifica el commit |
| **Un gate humano antes de producción** | *required reviewers* del environment | *protected environments* de GitLab · *approvals* de Azure |
| **Secrets con alcance de entorno** | environment secrets | *masked/protected variables* · *variable groups* |
| **Una base por entorno** | dos databases en Neon | RDS · Cloud SQL · Supabase · el Postgres del proveedor |
| **Crear el esquema en cada base** | `EnsureCreated()` al arrancar | migraciones de EF Core · Flyway · Alembic · Liquibase · `prisma migrate` |
| **Smoke post-deploy con reintentos** | `curl` en loop | igual en todos: es un script |
| **Release versionada con notas** | `gh release create` | *Releases* de GitLab · *Tags* de Azure |

Y las que dependen de **tu app**, no de tu nube:

| Lo que tenés que lograr | Los ejemplos de la guía | Cómo se llama en otro lado |
|---|---|---|
| **Pasarle la conexión a la app por el entorno** | `ConnectionStrings__Default` | `SPRING_DATASOURCE_URL` · `DATABASE_URL` · `DJANGO_DATABASE_URL` — lo que tu framework lea del ambiente |
| **El formato de la cadena de conexión** | pares `Clave=Valor` de Npgsql | URI `postgresql://…` · JDBC `jdbc:postgresql://…` — Neon te da las tres, elegí la tuya |
| **Un endpoint que diga «estoy vivo»** | `/health` | `/actuator/health` (Spring) · `/healthz` · el que escribas vos: alcanza con que devuelva 200 |
| **Un endpoint que pruebe que la BD responde** | `/api/tareas` | cualquiera que **toque la base**: si el smoke sólo pega al `/health`, una base caída pasa en verde |
| **Servir el front construido** | la imagen nginx del TP2, con `dist/` de Vite adentro | `build/` de CRA · `.next/` · `public/` — lo que tu build produzca, adentro de TU contenedor |
| **Que el front sepa dónde está SU backend** | variable `BACKEND_URL` que nginx lee al arrancar | la misma idea en cualquier servidor web: la dirección del backend no va escrita en la imagen |

> 🔴 **Tres cosas de TU app que hay que tener listas antes de desplegar, y que no se ven hasta que
> fallan.** Las tres dejan el build en verde y el entorno roto:
> 1. **`/health` tiene que existir.** El smoke lo llama tres veces, y la app de la cátedra lo trae; si
>    la tuya no, escribilo (una línea que devuelva 200) o cambiá el smoke por un endpoint tuyo que sí
>    exista y toque la base.
> 2. **Tu front no puede tener la dirección del backend horneada.** Si en el TP2 quedó un
>    `http://localhost:8080` dentro del build de Vite, en Render el navegador va a llamar a esa
>    dirección y no a tu api. El front tiene que pedirle a **su mismo origen** (`/api/...`) y dejar
>    que nginx lo mande al backend, que es lo que arma la plantilla del §3.3.
> 3. **Tu backend tiene que escuchar en `0.0.0.0`**, no en `localhost`. Adentro de un contenedor,
>    `localhost` es el propio contenedor: Render publica el servicio, dice **Live**, y la URL
>    contesta 502 para siempre. En .NET alcanza con `ASPNETCORE_URLS=http://0.0.0.0:8080` (la app de
>    la cátedra ya lo trae en su Dockerfile); en otros frameworks es el `host` del `listen`.

📌 **Si tu proveedor despliega por imagen** (Fly.io, Railway, Cloud Run, Kubernetes) tenés una
ventaja: le pasás el tag `sha-…` que acabás de publicar y la pregunta *«¿se desplegó lo mismo que
verifiqué?»* se contesta sola. Si el tuyo **reconstruye desde el repositorio** —como Render en este
TP—, esa garantía la tenés que pedir explícitamente, y ahí es donde va el `&ref=`. Contalo en
`decisiones.md`: es exactamente la diferencia que el TP7 viene a cerrar.

📌 **¿Tu stack no está?** El criterio es el mismo y averiguarlo es parte del trabajo: las doce filas
existen en todos lados. Contá en `decisiones.md` qué usaste para cada una.

---

### 3.0 El cierre del CI: publicar la imagen, y sólo la que pasó

Tu pipeline ya verifica. Falta que **entregue** (§2.0). Son tres cambios chicos sobre el `ci.yml`
que ya tenés, y el orden en que van importa más que el contenido.

**Uno — permiso para publicar.** El job necesita poder escribir paquetes. Va **adentro de cada
job**, a la altura de `runs-on:`:

```yaml
  build-backend:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      …
```

> 🔴 **Adentro del job, no arriba de todo.** Es tentador ponerlo al mismo nivel que `on:` y
> escribirlo una sola vez. Pero `permissions:` a nivel del workflow **reemplaza** el default para
> **todos** los jobs, no lo amplía: le das permiso de escritura de paquetes a jobs que no publican
> nada. Declararlo por job toca sólo al que publica, y ése es el argumento: **mínimo privilegio**.
> (Un job que sí necesite otro permiso puede declarar el suyo y gana sobre el del workflow — lo del
> nivel superior no lo deja sin permisos, simplemente le pone otros por defecto.)
>
> ⚠️ **Lo que no listás queda sin permiso.** Por eso `contents: read` no es de más: sin él, el
> checkout no puede leer tu código. Y si tu job usa una action que comenta el PR o crea checks,
> sumá acá ese permiso (`pull-requests: write`, `checks: write`).

> 🔑 **No hace falta ningún secret nuevo, y eso es parte de la gracia.** GitHub le entrega a cada
> corrida un token propio (`GITHUB_TOKEN`), que vive lo que dura el job y no lo tenés que guardar en
> ningún lado. Es el caso ideal del §2.6 del TP4: la credencial se le pasa al comando que la
> necesita y nunca se imprime.

**Dos — entrar al registry.** Un paso más en cada job, **al final**, después del paso que publica el
reporte (en el paso Tres, el build de la imagen se muda justo debajo de éste):

```yaml
      - name: Entrar al registry
        if: github.event_name == 'push'
        uses: docker/login-action@v4
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
```

**Tres — que el build final publique.** Éste es el cambio de fondo, y tiene dos partes: **mover el
paso, y después cambiarle tres líneas** (el nombre y dos parámetros).

🔴 **Primero el orden, que es lo que hace verdadera la frase que abre este práctico.** El paso que ya
construía la imagen (el del TP4, con `push: false`, el que **no** tiene `target: test`) hoy está
**arriba**, antes de los tests. (Hay **cuatro** `push: false`, dos por job: los de la etapa de tests
se quedan como están.) Cortalo y pegalo **al final del job**, debajo de «Entrar al registry». El job te tiene que quedar
así:

```
1 · checkout / preparar el constructor
2 · (acá estaba el build de la imagen: se fue abajo)
3 · construir la etapa test          6 · publicar el reporte
4 · docker run: correr los tests     7 · entrar al registry
5 · armar el reporte                 8 · construir y PUBLICAR la imagen  ← el que moviste
```

**Si no lo movés, todo te va a quedar en verde igual** —los tests corren, la imagen se publica— y
tu pipeline va a estar publicando una imagen que se construyó **antes** de saber si los tests
pasaban. No hay ningún error que te avise: la única señal es el orden de los pasos en el log. Y la
frase que vas a escribir en `decisiones.md` —«el artefacto sale sólo si la verificación pasó»— sería
falsa sobre tu propio repositorio.

✅ **Cómo lo comprobás**: abrí una corrida en Actions y mirá la lista de pasos del job. «Construir y
publicar la imagen» tiene que ser **el último de tus pasos** (después sólo aparecen los `Post …` y
*Complete job* que agrega GitHub). Si aparece antes de «Correr los tests», no lo
moviste.

Y ahora sí, las tres líneas que cambian: el nombre, `push` y `tags`:

```yaml
      - name: Construir y publicar la imagen del backend
        uses: docker/build-push-action@v7
        with:
          context: ./backend
          push: ${{ github.event_name == 'push' && github.ref == 'refs/heads/main' }}
          tags: ghcr.io/${{ github.repository }}-backend:sha-${{ github.sha }}
          cache-from: type=gha,scope=backend
          cache-to: type=gha,mode=max,scope=backend
```

En el job del frontend va lo mismo, cambiando `backend` por `frontend` en todas las líneas donde
aparece.

> 🔴 **Las dos condiciones, no una.** `github.event_name == 'push'` sola te alcanzaría *hoy*, porque
> tu workflow sólo escucha `main` y los Pull Requests. Pero el día que le agregues otra rama al
> disparador, un push a esa rama publicaría una imagen que nunca pasó por un Pull Request ni por los
> guardianes — y no te ibas a enterar. La garantía conviene tenerla **en el paso que publica**, no en
> el disparador del archivo.

#### Por qué el orden de los pasos ES la condición

Mirá lo que acabás de escribir: **no hay ningún `if` que diga «si los tests pasaron»**. No hace
falta. Los steps de un job corren **en orden y se cortan al primer error** (lo viste en el TP4): si
el `docker run` de los tests devuelve rojo, el job muere ahí y el paso de publicar **nunca llega a
correr**. La condición no está escrita: está en el orden.

> 🔴 **Por eso el paso de construir la imagen final se mueve abajo.** Si quedara donde estaba —antes
> de los tests, como en el TP4— publicaría una imagen que todavía no se verificó. Es un cambio de
> dos renglones y es todo el punto del práctico.

> 📌 **Y por eso también publica el MISMO paso que construye**, en vez de un job aparte que suba lo
> que otro construyó. Un job aparte tendría que **volver a construir** la imagen, y entonces lo
> publicado no sería lo verificado sino una reconstrucción. En el TP7 vas a ver esto convertido en
> regla: *la unidad de release se construye una sola vez*.

#### Qué significa cada pieza del nombre

`ghcr.io/tu-usuario/tu-repo-backend:sha-a1b2c3…` — el registry, tu espacio, el nombre de la imagen,
y después de los dos puntos la **etiqueta**: el commit que la produjo.

> 🔴 **Minúsculas.** El registry no acepta mayúsculas en el nombre. Si tu usuario o tu repositorio
> las tienen, `${{ github.repository }}` te va a dar un error que dice **`repository name must be lowercase`**:
> escribí el nombre a mano, todo en minúsculas.

> ⚠️ **En los Pull Requests no se publica nada, y está bien.** Con `push:` atado al evento, la
> corrida del PR construye y verifica igual, pero no sube nada — el registry guarda solamente lo que
> ya está integrado. La corrida que publica es la del `push` a `main`, esa que el TP4 configuró
> «sólo para que main tenga su corrida» y que ahora tiene un segundo trabajo.

**El merge.** Cuando los checks del PR estén en verde, mergealo con **Squash and merge**. Ese merge
dispara la corrida de `main`, que es la que publica: esperá a que termine en verde (si en *Packages*
no aparece nada, mirá esa corrida en *Actions*).

**Cuatro — hacer públicos los paquetes y comprobarlo desde afuera.** Andá a tu perfil → *Packages* →
**cada uno de los dos** paquetes → *Package settings* → abajo de todo, en **Danger Zone**, *Change
visibility* → *Public*, y escribí el nombre del paquete para confirmar. ⚠️ Hacer público un paquete
**no tiene vuelta atrás**: no se puede volver a privado. Y comprobalo sin credenciales (el `logout`
es para que una sesión vieja del TP2 no haga pasar el `pull` sin probar nada):

```bash
docker logout ghcr.io
docker pull --platform linux/amd64 ghcr.io/<tu-usuario>/<tu-repo>-backend:sha-<el-commit-del-merge>
docker pull --platform linux/amd64 ghcr.io/<tu-usuario>/<tu-repo>-frontend:sha-<el-commit-del-merge>
docker image inspect ghcr.io/<tu-usuario>/<tu-repo>-backend:sha-<el-commit> --format '{{.Created}}'
```

> ⚠️ El `--platform linux/amd64` es porque tu pipeline construye para la arquitectura del runner de
> GitHub. En una Mac con chip Apple, sin esa opción el `pull` falla con `no matching manifest for
> linux/arm64/v8`: no es que el paquete sea privado, es que no hay una versión para ese procesador.

> 📌 **Por qué público**: el mismo motivo que el repo. Quien corrige tiene que poder bajar tu imagen
> sin que le des acceso a nada. Y en unas secciones más, cuando el pipeline despliegue, un servicio
> de la nube va a tener que poder bajarla también.

#### Cómo te queda el orden, que es lo único que no podés equivocar

En esta sección tocaste el mismo job varias veces. Antes de mergear, releelo de arriba
abajo y comprobá que los pasos estén **en este orden** — no importa cómo se llamen los tuyos,
importa qué va antes de qué:

| # | Paso | Por qué ahí |
|---|---|---|
| 1 | Bajar el código · preparar el constructor | Sin esto no hay nada que construir (TP4) |
| 2 | Construir la **etapa de tests** y correrla | Si algo se rompe acá, el job queda en rojo y **los pasos que publican** no corren (los del reporte sí, a propósito: llevan `!cancelled()`) |
| 3 | Reporte de coverage y artefacto | Va después de los tests porque lee lo que ellos dejaron |
| 4 | Entrar al registry | Sólo cuando el evento es `push`; antes de publicar |
| 5 | **Construir y publicar la imagen final** | 🔴 **Último.** Acá está la condición: si llegaste, es porque todo lo anterior pasó |

> 🔧 **Si publicó una imagen con los tests en rojo**, no busques el error en los permisos ni en el
> registry: el paso de publicar te quedó **arriba** del de los tests, **o** le pusiste a «Entrar al
> registry» o a «Construir y publicar» un `if` con `!cancelled()` o `always()` (el del reporte del
> TP5 **no** va acá), o `continue-on-error` en los tests. Los pasos que publican no llevan nada de
> eso: su `if` sin función de estado ya exige que todo lo anterior haya pasado.

**✅ Checkpoint:** después de mergear, las **dos** imágenes están en el registry etiquetadas con el
commit del merge, se pueden bajar desde otra máquina, y podés explicar la cadena: nada llegó a
`main` sin pasar los guardianes automáticos, sólo lo que llega a `main` se publica, y el paso que publica es el ÚLTIMO del mismo job que corrió los tests — si fallan, nunca se llega a él. Son tres eslabones, no dos.

> 📌 **Va primero, y por eso es el §3.0**: es el cierre del CI, y todo lo que sigue lo da por hecho.
> 🔴 **Ojo con lo que este práctico NO hace todavía**: los entornos de Render que armás más abajo
> **reconstruyen** tu app desde el repositorio, así que la imagen que acabás de publicar queda
> guardada y etiquetada, pero no es la que corre en QA ni en PROD. Que *sí* lo sea es el TP7, y esta
> sección es lo que se lo hace posible: sin una imagen publicada e identificable, allá no habría
> nada que desplegar.

> 🎬 **Lo que se ve en el video, desde tu terminal** (ninguno es obligatorio: sirven para mirar tu
> trabajo mientras lo hacés).
>
> ```bash
> git switch -c feature/publicar-imagen        # la rama de esta sección
> git --no-pager diff -U1                      # qué cambiaste: + verde agregado, - rojo sacado
> git add .github/workflows/ci.yml             # aparta lo hecho: el próximo diff muestra sólo lo nuevo
> bat --paging=never --style=numbers --line-range 10:37 .github/workflows/ci.yml   # un tramo con número de línea (en el editor ves lo mismo)
> grep -n -B2 'push: false' .github/workflows/ci.yml   # los cuatro push: false y de qué paso es cada uno
> grep -n -E '^  build-|- name:|if:' .github/workflows/ci.yml   # jobs, pasos y if, en orden
> git commit -m "…" && git push -u origin feature/publicar-imagen
> gh pr create --fill                          # abre el PR con el título del commit
> gh run list --limit 3                        # las últimas corridas de Actions
> ```

### 3.1 La base de datos: Neon (Postgres serverless, gratis permanente)

1. Entrá a **neon.com** → *Sign up* con GitHub (sin tarjeta).
2. Creá un proyecto (ej: `miapp`) y **dos databases**: `app_qa` y `app_prod` (mismo servidor, entornos separados — suficiente para la materia). Neon crea el proyecto con **una** base (`neondb`): las tuyas se agregan desde *Databases → Add database*, y la `neondb` la podés borrar.
3. Con el botón **Connect** del proyecto (en algunas cuentas, *Connection Details*) **elegí la database** (`app_qa`, y después `app_prod`: si no, copiás la de `neondb`) y, en el selector de formato, bajo **.NET**, *Entity Framework (appsettings.json)*. Copiá sólo el valor, ya en formato Npgsql (`Host=…;Database=app_qa;Username=…;Password=…;SSL Mode=VerifyFull;Channel Binding=Require`). ⚠️ `Channel Binding` lo entiende **Npgsql 8 o posterior**: si tu app usa una versión anterior y al arrancar dice *«Keyword not supported: 'channel binding'»*, sacá ese par. Si preferís partir de la URI genérica (`postgresql://user:pass@host/db?sslmode=require&channel_binding=require`), la traducción es esa misma: host, database, usuario y password como pares `Clave=Valor` + `SSL Mode=Require` (el `channel_binding` Npgsql lo negocia solo; y si tu password tiene caracteres escapados con `%` en la URI, en el formato de pares va **decodificado**).

> 💡 ¿Por qué no el Postgres de Render? Su tier gratuito **expira a los 30 días** de creado (con 14 de gracia antes del borrado) — ni siquiera cubre un mes de TP. El de Neon es permanente (con límites de uso). Anotalo en `decisiones.md`: elegir infraestructura también es leer letra chica.

4. 🔴 **Y ahora el paso que falta en casi todas las entregas: las tablas.** Las dos bases nacen
   **vacías**. Tu app tiene que crear su esquema, y hay tres maneras según cómo esté escrita:
   - **Se crea sola al arrancar** — la app de la cátedra hace esto: un `db.Database.EnsureCreated()`
     en el `Program.cs`. Si la tuya lo tiene, no hacés nada acá: las tablas aparecen **la primera vez
     que tu app arranca**, y eso pasa **al crear el servicio en Render** (§3.2): Render lo despliega apenas
     lo creás. Por eso esta sección va antes que la de Render. 📌 Entonces el checkpoint de abajo lo
     cumplís en dos tiempos: ahora comprobás que podés **conectarte**, y el `count(*)` lo mirás
     cuando tus servicios de Render ya arrancaron. Volvé acá en ese momento: es un minuto y te ahorra el atasco.
   - **Tenés migraciones** (EF Core, Flyway, Alembic, Liquibase…): corrélas contra cada base
     **antes** del primer deploy, o agregá el paso al arranque de la app. Dos bases: dos corridas.
   - **No tenés ninguna de las dos**: creá las tablas a mano en el SQL Editor de Neon, en las **dos**
     bases, y decilo en `decisiones.md`. No es la forma profesional y por eso se cuenta.

> 🔴 **Por qué esto merece su propio paso.** Si la base queda vacía, tu app *levanta bien* y el
> síntoma aparece **recién en el smoke test**, entre diez y veinticinco minutos después (reintenta 30 veces cada 20 s), y
> con un error que no menciona la palabra «tabla» por ningún lado. Es el atasco más caro de este TP,
> y se evita mirando una vez el checkpoint de abajo.

> 🔴 **El error más silencioso de todo el TP: PROD apuntando a la base de QA.** Las dos cadenas de
> conexión difieren en **una palabra** (`Database=app_qa` contra `app_prod`), se copian y pegan una
> después de la otra, y si te equivocás **nada se pone rojo**: PROD levanta, contesta, y tu smoke da
> verde — sólo que está escribiendo en la base de QA. Lo vas a descubrir cuando un dato de prueba
> aparezca en producción.
>
> Se comprueba en treinta segundos, y **hacelo**: creá un dato con un nombre inconfundible **sólo en
> `app_prod`** desde el SQL Editor de Neon (en la app de la cátedra: `insert into "Tareas" ("Titulo", "CreadaEl") values ('SOY PROD', now());`), abrí la URL de PROD y fijate que
> aparezca, abrí la de QA y fijate que **no**. Si aparece en las dos, tenés las dos apuntando al
> mismo lado. Contalo en `decisiones.md`: «cómo comprobé que cada entorno usa su propia base» es una
> pregunta de defensa.

**✅ Checkpoint, en dos tiempos.** *Ahora*: tenés dos connection strings (QA y PROD) y podés
conectarte a `app_qa` —el camino de un click es el **SQL Editor** de la propia consola de Neon—.
*Ahora* el `count(*)` de abajo te contesta *«relation does not exist»*: es lo esperado, la base está vacía.
*Y apenas tus servicios de Render arrancaron (§3.2)*: **`SELECT count(*) FROM <tu tabla>;` devuelve un número**
(aunque sea 0) en vez de *«relation does not exist»*, en las **dos** bases. Ese `count` es el
checkpoint de verdad: el `SELECT 1;` sólo prueba que la base existe, no que tenga tus tablas.

> 🔴 **Ojo con cómo escribís el nombre de la tabla, porque este error se ve IDÉNTICO al de la base
> vacía.** Entity Framework crea las tablas con el nombre de la propiedad y **entre comillas**:
> en la app de la cátedra la tabla se llama `"Tareas"`, con mayúscula. Postgres distingue: si
> escribís `select count(*) from tareas;` —minúscula, sin comillas— te contesta *«relation "tareas"
> does not exist»* **aunque la tabla exista**, y vas a creer que tu base está vacía. La consulta
> que anda es `select count(*) from "Tareas";`. Medido contra Postgres real: la primera falla y la
> segunda devuelve el número. Si no sabés cómo se llama, preguntáselo a la base:
> `select table_name from information_schema.tables where table_schema = 'public';`

### 3.2 Los entornos de ejecución: Render (dos contenedores × 2 entornos)

> 🚨🚨 **ATENCIÓN: LO QUE ARMÁS EN ESTA SECCIÓN ESTÁ MAL A PROPÓSITO.** 🚨🚨
>
> Tu pipeline ya publica **la imagen que verificó** (§3.0). Pero acá le vas a pedir a Render que
> **vuelva a construir** tus dos contenedores **desde el repositorio**. O sea que lo que corre en QA y
> en PROD **NO es la imagen que tus tests aprobaron**: es **otra construcción** del mismo commit.
> Puede salir igual… o no (una dependencia que cambió entre un build y el otro, una imagen base que
> se actualizó). Eso rompe el principio de §2.3 —**se promueve lo mismo que se verificó**— y es
> **exactamente el problema que el TP7 viene a resolver**: ahí Render deja de construir y **ejecuta
> tu imagen del registry**.
>
> Se hace así en el TP6 para no mezclar dos temas: acá aprendés entornos, promoción y el gate humano.
> **Contalo en `decisiones.md`** con tus palabras: qué garantía perdés al reconstruir, y qué podría
> salir distinto. Es pregunta de defensa.

Creá la cuenta en **render.com** (sign up con GitHub, sin tarjeta) y autorizá tu repo. Vas a crear **cuatro** servicios, todos *Web Service* con **Docker**: back y front, por cada entorno (primero QA, después PROD).

**Backend** — *New → Web Service* → tu repo:
- **Root Directory**: `backend` · **Runtime/Language**: **Docker** (detecta tu Dockerfile del TP2) · **Instance type**: Free.
- **Environment variables**: `ConnectionStrings__Default` = la connection string de Neon del entorno (¡la de `app_qa` en el servicio QA!).
- **Settings → Auto-Deploy: OFF.** Clave conceptual: Render sabe desplegarse solo en cada push, pero entonces el deploy NO pasaría por tu pipeline (ni esperaría a CI verde, ni a la aprobación). El deploy lo dispara **tu workflow**, no el proveedor.
- Nombralos identificables: `miapp-api-qa` / `miapp-api-prod`.

> 🔴 **El runtime lo elegís al CREAR el servicio, y conviene no equivocarse.** Si elegiste otro
> (Node, Python…), Render ignora tu Dockerfile y construye a su manera. Se corrige sin perder nada:
> *Settings → Build → **Edit*** y ahí cambiás la fuente y el runtime; **no borres el servicio**,
> porque con él se van la URL —que vas a tener escrita en el smoke y en el `BACKEND_URL` del front—
> y el deploy hook. Comprobalo de un vistazo en el *Dashboard*: la columna **Runtime** tiene que
> decir `Docker` en los cuatro servicios (y en la página de cada uno, la etiqueta `Docker` al lado
> del nombre). En esa misma pantalla se editan **Root Directory** y la ruta del Dockerfile.

**Frontend** — también *New → Web Service* → tu repo, **con tu contenedor nginx del TP2**:
- **Root Directory**: `frontend` · **Runtime/Language**: **Docker** · **Instance type**: Free.
- **Environment variables**: `BACKEND_URL` = la URL de **la api de SU entorno**, **sin barra al final ni ruta** (`https://miapp-api-qa.onrender.com` en el front de QA, no `…onrender.com/`: con una barra, nginx manda todos los pedidos a `/` y cada llamada da 404; la de PROD en el de PROD) y `DNS_RESOLVER` = `8.8.8.8`. Tu nginx las va a leer recién después del cambio del §3.3: hasta ese merge, el front de Render abre pero la lista da error.
- Nombralos `miapp-front-qa` / `miapp-front-prod` (los smoke tests de §3.3/§3.4 usan esas URLs). Auto-Deploy también **OFF**.

La **URL pública** de cada servicio está arriba de todo en su página, debajo del nombre. Copiala de
ahí, exacta: si el nombre ya estaba tomado en `onrender.com`, Render le agrega un sufijo.

**✅ Checkpoint:** las 4 URLs (`…-api-qa`, `…-front-qa`, `…-api-prod`, `…-front-prod`) responden: el backend en `/health` y en `/api/tareas` (con **su** base), y el front abre la página. ⚠️ Las llamadas del front a la api **todavía fallan** (502): el front de Render sigue con el `nginx.conf` del TP2, que apunta a `backend:8080`. Se arregla en §3.3 con la plantilla, y la app completa se comprueba en el checkpoint del §3.3 (QA) y del §3.4 (PROD). (Primer request tras idle: paciencia — cold start, ahora también en el front.)

### 3.3 El deploy lo dispara el pipeline: deploy hooks + job de QA

Antes de empezar, traé a tu `main` local el merge del §3.0 —que hiciste en GitHub— y abrí la rama de
esta sección desde ahí: `git switch main && git pull && git switch -c feature/deploy-qa`. Sin el
`pull`, la rama sale del `main` viejo y el PR choca.

🔴 **Primero, un cambio en tu front, en esta misma rama (llega a `main` con el PR del deploy): el `nginx.conf` del TP2 tiene la dirección del backend ESCRITA
ADENTRO** (`http://backend:8080`, el nombre del servicio en compose). En Render ese nombre no
existe: el backend es otra URL, y distinta en QA y en PROD. Si la escribís fija en la imagen, la
imagen de QA no sirve para PROD — y en el TP7 vas a querer desplegar **la misma imagen** en los dos.
La salida es que nginx la lea **del entorno al arrancar**: la imagen oficial de nginx procesa los
archivos de `/etc/nginx/templates/` y reemplaza `${VARIABLE}` por su valor.

Renombralo con git, para que quede como el mismo archivo, y dejalo así:

```bash
git mv frontend/nginx.conf frontend/default.conf.template
```

`frontend/default.conf.template` (reemplaza a tu `nginx.conf`):

```nginx
server {
    listen 80;

    location / {
        root /usr/share/nginx/html;
        index index.html;
        try_files $uri $uri/ /index.html;
    }

    # La dirección del backend y el DNS llegan por variables de entorno: en compose,
    # el nombre del servicio y el DNS de Docker; en Render, la URL de la api de ESTE
    # entorno y un DNS público.
    resolver ${DNS_RESOLVER} valid=10s ipv6=off;
    set $backend_api ${BACKEND_URL};

    location /api/ {
        proxy_pass $backend_api;
        proxy_ssl_server_name on;   # la api de Render es https: sin esto el TLS no sabe a qué host va
    }
}
```

Y en `frontend/Dockerfile`, la última etapa copia la plantilla y deja los valores de compose por defecto:

```dockerfile
FROM nginx:alpine
COPY default.conf.template /etc/nginx/templates/default.conf.template
COPY --from=build /app/dist /usr/share/nginx/html
ENV BACKEND_URL=http://backend:8080 \
    DNS_RESOLVER=127.0.0.11
EXPOSE 80
```

> 📌 **Tu `docker compose up` del TP2 sigue andando sin tocarlo**: los valores por defecto son los de
> compose. Render los pisa con las variables del servicio. La plantilla va en `templates/` y no en
> `conf.d/` porque nginx leería `conf.d/` tal cual, con los `${…}` sin reemplazar, y no arrancaría.

**Y comprobalo en tu máquina antes de subirlo:**

```bash
docker build -t front-tp6 frontend
docker run -d --name sin-variables front-tp6          # como en compose: sin variables
docker exec sin-variables grep -E '^ *(resolver|set) ' /etc/nginx/conf.d/default.conf
#   → resolver 127.0.0.11 … · set $backend_api http://backend:8080;
docker run -d --name con-variables -e BACKEND_URL=https://<tu-api-qa>.onrender.com -e DNS_RESOLVER=8.8.8.8 front-tp6
docker exec con-variables grep -E '^ *(resolver|set) ' /etc/nginx/conf.d/default.conf
#   → resolver 8.8.8.8 … · set $backend_api https://<tu-api-qa>.onrender.com;
docker rm -f sin-variables con-variables              # limpieza
git add frontend
```

Misma imagen, dos configuraciones: es lo que va a pasar en Render.

Y creá los environments con intención: *Settings → Environments → New environment* → `qa` (sin reglas de protección — es automático **a propósito**). El de `production` lo creamos en §3.4, con reglas.

Cada servicio de Render tiene un **Deploy Hook** (*Settings → Deploy Hook*): una URL secreta que, al recibir un request, dispara el deploy de ese servicio. **Es un secret, y vive en el alcance de su entorno** (quien la tenga despliega tu app):

```bash
gh secret set RENDER_HOOK_API_QA   --env qa     # te lo pide: pegás la URL del hook y Enter
gh secret set RENDER_HOOK_FRONT_QA --env qa
# Sin --body, a propósito: si escribís el valor en el comando, la URL secreta queda en el
# historial de tu terminal. Así gh te la pide y no queda en ningún lado.
# --env qa: son ENVIRONMENT secrets — solo los ve un job con environment: qa
# (los de PROD van en §3.4, dentro del environment production)
```

Agregá al final de tu `ci.yml` el job de deploy a QA — **la promoción arranca donde termina la verificación** (⚠️ donde diga `miapp-…onrender.com`, va **TU** URL de Render):

```yaml
  deploy-qa:
    runs-on: ubuntu-latest
    needs: [build-backend, build-frontend]     # sin CI verde no hay deploy
    if: github.ref == 'refs/heads/main'        # los PRs verifican; solo main despliega
    environment: qa
    steps:
      # 🔴 EL `&ref=` NO ES OPCIONAL, Y ES EL CORAZÓN DEL PRÁCTICO. Un hook
      # pelado despliega **la punta de la rama**, no el commit que acabás de
      # verificar. Con dos merges seguidos eso alcanza para que la corrida de A
      # despliegue B — y en el §3.4, para que apruebes A y suba B. «Se promueve
      # lo mismo que se verificó» deja de ser cierto justo donde el TP lo enseña.
      # El hook de Render ya trae un `?key=…`, así que va `&`, no `?`.
      - name: Disparar deploy en Render (backend + front) — del commit verificado
        env:
          HOOK_API: ${{ secrets.RENDER_HOOK_API_QA }}
          HOOK_FRONT: ${{ secrets.RENDER_HOOK_FRONT_QA }}
        run: |
          curl -fsS --max-time 30 "$HOOK_API&ref=$GITHUB_SHA"
          curl -fsS --max-time 30 "$HOOK_FRONT&ref=$GITHUB_SHA"

      # 🔴 Y COMPROBALO LA PRIMERA VEZ, en vez de creerle a esta guía: en el
      # primer deploy abrí los dos servicios en Render y mirá qué commit dice
      # que desplegó cada uno. Tiene que ser el tuyo, en los dos.
      #
      # 🚨 Y ACORDATE DE §3.2: este hook le pide a Render que VUELVA A CONSTRUIR
      # ese commit. Desplegás el commit verificado, pero no la imagen verificada.
      # Está mal a propósito, y el TP7 lo cambia por desplegar la imagen.

      - name: Smoke test (espera a que QA conteste)
        shell: bash        # el loop es bash — en runners Windows el default es PowerShell
        env:               # ⚠️ TUS URLs de Render: se escriben UNA vez y el loop usa las variables
          URL_API: https://miapp-api-qa.onrender.com
          URL_FRONT: https://miapp-front-qa.onrender.com
        run: |
          echo "Esperando a que QA esté vivo..."
          for i in $(seq 1 30); do
            # 🔴 EL `--max-time` VA ACÁ, que es donde puede colgarse. Un servicio
            # despertando de un cold start acepta la conexión y no contesta: sin
            # tope, `curl` no tiene límite y puede quedarse esperando en CADA
            # vuelta hasta que el job muera por el límite del runner — no por tu
            # entorno, y el error no lo dice.
            if curl -fsS --max-time 10 "$URL_API/health" \
               && curl -fsS --max-time 10 "$URL_API/api/tareas" > /dev/null \
               && curl -fsS --max-time 10 "$URL_FRONT/" > /dev/null; then
              echo "✅ QA responde (API + BD + front)"; exit 0
            fi
            sleep 20
          done
          echo "❌ QA no respondió a tiempo"; exit 1
```

🔴 **La mitad más importante de este job es `&ref=$GITHUB_SHA`, y conviene que la puedas explicar
sin mirar el archivo.** El hook pelado despliega **la punta de la rama**; con `&ref` le decís a
Render **qué commit** desplegar, y ése es el que tu pipeline acaba de verificar. Con dos merges
seguidos —que pasa seguido en un equipo— la corrida del primero desplegaría el segundo, que todavía
no pasó por nada: «se promueve lo mismo que se verificó» dejaría de ser cierto justo donde el
práctico lo enseña. Y va con `&` y no con `?` porque la URL del hook ya trae un parámetro, la clave.

Cuatro decisiones de diseño para tu `decisiones.md`:
- `needs:` + `if:` implementan la **cadena con compuertas** de §2.3. 📌 El job de PROD (§3.4) **no
  repite el `if` de la rama**, y no es un olvido: depende de `deploy-qa`, que sí lo tiene, así que en
  un Pull Request ninguno de los dos corre. La condición se hereda por la cadena.
- `environment: qa` conecta el job al environment: hereda sus secrets (los hooks) y suma al historial. QA no lleva reviewers — es automático **a propósito**.
- El smoke test **reintenta** (30 × 20 s): el contrato del free tier incluye el build de Render + cold start. Un curl seco daría falsos rojos. Y verifica **tres cosas**: `/health` (proceso vivo), `/api/tareas` (la BD responde — un `/health` que no toca la BD puede dar verde con la connection string rota) y el front servido.
- **Limitación honesta para tu `decisiones.md`**: el deploy hook responde al instante y Render buildea en background, y **mientras construye** —o si ese build **falla**— sigue sirviendo la versión anterior. Tu smoke puede dar verde contra la versión VIEJA: en la corrida de la cátedra, el verde llegó 33 segundos después del hook, con el build todavía en curso. La mitigación es una línea y **te la vas a agradecer**: que tu `/health` devuelva también el commit
que corre (una variable de entorno que la imagen recibe al construirse) y que el smoke la compare con
`github.sha`, fallando si no coincide. Es opcional acá y **obligatoria en el TP7**, donde es además
la única forma de probar desde afuera qué está corriendo. La otra opción es preguntarle a la API de
Render por el estado del deploy. Reconocer los límites del propio pipeline también es criterio profesional.

> ⚠️ Si un job referencia un environment inexistente, GitHub lo crea **sin reglas ni secrets** — por eso los creamos nosotros primero, con intención.

**✅ Checkpoint:** mergeás un PR → CI corre → `deploy-qa` dispara solo → el smoke test pasa → cuando Render termina el deploy (mirá el commit y el estado **live** en *Deploys*, en el menú de cada servicio; *Events* lista lo que pasó, pero el detalle del deploy está en *Deploys*), tu cambio está visible en la URL de QA. En el historial de **Deployments** del repo (link en el sidebar de la home, filtrable por environment) queda el registro del deploy.

Y probalo de punta a punta: abrí la URL del front de QA —ahora la lista carga, porque sus llamadas a
`/api` pasan por la plantilla—, agregá una tarea, recargá, y buscala en Neon sobre `app_qa`:
`select "Id", "Titulo" from "Tareas" order by "Id" desc;` — tiene que estar arriba. Front, api y base
de QA, conectados. (Si la primera carga del front da 504, es la api despertando: recargá.)

> 🎬 **Lo que se ve en el video, desde tu terminal**:
>
> ```bash
> gh secret list                              # los del repositorio (no los del environment)
> gh secret list --env qa                      # los secrets del environment (nunca el valor)
> git rev-parse HEAD                           # el SHA del commit en el que estás
> grep -n 'ref=' .github/workflows/ci.yml      # las dos líneas del &ref
> git add .github/workflows/ci.yml && git commit -m "…" && git push -u origin feature/deploy-qa
> gh pr create --fill
> gh pr checks --watch                         # espera los checks del PR y muestra cómo terminaron
>                                              # (en un PR, deploy-qa sale salteado: es el if de la rama)
> gh pr merge --squash --delete-branch         # mergea en un solo commit y borra la rama
> ```

### 3.4 El gate humano: environment `production` con required reviewers

1. *Settings → Environments → New environment* → `production`.
2. Activá **Required reviewers** y agregate **a vos mismo**. Dejá **desactivado** el checkbox *Prevent self-review* — es lo que te permite aprobar tus propios deploys (a diferencia de los PRs, acá GitHub lo permite; ese checkbox existe justamente para equipos que quieren prohibirlo). Guardá.
3. Los secrets de PROD, igual que los de QA, van **dentro de su environment**, y sin `--body`: `gh secret set RENDER_HOOK_API_PROD --env production` (ídem `RENDER_HOOK_FRONT_PROD`), o desde la web: *Settings → Environments → production → Environment secrets → Add secret*. Así, **solo** los jobs aprobados hacia `production` pueden leerlos — el alcance de secrets de §2.2, aplicado con una diferencia clave respecto de QA: a estos, sin aprobación, no los lee nadie.

El job de PROD — igual al de QA, con el environment protegido. Como siempre, en una rama
(`git switch main && git pull && git switch -c feature/deploy-prod`), por PR, y mergeado cuando los
checks estén en verde:

```yaml
  deploy-prod:
    runs-on: ubuntu-latest
    needs: deploy-qa                            # PROD solo después de QA vivo
    environment: production                     # ⏸ acá el workflow SE PAUSA hasta el approve
    steps:
      # 🔴 Y ACÁ MÁS QUE EN NINGÚN LADO: entre que la corrida queda esperando
      # aprobación y que alguien la aprueba pueden pasar horas, y `main` se
      # mueve. Sin el `&ref=`, aprobás la corrida del commit A y a PROD sube lo
      # último que haya. El aprobador estaría firmando algo que no se despliega.
      - name: Disparar deploy en Render (backend + front) — del commit aprobado
        env:
          HOOK_API: ${{ secrets.RENDER_HOOK_API_PROD }}
          HOOK_FRONT: ${{ secrets.RENDER_HOOK_FRONT_PROD }}
        run: |
          curl -fsS --max-time 30 "$HOOK_API&ref=$GITHUB_SHA"
          curl -fsS --max-time 30 "$HOOK_FRONT&ref=$GITHUB_SHA"

      - name: Smoke test PROD
        shell: bash
        env:               # ⚠️ TUS URLs de PROD: el loop es el mismo que el de QA
          URL_API: https://miapp-api-prod.onrender.com
          URL_FRONT: https://miapp-front-prod.onrender.com
        run: |
          for i in $(seq 1 30); do
            # 🔴 EL `--max-time` VA ACÁ, que es donde puede colgarse. Un servicio
            # despertando de un cold start acepta la conexión y no contesta: sin
            # tope, `curl` no tiene límite y puede quedarse esperando en CADA
            # vuelta hasta que el job muera por el límite del runner — no por tu
            # entorno, y el error no lo dice.
            if curl -fsS --max-time 10 "$URL_API/health" \
               && curl -fsS --max-time 10 "$URL_API/api/tareas" > /dev/null \
               && curl -fsS --max-time 10 "$URL_FRONT/" > /dev/null; then
              echo "✅ PROD responde (API + BD + front)"; exit 0
            fi
            sleep 20
          done
          exit 1
```

Al llegar acá, el run muestra **"Waiting for review"**: te llega la notificación, entrás al run, ves *Review deployments* y decidís — aprobar o rechazar, con comentario. Practicá **los dos caminos**:
- **Aprobación**: deploy-prod corre → smoke verde → el cambio está en PROD. Queda a la vista en *Actions* y en *Deployments*: no hace falta capturarlo.
- **Rechazo** (evidencia obligatoria del TP): rechazá un deploy **con un motivo tuyo y específico**, escrito en el comentario: qué viste en **esa** corrida que te hizo decir que no. 🔴 El video usa «viernes siete de la tarde, no desplegamos» como ejemplo; ése ya está tomado — si nos llegan veinte rechazos con esa frase, ninguno prueba que alguien miró algo. Sirven: «el smoke tardó tres vueltas y quiero revisar por qué», «este PR toca la connection string y prefiero mirarlo con tiempo», «entra otro merge en cinco minutos, apruebo los dos juntos». El run queda como *failed* en ese job: la máquina obedeció al humano. Rechazarte a vos mismo parece raro — pero es exactamente el criterio que el gate te pide practicar: la evidencia dice que QA no estaba OK, y el deploy no sale.

Cada camino necesita **su propia corrida**: la rechazada ya no se aprueba. Hacelos en este orden, que
es el del video: **primero el rechazo** —así la evidencia obligatoria queda hecha— y después mergeá
otro Pull Request para aprobar.

🔴 **Y que ese segundo cambio SE VEA en la pantalla**: cambiá un texto del front (el título, el
subtítulo, el texto de un botón), no un comentario del YAML. Es lo que te permite abrir la URL de
PROD después de aprobar y mostrar que el cambio llegó — y es lo que se te va a pedir en la defensa.
Un cambio invisible te deja sin nada que mostrar.

> ⚠️ **Si aprobás una corrida vieja después de una nueva, PROD retrocede.** Con dos merges seguidos,
> dos corridas pueden quedar esperando tu aprobación a la vez, y aprobar la de abajo manda a PROD un
> commit anterior al que ya está.
> - **La regla, y es tuya: rechazá la vieja a mano**, con su motivo. Es lo único que funciona siempre.
> - `concurrency: { group: deploy-prod, cancel-in-progress: false }` en el job ayuda, pero **no te
>   cubre este caso**: la documentación de GitHub dice que una corrida nueva cancela a la que está
>   *en cola* esperando un carril — y un job detenido **esperando aprobación** no está en cola, está
>   en otro estado. Lo reportado es que la vieja sobrevive y la nueva queda bloqueada detrás.
>   🔴 La cátedra **no lo midió**: lo que sabemos sale de la documentación y de los foros de GitHub
>   (discusión 17401). Poné el `concurrency` igual —evita dos deploys a PROD pisándose—, pero no
>   cuentes con que te ordene la cola de aprobaciones.

**✅ Checkpoint:** en *Actions* se ve el flujo completo (QA automático → esperando aprobación → aprobado → PROD verde) **y** una corrida rechazada con su motivo.

#### Así quedó tu `ci.yml`

Lo fuiste armando por partes, y nunca lo viste entero. No te damos el archivo completo a propósito
—si lo copiás, no lo entendés— pero sí el **mapa**, para que compruebes la estructura:

```
build-backend    (sin needs)                                    ← del TP4, + publicar la imagen (§3.0)
build-frontend   (sin needs)                                    ← ídem
deploy-qa        needs: [build-backend, build-frontend]         · environment: qa          (§3.3)
                 if: solo en main · smoke con reintentos
deploy-prod      needs: deploy-qa                               · environment: production  (§3.4)
                 if: solo en main · concurrency: deploy-prod
```

Cuatro jobs, todos al mismo nivel de indentación dentro de `jobs:`. Si te queda uno anidado adentro
de otro, GitHub no se queja con un error claro: simplemente ese job **no aparece** en la corrida. Es
el error más común de este TP, y se ve mirando la lista de jobs en *Actions*: tienen que estar los
cuatro.

> 🎬 **Lo que se ve en el video, desde tu terminal** (§3.4):
>
> ```bash
> gh secret set RENDER_HOOK_API_PROD --env production   # y el del front: te pide el valor
> gh secret list --env production                       # los dos, con su fecha; el valor nunca
> git switch main && git pull && git switch -c feature/deploy-prod
> git --no-pager diff -U1                               # el job de PROD entero, en verde
> gh pr checks                                          # cómo terminaron los checks del PR
> gh pr merge --squash --delete-branch
> ```

### 3.5 La release: el tag de entrega y sus notas

Con PROD actualizado, etiquetá **lo que efectivamente llegó**:

```bash
git switch main && git pull                    # los merges se hicieron en GitHub: traelos
git tag v6.0.0 <sha-del-commit-desplegado>     # v6 = TP6; la regla del semestre, no una elección
# el sha sale de Deployments (§3.3), no de la punta de main: entre la aprobación y este
# comando pudo entrar otro merge, que nadie aprobó todavía
git push origin v6.0.0
gh release create v6.0.0 --generate-notes --verify-tag
```

> 🔴 **Es `v6.0.0`, no `v1.0.0`.** Cada práctico cierra con su número (el encabezado de esta guía lo
> fija), y `v1.0.0` ya existe: es tu TP1. Si copiás el comando con el número viejo, `git tag` falla
> con *«already exists»* y el error no menciona nada de esto.

`--generate-notes` arma las notas desde los PRs mergeados desde la release anterior (los buenos mensajes que venís escribiendo desde el TP1 — que con squash se vuelven el título del PR — acá cobran). ¿El `<sha-del-commit-desplegado>`? Lo ves en el run del deploy o en el historial de Deployments del repo (§3.3).

> 📌 **¿Y por qué esto a mano, si todo lo demás lo hace el pipeline?** Por dos motivos, y los dos
> son del práctico, no de la herramienta. Primero, el **orden**: el tag tiene que apuntar a lo que
> llegó a PROD, y PROD sólo se actualiza después de que vos aprobaste. Segundo, `v6.0.0` **no es una
> versión de producto**: es la marca de cierre del práctico, y la pone una persona cuando decide
> «esto es lo que entrego».
> En la industria lo normal es automatizarlo —último paso del job de PROD, después del smoke:
> taggear el commit desplegado y crear la release—. Si querés hacerlo, es un paso más con el
> `GITHUB_TOKEN` que ya tenés. Lo que no se hace **nunca**, ni a mano ni automático, es taggear la
> punta de la rama.

**✅ Checkpoint:** la release `v6.0.0` publicada en GitHub, con notas, apuntando al commit que está corriendo en PROD.

> 🔄 **Y con esto ya tenés el rollback, que es la otra mitad de la Tarea 6.** Como el deploy se
> dispara con el commit explícito (`&ref=$GITHUB_SHA` del §3.3), volver atrás es **disparar los mismos
> hooks con el sha del último deploy bueno anterior** — lo ves en *Deployments* → `production`, y
> cuando tengas releases, es el de la anterior:
>
> ```bash
> gh run list --workflow=ci.yml --limit 3            # las últimas corridas, con su commit
> export SHA_ANTERIOR=<sha-del-deploy-bueno-anterior>
> read -rs HOOK_API_PROD && export HOOK_API_PROD      # pegás el Deploy Hook de tu api de PROD (Render → Settings) y Enter: no queda en el historial
> read -rs HOOK_FRONT_PROD && export HOOK_FRONT_PROD  # ídem, el del front de PROD
> INICIO=$(date +%s)
> curl -fsS "$HOOK_API_PROD&ref=$SHA_ANTERIOR"; echo
> curl -fsS "$HOOK_FRONT_PROD&ref=$SHA_ANTERIOR"; echo
> # … cuando Render muestre ese commit como live en los dos servicios:
> echo "rollback: $(( $(date +%s) - INICIO )) s"
> ```
>
> El secret de GitHub no se puede volver a leer: el hook lo copiás otra vez de Render. ⚠️ **No uses
> `v5.0.0`**: ese commit es anterior a la plantilla de nginx y a los jobs de deploy, y en Render deja
> el front sin backend. Para practicarlo: después de la release `v6.0.0`, mergeá y aprobá un cambio
> más que se note (en el video, `/health` pasa a informar la versión), y volvé a `v6.0.0` con
> `export SHA_ANTERIOR=$(git rev-list -n1 v6.0.0)`.
>
> El cronómetro corre desde que llamás al hook hasta que en Render → tu servicio de PROD →
> **Deploys** el deploy de ese commit figura como **live** (la hora está en cada fila; *Events*
> también lo lista, pero el estado lo manda *Deploys*). No lo pares con el `/health`: contesta verde con la
> versión que estabas revirtiendo.
>
> 📌 **El video cronometra en *Events*, y está bien**: las dos solapas dan la misma hora. La
> diferencia es que *Deploys* además te dice en qué estado quedó el deploy, y **es la que miramos en
> la defensa** — así que acostumbrate a ésa.
>
> También está la interfaz: *Actions* → la corrida de ese deploy → el job **`deploy-prod`** → re-correr
> sólo ese job (no *Re-run all jobs*, que también redespliega QA). Pero vuelve a pedir la aprobación
> —es un deploy a producción: el gate actúa igual— y GitHub sólo deja re-correr corridas de hasta
> **30 días**. El que siempre funciona es el hook con el SHA. Cronometralo una vez y poné el número en
> `decisiones.md`: «cuánto tarda» es parte de lo que se pide. 🔴 Y anotá lo que el rollback de código
> **no** deshace: una migración que borró una columna sigue borrada. Ésa es la respuesta que la
> defensa busca.

> 🎬 **Lo que se ve en el video, desde tu terminal** (§3.5):
>
> ```bash
> git switch main && git pull
> # ¿qué commit está en PROD? se lo preguntás a Deployments, no a la punta de la rama:
> export SHA_PROD=$(gh api 'repos/{owner}/{repo}/deployments?environment=production' --jq '.[0].sha')
> echo $SHA_PROD
> git tag v6.0.0 $SHA_PROD                    # el tag marca ESE commit, no «lo último»
> git push origin v6.0.0                      # sin este push, en GitHub no existe
> gh release create v6.0.0 --generate-notes --verify-tag
> git tag --list "v*"                         # tus releases, para el rollback
> export SHA_ANTERIOR=$(git rev-list -n1 v6.0.0)
> read -rs HOOK_API_PROD && export HOOK_API_PROD    # lo copiás de Render: el secret no se relee
> date +%T                                    # la hora del disparo, para comparar con Render
> ```

### 3.6 Fallback local (plan C garantizado — y entrega válida)

Sin tarjeta, sin nube, mismos conceptos: tu máquina como runner + compose como entornos.

> 🚨 **Plan C NO es «levanto el compose a mano y lo muestro andando».** Eso es el TP2, y acá vale
> **cero**: lo que se evalúa es que **el pipeline despliegue**, no que la app corra. Para que el plan
> C cuente, tiene que pasar exactamente lo mismo que en la nube, y se ve en *Actions*:
> 1. El deploy lo dispara **una corrida** de GitHub Actions sobre tu **self-hosted runner** — vos no
>    ejecutás ningún `docker compose up`: lo ejecuta el job.
> 2. **Dos entornos separados y simultáneos**, cada uno su proyecto de Compose, sus puertos y su
>    **base de datos propia** (volúmenes distintos), front y back como contenedores.
> 3. **QA automático** al mergear, con **smoke test con reintentos** que puede poner el job en rojo.
> 4. **PROD detrás del environment `production`** con required reviewer: aprobación y rechazo con
>    motivo, en GitHub, igual que en la nube.
> 5. **En la defensa lo vas a ver arrancar**: mergeás un cambio chico delante nuestro y tus dos
>    entornos se actualizan **solos**, disparados por la corrida. Si hay que levantar algo a mano
>    para que la demo funcione, el plan C no está cumplido.
>
> Y decilo en `decisiones.md`: por qué usaste el plan C y qué pierde respecto de la nube (URLs
> públicas, entorno ajeno a tu máquina, cold start real).

1. **Registrá el runner**: *Settings → Actions → Runners → New self-hosted runner* → seguí los comandos (`./config.sh --url … --token …` y `./run.sh`; en Windows, `config.cmd`/`run.cmd`). El runner queda escuchando jobs de TU repo.
2. **Dos entornos compose**: copiá tu `docker-compose.yml` a `compose.qa.yml` y `compose.prod.yml`, con **tres cambios quirúrgicos** cada uno:
   - **`name:` como clave top-level** (`name: qa` / `name: prod`). ⚠️ Este es EL paso que todos saltean: sin él, Compose usa el directorio como project name y **el deploy de PROD recrea (mata) los contenedores de QA** — probálo y lo ves. Con `name:` distinto, son dos proyectos aislados de verdad.
   - **Puertos — solo el lado HOST del mapping** (`host:contenedor`): en QA quedan `"8080:8080"` y `"3000:80"`; en PROD pasan a `"8081:8080"` y `"3001:80"`. El lado derecho NO se toca: adentro del contenedor el backend sigue escuchando 8080 y nginx 80 — si cambiás el lado interno, el contenedor "levanta" pero nadie escucha ahí.
   - Si tu compose tiene **`container_name:`**, sacalo o hacelo distinto por entorno: con el mismo nombre, QA y PROD chocan (*«container name already in use»*) aunque tengan `name:` distinto.
   - **Nombres de volumen** distintos (`db_data_qa` / `db_data_prod`), para que cada entorno tenga sus datos. (El `POSTGRES_DB` puede quedar igual en ambos: el aislamiento real lo dan el proyecto y el volumen separados — si lo cambiás, acordate de cambiar también `Database=` en la connection string del mismo archivo, o tu app va a seguir escribiendo en la BD vieja.)
3. 📌 **La plantilla de nginx de la Tarea 2 sigue siendo obligatoria, aunque acá no se note.** En
   local, el front le habla al backend por el nombre del servicio (`http://backend:8080`), y como
   cada entorno es un proyecto de Compose aparte, **cada uno tiene su propio `backend`** en su
   propia red: la misma dirección resuelve bien en QA y en PROD. Eso NO quiere decir que puedas
   hornearla adentro de la imagen y sacar la plantilla — la Tarea 2 la pide igual, y es lo que hace
   que una misma imagen sirva en dos entornos. Acá simplemente recibe el mismo valor en los dos.

4. **El password de la BD tiene que viajar al runner**: tu `.env` local está gitignoreado (bien), así que el checkout del runner NO lo trae y el compose fallaría con "superuser password is not specified". Se resuelve como todo en esta materia: secret + env:

```bash
gh secret set DB_PASSWORD --body "un-password-para-los-entornos-locales"
```

5. **Los jobs de deploy cambian solo el "cómo"**:

```yaml
  deploy-qa:
    runs-on: self-hosted                       # ← tu máquina
    needs: [build-backend, build-frontend]
    if: github.ref == 'refs/heads/main'
    environment: qa
    env:
      DB_PASSWORD: ${{ secrets.DB_PASSWORD }}  # el .env no viaja: el secret sí
    steps:
      - uses: actions/checkout@v6
      - name: Levantar QA local
        run: docker compose -f compose.qa.yml up -d --build
      - name: Smoke test
        shell: bash          # en un runner Windows el default es PowerShell (y su curl no es curl)
        env:
          URL_API: http://localhost:8080
          URL_FRONT: http://localhost:3000
        run: |
          for i in $(seq 1 30); do
            if curl -fsS --max-time 10 "$URL_API/health" \
               && curl -fsS --max-time 10 "$URL_API/api/tareas" > /dev/null \
               && curl -fsS --max-time 10 "$URL_FRONT/" > /dev/null; then
              echo "✅ QA responde (API + BD + front)"; exit 0
            fi
            sleep 20
          done
          echo "❌ QA no respondió a tiempo"; exit 1
```

(`deploy-prod` idéntico con `compose.prod.yml`, puertos 8081/3001 y `environment: production` — **la aprobación funciona exactamente igual**: las protection rules viven en GitHub, no en el destino. En Windows, los steps con `shell: bash` requieren Git Bash — viene con Git.)

> ⚠️ **Seguridad del runner — en serio**: GitHub **desaconseja** self-hosted runners en repos públicos, porque un fork malicioso podría intentar ejecutar código en tu máquina vía PR. Mitigación para la materia: (1) en *Settings → Actions → General*, poné **"Require approval for all external contributors"** (el default solo frena a los primerizos); (2) no aceptes PRs de desconocidos mientras el runner esté activo; (3) apagalo cuando no lo uses (`Ctrl+C`). Con el flujo de la materia (vos como único contributor) el riesgo es manejable — pero tenés que poder EXPLICARLO en la defensa.

**✅ Checkpoint (fallback):** mismos checkpoints de §3.3–§3.4 con URLs `localhost:8080/8081` — promoción, aprobación, rechazo e historial idénticos. 🔴 Y el que manda: **los dos entornos los levantó una corrida de Actions, no vos** — se prueba abriendo el job `deploy-qa` en *Actions* y viendo ahí el `docker compose up` con su salida, sobre el runner `self-hosted`.

## 4- Riel alternativo: Azure Pipelines

| Concepto | GitHub Actions (canónico) | Azure Pipelines |
|---|---|---|
| Entornos | Environments (`qa`, `production`) | **Environments** (Pipelines → Environments) |
| Aprobación manual | Environment → Required reviewers | Environment → **Approvals and checks** → Approvals |
| Secrets por entorno | Environment secrets | Variable groups vinculados + service connections |
| Cadena de promoción | jobs con `needs:` + `environment:` | **Stages** (`stage: DeployQA` → `stage: DeployProd`) con `dependsOn` |
| Destino de deploy | Render (hooks) / self-hosted | 2 **Azure Web Apps F1** (task `AzureWebApp@1` + service connection) |
| Historial de deploys | Link Deployments (sidebar de la home del repo) | Environments → Deployments |
| Release/tag | `gh release create` + el tag del práctico | Tags de Git + (opcional) release notes en wiki/pipeline |
| Costo | $0 (Render/Neon free + Actions público) | Web Apps F1 gratis (60 min CPU/día, sin SLA, sin dominio propio) — pipeline: ver advertencia de minutos del TP4 |

**Checkpoints riel Azure:** los mismos 6 (BD/entornos arriba → deploy QA automático post-CI → smoke test con reintentos → aprobación en PROD + rechazo documentado → release etiquetada → fallback local disponible). Base útil: tu material AZ-400 de Web Apps (guía 2025 TP05-4A) sigue siendo válido para crear las F1.

> 📌 Recordatorio 2026: los minutos hosted de Azure Pipelines exigen org vinculada a suscripción; sin tarjeta, self-hosted agent — que además te deja TODO este TP funcionando local.

---
---

# 📋 Trabajo Práctico 06 – CD: environments, aprobaciones y deployment patterns (2026)

## ⚠️ Este es el TP que debés entregar y defender

## 🎯 Objetivo

Que tu pipeline deje de solo verificar y empiece a **entregar**: cada cambio integrado llega automáticamente a un entorno QA real, y a producción **solo** con aprobación humana explícita — con smoke tests, historial, release versionada y plan de rollback.

Este trabajo se aprueba **solo si podés explicar qué hiciste, por qué lo hiciste y cómo lo resolviste**.

## 🧩 Escenario

Tu app tiene CI con calidad (TP4/TP5), pero el "deploy" sigue siendo artesanal: cada demo se levanta a mano en la notebook de alguien, dos veces se mostró una versión vieja, y la última vez el "ambiente de prueba" era la máquina del que faltó. El equipo decidió profesionalizar la entrega: **un QA siempre actualizado solo, una producción a la que únicamente se llega con verificación + aprobación**, y una respuesta clara a "¿qué versión está corriendo y cómo volvemos atrás?".

## 📋 Tareas que debés cumplir

### 1. El artefacto: el pipeline entrega lo que verificó
- **Las dos imágenes publicadas** en un registry (`ghcr.io` canónico), **etiquetadas con el commit**
  que las produjo, y publicadas **sólo cuando el cambio entra a `main`**.
- **Paquetes públicos**: se tienen que poder bajar con `docker pull` desde cualquier máquina, sin
  credenciales.
- **La cadena mostrada**, que es la evidencia de que el registry es confiable. Son **dos enlaces**
  —no capturas: las dos cosas tienen dirección propia, y desde el TP3 lo que tiene URL se entrega
  como URL—:
  1. **La corrida de Actions de un PR con los tests en verde** —por ejemplo, el del §3.0—, abierta en
     el job que publica, donde «Entrar al registry» aparece **salteado**. Pasó todo y aun así no
     publicó: muestra el segundo eslabón, sólo `main` publica.
     (Ojo con dos atajos que no prueban nada: en un job con los tests en **rojo** los pasos que
     publican salen salteados por el error, no por el `if`; y buscar el commit del PR en el listado del paquete
     tampoco sirve, porque con *squash merge* el commit de la rama no llega al registry ni siquiera
     cuando el PR se mergea.)
  2. 🔴 **La lista de pasos de una corrida de `main`**, donde se vea que **«construir y publicar la
     imagen» es el ÚLTIMO de tus pasos**, después de los tests (después sólo aparecen los `Post …` que
     agrega GitHub). Ésta es la que prueba el tercer eslabón, y
     es la única que distingue un pipeline bien armado de uno que publica antes de testear — la
     primera sale igual en los dos casos, porque ningún PR publica, esté bien armado el orden o no.

> 📌 **¿Tu app tiene un solo Dockerfile?** Se entrega **un solo paquete** y no se descuenta nada.
> Decilo en `decisiones.md` en una línea. Lo que **no** vale es inventar un segundo job o un
> segundo paquete vacío para llegar al número.

> 📌 **Por qué no se pide «una corrida en rojo que no publicó»**: con este diseño ese caso no puede
> ocurrir, y ésa es justamente la gracia. Nada llega a `main` sin verde y sólo `main` publica, así
> que la garantía se lee en la configuración en vez de escenificarse. Lo que se muestra es la
> cadena, no un accidente.

### 2. Dos entornos reales
- **QA y PROD desplegados y accesibles por URL** (Render+Neon, tu nube, o el fallback local documentado — cualquiera vale, con el mismo contrato). 🚨 En los tres casos, **el deploy lo dispara el pipeline**: un `docker compose up` hecho a mano no cuenta como entorno desplegado (§3.6).
- App completa en cada entorno (front + back + BD **separada por entorno**).
- 🔴 **La dirección del backend NO va adentro de la imagen del front**: se lee del entorno al
  arrancar (la plantilla de nginx del §3.3). La prueba es que **la misma imagen** sirve en QA y en
  PROD y cada una habla con su api. Contá en `decisiones.md` qué quedó por variable y qué quedó
  adentro de la imagen — es lo que hace posible el TP7.
- Environments `qa` y `production` creados en GitHub, con los **secrets de deploy en su alcance correcto** (los de PROD, dentro del environment).

### 3. Promoción automática a QA
- Push a `main` → CI verde → **deploy automático a QA** (disparado por el pipeline, no por auto-deploy del proveedor) → **smoke test post-deploy con reintentos** que falla el job si el entorno no responde.
- 🔴 **Y tiene que poder comprobarse, no declararse.** En la defensa vamos a mirar, en tu pantalla:
  (a) **Auto-Deploy en `Off`** en los cuatro servicios —si está prendido, tus entornos se actualizan
  solos y tu pipeline es decorado—; (b) que los **deploy hooks de producción apunten a los servicios
  de producción** (los abrís en Render y comparamos el id del servicio); y (c) que las dos bases sean
  **distintas de verdad**: creás un dato sólo en `app_prod` desde el SQL Editor y mostrás que en QA
  **no** aparece (§3.1). Las tres cosas llevan dos minutos y son las que separan un entorno de una
  apariencia de entorno.

### 4. El gate humano hacia PROD
- Environment `production` con **required reviewer** (vos mismo, con *Prevent self-review* desactivado).
- **Evidencia del flujo completo**: QA verde → "Waiting for review" → aprobación → PROD desplegado y smoke verde.
- **Evidencia de un rechazo** con motivo escrito: el gate tiene que haber dicho "no" al menos una vez.
  🔴 El motivo tiene que ser **tuyo y específico** —qué viste en esa corrida—, no el ejemplo de esta
  guía: dos rechazos con la misma frase que el video se leen como lo que son.

### 5. Release versionada
- La llegada a PROD etiquetada con el tag de cierre del práctico —**`v6.0.0`**, la regla del semestre— y publicada como **Release de GitHub con notas**.
- 📌 El número **no se elige**: lo fija el práctico. Versionar por lo que cambió —SemVer, §2.6— es otra cosa, y en este TP no se entrega nada sobre eso.

### 6. Estrategia y rollback (en `decisiones.md`)
- Qué **deployment pattern** usarías para esta app en una producción real con usuarios (blue-green / canary / rolling / flags) y **por qué ese** — costo, riesgo, rollback y qué observabilidad te falta hoy para ejecutarlo.
- Tu **plan de rollback actual**: si el deploy aprobado sale mal, ¿qué hacés, paso a paso, y cuánto tarda?
- 🔴 **El número es medido, no estimado**: hacelo una vez de verdad (§3.5) y dejá el rastro que lo
  prueba — en Render, *Deploys* del servicio de producción muestra el deploy del commit anterior con
  su hora de inicio y su hora de *live*. Esas dos horas son tu número, y son lo que vamos a mirar en
  la defensa; un «unos minutos» escrito de memoria no cuenta.

## 📄 Entregables

1. **URL del repositorio público** (formulario de la cátedra). **Es lo único que va al formulario**, como en el TP5: todo lo demás se llega desde ahí.
2. **En `decisiones.md`, arriba de todo, una sección «Enlaces de este TP»** con: (a) los **dos
   paquetes públicos** del registry, los dos etiquetados con el commit de un merge (el `docker pull`
   tiene que funcionar sin credenciales, desde cualquier máquina); (b) los **dos enlaces de la
   cadena** de la Tarea 1 (el job del PR con «Entrar al registry» salteado, y la lista de pasos de la
   corrida de `main`); y (c) **las URLs de QA y PROD**, vivas hasta la defensa, porque en P2 se navegan
   en vivo. Los paquetes y las URLs viven afuera del repo, y
   las dos corridas son dos entre muchas: si no están ahí, la corrección no las busca.
3. **`decisiones.md`** (acumulativo) explicando:
   - Por qué el artefacto se publica **sólo** con la verificación en verde, y qué dejaría de
     significar el registry si se publicara igual.
   - Continuous Delivery vs Deployment: cuál implementaste y por qué corresponde a tu contexto.
   - El diseño de la cadena (`needs`/`if`/environments) y el alcance de cada secret.
   - Qué mira tu aprobador antes de aprobar (los criterios del gate).
   - La letra chica de tu free tier (cold start, sleep, horas y minutos de build) y cómo la maneja tu pipeline.
   - Qué garantía perdés porque Render **reconstruye** desde el repo en vez de usar tu imagen (§3.2).
   - Qué prueba tu smoke test y qué **no**: contesta, pero no dice qué versión corre (§3.3).
   - Deployment pattern elegido para producción real + plan de rollback (Tarea 6).
   - Problemas encontrados y cómo los resolviste. Declaración de uso de IA.
> 📌 **En este TP tampoco hay `evidencias.md`**, igual que desde el TP3 y por el mismo motivo: tu
> repositorio es público, así que quien corrige abre *Actions* y ve el deploy a QA disparándose tras
> el merge, el smoke reintentando, el `Waiting for review`, la aprobación y **el rechazo con su
> motivo**; abre *Deployments* y ve qué commit está en cada entorno y desde cuándo; y abre
> *Releases* y ve la `v6.0.0` con sus notas. Sacar capturas de eso es duplicar lo que ya está a la
> vista.
>
> 🔴 **Lo único que hay que enlazar es lo que no se encuentra navegando**: los paquetes, las URLs de
> QA y PROD, y las dos corridas de la cadena. Por eso van en la sección «Enlaces de este TP» del
> punto 2 — si no están ahí, la corrección no los busca.

## 🗣️ Defensa Oral Obligatoria

Se realiza en **P2**, junto con los TPs 5 a 9. Vas a mostrar tu trabajo y responder preguntas como:
- ¿Qué produce tu pipeline y dónde queda? Mostrame la imagen publicada y decime de qué commit salió.
- ¿Cómo probás que lo que está publicado es exactamente lo que pasó la verificación? (Pista: no hay
  un control que lo garantice — hay **tres** que, encadenados, lo hacen imposible por accidente:
  nada entra a `main` sin verde, sólo `main` publica, y publicar es el último paso del job que
  testea.)
- CI, Continuous Delivery, Continuous Deployment: definí las tres y decime cuál implementaste. ¿Qué te faltaría para la tercera? ¿La querrías?
- ¿Qué es un environment de GitHub? Nombrá las tres cosas que te da y mostrámelas en TU repo.
- ¿Por qué QA se despliega solo y PROD no? ¿Qué "compra" la aprobación humana? ¿Cuándo NO agregaría valor?
- ¿Por qué los secrets de PROD viven en el environment y no en el repo? ¿Qué pasaría si estuvieran en el repo?
- ¿Qué es un smoke test? ¿Por qué el tuyo reintenta? Mostrame qué pasa si el deploy "termina" pero la app no responde.
- Mostrame el rechazo documentado: ¿por qué se rechazó y qué pasó con ese run?
- Blue-green vs canary: explicá ambos, y por qué elegiste el de tu `decisiones.md` para TU app. ¿Qué te falta hoy para hacer canary en serio?
- Deploy ≠ release: explicalo con feature flags. ¿Para qué usarías un flag en tu app?
- PROD está roto tras un deploy aprobado: ¿qué hacés? (paso a paso, con tiempos). ¿Qué métrica DORA estás ejercitando?
- ¿Por qué `v1.2.0` y no `v2.0.0`? ¿Qué prometés con ese número?
- Si Render desaparece mañana, ¿qué partes de tu TP sobreviven sin cambios y cuáles migran? (pista: casi todo sobrevive — ¿por qué?)

## ✅ Evaluación

| Criterio | Peso |
|---|---|
| Configuración técnica (entornos, promoción, gate, smoke, release, corridas navegables) | 25% |
| Claridad y justificación en `decisiones.md` | 25% |
| Defensa oral: comprensión y argumentación | 50% |

> ⚖️ Peso orientativo de este TP en la nota de **P2**: **25%** (la ponderación completa de los 9 TPs está en el reglamento, §5).

**Cómo se distingue un 4 de un 8.** Los mínimos de las tareas te dan el 4: están o no están. Lo que
mueve la nota de ahí para arriba es el criterio, y se ve en estas cuatro cosas:

| | **Suficiente (4-5)** | **Bien (6-7)** | **Muy bien (8-10)** |
|---|---|---|---|
| **El artefacto** | Las imágenes están publicadas y se bajan | Están etiquetadas por commit y explicás la cadena de tres eslabones | Además explicás qué **no** garantiza (un `docker push` a mano entra igual) y qué sería un digest |
| **La promoción** | QA se despliega solo después del merge | El deploy lleva **el commit verificado**, no la punta de la rama, y sabés por qué importa | Además contás qué pasa entre dos merges seguidos y cómo lo comprobaste |
| **El gate** | Hay required reviewer, un «Waiting for review» y **el rechazo con su motivo** — los tres los pide la Tarea 4 | Además decís qué mirás antes de aprobar: los criterios, no el trámite | Además: qué NO puede ver tu aprobador, y qué observabilidad te falta para decidir mejor |
| **El rollback** | Hay un plan escrito **y cronometrado**, como pide la Tarea 6 | Lo probaste de verdad una vez, y el número sale de esa prueba | Además distingue lo que el rollback de código no deshace (los datos) y qué haría falta para eso |
| **La defensa** (50%) | Contesta lo conceptual: qué es un artefacto, qué es un environment | Muestra **su** repo y explica **sus** decisiones cuando se le pregunta | Sostiene la repregunta: por qué ESE patrón, qué garantiza su cadena y qué no, y qué haría distinto |

📌 **Cómo se usa**: la columna «Suficiente» es lo que ya piden las tareas — con eso se aprueba. Las
otras dos no piden **más trabajo**, piden **más criterio** sobre el mismo trabajo, y casi todo se ve
en `decisiones.md` y en la defensa.

## ⚠️ Uso de IA

Podés usar IA (ChatGPT, Copilot, Claude), pero **deberás declarar en `decisiones.md` qué parte fue asistida por IA** y justificar cómo la verificaste. En este TP en particular: si la IA te escribió el workflow de deploy, tenés que poder explicar **qué pasa exactamente entre el merge y PROD** — cada compuerta, cada secret, cada espera. Si no podés defenderlo, **no se aprueba**.
