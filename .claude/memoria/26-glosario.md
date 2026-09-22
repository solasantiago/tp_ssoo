# Glosario de términos — en el orden de las diapositivas de la cátedra

Términos nuevos de la materia, en el **orden de aparición** dentro de cada diapositiva de clase (mismo orden que se usó para estudiar, no orden alfabético). Las clases van en su orden cronológico real (`contenido_drive/Clase 1 y 2...`, `Clase 3...`, etc.); dentro de una clase con más de un `.pptx`, van en el orden en que se leyeron. Definiciones cortas, tipo diccionario — para la explicación completa de un concepto, ir al archivo `2x-teoria-*.md` o `1x-*.md` que se indica entre paréntesis.

## Clase 1 y 2 — Planificación de Procesos e Hilos

### `Planificación.pptx`

- **Planificador (de procesos):** módulo del SO que mueve los procesos entre las colas de planificación (NEW/READY/BLOCK/etc.), decidiendo el orden en que se ejecutan. (`20-teoria-planificacion.md`)
- **Ráfaga de CPU (CPU burst):** tramo en el que un proceso usa la CPU activamente.
- **Ráfaga de E/S (I/O burst):** tramo en el que un proceso espera una operación de entrada/salida, sin necesitar la CPU.
- **Proceso I/O-bound:** pasa más tiempo haciendo E/S que usando la CPU — ráfagas de CPU cortas.
- **Proceso CPU-bound:** pasa más tiempo procesando que haciendo E/S — ráfagas de CPU largas.
- **Planificador de largo plazo:** decide qué proceso nuevo entra al sistema (NEW → READY) y controla el grado de multiprogramación.
- **Grado de multiprogramación:** cantidad de procesos que conviven activos en el sistema a la vez.
- **Planificador de mediano plazo:** decide suspender procesos (sacarlos de RAM) o volver a cargarlos, vía swapping.
- **Swapping / Swap in / Swap out:** intercambio de un proceso entero entre RAM y almacenamiento secundario; swap in = vuelve a RAM, swap out = sale de RAM.
- **Planificador de corto plazo:** decide, de los procesos en READY, a cuál se le asigna la CPU ahora. Es el más frecuente de los tres.
- **Eventos de replanificación:** los sucesos que disparan al planificador de corto plazo (fin de proceso, bloqueo, cesión voluntaria, y opcionalmente desbloqueo, llegada de un nuevo proceso, o timer).
- **Con desalojo / apropiativo (preemptive):** el planificador puede interrumpir un proceso en ejecución para darle la CPU a otro.
- **Sin desalojo / no apropiativo (non-preemptive):** el planificador espera a que el proceso en ejecución devuelva la CPU por su cuenta.
- **FIFO / FCFS (First In, First Out / First Come, First Served):** se ejecuta en orden de llegada a READY, sin desalojo.
- **SJF (Shortest Job First):** se ejecuta primero el proceso con la ráfaga de CPU (estimada) más corta; sin desalojo.
- **SRT (Shortest Remaining Time):** SJF con desalojo — si llega uno con ráfaga estimada más corta que el tiempo restante del que está corriendo, lo expropia.
- **Starvation (inanición):** un proceso nunca (o casi nunca) llega a ejecutar porque siempre hay otros "mejor calificados" antes que él.
- **Estimación de ráfaga (aging exponencial):** fórmula `Est(n+1) = α·R(n) + (1-α)·Est(n)` para predecir la duración de la próxima ráfaga a partir de las anteriores.
- **Planificación por prioridad:** la CPU va al proceso de prioridad más alta (número más chico = más prioridad, por convención).
- **Aging (envejecimiento):** subir la prioridad de un proceso cuanto más tiempo lleva esperando, para evitar starvation.
- **HRRN (Highest Response Ratio Next):** adaptación de SJF que prioriza por `Response Ratio = (S+W)/S`, rompiendo la inanición.
- **Response Ratio:** relación entre tiempo de espera + ráfaga estimada, y la ráfaga estimada sola; a mayor espera, mayor prioridad.
- **Round Robin (RR):** cada proceso obtiene la CPU por un tiempo máximo (quantum); siempre con desalojo.
- **Quantum:** tiempo máximo que un proceso puede tener la CPU en RR antes de ser desalojado.
- **VRR (Virtual Round Robin):** variante de RR donde un proceso que se bloqueó antes de agotar su quantum conserva el resto para la próxima vez. *(no la pide el TP)*
- **Colas multinivel:** varias colas de READY organizadas por prioridad, cada una con su propio algoritmo. *(no las pide el TP)*
- **Colas multinivel retroalimentadas (feedback):** colas multinivel donde un proceso puede cambiar de cola según su comportamiento. *(no las pide el TP)*

