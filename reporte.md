# Exploración de la planificación de procesos en xv6-riscv

**Estudiante:** Brian Rodríguez
**Código:** 2420014
**Repositorio:** https://github.com/Brianbrd/xv6-riscv
**Fecha:** Noviembre de 2026

## 1. Introducción

El objetivo de esta actividad es explorar el funcionamiento del planificador de procesos del sistema operativo xv6-riscv, identificar sus mecanismos de selección y analizar qué cambios serían necesarios para incorporar un algoritmo de planificación MLFQ (*Multilevel Feedback Queue*).

El trabajo se desarrolla como una propuesta de diseño. No se modifica el código del planificador ni se implementa MLFQ. Se estudia el funcionamiento original de xv6 y se relacionan sus estructuras y funciones con las reglas que tendría la nueva política de planificación.

## 2. Entorno de trabajo

La exploración se realizó en Google Cloud Shell, utilizando un entorno Linux y el repositorio xv6-riscv.

### 2.1. Información del entorno

- Sistema operativo: Ubuntu 24.04.5 LTS.
- Kernel de Linux: 6.18.59+.
- Compilador RISC-V: GCC 13.3.0.
- Emulador: QEMU 8.2.2.
- Herramienta de compilación: GNU Make 4.3.
- Repositorio: https://github.com/Brianbrd/xv6-riscv
- Rama de trabajo: `riscv`.
- Commit inicial: `fa2ddcb7ca87a77af579380a6eafd5aff163b44f`.

**Evidencia:** `evidencias/entorno_versiones.png`

### 2.2. Compilación y ejecución

Se utilizó el comando:

```bash
make qemu CPUS=1
```

Este comando compila xv6 y lo ejecuta mediante QEMU utilizando un solo procesador virtual.

La configuración de QEMU se encuentra en el `Makefile`, donde la opción `-smp $(CPUS)` permite definir la cantidad de CPU virtuales. Al ejecutar el comando con `CPUS=1`, se utiliza un único procesador virtual.

**Evidencia:** `evidencias/boot_cpus1.png`

### 2.3. Comprobación del funcionamiento

Una vez iniciado xv6, se ejecutaron comandos básicos para comprobar que el sistema arrancaba y permitía interactuar con su consola.

Se utilizó `ls` para consultar los archivos y programas disponibles, y `cat README` para leer el archivo README del sistema.

**Evidencias:**

- `evidencias/xv6_ls.png`
- `evidencias/xv6_cat_readme.png`
- `evidencias/xv6_cat_readme_2.png`

Estas pruebas permitieron comprobar que xv6 compilaba correctamente, arrancaba en QEMU y permitía ejecutar comandos desde su consola.

## 3. Análisis del planificador original de xv6

### 3.1. Política de planificación

El xv6 original utiliza un planificador preventivo sencillo, sin niveles explícitos de prioridad.

La función `scheduler()`, ubicada en `kernel/proc.c`, recorre secuencialmente el arreglo estático `proc[NPROC]`, que contiene las estructuras de los procesos. Durante el recorrido busca procesos cuyo estado sea `RUNNABLE`.

Cuando encuentra uno, adquiere su bloqueo, cambia su estado a `RUNNING` y realiza el cambio de contexto mediante:

```c
swtch(&c->context, &p->context);
```

El planificador no mantiene una estructura explícita de colas de prioridad ni un cursor circular para seleccionar procesos. Recorre el arreglo de procesos secuencialmente.

### 3.2. Estados de los procesos

En `kernel/proc.h` se encuentra la enumeración `enum procstate`, que define los estados posibles de un proceso:

- `UNUSED`: posición sin utilizar.
- `USED`: proceso creado o reservado.
- `SLEEPING`: proceso que está esperando un evento.
- `RUNNABLE`: proceso preparado para ejecutarse.
- `RUNNING`: proceso que está utilizando la CPU.
- `ZOMBIE`: proceso que terminó su ejecución, pero aún no ha sido recogido por su proceso padre.

Estos estados permiten que el planificador identifique qué procesos pueden utilizar la CPU.

### 3.3. Cesión y recuperación de la CPU

En `kernel/proc.c` se encuentra la función `yield()`. Esta función permite que un proceso ceda voluntariamente la CPU o lo haga como consecuencia de una interrupción de temporizador.

