# Teoría de la cátedra — Planificación de Procesos e Hilos

Fuente: `contenido_drive/Clase 1 y 2 - Planificación de Procesos e Hilos/Planificación.pptx` y `Procesos - Hilos.pptx` (diapositivas de la cátedra, no el enunciado del TP). Este archivo es **teoría de base**; la letra obligatoria del TP está en `11-planificador.md`. Cuando algo de acá no coincide con el enunciado, **gana el enunciado** (es una simplificación académica).

## Qué es un planificador (definición, antes de ver los niveles)

**Planificación de procesos** es el conjunto de políticas y mecanismos del SO que gobiernan **el orden en que se ejecutan los procesos**. Un **planificador de procesos** es, concretamente, un **módulo del SO que mueve los procesos entre las distintas colas de planificación** (NEW, READY, BLOCK, etc.) — es decir, decide las transiciones del diagrama de estados, no ejecuta él mismo las instrucciones del proceso.

La razón de que existan **varios niveles de planificador** (y no uno solo) es que la ejecución de un proceso alterna entre dos tipos de actividad:

- **Ráfaga de CPU** (CPU burst): tramo en el que el proceso usa la CPU activamente.
- **Ráfaga de E/S** (I/O burst): tramo en el que el proceso espera una operación de entrada/salida y no necesita la CPU.

Según cómo sea esa alternancia, un proceso se clasifica en:
- **CPU-bound** (limitado por CPU): pasa más tiempo procesando que haciendo E/S — tiene ráfagas de CPU **largas**.
- **I/O-bound** (limitado por E/S): pasa más tiempo haciendo E/S que usando la CPU — tiene ráfagas de CPU **cortas**.

Cada nivel de planificador atiende una preocupación distinta sobre esta alternancia: cuántos procesos conviven (largo plazo), qué mezcla de CPU-bound/I/O-bound conviene tener activa a la vez (mediano plazo), y a cuál de los que ya están listos se le da la CPU en este instante (corto plazo). *[derivado]* En EntrenadOS, la alternancia CPU/E-S de un Job real se traduce en: ejecuta instrucciones (ráfaga de "CPU" en el Core) hasta que pide una syscall bloqueante, un Servicio o sufre un Page Fault (equivalente a una "ráfaga de E/S": el Job se bloquea esperando algo externo).

## Planificadores por plazo (visión completa de la materia)

La cátedra distingue **4** niveles (el TP solo pide implementar 2: largo y corto plazo):

### Planificador de largo plazo
- **Qué decide:** si se agrega un nuevo proceso al conjunto de procesos activos del sistema — es decir, **qué Job entra** y **en qué momento** se le permite pasar de NEW a READY.
- **Cuándo se ejecuta:** cuando se crea un proceso nuevo.
- **Qué controla:** el **grado de multiprogramación** — cuántos procesos conviven "activos" en el sistema a la vez.
  - **Alto** grado de multiprogramación → la CPU no queda ociosa, pero cada proceso recibe un porcentaje menor de CPU.
  - **Bajo** grado de multiprogramación → la CPU puede quedar ociosa, pero se brinda un servicio más satisfactorio a los procesos que sí están LISTOS.
  - Otras señales que puede mirar (más allá del criterio simple de "hay lugar o no"): que un proceso finalice (libera lugar → baja el grado de multiprogramación), monitorear tiempo de CPU ociosa, prioridad del Job que quiere entrar, y buscar una **buena mezcla de procesos CPU-bound / I/O-bound** para optimizar el uso de la CPU (si entran muchos I/O-bound, la CPU tiende a quedar más libre para los CPU-bound, y viceversa).
- **En EntrenadOS:** corresponde a `GRADO_MULTIPROGRAMACION` y al pasaje NEW → READY por FIFO. La transición `NEW → READY` y `cualquier estado → EXIT` son las que gestiona este nivel. Como acá no hay ráfagas de E/S reales que perfilar de antemano (no se sabe si un Job es "CPU-bound" o "I/O-bound" antes de correrlo), el criterio de "buena mezcla" no aplica — el TP simplifica a FIFO puro limitado por el grado de multiprogramación.

