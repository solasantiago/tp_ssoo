# Teoría de la cátedra — Planificación de Procesos e Hilos

Fuente: `contenido_drive/Clase 1 y 2 - Planificación de Procesos e Hilos/Planificación.pptx` y `Procesos - Hilos.pptx` (diapositivas de la cátedra, no el enunciado del TP). Este archivo es **teoría de base**; la letra obligatoria del TP está en `11-planificador.md`. Cuando algo de acá no coincide con el enunciado, **gana el enunciado** (es una simplificación académica).

## Planificadores por plazo (visión completa de la materia)

La cátedra distingue **4** niveles (el TP solo pide implementar 2: largo y corto plazo):

- **Extra largo plazo**: lo hace el administrador del sistema (fuera de alcance del TP).
- **Largo plazo**: controla el **grado de multiprogramación**. Transiciones: `NEW → READY` y `cualquier estado → EXIT`. Corresponde en EntrenadOS a `GRADO_MULTIPROGRAMACION` y al pasaje NEW→READY por FIFO.
- **Mediano plazo**: controla la **suspensión** de procesos vía swapping (`READY/BLOCK ↔ SUSPENDIDO`). **No existe como tal en el TP** — la Placa no tiene estados "suspendido", pero el **Offload** cumple un rol análogo a nivel de páginas (no de Jobs completos): páginas que no están en la Placa "viven" en el Offload, igual que un proceso suspendido vive en la partición de swap.
- **Corto plazo**: controla qué proceso en READY pasa a EXEC (`READY → EXEC` y viceversa). Es el que el TP pide como `ALGORITMO_PLANIFICACION`.

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