La función cambia el estado del proceso de `RUNNING` a `RUNNABLE` y llama a `sched()` para realizar el cambio de contexto.

Cuando un proceso necesita esperar un evento, puede pasar al estado `SLEEPING` mediante `sleep()`. Posteriormente, cuando ocurre el evento esperado, `wakeup()` permite que el proceso vuelva al estado `RUNNABLE`.

En `kernel/trap.c`, las interrupciones del temporizador se identifican mediante `which_dev == 2`. Tanto `usertrap()` como `kerneltrap()` pueden invocar `yield()` cuando se presenta una interrupción de reloj que requiere ceder la CPU.

La función `clockintr()` incrementa el contador global de ticks. En la configuración explorada, cada tick corresponde aproximadamente a 0.1 segundos.

### 3.4. Incorporación del planificador a la compilación

El archivo `Makefile` contiene la lista de objetos que se utilizan para construir el kernel.

En la variable `OBJS` se incluye `kernel/proc.o`, que se genera a partir del archivo fuente `kernel/proc.c`.

Posteriormente, los objetos se enlazan utilizando el archivo `kernel/kernel.ld` para generar el ejecutable del kernel:

```text
kernel/kernel
```

Este archivo es el que QEMU ejecuta al iniciar xv6.

Por lo tanto, los cambios futuros en el planificador tendrían que realizarse en los archivos fuente correspondientes y pasar nuevamente por el proceso de compilación y enlace del kernel.

## 4. Propuesta de planificación MLFQ

### 4.1. Descripción inicial

Antes de consultar la inteligencia artificial, se planteó que MLFQ organiza los procesos en varias colas con distintos niveles de prioridad.

El planificador atiende primero la cola no vacía de mayor prioridad y, cuando hay varios procesos en el mismo nivel, utiliza Round Robin para alternarlos.

Los procesos nuevos ingresan a la cola de mayor prioridad. Cuando consumen su tiempo asignado, bajan de prioridad. Si ceden la CPU antes de agotar ese tiempo, conservan su nivel de prioridad.

Además, cada cierto tiempo se realiza un *Priority Boost*, mediante el cual los procesos recuperan la prioridad más alta para evitar la inanición (*starvation*).

### 4.2. Variante propuesta

A partir del análisis y de las preguntas realizadas durante la exploración, se propone la siguiente configuración:

| Nivel | Prioridad | Quantum por turno | Presupuesto acumulado |
|---|---|---:|---:|
| Cola 0 | Alta | 1 tick | 2 ticks |
| Cola 1 | Media | 1 tick | 4 ticks |
| Cola 2 | Baja | 1 tick | 8 ticks |

Se propone realizar un *Priority Boost* cada 100 ticks, equivalentes aproximadamente a 10 segundos.

Los procesos nuevos ingresarían a la cola de mayor prioridad. El tiempo de CPU consumido se acumularía mientras el proceso permanezca en el mismo nivel, incluso si cede la CPU por una operación de entrada/salida. Cuando agote el presupuesto correspondiente a su nivel, descendería al siguiente.

El quantum de un tick determina cuánto puede ejecutarse un proceso en cada turno antes de ceder la CPU. El presupuesto acumulado determina cuánto tiempo puede consumir en total dentro de un nivel antes de descender de prioridad.

Esta regla busca evitar que un proceso pueda conservar indefinidamente una prioridad alta simplemente cediendo la CPU antes de terminar su turno.

El *Priority Boost* permitiría recuperar la prioridad de los procesos periódicamente y reducir el riesgo de inanición.

En cuanto a la expulsión por prioridad, se propone que si un proceso de mayor prioridad pasa a estar `RUNNABLE` mientras se ejecuta uno de menor prioridad, este último ceda la CPU en la siguiente interrupción del temporizador. De esta manera, no se exige una interrupción inmediata dentro de `wakeup()`.

### 4.3. Relación con el código de xv6

La incorporación de MLFQ requeriría estudiar los siguientes componentes:

| Archivo o función | Propósito del posible cambio |
|---|---|
| `kernel/proc.h` – `struct proc` | Incorporar información para identificar el nivel de prioridad y el tiempo de CPU acumulado. |
| `kernel/proc.c` – `allocproc()` | Inicializar los nuevos datos de planificación cuando se crea un proceso. |
| `kernel/proc.c` – `scheduler()` | Seleccionar primero procesos de la cola de mayor prioridad y mantener la alternancia entre procesos del mismo nivel. |
| `kernel/proc.c` – `yield()` | Considerar cómo se registra la cesión de CPU sin perder el tiempo acumulado en el nivel. |
| `kernel/proc.c` – `sleep()` y `wakeup()` | Conservar la información de prioridad y consumo de CPU cuando un proceso pasa a dormir y posteriormente vuelve a estar disponible. |
| `kernel/trap.c` – `clockintr()` | Relacionar el contador de ticks con el control periódico del tiempo y el *Priority Boost*. |
| `kernel/trap.c` – `usertrap()` y `kerneltrap()` | Analizar cómo contabilizar el tiempo de CPU y cuándo debe cederse la CPU por agotamiento del presupuesto. |

Una posible alternativa para representar la planificación sería conservar el arreglo estático de procesos y agregar campos a `struct proc`, en lugar de reemplazarlo necesariamente por listas enlazadas.

El planificador tendría que consultar esos campos para identificar la prioridad de los procesos y seleccionar los que estén en estado `RUNNABLE`.

Estos cambios son propuestas de análisis y no fueron implementados ni probados.

### 4.4. Aspectos pendientes de investigación

Durante el análisis se identificaron algunos aspectos que necesitan mayor estudio antes de implementar la propuesta:

- Determinar cómo contabilizar correctamente los ticks consumidos por cada proceso.
- Analizar cómo realizar el *Priority Boost* sin generar conflictos con los bloqueos de los procesos.
- Investigar cómo se comportaría la propuesta en una configuración con más de un procesador virtual.
- Definir qué prioridad heredaría un proceso creado mediante `fork()`.
- Estudiar cómo garantizar que la expulsión por prioridad se realice en la siguiente interrupción del temporizador.

En particular, los bloqueos (*locks*) requieren especial atención, porque el planificador trabaja con bloqueos asociados a los procesos. Cualquier modificación de las prioridades o de los estados debe respetar las reglas de sincronización del kernel.

## 5. Escenarios propuestos para una futura comprobación

Los siguientes escenarios permitirían comprobar el comportamiento de una futura implementación de MLFQ. No fueron ejecutados porque la actividad se limita al diseño y análisis.

### Escenario 1. Procesos intensivos en CPU

- **Procesos participantes:** dos procesos que requieren utilizar continuamente la CPU.
- **Acciones:** ambos procesos ingresan a la cola de mayor prioridad y continúan ejecutándose sin realizar operaciones de entrada/salida.
- **Regla que se comprobaría:** descenso de prioridad cuando se agota el presupuesto de cada nivel.
- **Comportamiento esperado:** los procesos deberían descender progresivamente de prioridad al consumir el presupuesto de sus respectivos niveles.

### Escenario 2. Proceso interactivo y acumulación del tiempo

- **Procesos participantes:** un proceso que realiza operaciones de entrada/salida y otro que consume continuamente CPU.
- **Acciones:** el proceso interactivo utiliza una parte de su presupuesto, se bloquea y posteriormente vuelve a estar disponible.
- **Regla que se comprobaría:** acumulación del tiempo de CPU consumido aunque el proceso ceda la CPU por entrada/salida.
- **Comportamiento esperado:** el tiempo utilizado antes de bloquearse debería conservarse, de manera que el proceso no pueda permanecer indefinidamente en una prioridad alta.

### Escenario 3. Priority Boost e inanición

- **Procesos participantes:** procesos que permanecen en niveles de prioridad diferentes, incluyendo uno que consume continuamente CPU.
- **Acciones:** los procesos se ejecutan durante un período suficiente para alcanzar el momento establecido para el *Priority Boost*.
- **Regla que se comprobaría:** promoción periódica de los procesos a la prioridad más alta.
- **Comportamiento esperado:** los procesos de menor prioridad deberían recuperar la posibilidad de competir por la CPU y evitar quedar relegados indefinidamente.

## 6. Evidencia del uso de inteligencia artificial

Para apoyar el análisis se utilizó **Google Gemini** como tutor socrático. Se proporcionó una descripción inicial de MLFQ y se solicitó que las preguntas se realizaran de forma progresiva, sin entregar una implementación ni resolver directamente el diseño.

