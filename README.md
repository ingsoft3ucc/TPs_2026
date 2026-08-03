# Ingeniería del Software 3 — Práctica · UCC 2026

> **Ingeniería en Sistemas · 4to año** — Cátedra práctica: DevOps de punta a punta, sobre una aplicación tuya.
> Docente: Ing. Ariel Schwindt

Este documento es el **reglamento operativo** de la práctica: cómo funciona la cursada, cómo se evalúa, qué reglas rigen y dónde está cada cosa. Leelo entero una vez; volvé cuando tengas dudas.

> 📌 **Material publicado por clase.** Este repo se va llenando a medida que avanza la cursada: el enunciado de cada TP y las filminas de cada clase se publican en la semana que corresponde. Si un TP todavía no tiene link, es porque aún no se dictó su clase.

---

## 1. La idea de la materia (en 5 líneas)

Vas a construir, con tus manos y semana a semana, el **sistema de entrega profesional** completo de una aplicación: integración continua con gates de calidad, entrega continua con aprobaciones, releases inmutables verificadas end-to-end, infraestructura como código, seguridad automatizada y observabilidad. No es una materia de "ver herramientas": es una materia de **construir el sistema y poder defender cada decisión**. Al final tenés un pipeline real, público, en tu portfolio — y el modelo mental que la industria espera de un ingeniero 2026.

## 2. Estructura de la cursada

| # | Clase | Tipo | Tema | Filminas | TP |
|---|---|---|---|---|---|
| 1 | C1 | Contenido | Cultura DevOps + Git para equipos | [PDF](clases/Clase01/Clase01-presentacion.pdf) | [TP1](trabajos/01-git-colaborativo.md) |
| 2 | C2 | Contenido | Contenedores: Docker + Compose | — | *próximamente* |
| 3 | C3 | Contenido | Plataformas DevOps + planificación ágil | — | *próximamente* |
| 4 | C4 | Contenido | CI: Pipelines as Code | — | *próximamente* |
| 5 | **P1** | **Defensa** | **Defensa oral de TPs 1–4** | — | — |
| 6 | C5 | Contenido | Testing en el pipeline: unit tests + coverage + análisis estático | — | *próximamente* |
| 7 | C6 | Contenido | CD: environments, aprobaciones y deployment patterns | — | *próximamente* |
| 8 | C7 | Contenido | Contenedores en el pipeline + pruebas e2e | — | *próximamente* |
| 9 | C8 | Contenido | Infraestructura como Código | — | *próximamente* |
| 10 | C9 | Contenido | DevSecOps + Observabilidad + Continuous Feedback | — | *próximamente* |
| 11 | **P2** | **Defensa** | **Defensa oral de TPs 5–9** | — | — |
| 12 | **R** | **Recuperatorio** | Se vuelve a presentar el bloque que quedó desaprobado | — | — |

**No hay parciales escritos**: tus notas son **dos**, y salen de las dos presentaciones (P1 sobre los TPs 1–4, P2 sobre los TPs 5–9). La materia se aprueba **trabajando todas las semanas** — está diseñada para que cada TP tome una semana si venís al día, y para que se acumule mal si no.

## 3. La app del semestre (el hilo conductor)

A partir del **TP2** elegís una aplicación **full-stack (frontend + backend + base de datos)** — propia o adaptada de un repo público — y esa misma app te acompaña toda la materia: cada TP le agrega una capa (CI, calidad, CD, imagen, e2e, IaC, seguridad, observabilidad). El **TP Integrador** — condición para rendir el final — pide exactamente esa cadena completa: si venís al día, al llegar a P2 lo tenés ~80% hecho.