### Planificador de mediano plazo
- **Qué decide:** si hace falta **suspender** un proceso (sacarlo de RAM y llevarlo a almacenamiento secundario) o volver a cargar en RAM un proceso previamente suspendido. Esta operación de intercambio se llama **swapping**: `SWAP OUT` (sale de RAM, disminuye el grado de multiprogramación efectivo) y `SWAP IN` (vuelve a RAM, lo aumenta).
- **Para qué sirve:** igual que el largo plazo, busca una buena mezcla de procesos CPU-bound/I/O-bound activos, pero actuando sobre procesos que **ya estaban activos** (a diferencia del largo plazo, que decide sobre los que recién quieren entrar).
- **Ejemplos de cuándo actúa** (de la propia diapositiva de la cátedra):
  - Muchos procesos I/O-bound, todos bloqueados esperando E/S → la CPU queda ociosa (IDLE) → conviene suspender alguno de esos y cargar en su lugar procesos CPU-bound desde el swap.
  - Muchos procesos CPU-bound compitiendo → mal uso de los dispositivos de E/S (nadie los usa) → conviene suspender alguno y cargar procesos I/O-bound desde el swap.
  - Llega un proceso de mayor prioridad y no hay RAM libre → se suspende uno de menor prioridad para hacerle lugar.
  - Un proceso suspendido está por desbloquearse (ya casi termina la espera que lo tenía afuera) y hay RAM libre → conviene cargarlo ya, para acelerar su vuelta a ejecución.
- **En EntrenadOS:** **no existe como tal** — la Placa no tiene un estado "Job suspendido" (los Jobs son NEW/READY/EXEC/BLOCK/EXIT, nada más). El **Offload** cumple un rol *análogo* pero a nivel de **páginas individuales**, no de Jobs completos: una página que no está en la Placa "vive" en el Offload, igual que un proceso suspendido vive en la partición de swap real — pero decidir qué página bajar es el algoritmo de **reemplazo** (LRU/CLOCK-M) de la Placa, no un "planificador de mediano plazo" separado.

### Planificador de corto plazo
- **Qué decide:** de todos los procesos que ya están en RAM y **listos** para ejecutar (READY), a **cuál** se le asigna la CPU ahora.
- **Frecuencia:** es el que se ejecuta **con más frecuencia que todos los demás** — por eso tiene dos requisitos en tensión: debe tomar buenas decisiones, pero su propio **overhead debe ser mínimo** (no puede ser costoso de correr, porque se corre todo el tiempo).
- **Cuándo se invoca:** cada vez que ocurre un evento que **libera la CPU** o que da la oportunidad de elegir un proceso "más prioritario" — interrupciones, llamadas al sistema, señales (el detalle exacto de estos eventos está en la sección siguiente).
- **En EntrenadOS:** es el que el TP pide implementar como `ALGORITMO_PLANIFICACION` (FIFO/RR/HRRN), decidiendo la transición `READY → EXEC`.

### Planificador de extra largo plazo
Lo hace el administrador del sistema (una persona, no un algoritmo del SO) — está fuera de alcance tanto de la teoría en detalle como del TP, se menciona solo para completar los 4 niveles.

## Eventos de replanificación (corto plazo)

Siempre se consideran (obligan a replanificar porque liberan la CPU):
- Proceso finaliza (EXEC → EXIT)
- Proceso se bloquea (EXEC → BLOCK)
- Proceso cede voluntariamente la CPU (EXEC → READY)

Pueden considerarse además (definen si el algoritmo es **con o sin desalojo**):
- Proceso recibe el evento que esperaba (BLOCK → READY)
- Proceso nuevo entra a READY (NEW → READY)
- Interrupción por timer (fin de quantum)

- **Sin desalojo / No apropiativo (non-preemptive)**: solo mira los eventos obligatorios; espera a que el proceso devuelva el control.
- **Con desalojo / Apropiativo (preemptive)**: además considera al menos uno de los otros eventos; puede interrumpir un proceso en EXEC y devolverlo a READY para correr uno "más prioritario".

*[derivado]* **RR es siempre con desalojo** (necesita la interrupción de timer del quantum). **FIFO tal como lo pide el TP es sin desalojo** entre los Jobs que compiten por CPU (aunque en EntrenadOS los Jobs igual salen de EXEC por syscalls bloqueantes o Page Fault, que son eventos "obligatorios", no desalojo por prioridad). **HRRN es sin desalojo**: se decide solo al liberarse la CPU, no interrumpe una ráfaga en curso.

## Criterios de evaluación de un algoritmo de corto plazo

- **Orientados al usuario:** tiempo de respuesta, predictibilidad, cumplimiento de deadlines.
- **Orientados al sistema:** utilización de CPU, throughput, tiempo de espera, tiempo de ejecución, utilización de recursos, respetar prioridades.

