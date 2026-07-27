# Smoke test de vigencia — free tiers y versiones citados en las guías

> **Documento operativo de la cátedra (público por transparencia).** Los servicios gratuitos y las versiones que las guías citan **mutan** — y en las dos direcciones (2026 lo probó: Koyeb empezó a pedir tarjeta; Render Postgres bajó de 90 a 30 días; UptimeRobot *revirtió* su restricción no-comercial). Este checklist se corre **completo antes de arrancar cada cohorte** y **la fila correspondiente antes de publicar cada TP**.
>
> **Método**: verificar cada claim contra la **fuente oficial** de la columna "Dónde verificar" (no contra blogs). Si un claim cambió: (1) actualizar la guía afectada, (2) actualizar esta tabla con la fecha, (3) si el riel canónico queda comprometido → activar/documentar el fallback local del TP correspondiente.
>
> **Última corrida completa: 2026-07-26** (durante la autoría de C5–C9, con fact-checkers contra docs oficiales).

## 1. GitHub (afecta a TODOS los TPs)

| Claim en las guías | TP(s) | Dónde verificar | Estado 2026-07-26 |
|---|---|---|---|
| Actions **ilimitado y gratis en repos públicos** | TP4+ | docs.github.com → billing for Actions | ✅ |
| Branch protection + `enforce_admins` gratis en públicos | TP1 | docs.github.com → branch protection | ✅ |
| Environments con **required reviewers gratis en públicos** (privados: Enterprise) | TP6 | docs.github.com → deployment environments | ✅ |
| Secret scanning + **push protection always-on** en públicos | TP9 | docs.github.com → secret scanning | ✅ |
| Dependabot gratis (alertas + PRs; transitivas según grafo/lockfile) | TP9 | docs.github.com → Dependabot | ✅ |
| CodeQL default setup gratis en públicos; action **@v4** (v3 muere dic-2026) | TP5, TP9 | github.blog/changelog | ✅ — ⚠️ re-chequear el major en 2027 |
| ghcr.io: storage/bandwidth gratis, packages nacen privados | TP2, TP7 | docs.github.com → Container registry billing | ✅ ("currently free" — lenguaje no garantizado) |
| Docker Hub (alternativa de registry del TP2): push/pull público sin tarjeta; **límites de pull** vigentes para anónimos/free | TP2 | docs.docker.com → usage/pricing | ✅ — ⚠️ historial de cambios en rate limits: re-leer |
| Majors de actions usados: `actions/*@v6` · `docker/login@v4` · `docker/build-push@v7` · `docker/setup-buildx@v4` · `hashicorp/setup-terraform@v4` | TP4–TP8 | releases de cada repo | ✅ — ⚠️ existe v7 de `actions/*`: evaluar bump coordinado de TODO el material en la próxima cohorte |

## 2. Render (TP6, TP7)

| Claim | Dónde verificar | Estado 2026-07-26 |
|---|---|---|
| Web services free **sin tarjeta**; sleep a los 15 min; despertar ~1 min; **750 hs de instancia/mes por workspace** | render.com/docs/free — ⚠️ el "sin tarjeta" NO está textual en la doc: verificarlo con signup en vivo (cuenta `ingsoft3ucc`) | ✅ |
| **Postgres free EXPIRA a los 30 días** (por eso NO se usa — BD en Neon) | render.com/docs/free | ✅ (era 90 antes — ya mordió una vez) |
| Deploy hooks (GET/POST, URL secreta) + parámetro `imgURL` (tag/digest; el URL-encoding de `:`/`/` es convención de implementación consistente con los ejemplos) | render.com/docs/deploy-hooks | ✅ |
| Image-backed services desde registries públicos sin credenciales, en free | render.com/docs/deploying-an-image | ✅ |
| Static sites gratis con Redirects/Rewrites | render.com/docs | ✅ |
| Si el build falla, sigue sirviendo la versión anterior (→ smoke puede dar falso verde) | render.com/docs/deploys | ✅ (documentado textual) |

## 3. Neon (TP6+)

| Claim | Dónde verificar | Estado 2026-07-26 |
|---|---|---|
| Free plan **sin tarjeta y permanente** (no trial); ~0.5 GB/proyecto; compute suspende ~5 min idle | neon.com/pricing | ✅ |
| Múltiples databases por proyecto; selector ".NET" en Connection Details | console + docs | ✅ |

## 4. Monitoreo (TP9)

| Claim | Dónde verificar | Estado 2026-07-26 |
|---|---|---|
| UptimeRobot free: 50 monitores, 5 min, email, sin tarjeta; **permite cualquier uso (Fair Use Policy)** | uptimerobot.com/pricing + /terms | ✅ — ⚠️ el ToS ya cambió 2 veces (dic-2024 restringió; may-2026 revirtió): re-leer SIEMPRE |
| Better Stack free: 10 monitores, 3 min, sin tarjeta | betterstack.com/pricing | ✅ |