- **Un solo repo**: la app entra al **mismo repo del TP1** (el de las protecciones) — tu historial del semestre es evidencia del Integrador. Si preferís repo nuevo al elegir la app, el TP2 (§3.2) te dice cómo hacerlo bien (migrar `decisiones.md`, recrear protecciones, avisar).
- **Política de cambio de app**: podés cambiarla **hasta el TP4 inclusive** (rehaciendo lo entregado, que hasta ahí es chico). Después, consultá a la cátedra antes — el costo crece mucho.
- La cátedra mantiene un **sample de referencia** con la estructura que las guías asumen: [`ingsoft3ucc/demo-fullstack`](https://github.com/ingsoft3ucc/demo-fullstack) (.NET 8 + React/Vite + PostgreSQL, `./backend` + `./frontend`). Es el repo de las demos en clase — podés inspirarte en su estructura, **no** entregarlo como tu app.

## 4. Las herramientas: dos rieles + tier libre

Ninguna nube es obligatoria. Cada TP incluye:

- **Riel GitHub (canónico)** — GitHub Actions + servicios gratuitos verificados **sin tarjeta de crédito**. Es el riel de las demos y trae la **guía paso a paso completa**. Todo lo evaluable se puede completar por acá **sin costo**, y hay **fallback local documentado** (self-hosted runner + docker-compose) por si un servicio gratuito cambia sus condiciones a mitad de semestre.
- **Riel Azure** — Azure DevOps / Azure (el stack de la certificación AZ-400). Cada TP trae la **tabla de equivalencias + checkpoints**. Estado 2026: los minutos hosted de Azure Pipelines requieren organización vinculada a una suscripción (Azure for Students no pide tarjeta — verificalo antes de apostar tu TP); la alternativa sin tarjeta en ese riel es el agente self-hosted.
- **Tier libre** — AWS, GCP, GitLab u otra: permitido con el **mismo contrato de entregables**, soporte limitado de la cátedra. Ojo: GitLab pide tarjeta para validar sus runners compartidos.

**Repos públicos: obligatorio.** No es capricho — las funciones que la materia usa (protecciones, Actions ilimitado, environments con aprobaciones, secret scanning, CodeQL, SonarQube Cloud) son gratuitas **solo** en repos públicos. Bonus: terminás con un portfolio público real.

## 5. Cómo se evalúa

### Cada TP
Así se mira cada TP dentro de su presentación (la nota, como ves más abajo, se pone por presentación):

| Criterio | Peso |
|---|---|
| Configuración técnica (funcionando y **reproducible** — evidencias) | 25% |
| Claridad y justificación en `decisiones.md` + `evidencias.md` | 25% |
| **Defensa oral: comprensión y argumentación** | **50%** |

- **Entrega**: la URL de tu repo público se carga en el **formulario de la cátedra** — hay uno por presentación, los links están en el aula virtual — antes de la clase de defensa correspondiente. Es un formulario, no una planilla compartida: nadie puede pisar la entrega de otro, y podés volver a abrir la tuya para corregirla. `decisiones.md` y `evidencias.md` viven **en el repo** y se acumulan TP a TP. No hay checkpoints semanales con nota — pero el historial de Git cuenta **cuándo** trabajaste, y un bloque de 4 TPs commiteados la noche anterior a P1 se nota (y se pregunta) en la defensa.
- **Defensa**: en P1 (TPs 1–4) y P2 (TPs 5–9). Individual, con tu repo abierto, **navegando en vivo** (y desde P2, con tus **entornos vivos**: URLs de QA/PROD, dashboard de análisis, monitor — o tu **fallback local** del TP6/TP7 si el free tier te falló: avisá antes). Las **preguntas están publicadas** en cada enunciado — no hay sorpresas: salen de la teoría enseñada.
- **La regla innegociable**: *si no lo podés explicar, no lo aprobás — aunque funcione.* Y su reversa: un repo con cicatrices bien explicadas vale más que uno perfecto defendido con silencios.
- **La nota es por presentación, no por TP.** Los 9 TPs **no llevan nota propia**: son lo que defendés en la presentación que les toca (P1 → TPs 1–4 · P2 → TPs 5–9). De cada presentación sale **una nota**, y esas dos notas son las de tu cursada — **van por separado: no se promedian ni se compensan entre sí**.
- **Aprobación y regularidad**: cada presentación se aprueba con **nota ≥ 4** (escala 1–10). Para **regularizar** necesitás **las dos presentaciones aprobadas**.
- **Recuperatorio (clase 12): la red es una sola.** Si una presentación quedó abajo de 4, en la clase 12 volvés a presentar **ese bloque completo** — los TPs 1–4 o los 5–9, según cuál haya sido. Si **las dos** quedaron abajo de 4, la materia queda **libre** (la recursás) — y lo mismo si vas al recuperatorio y tampoco lo aprobás. Ante la duda, avisá temprano y lo acomodamos en el momento, no en la clase 12.

### El Integrador
Condición para **rendir el final**. Es tu app del semestre con la cadena completa — el enunciado se publica en `trabajos/` durante la cursada. Se valida **en vivo** en la mesa.

## 6. Uso de Inteligencia Artificial

La IA (ChatGPT, Copilot, Claude…) está **permitida y alentada** — es como se trabaja hoy. Con una condición innegociable:

1. **Declarás** en `decisiones.md` qué partes fueron asistidas por IA.
2. **Verificaste** lo que la IA produjo (y contás cómo).
3. **Podés defenderlo**: si en la defensa no podés explicar una decisión que la IA tomó por vos, ese punto **no se aprueba**.

**Las tres valen también para la IA que opera, no solo para la que escribe.** Hoy hay agentes que investigan un incidente y proponen la causa raíz, que generan y corren el plan de pruebas de un release, o que configuran el pipeline solos. Si usás uno, la vara es la misma: declarado, verificado y defendible.

La IA es una herramienta de productividad, no un reemplazo del entendimiento. Copiar sin entender no es un atajo: es la forma más cara de desaprobar la defensa.

## 7. Trabajo en equipo y honestidad

- Los TPs son **individuales**: tu repo, tus decisiones, tu defensa. Los mecanismos de colaboración de un equipo real (PRs, revisiones, aprobaciones de deploy) los operás vos mismo sobre tu repo — lo que se evalúa es que entiendas y puedas defender el flujo. Ayudarse entre compañeros está muy bien; entregar el trabajo de otro, no (ver el punto siguiente).
- El trabajo es de ustedes. Repos clonados de otros equipos, historiales fabricados o evidencias ajenas son plagio y se tratan según el reglamento de la universidad. (El historial de Git cuenta la historia real — y lo sabemos leer.)
- Los secretos jamás se commitean (lo vas a aprender en capas, TP a TP). Si filtrás una credencial real: **rotala ya** y contalo en `decisiones.md` — manejar bien un incidente también es aprendizaje.

## 8. Logística

- **Cómo es una clase**: bloque teórico + pausa + demo en vivo + presentación del TP + **manos a la obra** (trabajás sobre tu propio TP en el aula, con soporte). La teoría acá es **la de la práctica**: los fundamentos que necesitás para resolver el TP de la semana y para defenderlo — no repite ni reemplaza a la materia teórica. Traé tu notebook con Docker funcionando desde la C2.
- **Consultas**: en clase (el bloque de manos a la obra existe para eso) y por los canales del aula virtual.
- **Los enunciados** de la carpeta [`trabajos/`](trabajos/) son la fuente de verdad de cada TP: guía sugerida (aprendizaje) + TP entregable (evaluación). La guía no se entrega; el TP sí.
- **Las filminas** de cada clase, en PDF, están en [`clases/`](clases/) — se publican después de dictada la clase.

---

*Los servicios gratuitos citados en las guías se re-verifican antes de cada cohorte ([smoke test de vigencia](smoke-test-vigencia.md) — documento operativo de la cátedra, público por transparencia). Si encontrás que alguno cambió sus condiciones, avisá a la cátedra — es exactamente el tipo de ojo profesional que esta materia entrena.*