## FIFO / FCFS (First In, First Out / First Come, First Served)

Se ejecutan en el orden de llegada a READY, sin interrupciones entre sí. Simple, pero el tiempo de espera medio depende mucho del orden de llegada (un proceso largo al principio hace esperar mucho a los demás — "efecto convoy").

## SJF / SRT (Shortest Job First / Shortest Remaining Time)

- Asigna la CPU al proceso con la **ráfaga de CPU más corta** (estimada). Empates se resuelven por FIFO.
- **Sin desalojo (SJF):** una vez asignada la CPU, no se expropia hasta terminar la ráfaga.
- **Con desalojo (SRT — Shortest Remaining Time Next):** si llega un proceso a READY con una ráfaga más corta que el tiempo **restante** del que está en EXEC, se lo expropia.
- SJF es **óptimo**: da el mínimo tiempo de espera medio para un conjunto de procesos dado. Requiere conocer (o estimar) de antemano la duración de la ráfaga. Puede producir **starvation** de procesos con ráfagas largas.

### Estimación de la próxima ráfaga (clave para HRRN en el TP)

No se puede saber cuánto va a durar la próxima ráfaga, pero se puede **estimar** como un promedio ponderado (exponential aging) de las ráfagas anteriores:

```
Est(n)   = estimado de la ráfaga anterior
R(n)     = lo que realmente ejecutó la ráfaga anterior en la CPU
Est(n+1) = α · R(n) + (1 − α) · Est(n)        con α ∈ [0, 1]
```

**Esta es la fórmula que corresponde a `HRRN_ALFA` y `ESTIMACION_INICIAL` del Planificador** (confirma lo que en `11-planificador.md` estaba marcado como *[derivado]*: `ESTIMACION_INICIAL` es `Est(0)`, la estimación antes de la primera ráfaga real). Cuanto más grande α, más peso tiene lo último ejecutado (reacciona rápido a cambios); más chico α, más peso tiene el historial (más estable).

## Prioridades

- Se asocia a cada proceso una prioridad (entero); la CPU va al de prioridad más alta (por convención, número más chico = más prioridad).
- Puede ser con o sin desalojo.
- **SJF es un caso particular de planificación por prioridad**, donde la prioridad es la duración estimada de la próxima ráfaga (menor duración = mayor prioridad).
- Problema: **starvation** de los procesos de baja prioridad. Solución: **aging** — la prioridad de un proceso aumenta cuanto más tiempo lleva esperando.

## HRRN (Highest Response Ratio Next)

Adaptación de SJF que **rompe la inanición** combinando espera y duración de ráfaga:

```
S = duración de la próxima ráfaga de CPU (service time) — la estimación de arriba
W = tiempo de espera en READY (wait time)

RR (Response Ratio) = (S + W) / S = 1 + W / S
```

Se ejecuta el proceso con **mayor Response Ratio**. A mayor tiempo de espera, mayor RR (evita starvation); a mayor duración de ráfaga estimada, menor RR (favorece procesos cortos, como SJF). **Es sin desalojo**: la decisión se toma al liberarse la CPU, no interrumpe una ráfaga en curso.

**Esta es la fórmula de "Prioridad HRRN calculada"** que pide loguear el enunciado (`## (<JID>) - Prioridad HRRN calculada: <PRIORIDAD> - Estimación próxima ráfaga: <ESTIMACION>`). Confirma el *[derivado]* de `11-planificador.md`.

*[derivado]* Qué cuenta como "W" (tiempo de espera) para un Job que vuelve de BLOCK a READY (p. ej. tras resolver un Page Fault o terminar un servicio) es una decisión de diseño del grupo: el enunciado no lo aclara y hay que documentarlo en `03-decisiones-de-diseno.md` cuando se resuelva. La teoría solo habla de "tiempo de espera en READY" en términos generales.

## Round Robin (RR)

- Cada proceso obtiene la CPU durante una cantidad máxima de tiempo: el **quantum**.
- **Siempre con desalojo**: al vencer el quantum se dispara una interrupción de timer, el proceso es expropiado (vuelve a READY) e insertado **al final** de la cola de listos.
- Con `n` procesos en READY y quantum `q`, ningún proceso espera más de `q · (n − 1)` unidades de tiempo.
- No minimiza el tiempo de espera medio, pero lo hace **muy previsible**.
- Tamaño del quantum: **muy grande** → se comporta como FIFO; **muy chico** → demasiado overhead de cambios de contexto.

