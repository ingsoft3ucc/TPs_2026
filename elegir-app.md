# Cómo elegir la app del semestre

> **Aplica desde el TP2.** En el TP2 seleccionan la aplicación que utilizarán durante toda la
> materia: en el TP4 se le construye el pipeline, en el TP5 los tests, en el TP6 se despliega, en el
> TP7 se empaqueta, en el TP8 se define su infraestructura, en el TP9 se asegura, y es la base del
> Trabajo Integrador. Una elección adecuada hoy les evita dificultades hasta noviembre.

Puede ser propia (de otra materia o de un proyecto personal) o tomada de GitHub, siempre que la
comprendan. Requisito mínimo: **frontend + backend + base de datos**.

**La tecnología es libre.** .NET, Java, Node, Python, Go, PHP, Ruby; React, Angular, Vue, Svelte o
plantillas del lado del servidor; PostgreSQL, MySQL, SQL Server, MariaDB, MongoDB. La cátedra no
exige ningún stack y ninguno da puntos extra: lo que se evalúa es el sistema de entrega que le
construyen alrededor, no el lenguaje en el que está escrita. La única condición es que la
combinación que elijan cumpla los cinco criterios que siguen — en particular, que la entiendan lo
suficiente como para explicarla y modificarla en la defensa. Las guías de los TPs están escritas
sobre un stack de ejemplo, pero el contenido evaluable es el mismo para todos; si su stack requiere
un paso distinto (otro comando de compilación, otra imagen base, otro cliente de base de datos), ese
ajuste es parte del trabajo y se documenta en `decisiones.md`.

---

## Criterios de selección, en orden de importancia

### 1. Que puedan ejecutarla hoy

Clónenla, levántenla y úsenla antes de comprometerse. Si no arranca en su máquina en el transcurso
de una tarde, no va a arrancar más adelante. Es el criterio que con más frecuencia se omite y el que
mayor costo tiene.

### 2. Que conozcan los comandos de compilación y ejecución

Concretamente: ¿es `dotnet build`? ¿`npm run build`? ¿`mvn package`? En el TP2 escribirán un
Dockerfile, que consiste exactamente en eso: los comandos de compilar y arrancar, expresados para
que los ejecute otra máquina. Sin esa información no es posible redactarlo.

### 3. Que identifiquen dónde se configura la conexión a la base

Localicen ahora la cadena de conexión: ¿está en `appsettings.json`, en un `.env`, en
`application.properties`? Y la pregunta siguiente: **¿puede modificarse mediante una variable de
entorno sin tocar el código?**

Esto es determinante: en el TP2 la base pasa a ser un contenedor (cambia el host) y en el TP6 la
misma aplicación debe apuntar a una base para QA y a otra para producción. Si la configuración está
fijada en el código en cinco lugares, el problema los acompaña todo el semestre; si está
centralizada y es parametrizable por variable de entorno, ya tienen mucho resuelto.

### 4. Que tenga lógica para testear, o que puedan incorporársela

En el TP5 se piden **8 tests en el backend y 4 en el frontend**, y tienen que ser tests con sentido:
no ocho variantes de "el endpoint responde 200". Lo que se necesita no es tamaño, sino **reglas**:
validaciones, cálculos, cambios de estado, restricciones.

Cuenten cuántas tienen. Como referencia, para llegar a 8 tests backend con comodidad hacen falta
**entre 4 y 6 reglas** (cada una suele dar un caso válido y uno o dos casos borde), y para los 4 del
frontend, **dos o tres comportamientos** propios de la interfaz: un formulario que no deja enviar
con datos inválidos, un total que se recalcula, un botón que se deshabilita según el estado.

Ejemplos de reglas que sirven:

| Tipo | Ejemplo concreto | Tests que habilita |
|---|---|---|
| Validación | El email debe ser único / la fecha de fin no puede ser anterior a la de inicio | válido, inválido, límite |
| Cálculo | Total con descuento por cantidad, promedio, saldo acumulado | caso típico, cero, borde del tramo |
| Transición de estado | Un pedido `pendiente` puede pasar a `pagado`, pero no a `entregado` | transición permitida, transición prohibida |
| Restricción | No se puede eliminar una categoría con productos asociados | permite, bloquea |
| Autorización | Un usuario solo ve sus propios registros | propio, ajeno |

Si la aplicación se limita a altas, bajas y modificaciones sin ninguna regla asociada, tienen dos
caminos igualmente válidos: elegir otra, o **agregarle ustedes las reglas que falten** hasta llegar
a ese número. Agregarlas es trabajo legítimo de la materia y conviene hacerlo en el TP2 o el TP3, no
la semana del TP5. Lo que no conviene es llegar al TP5 sin nada que verificar.

### 5. Que la entiendan lo suficiente para modificarla

En la mesa del final del Integrador se les solicitará un cambio de código en vivo. No hace falta que
la hayan escrito; sí que sepan dónde intervenir.

---

## Dos consideraciones adicionales

- **Tamaño: reducido.** Dos o tres pantallas son suficientes. Un sistema más grande no mejora la
  calificación: implica compilaciones más lentas, más puntos de falla y más tiempo de espera. La
  aplicación de ejemplo de la cátedra expone tres endpoints.
- **Sin dependencias exóticas.** Frontend, backend y una base. Si además requiere Redis, Kafka, una
  API paga o credenciales de las que no disponen, cada TP se complica. Presten particular atención a
  las APIs de terceros: si el servicio deja de estar disponible o vence el período gratuito, el TP
  queda comprometido.

---

## Verificación previa a la decisión (20 minutos)

1. Clonarla y ejecutarla localmente.
2. Enunciar el comando con el que se compila.
3. Señalar el archivo donde se configura la base de datos.
4. Modificar un texto visible en pantalla y comprobar el cambio.
5. **Enumerar en voz alta las reglas de negocio que tiene** — validaciones, cálculos, transiciones
   de estado, restricciones — y contarlas. Si salen 4 o más en el backend y 2 o más en el frontend,
   el TP5 está cubierto. Si salen menos, escriban ahí mismo cuáles le van a agregar y en qué
   archivo: eso es parte de la decisión, no un pendiente para septiembre.

Si completan los cinco pasos, esa es su aplicación. Si se detienen en alguno, busquen otra ahora —
no en septiembre.

---

> ⚠️ **Plazo.** Pueden cambiar de aplicación **hasta el TP4 inclusive**, rehaciendo lo entregado
> hasta ese punto (que es poco). A partir de ahí el costo es alto.