### `Procesos - Hilos.pptx`

- **Proceso:** un programa en ejecución; la unidad de asignación de recursos del SO.
- **Concurrencia vs. paralelismo:** concurrencia = varios procesos avanzan intercalados (pueden compartir una sola CPU); paralelismo = avanzan literalmente al mismo tiempo (requiere varias CPUs/cores).
- **PCB (Process Control Block):** estructura con toda la info que el SO necesita para administrar un proceso (ID, estado, PC, registros, info de planificación/memoria/E-S). Siempre en RAM.
- **Stack (pila) del proceso:** guarda el contexto dinámico de las llamadas a función — en qué función está, con qué parámetros y variables locales.
- **Cambio de contexto:** guardar el contexto de un proceso y cargar el de otro al cambiar quién usa la CPU. Es overhead puro.
- **Process switch / Context switch / Mode switch:** tres niveles de "cambio" — de proceso completo, de contexto de ejecución, o de modo usuario↔kernel. Pueden anidarse entre sí.
- **PID:** identificador único de proceso.
- **`fork()`:** syscall que crea un proceso hijo como copia del padre.
- **Fork bomb:** ataque/bug que crea procesos en bucle infinito hasta agotar los recursos del sistema.
- **`exit()` / `wait()` / `abort()`:** syscalls de finalización — terminar el propio proceso, esperar a que termine un hijo, o forzar la terminación de un hijo desde el padre.
- **Terminación en cascada:** cuando el padre termina, el SO fuerza la terminación de todos sus hijos.
- **IPC (Inter-Process Communication):** mecanismos para que procesos cooperativos se comuniquen.
- **Memoria compartida (IPC):** los procesos comparten una zona de memoria para leer/escribir datos directamente — rápido, sin intervención del SO una vez establecida.
- **Paso de mensajes (IPC):** comunicación vía syscalls — más lento (requiere cambios de contexto) pero no necesita memoria compartida.
- **Hilo (thread):** unidad básica de utilización de CPU dentro de un proceso — un juego de registros + una pila. Comparte código/datos/recursos con sus hilos pares.
- **TCB (Thread Control Block):** estructura con la info propia de un hilo (registros, PC, pila), referenciada desde el PCB del proceso.
- **KLT (Kernel-Level Thread):** hilo que el SO conoce y planifica directamente. EntrenadOS usa estos (`pthread`).
- **ULT (User-Level Thread) / Green Thread:** hilo gestionado por una biblioteca en modo usuario; el SO no lo ve. No permite paralelismo real entre hilos pares.
- **Jacketing:** técnica para que un ULT que hace una syscall bloqueante no bloquee a todos sus hilos pares.

## Clase 3 — Sincronización

