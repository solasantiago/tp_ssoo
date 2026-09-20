# Teoría de la cátedra — Sincronización

Fuente: `contenido_drive/Clase 3 - Sincronización/Sincronización.pptx`. Teoría de base; el enunciado no tiene una sección propia de "sincronización" (es transversal a los 3 módulos servidor), por eso este archivo no tiene un `1x-*` equivalente al que contrastar — sirve para justificar decisiones de diseño en el coloquio.

## Condición de carrera

Situación en la que varios procesos/hilos manipulan datos **compartidos concurrentemente**, de forma que el resultado depende del **orden** particular en que terminan de ejecutarse. Se da cuando se cumplen las **condiciones de Bernstein**: (1) más de un proceso/hilo usa el mismo recurso, (2) al menos uno lo modifica, (3) los accesos son concurrentes. Si dos procesos son **independientes** (sus conjuntos de lectura/escritura no se solapan: `Wa∩Wb = Wa∩Rb = Ra∩Wb = {}`), no hace falta sincronizarlos.

## Región / Sección crítica

El fragmento de código que accede a un recurso compartido debe ser lo más chico posible y ejecutarse en forma **atómica**. Protocolo: sección de entrada (pedir permiso) → sección crítica → sección de salida (liberar).

**Requisitos de una solución correcta:**
- **Mutua exclusión:** solo un proceso/hilo a la vez en la sección crítica.
- **Progreso:** solo los que están en la sección de entrada/salida deciden quién entra después (no puede bloquear indefinidamente por otros que ni siquiera quieren entrar).
- **Espera limitada:** un proceso que pide entrar no espera indefinidamente (sin starvation).
- **Independencia de velocidad:** la solución funciona sin importar cuánto tarden o cuántas veces se ejecuten las secciones críticas.

## Herramientas de sincronización (de más simple/limitada a más general)

- **Software puro** (algoritmo de Peterson, etc.): solo para 2 procesos, asume operaciones atómicas de por sí — no escalan.
- **Hardware — deshabilitar interrupciones:** evita que la sección crítica sea interrumpida. No sirve en multiprocesador (hay que avisarle a cada CPU) y es costoso.
- **Hardware — instrucciones atómicas (Test-and-Set, Compare-and-Swap):** eficientes incluso en multiprocesador, pero implican **espera activa** (busy waiting): el proceso consume CPU mientras espera.
- **Semáforos (a nivel de SO):** variable entera accedida únicamente por dos operaciones atómicas, `wait` (P) y `signal` (V). A diferencia de las anteriores, se puede implementar **con o sin espera activa** (con bloqueo: el proceso que hace `wait` sobre un semáforo agotado se bloquea y se encola; `signal` despierta al primero de la cola).

### Tipos de semáforo
- **Mutex (binario):** resuelve exclusión mutua. Se inicializa en 1. Representa libre/ocupado.
- **Contador:** controla el acceso a **N instancias** de un recurso. Se inicializa en N.
- Con bloqueo: el valor puede pensarse como negativo = cantidad de procesos bloqueados esperando; positivo = cantidad de recursos libres.

### Usos típicos de semáforos
1. **Mutua exclusión:** `wait(mutex); <SC>; signal(mutex);` alrededor del acceso al recurso compartido.
2. **Limitar acceso a N instancias:** semáforo contador inicializado en N.
3. **Ordenar ejecución:** un semáforo que arranca en 0 fuerza a que un hilo espere hasta que otro haga `signal`.
4. **Productor-consumidor:** patrón con 3 semáforos — uno mutex para la lista compartida, uno contador de "tareas pendientes" (arranca en 0, el productor hace `signal` y el consumidor `wait`) y uno de "lugares libres" (arranca en el tamaño del buffer, para no desbordarlo).

**Aplicación directa a los Servicios del TP** *[derivado]*: Loader, Labeler y Logger son colas FIFO de "una petición a la vez" — es exactamente el patrón productor-consumidor con un semáforo mutex protegiendo la cola y (opcionalmente) un semáforo contador de "hay trabajo pendiente" para no hacer polling activo mientras la cola de un servicio está vacía.

### Monitores (mencionado por completitud)
Mecanismo que provee mutua exclusión encapsulando el acceso a una estructura de datos: expone operaciones "seguras" (thread-safe) en vez de exponer `wait`/`signal` sueltos. Es una abstracción de más alto nivel sobre mutex/semáforos; el enunciado no exige usar monitores específicamente, es una alternativa de implementación.

### Inversión de prioridades y herencia de prioridades (contexto, no crítico para el TP)
Si un proceso de baja prioridad retiene un recurso que necesita uno de alta prioridad, y uno de prioridad intermedia lo desaloja, el de alta prioridad queda esperando indirectamente al de prioridad intermedia — **inversión de prioridades**. La solución (**herencia de prioridades**) es subirle temporalmente la prioridad al que tiene el lock mientras lo retiene. EntrenadOS no define prioridades de hilos del SO real (los Jobs tienen prioridad HRRN, que es otra cosa), así que esto es contexto teórico, no un requisito.

## Relación con el "page locking" del enunciado

El page locking de la Placa (`13-placa.md`) es conceptualmente un **mutex por página**: mientras una página está lockeada (en uso por una syscall o por el Core), no puede ser tocada por el algoritmo de reemplazo — evita que el hilo que hace reemplazo y el hilo que atiende al Core/Planificador "compitan" por la misma página al mismo tiempo. No es necesariamente un semáforo por página literal (puede implementarse con un flag protegido por el lock general de la tabla de páginas), pero el problema que resuelve es el mismo que ilustran los usos de semáforos de esta clase.
