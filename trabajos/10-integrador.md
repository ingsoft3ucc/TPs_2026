# 📋 Trabajo Integrador – Para rendir el final (2026)

## ⚠️ Qué es (y qué no es)

El Integrador **no es un trabajo nuevo**: son tus **TPs 1 a 9**, tu app del semestre con toda la cadena que fuiste construyendo, en el **mismo repositorio de todo el semestre** (el que creaste en el TP1, con sus tags `v1.0.0` … `v9.0.0`). No hay un repo nuevo.

**Para rendir el final, defendés el Integrador completo**: como defendiste la P1 (TPs 1 a 4) y la P2 (TPs 5 a 9), pero ahora los nueve juntos, en la mesa de final, y **con la cadena funcionando en vivo**.

> 🔴 **La regla de siempre: si no lo podés explicar, no lo aprobás — aunque funcione.** Lo que está en tu repo y no sabés defender no cuenta. Y su reversa: un problema que te pasó y sabés explicar no resta (eso no reemplaza que la cadena ande).

## 🎯 Objetivo

Demostrar, sobre tu propia aplicación, que dominás el ciclo DevOps de punta a punta —del commit a producción, con verificación automática en cada eslabón— y que podés explicar **por qué** cada pieza está donde está.

## 📋 Qué tiene que tener tu repo

**Lo que pide cada guía de [`trabajos/`](./), del TP1 al TP9. No hay requisitos nuevos**, salvo la sección de cierre de tu `decisiones.md` (abajo). En resumen, la cadena completa sobre tu app (frontend + backend + base de datos), en `main`:

| | Qué | TPs |
|---|---|---|
| 1 | Todo cambio entra por **PR** a `main` protegida; cada PR corre **build + tests con reporte** y **sin verde no hay merge**; la **cobertura** tiene un umbral que frena | TP1, TP4, TP5 |
| 2 | La app corre en **contenedores**; CI verde → **deploy automático a QA** de la imagen del registry con la etiqueta del commit; **e2e contra QA** que frenan la promoción; **aprobación manual** → **la misma imagen** en PROD | TP2, TP6, TP7 |
| 3 | La **infraestructura declarada como código** (`infra/`), y los **controles de seguridad y la observabilidad** del TP9 activos | TP8, TP9 |
| 4 | La **planificación** del TP3 y tu `decisiones.md` con una sección por TP | TP3 y todos |

Cualquier herramienta de CI/CD y cualquier nube valen (riel GitHub, Azure u otra), **o el fallback local documentado** (self-hosted runner + docker-compose como QA/PROD), con el mismo contrato.

## 📄 Qué traés a la mesa

**No se entrega nada antes ni se sube a ningún lado**: el día del examen venís con tu notebook y lo presentás ahí. Tu repo, tus entornos y tu `decisiones.md` se miran en vivo, en tu pantalla.

Ese día, en `main` de tu repo:

1. **Los tags y releases `v1.0.0` … `v9.0.0`**: marcan dónde cerró cada TP. Si falta alguno, ponelo en el commit donde terminaste ese TP y publicá su release. En la mesa se mira `main` tal como está ese día: si completaste algo después del tag, está ahí y se defiende ahí.
2. **`decisiones.md` consolidado**: una sección por TP (TP1 a TP9) y, al final, una **sección de cierre** (~1 página): el diagrama de TU cadena completa, las 3 decisiones de diseño que más defendés, y qué harías distinto si empezaras de nuevo.
3. **Nada de `evidencias.md` ni capturas**: tu repo es público y se navega — *Actions*, *Deployments*, *Releases*, los PRs, *Security*. Lo que no deja rastro ahí ya está contado en `decisiones.md`.

## 🟢 Lo que tiene que estar vivo el día de la mesa

Se defiende lo que **funciona hoy**, no lo que «anduvo en su momento»:

- **QA y PROD**, despiertos: abrilos unos minutos antes de tu turno, porque el free tier se duerme y tarda en arrancar.
- **Tu `infra/`**, lista para correr un `plan` sin errores y que sepas leer. Si destruiste preprod como pide el TP8, el plan va a decir que lo crea, y eso está bien. El estado lo necesitás en la notebook que traés (o donde lo hayas guardado). Un `infra/` que anduvo en octubre y ya no corre el día de la mesa no se puede defender.
- **Los controles de seguridad del TP9** activos: Dependabot, CodeQL y push protection siguen corriendo solos.
- **Tu Sentry y tu monitor**, con historial: guardan 30 días. Si tu error o tu caída son más viejos, repetilos antes de la mesa. El monitor queda pausado como en el TP9: prendelo, repetí la caída y volvé a pausarlo.

Si un free tier murió, el **fallback local** mostrado en vivo vale igual: avisá antes de la mesa.