## 5. Análisis de calidad (TP5)

| Claim | Dónde verificar | Estado 2026-07-26 |
|---|---|---|
| SonarQube Cloud (ex SonarCloud) **gratis para repos públicos/OSS** | sonarsource.com → pricing | ✅ (plan "SonarQube for OSS") |
| C# requiere SonarScanner for .NET; check del PR se llama "SonarCloud Code Analysis" | docs.sonarsource.com | ✅ (nombre legacy — puede renombrarse: rompería los required checks configurados) |
| Automatic Analysis soporta .NET pero conflictúa con CI-based (apagar) | docs.sonarsource.com | ✅ |
| Tasks Azure: `SonarCloud*@4` (las @3 deprecadas) | docs.sonarsource.com | ✅ |

## 6. IaC (TP8)

| Claim | Dónde verificar | Estado 2026-07-26 |
|---|---|---|
| `brew install terraform` NO existe en core (post-BSL) → tap `hashicorp/tap`; **opentofu SÍ está en core** | formulae.brew.sh + developer.hashicorp.com/terraform/install | ✅ |
| Provider `kreuzwerker/docker` major vigente `~> 4.0` (MPL-2.0) | registry.terraform.io | ✅ (4.5.0 a jun-2026) |
| OpenTofu: compatible para el workflow estándar, **ya no 100% intercambiable** | opentofu.org | ✅ |

## 7. Riel Azure (todos los TPs — tabla de equivalencias)

| Claim | Dónde verificar | Estado 2026-07-26 |
|---|---|---|
| **Azure DevOps Basic gratis hasta 5 usuarios** sin suscripción (claim del TP3 para Boards/Repos) | azure.microsoft.com/pricing/details/devops | ✅ — ⚠️ área que YA mutó en 2026 (proyectos públicos, minutos): vigilar |
| **Azure for Students sin tarjeta** (USD 100/año, renovable) | azure.microsoft.com/free/students | ✅ |
| Minutos hosted de Azure Pipelines requieren org **vinculada a suscripción**; proyectos públicos de ADO retirados (abr-2026) | learn.microsoft.com / devblogs | ✅ — el riel sin tarjeta garantizado sigue siendo SOLO GitHub |
| App Service **F1** vigente (60 min CPU/día, sin SLA) | azure.microsoft.com/pricing | ✅ |
| ACR Basic ~USD 5/mes (único componente con costo citado — con advertencia) | azure.microsoft.com/pricing | ✅ |
| App Insights: **URL ping tests se RETIRAN el 30/09/2026**; standard tests se cobran por ejecución | learn.microsoft.com | ✅ — ⚠️ tras esa fecha, revisar la fila de la tabla §4 del TP9 |
| GHAS for Azure DevOps de pago (Secret $19 + Code $30 /committer/mes) | learn.microsoft.com | ✅ |

## 8. Degradados / no garantizados (mantener las advertencias)

| Servicio | Estado | Dónde está la advertencia |
|---|---|---|
| **Koyeb** | Puede pedir tarjeta para verificación (hold USD 29) | TP6 intro |
| **GitLab** | Pide tarjeta para validar shared runners | README + TP4 |
| **Railway / Fly.io** | Trial / piden tarjeta | TP6 intro |

## Procedimiento de la corrida

0. **Pre-flight del repo demo**: verificar que `github.com/ingsoft3ucc/demo-fullstack` existe, es público, su CI está verde (badge) y las ramas/PRs de demo siguen en su lugar (demo-c2-inicio, demo-c4-inicio, demo-c4-paso1..3).
1. **Cuándo**: (a) 2-3 semanas antes del inicio de cursada (margen para reescribir guías); (b) la fila del TP correspondiente, la semana antes de publicarlo; (c) tras cualquier anuncio relevante (changelogs de GitHub/Render/Sonar).
2. **Cómo**: abrir la fuente oficial de cada fila y confirmar el claim **textual**. Ante ambigüedad, probar en vivo con la cuenta de la cátedra (`ingsoft3ucc`).
3. **Registrar**: actualizar la columna de estado con la fecha; si algo cambió, anotar QUÉ decía antes (el historial de cambios es material didáctico — casos Koyeb/Render/UptimeRobot ya enseñados en clase).
4. **Además, antes de cada demo**: el pre-calentado de cada clase (listado en el ⚡ Quick Reference del slide de demo de cada guion) — corridas, ramas, servicios despiertos, tokens de prueba.