- **Condición de carrera (race condition):** el resultado de la ejecución depende del orden particular en que terminan varios procesos/hilos que comparten datos. (`21-teoria-sincronizacion.md`)
- **Condiciones de Bernstein:** las 3 condiciones que, si se cumplen todas, generan una condición de carrera (mismo recurso, al menos una escritura, acceso concurrente).
- **Procesos independientes / cooperativos / competitivos:** independientes = no se afectan entre sí; cooperativos = comparten datos y colaboran; competitivos = compiten por el mismo recurso.
- **Sección crítica (región crítica):** el fragmento de código que accede a un recurso compartido y debe ejecutarse de forma atómica.
- **Sección de entrada / sección de salida:** el código antes y después de la sección crítica, donde se pide y se libera el permiso de acceso.
- **Mutua exclusión:** solo un proceso/hilo a la vez puede estar en la sección crítica.
- **Progreso:** solo los que quieren entrar a la SC deciden quién entra después — no puede bloquear por otros que no la necesitan.
- **Espera limitada:** un proceso que pide entrar a la SC no espera indefinidamente.
- **Solución de Peterson:** algoritmo por software para 2 procesos que garantiza mutua exclusión, progreso y espera limitada.
- **Deshabilitar interrupciones:** mecanismo de hardware para proteger una SC evitando que se interrumpa; no sirve en multiprocesador.
- **Instrucciones atómicas (Test-and-Set):** instrucciones de hardware que leen y modifican una variable en un solo paso indivisible.
- **Espera activa (busy waiting):** el proceso consume CPU mientras espera poder entrar a la SC, en vez de bloquearse.
- **Semáforo:** variable entera accedida solo por `wait`/`signal` (operaciones atómicas), para sincronizar procesos/hilos.
- **`wait` / `signal` (P/V):** las dos operaciones atómicas de un semáforo — decrementar (y bloquear si queda negativo) / incrementar (y despertar a alguien si corresponde).
- **Semáforo mutex (binario):** resuelve exclusión mutua; se inicializa en 1.
- **Semáforo contador:** controla el acceso a N instancias de un recurso; se inicializa en N.
- **Spinlock:** mutex implementado con espera activa — conviene con secciones críticas muy cortas o en multiprocesador.
- **Monitor:** mecanismo de más alto nivel que encapsula el acceso a una estructura de datos, exponiendo operaciones "thread-safe" en vez de `wait`/`signal` sueltos.
- **Inversión de prioridades:** un proceso de alta prioridad queda esperando indirectamente a uno de prioridad intermedia, porque este desalojó al de baja prioridad que tenía el recurso.
- **Herencia de prioridades:** solución a la inversión — subirle temporalmente la prioridad al proceso que retiene el recurso mientras lo tiene.

## Clase 4 — Deadlocks *(no lo pide el enunciado de EntrenadOS)*

- **Deadlock (interbloqueo):** un conjunto de procesos donde todos esperan un suceso que solo puede producir otro proceso del mismo conjunto — ninguno avanza nunca. (`24-teoria-deadlocks.md`)
- **Recursos consumibles / reusables:** consumibles se generan y desaparecen (interrupción, señal); reusables se piden y liberan repetidamente (memoria, archivos, semáforos).
- **Exclusión mutua (condición de deadlock):** al menos un recurso involucrado es de uso no compartido.
- **Retención y espera:** un proceso retiene un recurso mientras espera adquirir otros.
- **Sin desalojo (condición de deadlock):** un recurso solo se libera voluntariamente.
- **Espera circular:** existe un ciclo de procesos, cada uno esperando un recurso que retiene el siguiente.
- **Prevención de deadlocks:** política que ataca una de las 4 condiciones necesarias para que nunca se den todas juntas.
- **Evasión de deadlocks:** simula cada asignación de recursos para verificar que el sistema quede en estado seguro antes de otorgarla (Algoritmo del Banquero).
- **Algoritmo del Banquero:** algoritmo de evasión que usa matrices de peticiones máximas/asignadas y el vector de recursos totales.
- **Estado seguro / inseguro:** seguro = existe algún orden en que todos los procesos podrían terminar sin deadlock; inseguro = podría ocurrir deadlock (no es seguro que ocurra, pero podría).
- **Detección y recuperación de deadlocks:** se permite pedir libremente, se detecta el deadlock periódicamente y se recupera (finalizando procesos o desalojando recursos).
- **Livelock:** similar al deadlock, pero los procesos siguen ejecutándose sin lograr avanzar (no están bloqueados) — más difícil de detectar.

## Clase 5 y 6 — Memoria y Memoria Virtual

### `Memoria.pptx`