*[derivado]* Esto es exactamente `RR_QUANTUM` del TP: el desalojo por fin de quantum es la transición **EXEC → READY** con el log `"## (<JID>) - Desalojado por fin de quantum"`.

### VRR (Virtual Round Robin) — variante, no pedida por el TP

Si un proceso se bloquea **antes** de consumir todo su quantum (por ejemplo, por I/O), guarda el remanente `Q' = Q - (lo que ejecutó)` y, al volver a READY, se le da prioridad para consumir ese resto antes que un quantum nuevo. Es una optimización sobre RR puro; el enunciado de EntrenadOS **no la pide** (no hay parámetro de config para esto), se menciona solo como contexto teórico.

## Colas multinivel (mencionado por completitud, no lo pide el TP)

Varias colas READY organizadas por prioridad, cada una con su propio algoritmo. Las **colas multinivel retroalimentadas (feedback)** permiten que un proceso cambie de cola según su comportamiento. Para definir un esquema de este tipo hace falta decidir: número de colas, algoritmo de cada cola, a qué cola entran los procesos nuevos, criterio de cambio de cola, y si el algoritmo entre colas es con desalojo. **EntrenadOS usa una sola cola de READY** con un único algoritmo global (FIFO, RR o HRRN, fijo por config) — no hay colas multinivel en este TP.

## Contexto de ejecución y cambio de contexto

- **PCB (Process Control Block):** estructura con toda la info que el SO necesita para administrar un proceso — ID, estado, PC, registros de CPU, info de planificación, de memoria, contable, de E/S, punteros. Está **siempre en RAM**. *[derivado]* El "contexto de ejecución" del TP (copia de los registros del Core por Job) es la parte de un PCB real que le interesa a este TP; el resto de la info administrativa (estado, estadísticas, tabla de páginas) vive en las estructuras propias del Planificador/Placa.
- **Cambio de contexto:** al cambiar qué proceso tiene la CPU, hay que guardar su contexto para poder reanudarlo después. Puede darse por: ejecutar otro proceso, atender una interrupción, o ejecutar una syscall. **Es overhead puro**: durante el cambio de contexto el sistema no hace trabajo útil para el usuario. *[derivado]* En EntrenadOS, cada vez que el Planificador manda un Job distinto a un Core (tras un Page Fault, una syscall bloqueante, fin de quantum, etc.) hay un cambio de contexto real: el contexto viejo se guarda en las estructuras del Job y el nuevo se envía al Core.

## Hilos (relevante para "servidor multihilo")

- Un **hilo** es la unidad básica de utilización de CPU: un juego de registros + una pila. Comparte **código, datos y recursos** con sus hilos pares dentro del mismo proceso. Cada hilo tiene su propio **TCB**, referenciado desde el PCB del proceso.
- Un proceso multihilo puede tener varios hilos ejecutando (o listos) simultáneamente sin usar mecanismos de IPC entre ellos, porque comparten memoria — pero **no hay protección entre hilos del mismo proceso**.
- **KLT (Kernel-Level Threads)** vs **ULT (User-Level Threads)**: los KLT los conoce y planifica el SO; los ULT los gestiona una biblioteca en modo usuario y el SO no los ve. EntrenadOS usa hilos reales del SO (p. ej. `pthread`), es decir, **KLT**.

### Aplicación directa al TP (según la propia diapositiva de cierre de la cátedra)

> "Creación de Hilos → `pthread_create`. Esperar a finalización de otro hilo → `pthread_join`. Ejecución con hilos, ejemplos de uso: esperar conexiones de otros módulos mientras se siguen realizando tareas; atender mensajes concurrentemente de otros módulos (Placa ← Core, Placa ← Planificador); mandar mensajes concurrentemente a otros módulos (Planificador → Placa, Planificador → Core)."

Esto es exactamente el requisito de "servidor multihilo" de Planificador, Placa y Storage: un hilo aceptando conexiones nuevas mientras otros hilos atienden a los clientes ya conectados.

## Y esto cómo afecta al TP (síntesis de la propia cátedra)

La diapositiva final de "Planificación" lista textualmente lo que hay que llevarse de este tema para EntrenadOS: contexto de ejecución (qué se guarda y por qué), diagrama de 5 estados, planificación de largo plazo, planificación de corto plazo (FIFO, RR y HRRN), bloqueo por uso de recursos, y el modelo cliente-servidor (servidor monohilo vs. multihilo).