Se seleccionaron tres interacciones que muestran decisiones tomadas durante el análisis.

### Interacción 1. Conservación del tiempo restante

**Pregunta de Gemini:**

Si un proceso consume un tick de un quantum de dos, se bloquea por una operación de entrada/salida y posteriormente vuelve a estar listo junto con otro proceso nuevo, ¿en qué orden deberían ejecutarse y cuánto tiempo le quedaría al primero?

**Respuesta del estudiante:**

Como B llegó antes de que A terminara su E/S, se ejecutaría primero B. En cuanto a los ticks, como sigue manteniendo la prioridad quiere decir que conserva los mismos ticks que ya tenía, en este caso si usó uno le queda 1.

**Aprendizaje:** se identificó la importancia de conservar el tiempo restante de un proceso cuando se bloquea y posteriormente vuelve a estar disponible.

### Interacción 2. Acumulación del tiempo de CPU

**Pregunta de Gemini:**

Si un proceso interactivo consume 0.5 ticks en cada una de cuatro interacciones, ¿qué ocurriría con su prioridad si el tiempo consumido se acumula aunque el proceso ceda la CPU?

**Respuesta del estudiante:**

Dado el ejemplo que mencionas, luego de todas esas interacciones con la CPU ese proceso consume la totalidad de su quantum en ese nivel de prioridad así que debe bajar de prioridad.

Este ejemplo es ilustrativo. En una implementación de xv6, el mecanismo de contabilización tendría que adaptarse a sus ticks enteros y a las interrupciones del temporizador.

**Aprendizaje:** se comprendió que el tiempo de CPU debe acumularse dentro del nivel de prioridad para evitar que un proceso conserve indefinidamente una prioridad alta mediante cesiones frecuentes de CPU.

### Interacción 3. Expulsión por prioridad

**Pregunta de Gemini:**

¿Qué debe hacer el planificador cuando un proceso de mayor prioridad se vuelve RUNNABLE mientras otro proceso de menor prioridad está ejecutándose?

**Respuesta del estudiante:**

En este caso, el planificador debe interrumpir el proceso actual y ceder CPU al proceso nuevo de mayor prioridad.

**Aprendizaje:** se identificó la necesidad de considerar la prioridad al decidir qué proceso debe utilizar la CPU. También se reconoció que esta regla debe analizarse en relación con las interrupciones y mecanismos de cesión de CPU existentes en xv6.

**Servicio utilizado:** Google Gemini.
**Modelo específico:** no registrado durante la consulta.

## 7. Conclusiones individuales

### Brian Rodríguez

Comprendí que el planificador original de xv6 recorre los procesos y selecciona aquellos que están en estado `RUNNABLE`, pero no establece diferentes niveles de prioridad. Al incorporar MLFQ, se podría mejorar la administración de la CPU mediante niveles de prioridad y presupuestos de tiempo de ejecución para cada nivel. Esto permitiría diferenciar el tratamiento de los procesos según su consumo de CPU y ofrecer oportunidades periódicas de ejecución a los procesos de menor prioridad.

Mi principal aporte fue proponer los parámetros de la variante de MLFQ, definiendo tres niveles de prioridad con presupuestos de 2, 4 y 8 ticks, respectivamente, y un *Priority Boost* cada 100 ticks. Estas decisiones buscan distribuir el tiempo de CPU según la prioridad de los procesos y reducir el riesgo de inanición (*starvation*).

Considero que todavía necesito profundizar en el funcionamiento de los bloqueos (*locks*) dentro del kernel de xv6, especialmente en cómo se deben adquirir y liberar al modificar las prioridades y los estados de los procesos. También necesito comprender mejor cómo evitar conflictos entre el planificador y las interrupciones del reloj.

## 8. Referencias

- MIT. *xv6-riscv: A simple, Unix-like teaching operating system*. https://github.com/mit-pdos/xv6-riscv
- Repositorio de trabajo: https://github.com/Brianbrd/xv6-riscv
- OSTEP. *Operating Systems: Three Easy Pieces*. Capítulo sobre planificación de CPU y Multilevel Feedback Queues. https://pages.cs.wisc.edu/~remzi/OSTEP/