- **MMU (Memory Management Unit):** hardware que traduce direcciones lógicas a físicas. (`22-teoria-memoria-virtual.md`)
- **Dirección lógica / dirección física:** la que usa el proceso (relativa a su propio espacio) vs. la ubicación real en la memoria física.
- **Realocación:** capacidad de reasignar a un proceso una ubicación distinta en memoria (en tiempo de compilación, carga o ejecución).
- **Asignación contigua:** cada proceso ocupa un bloque contiguo de memoria física.
- **Particiones fijas / dinámicas:** fijas = tamaño predefinido, N particiones; dinámicas = el tamaño de la partición se ajusta al del proceso.
- **Fragmentación externa:** espacio libre total suficiente, pero repartido en huecos chicos no contiguos que no alcanzan para una nueva asignación.
- **Compactación:** reubicar los procesos en memoria para juntar todo el espacio libre en un solo bloque contiguo.
- **Primer ajuste / mejor ajuste / peor ajuste / siguiente ajuste:** estrategias para elegir en qué hueco libre ubicar una nueva asignación.
- **Fragmentación interna:** espacio asignado a un proceso que le sobra y queda desperdiciado (por ejemplo, dentro de la última página).
- **Segmentación:** el proceso se divide en segmentos lógicos (código/datos/pila) de tamaño variable. *(EntrenadOS no usa segmentación)*
- **Paginación (simple):** memoria física dividida en marcos y proceso dividido en páginas, ambos del mismo tamaño fijo.
- **Página / marco (frame):** unidad de división del espacio lógico de un proceso / unidad de división de la memoria física.
- **Tabla de páginas:** estructura por proceso que indica en qué marco está cada página (y si es válida).
- **Bit de validez:** indica si una página pertenece al espacio de direcciones del proceso.
- **Bitmap de marcos libres:** estructura que indica qué marcos de memoria física están libres.
- **PTBR (Page Table Base Register):** registro de CPU con el puntero a la tabla de páginas del proceso en ejecución.
- **TLB (Translation Look-aside Buffer):** caché de hardware con traducciones recientes de página→marco, para no ir siempre a la tabla de páginas. *(EntrenadOS no lo pide)*
- **Paginación jerárquica:** tabla de páginas dividida en niveles, para no necesitar una tabla contigua gigante. *(EntrenadOS no la usa)*
- **Tabla de páginas invertida:** una sola tabla para todo el sistema, indexada por marco en vez de por página. *(EntrenadOS no la usa)*
- **Segmentación paginada:** cada segmento tiene su propia tabla de páginas. *(EntrenadOS no la usa)*

### `Memoria Virtual.pptx`

- **Overlay:** técnica manual (previa a la memoria virtual) para que un proceso grande quepa en poca RAM, cargando y descargando secciones a mano.
- **Memoria virtual:** permite que un proceso tenga partes en RAM y partes en disco, de forma transparente para el programador.
- **Paginación bajo demanda (demand paging):** una página se carga a memoria física recién cuando el proceso la accede, no al crear el proceso.
- **Bit de presencia:** indica si una página está actualmente cargada en un marco (`P=1`) o no (`P=0`).
- **Page Fault (fallo de página):** interrupción que se dispara al acceder a una página con `P=0`; el SO debe cargarla antes de continuar.
- **Asignación fija / dinámica (de frames):** fija = un proceso siempre tiene N marcos asignados; dinámica = puede variar.
- **Sustitución local / global:** local = la víctima sale del propio conjunto de marcos del proceso; global = puede ser de cualquier proceso.
- **Anomalía de Belady:** en ciertos algoritmos (como FIFO), aumentar la cantidad de marcos puede generar *más* Page Faults, no menos.
- **Algoritmo FIFO (reemplazo):** víctima = la página cargada hace más tiempo. Sufre la Anomalía de Belady.
- **Algoritmo óptimo (OPT/MIN):** víctima = la que no se va a usar por más tiempo hacia adelante. Da el mínimo teórico de fallos, pero requiere conocer el futuro.
- **Algoritmo LRU (Least Recently Used):** víctima = la menos recientemente usada. No sufre la Anomalía de Belady. *(uno de los dos que pide el TP)*
- **Algoritmo CLOCK:** aproximación barata de LRU con un bit de uso y un puntero circular.
- **Algoritmo CLOCK-M (Clock Modificado):** CLOCK + bit de modificado, para minimizar escrituras al respaldo eligiendo primero víctimas no modificadas. *(el otro algoritmo que pide el TP)*
- **Thrashing (sobrepaginación):** un proceso con menos marcos de los que necesita genera Page Faults constantemente sin avanzar trabajo útil.
- **Localidad:** conjunto de páginas que un proceso referencia activamente en un intervalo de tiempo.
- **Conjunto de trabajo (working set):** aproximación práctica de la localidad de un proceso, usada para prevenir thrashing.
- **Lockeo de páginas (page locking):** marcar una página/marco para que no pueda ser elegido como víctima mientras está en uso activo.
- **Page buffering:** mantener un pool de marcos ya libres para no tener que esperar a escribir la víctima antes de poder usar el marco nuevo.