## 🎤 Cómo es la defensa

**Individual, con tu notebook. Esto es la parte práctica del final**, y se aprueba con **4 (60 %)**. El profesor de la teoría puede hacer, además, las preguntas de la teoría que considere.

> 🔴 **Cada decisión que tomaste, la tenés que poder mostrar y explicar cómo la implementaste en tu código.** No alcanza con contar el concepto: «mostrame en tu repo dónde está y cómo funciona». Si no la podés mostrar ni explicar, no se aprueba, aunque funcione.

En la mesa te puedo pedir cualquiera de estas dos cosas, o las dos.

### 🔴 La validación en vivo

Con tu repo y tus entornos abiertos, te puedo pedir:

1. **Un cambio de código** (algo visible y chico, que te indico en el momento): lo hacés por el flujo completo —rama, PR, pipeline— y lo seguimos juntos hasta **QA**.
2. **Un test roto a propósito**: rompés un test, abrís el PR, y mostramos el **gate bloqueando** el merge (y las e2e frenando la promoción, si el rojo llega ahí). Después lo arreglás y el sistema se destraba solo.
3. **La promoción a PROD**: con el verde completo, aprobás el gate de producción, y PROD recibe **la misma imagen**.
4. **Tu `infra/` y tus controles, hoy**: corrés un `plan` y lo leemos, y abrís los controles del TP9 con evidencia reciente.

> ⏱ Como te la puedo pedir, traé tu sistema **pre-calentado**: la noche anterior o esa mañana, **una corrida completa del pipeline en verde** y **tu `plan` probado**; unos minutos antes de tu turno, **QA y PROD despiertos** (pegales un curl: los free tiers duermen). Es lo mismo que viste hacer a la cátedra antes de cada demo.

### 🎤 Las preguntas

Sobre **cualquiera de los TPs 1 a 9** y sobre **lo que escribiste en tu `decisiones.md`**. Quiero saber tres cosas: si tenés los **conceptos**, si sabés **por qué** hiciste lo que hiciste, y **cómo** lo hiciste en tu código.

Pueden tener el **formato del recuperatorio**: te pido que me muestres algo de tu repo o de tus entornos, después te pregunto para qué está, y si lo explicás bien, repregunto. Por ejemplo:

> **Mostrame** en tu workflow dónde se etiqueta la imagen con el commit y cómo QA y PROD reciben esa misma imagen.
> → **¿Para qué está eso?**
> → **Repregunta:** ¿qué jobs tienen que pasar antes del deploy a PROD, y qué le pasa a PROD si la e2e se pone roja?

**Qué tenés que poder explicar de cada TP** está en las filminas de defensa de los parciales, con preguntas de ejemplo: [`defensa-P1.pdf`](../defensas/defensa-P1.pdf) (TPs 1 a 4) y [`defensa-P2.pdf`](../defensas/defensa-P2.pdf) (TPs 5 a 9). Nada nuevo: todo lo que se pregunta se enseñó.

**Tené abierto en la notebook**, por si te pido mostrar algo: tu repo (*Actions*, *Pull Requests*, *Deployments*, *Releases*, *Security*) · tu `decisiones.md` · QA y PROD · tu panel de la nube · tu `infra/` en una terminal, con las credenciales cargadas (sin mostrarlas) · tu Sentry · tu monitor · tu tablero. Batería o cargador. Si te pido algo y no lo tenés abierto, el tiempo de abrirlo es tuyo.

## ⚠️ Uso de IA

Misma regla de todo el semestre: IA permitida y alentada, **declarada** en `decisiones.md`, **verificada** por vos, y **defendible**. En el final pesa doble: lo que la IA hizo por vos y no sabés explicar, no se aprueba. **Durante la mesa no se usa IA.**

## ❓ Preguntas frecuentes

- **Aprobé la P1 y la P2, ¿alcanza?** Para regularizar, sí. Para el final defendés los nueve TPs otra vez, juntos, y la cadena tiene que andar ese día, en vivo.
- **Me faltaron cosas en los parciales, ¿las puedo completar?** Sí: se mira tu repo como esté el día del examen. Pero completar no alcanza: tenés que poder defenderlo.
- **¿Puedo usar otra app distinta a la del semestre?** No. Se defiende la app que hiciste durante el año, en el mismo repo de los TPs 1 a 9.
- **¿Individual o en equipo?** **Todo es individual**: tu propia app, tu propio repo, tu propia defensa. Dos personas no pueden presentar el mismo sistema.
- **¿Qué pasa si un free tier murió justo antes de la mesa?** Fallback local (TP6 §3.6 / TP7 §3.6) mostrado en vivo: vale igual. Avisá antes para coordinar.