## Clase 7 y 8 — File Systems

### `File Systems.pptx`

- **Archivo:** conjunto de datos etiquetado con un nombre, de existencia duradera y compartible entre procesos. (`23-teoria-filesystems.md`)
- **FCB (File Control Block):** estructura administrativa de un archivo (metadata). *(FAT no usa FCB separado — la info va directo en la entrada de directorio)*
- **Directorio:** archivo que lista nombres de otros archivos y sus atributos, mapeando nombres a archivos reales.
- **Rutas relativas / absolutas:** ubicación de un archivo expresada desde el directorio actual, o desde la raíz.
- **Grafo acíclico (estructura de directorios):** estructura de directorios que permite links pero sin ciclos.
- **Tabla de archivos abiertos (global / por proceso):** estructuras que llevan el registro de qué archivos están abiertos y por quién.
- **Locks de archivo:** exclusivo (uno solo) vs. compartido (varios lectores); obligatorio (el SO lo garantiza) vs. sugerido (el programador debe respetarlo).
- **ACL (Access Control List):** lista, por archivo, de qué usuarios tienen qué permisos.
- **MBR:** bloque de control de arranque de un disco físico, que puede tener varias particiones.
- **Volumen:** una partición formateada con un filesystem específico.
- **Bloque lógico:** unidad mínima de asignación de un filesystem (`tam_bloque = N × tam_sector`).
- **Asignación contigua / enlazada / indexada (de bloques):** tres estrategias para asignar los bloques de disco a un archivo. FAT es una variante de la enlazada.
- **Bloque de índice:** en asignación indexada, un bloque aparte con los punteros a todos los bloques de datos del archivo.
- **Índice multinivel:** un bloque de índice de primer nivel apunta a bloques de índice de segundo nivel, para soportar archivos más grandes.
- **Gestión de espacio libre (bit vector, lista enlazada):** formas de llevar el registro de qué bloques del disco están libres. *(en FAT esto ya lo resuelve la propia tabla, con el valor 0)*
- **Journaling:** registrar las modificaciones en un log secuencial antes de aplicarlas al filesystem real, para poder recuperarse ante una falla a mitad de una operación.
- **Archivos mapeados a memoria:** tratar la E/S de un archivo como accesos directos a memoria, evitando el overhead de `read`/`write`.

### `FAT y EXT2.pptx`

- **FAT (File Allocation Table):** tabla con una entrada por bloque de disco, donde cada entrada indica el bloque siguiente de la cadena — variante de asignación enlazada, sin FCB separado. Base de `FAT32_TRAIN` del TP. (`25-teoria-libro-complementos.md`)
- **Cluster:** agrupación de varios bloques físicos tratados como una sola unidad de asignación, para reducir el overhead de punteros.
- **Inodo:** FCB de un archivo en filesystems tipo EXT — contiene metadata y punteros directos e indirectos a los bloques de datos.
- **Punteros directos / indirectos (simples, dobles, triples):** en un inodo, apuntan directo a un bloque de datos, o a un bloque de punteros (con 1, 2 o 3 niveles de indirección).
- **EXT2:** filesystem de asignación indexada pura, con inodos y grupos de bloques.
- **Grupo de bloques:** región de un volumen EXT2 con su propio superbloque, bitmap de datos/inodos e inodos, para acelerar el acceso.
- **Softlink (symbolic link):** archivo independiente cuyo contenido es la ruta a otro archivo; tiene su propio inodo.
- **Hardlink:** una referencia adicional al mismo inodo de un archivo ya existente; no puede cruzar filesystems.
