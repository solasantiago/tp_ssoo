# CLAUDE.md

## Memoria persistente del proyecto

La memoria del asistente vive en `.claude/memoria/` (dentro del repo, para que viaje entre computadoras). Los tres archivos base se cargan acá en cada sesión; el resto se lee bajo demanda según indica el índice.

@.claude/memoria/00-indice.md
@.claude/memoria/01-estado-actual.md
@.claude/memoria/02-como-trabajamos.md

Regla: ante cualquier detalle fino de un módulo (formato de log obligatorio, campo de config, layout de bytes, semántica de una syscall), leer el archivo `1x-*` correspondiente antes de responder. Lo que sigue en este archivo es un resumen orientativo; si contradice a `.claude/memoria/1x-*`, ganan estos (y sobre todos, el PDF).

## Proyecto: EntrenadOS

TP Cuatrimestral de la Cátedra de Sistemas Operativos (UTN FRBA, 2C2026, v1.0 del 25/08/2026). Consiste en diseñar e implementar, en C, un **sistema distribuido** que simula un sistema operativo para un cluster de entrenamiento de modelos de IA. Los usuarios envían Jobs de entrenamiento que el sistema planifica, ejecuta en una CPU simulada (Core) y administra en memoria (Placa), con persistencia de checkpoints (Storage).

- **Modalidad:** grupal (5 integrantes ± 0), obligatorio.
- **Fecha de comienzo:** 25/08.
- **Entregas finales:** 28/11, 12/12, 19/12.
- **Lugar de corrección:** Laboratorio de Sistemas - Medrano.
- **Evaluación:** 2 etapas — pruebas de laboratorio (requiere logs mínimos y obligatorios, si no se cumplen el TP queda no evaluable/desaprobado) y, si se aprueba, un coloquio individual. El coloquio no es recuperable. Una implementación que contradiga la teoría vista en clase es motivo de desaprobación directa.
- Desarrollar únicamente temas de conectividad, serialización o sincronización es **insuficiente** — motivo de desaprobación directa.

## Los dos problemas que definen el diseño

1. **La memoria de la Placa es más chica que el espacio que necesitan los Jobs**, y varios Jobs conviven en ella al mismo tiempo → paginación bajo demanda + mecanismo de swap llamado **Offload**.
2. **Un entrenamiento puede durar días** → los Jobs deben poder guardar su estado (checkpoints) y retomarlo más adelante, incluso tras una caída del sistema.

### Preconceptos teóricos detrás de estos dos problemas

**Paginación bajo demanda (demand paging):** técnica de gestión de memoria virtual donde una página lógica no se carga en un marco físico hasta el momento en que efectivamente se accede a ella (lazy loading), en vez de cargar todo el espacio de direcciones del proceso al crearlo. Esto permite que procesos con espacios de direcciones grandes convivan en memoria física reducida, ya que en un instante dado solo ocupan marcos las páginas realmente en uso (working set). El costo es el **Page Fault**: cuando se accede a una página no residente, el sistema debe interrumpir la ejecución, ubicar o crear un marco libre, traer la página desde el medio de respaldo y recién ahí reanudar. En EntrenadOS, la Placa es quien decide "no está en memoria" y dispara ese flujo hacia el Planificador (que bloquea el Job) y de vuelta a la Placa (que hace el fetch real).

**Mecanismo de swap (Offload):** cuando la memoria física está llena y hace falta un marco para una página entrante, el sistema debe elegir una **página víctima** entre las residentes no lockeadas y desalojarla a un área de respaldo en disco (aquí, el archivo de Offload) para liberar su marco — esto es swap-out; si esa página se vuelve a necesitar más adelante, se hace swap-in. La elección de víctima es el problema central del *reemplazo de páginas*: algoritmos como LRU (Least Recently Used, desaloja la que hace más tiempo no se usa) o CLOCK-M (aproximación de LRU con bit de uso y bit de modificación, más barata de mantener) buscan minimizar la cantidad de swaps futuros. El "lock" de páginas (mientras una syscall o el Core las está usando) existe para que no sean elegidas como víctimas mientras están en uso activo — evita condiciones de carrera entre el reemplazo y el acceso concurrente.

**Guardado de estado de Jobs (checkpointing):** mecanismo para persistir periódicamente el estado necesario para reanudar un cómputo largo desde un punto intermedio, en vez de tener que rehacerlo desde cero ante una caída o corte. Es la técnica estándar para trabajos de larga duración (entrenamientos de ML, simulaciones, cómputo científico) donde el costo de perder todo el progreso por una falla es inaceptable. Acá el checkpoint no es responsabilidad del proceso mismo sino un servicio explícito del sistema operativo simulado: el Job pide `SAVE_CHECKPOINT`/`LOAD_CHECKPOINT` como syscall, el Planificador la delega al Storage, y el Storage debe garantizar que el propio guardado sea resistente a fallas (de ahí el journaling: si el sistema se cae *mientras* se escribe un checkpoint, no puede quedar corrupto ni a medio escribir).

## Los cuatro módulos

| Módulo | Rol | Binario |
|---|---|---|
| Planificador | Administra la cola de Jobs y su planificación (largo y corto plazo) | `./bin/planificador [Archivo Config] [Path Job Inicial]` |
| Core | Ejecuta las instrucciones de un Job (CPU simulada) | `./bin/core [Archivo Config] [Identificador]` |
| Placa | Administra la memoria (instrucciones + memoria de usuario paginada) | `./bin/placa [Archivo Config]` |
| Storage | Persiste los checkpoints en un filesystem propio (FAT32_TRAIN) | `./bin/storage [Archivo Config]` |

Cada módulo es un proceso real de Linux en C, compilado y ejecutado en la VM; pueden correr en máquinas/VMs distintas (sistema distribuido real).

**Orden de arranque (por dependencias):**
1. Placa y Storage (no dependen de nadie).
2. Planificador (se conecta a Placa y Storage al iniciar).
3. Core (se conecta a Planificador y Placa al iniciar).

Todos los módulos servidor (Planificador, Placa, Storage) deben ser **multihilo**, atendiendo conexiones concurrentes, y deben poder aceptar nuevas conexiones de Cores en cualquier momento (los Cores se pueden conectar/desconectar dinámicamente).

**Quién le habla a quién:**
- Core → Placa: fetch de instrucciones (por PC) y traducción de páginas (MMU / obtención de marco).
- Core → Planificador: reportar Page Faults y Syscalls.
- Planificador → Placa: carga de página, lectura/escritura de datos, deslockeo de páginas, creación/liberación de espacio y de Job.
- Planificador → Storage: operaciones de checkpoint (guardar / cargar / eliminar).
- El Core **nunca** habla directo con el Storage; las syscalls de checkpoint se delegan al Planificador.

## Módulo: Planificador

Administra la cola de Jobs del cluster. Al iniciar se conecta a Placa y Storage, y levanta un servidor multihilo para atender a los Cores.

### Modelo de 5 estados (corregido)

Estados: **NEW, READY, EXEC, BLOCK, EXIT**. Transiciones válidas:

- `NEW → READY`
- `READY → EXEC` | `READY → EXIT`
- `EXEC → READY` | `EXEC → BLOCK` | `EXEC → EXIT`
- `BLOCK → READY` | `BLOCK → EXIT`

No existe, por ejemplo, ir de BLOCK directo a EXEC. Diferencia clave entre BLOCK y READY: READY tiene todo lo que necesita y solo espera que le asignen un Core; BLOCK depende de que termine algo externo (I/O, carga de página, un Servicio) para poder avanzar.

> Nota: el enunciado no detalla explícitamente `READY → EXIT` y `BLOCK → EXIT`, pero el diagrama oficial del PDF (pág. 10) sí las incluye — quedan confirmadas como válidas, no es una suposición nuestra.

**Contexto de ejecución:** por cada Job se almacena una copia de todos los registros del Core, inicializados en 0 al crear el Job. Se envía al Core al ponerlo a ejecutar, y este lo devuelve actualizado al liberarlo (o al ser interrumpido/bloqueado).

### Planificación de Largo Plazo

Mueve Jobs de NEW a READY, siempre con **FIFO**, limitado por `GRADO_MULTIPROGRAMACION` (config). Como los Jobs inician sin memoria asignada, este pasaje no requiere verificar espacio disponible en la Placa. Cuando un Job finaliza (EXIT), se liberan sus estructuras en todo el sistema.

### Planificación de Corto Plazo

Decide qué Job en READY pasa a EXEC. Algoritmo configurable (no cambia durante una prueba): **FIFO, RR o HRRN**. Para HRRN hay un valor de estimación inicial por config (`ESTIMACION_INICIAL`, `HRRN_ALFA`).

### Ciclo de vida de un Job

- **Nace** con la syscall `INIT_JOB` (no bloqueante): el Core le pasa al Planificador el nombre de un archivo de pseudocódigo, y el Planificador crea el Job en NEW. Caso especial: el **Job 0** no nace por `INIT_JOB` — su archivo se pasa como segundo parámetro al arrancar el Planificador (es el JID 0, a partir del cual se crean los demás).
- **Muere** con la syscall `EXIT`: el Planificador lo pasa a EXIT según lo definido en planificación de largo plazo, liberando todas sus estructuras (memoria en la Placa, estructuras administrativas). Si el grado de multiprogramación lo permite, entra un nuevo Job de NEW a READY.

### Page Fault y Syscalls

- **Page Fault:** cuando un Core lo informa, se bloquea al Job y se le pide a la Placa la carga de la página. Una vez cargada, el Job pasa a READY.
- Al liberarse un Core por una syscall bloqueante, el Planificador debe invocar al planificador de corto plazo para asignar un nuevo Job de la cola de READY.
- Syscalls que **no bloquean** al Job (no liberan el Core): `INIT_JOB`, `ALLOC`, `FREE`.
- Syscalls que **sí bloquean** (liberan el Core, se debe reasignar): `LOAD_BATCH`, `LABEL`, `REPORT` (Servicios); `SAVE_CHECKPOINT`, `LOAD_CHECKPOINT`, `DELETE_CHECKPOINT` (Checkpoints).
- Las syscalls que operan sobre memoria del Job (`LOAD_BATCH`, `LABEL`, `REPORT`, `SAVE_CHECKPOINT`, `LOAD_CHECKPOINT`) llegan con direcciones físicas ya traducidas por el Core y páginas lockeadas en la Placa; al terminar, el Planificador debe pedir el deslockeo de esas páginas antes de pasar el Job a READY.
- Ante la desconexión de un Core, el Job que estuviera ejecutando en él vuelve a READY y se pide a la Placa el deslockeo de sus páginas.

### Servicios (implementados por el Planificador, FIFO, una petición a la vez, con cola propia)

- **Loader** (`LOAD_BATCH`): recibe nombre de archivo + rango; lee bytes de ese archivo (en `PATH_DATOS`) y los escribe en memoria del Job. Retardo configurable (`RETARDO_LOADER`).
- **Labeler** (`LABEL`): recibe tamaño + rango; pide al usuario esa cantidad de caracteres por teclado y los escribe en memoria del Job.
- **Logger** (`REPORT`): recibe tamaño + rango; lee esos bytes de memoria del Job y los escribe en ASCII Hexadecimal en el archivo de reportes (`PATH_REPORTES`, distinto del log del módulo).

### Estadísticas por Job (se imprimen en el Log al finalizar)

Tiempo total de ejecución, tiempo acumulado en READY (espera), tiempo acumulado en NEW (para ingresar), tamaño máximo de memoria dinámica ocupada, cantidad acumulada de syscalls solicitadas, cantidad acumulada de page faults, cantidad acumulada de transiciones READY→EXEC.

### Archivo de configuración del Planificador

`PUERTO_ESCUCHA, IP_PLACA, PUERTO_PLACA, IP_STORAGE, PUERTO_STORAGE, LOG_LEVEL, ALGORITMO_PLANIFICACION (FIFO|RR|HRRN), RR_QUANTUM, ESTIMACION_INICIAL, HRRN_ALFA, GRADO_MULTIPROGRAMACION, PATH_DATOS, RETARDO_LOADER, PATH_REPORTES`.

## Módulo: Core

Ejecuta el ciclo de instrucción de un Job: **Fetch → Decode → Execute → Check Interrupt**. Cada Core tiene un identificador propio (por parámetro) y su propio log/config.

### Registros

`PC` (uint32_t, Program Counter — al modificarse por una instrucción, no se le suma 1 al final del ciclo), `P0..P5` (direcciones lógicas), `AX, BX, CX, DX` (propósito general), `E1, E2, E3` (operandos de instrucciones de entrenamiento).

### Ciclo de instrucción

- **Fetch:** se pide la próxima instrucción a la Placa usando el PC (relativo al Job).
- **Decode:** se interpreta la instrucción y si requiere traducción lógica→física.
- **MMU:** paginación. `nro_pagina = floor(dir_logica / tamaño_pagina)`, `desplazamiento = dir_logica % tamaño_pagina`. Una petición puede abarcar más de una página — el Core debe partirla. Todas las traducciones deben resolverse antes de modificar memoria o registros. Si una página no está en la Placa, se devuelve el Job al Planificador con motivo Page Fault, **sin actualizar el PC**. Las páginas obtenidas quedan lockeadas hasta que el Core informe su liberación al terminar la instrucción o al devolver el Job (excepto en syscalls sobre memoria del Job, donde el Planificador libera el lock).
- **Execute:** ver instrucciones abajo.
- **Check Interrupt:** si el Planificador envió una interrupción para el Job en ejecución, se devuelve el JID + contexto con motivo de interrupción; si no, sigue el ciclo siguiente.

### Instrucciones

**Básicas:** `NOOP`, `SET (Registro, Valor)`, `SUM (Registro, Valor)`, `SUB (Registro, Valor)`, `JNZ (Registro, Instrucción)`.

**De entrenamiento** (E1/E2/E3 se cargan con el contenido de las direcciones que la instrucción vaya a leer; al terminar, los registros de entrenamiento modificados se escriben en memoria del Job):
- `FORWARD (Pesos, Entrada, Salida, Acumulador)`: `acumulador += E1*E2; E3 = acumulador`
- `BACKWARD (Pesos, Activaciones, Gradientes, Acumulador)`: `acumulador -= E1*E2; E3 = acumulador`
- `UPDATE (Gradientes, Estado, Pesos)`: `E2 = (E2+E1)/2; E3 = E3 - E2`

**Syscalls** (dependen del Planificador; se le envían JID + parámetros):
`INIT_JOB(archivo)`, `ALLOC(RegDirLogica, Tamaño)`, `FREE(RegDirLogica)`, `LOAD_BATCH(archivo, RegDirLogica, RegTamaño)`, `LABEL(RegDirLogica, RegTamaño)`, `REPORT(RegDirLogica, RegTamaño)`, `SAVE_CHECKPOINT(nombre, RegDirLogica, RegTamaño)`, `LOAD_CHECKPOINT(nombre, RegDirLogica, RegTamaño)`, `DELETE_CHECKPOINT(nombre)`, `EXIT`. `INIT_JOB`, `ALLOC` y `FREE` **no bloquean** al Job (el Core espera la respuesta y sigue el ciclo); en `ALLOC` el Core debe guardar en el registro de dirección lógica la dirección devuelta.

### Archivo de configuración del Core

`LOG_LEVEL, IP_PLANIFICADOR, PUERTO_PLANIFICADOR, IP_PLACA, PUERTO_PLACA`.

## Módulo: Placa

Administra toda la memoria del sistema: instrucciones de los Jobs (memoria ilimitada, un archivo de pseudocódigo por Job) + memoria de usuario (paginada, tamaño y tamaño de página fijados por config).

### Memoria de usuario

- Esquema de **paginación con memoria virtual**, asignación dinámica, alcance global. **Paginación bajo demanda**: los Jobs empiezan sin marcos asignados; una página se asigna recién cuando el Job la accede.
- Estructuras mínimas: un espacio contiguo de memoria (`void*`, la memoria de datos) + una tabla de páginas por Job (no debe vivir dentro de la memoria de datos).
- **Offload:** archivo administrado por la Placa (path y tamaño por config) donde se guardan las páginas de Jobs que no están en memoria. Simula acceso a disco: un único hilo de ejecución, sin lecturas/escrituras paralelas.
- **Reemplazo de páginas:** cuando hay que cargar una página y no hay marcos libres, se elige una página víctima entre todas las páginas residentes no lockeadas (de cualquier Job). Algoritmo configurable: **LRU o Clock Modificado (CLOCK-M)**.

### Operaciones que atiende

Creación de Job (JID + archivo de pseudocódigo), Asignación de espacio (JID + tamaño → devuelve dirección lógica; reserva lugar en Offload; si no hay espacio, informa al Planificador para finalizar el Job), Liberación de espacio, Obtención de marco (para la MMU del Core — devuelve marco o Page Fault), Carga de página (pedida por el Planificador tras un Page Fault), Lectura de datos, Escritura de datos, Deslockeo de páginas, Finalización de Job (libera marcos, espacio en Offload y estructuras).

### Comandos locales (consola por stdin)

`INFO` (porcentaje memoria/offload ocupada, frames libres/lockeados/totales, cantidad de Jobs), `TLS` (lista Jobs: ID, páginas del conjunto residente, páginas totales). Se pueden agregar otros mientras no contradigan lo especificado.

### Archivo de configuración de la Placa

`PUERTO_ESCUCHA, LOG_LEVEL, TAM_MEMORIA (potencia de 2), TAM_PAGINA (potencia de 2), RETARDO_MEMORIA, ALGORITMO_REEMPLAZO (LRU|CLOCK-M), PATH_INSTRUCCIONES, PATH_OFFLOAD, TAM_OFFLOAD, RETARDO_OFFLOAD`.

## Módulo: Storage

Persiste los Checkpoints (sobreviven a la finalización del Job que los generó — un Job puede cargar un Checkpoint creado por otro). Servidor multihilo, atiende al Planificador de forma concurrente.

### Filesystem propio: FAT32_TRAIN

Se persiste en un archivo real del SO anfitrión, dividido en bloques (cantidad y tamaño por config, potencia de 2 desde 32). Regiones: **Superbloque** (bloque 0), **File Allocation Table**, **Directorio raíz** (bloques por config), **bloques de datos**. Si el archivo no existe al iniciar, se crea y formatea.

- **Superbloque:** firma "FAT32_TRAIN" (11 bytes), tamaño de bloque (4B), cantidad de bloques del filesystem (4B), cantidad de bloques del directorio (4B); resto del bloque en padding.
- **FAT:** una entrada uint32_t por bloque. 28 bits menos significativos = número de bloque siguiente; 4 más significativos reservados. `0` = bloque libre, `0x0FFFFFFF` = fin de checkpoint, `0x0FFFFFFE` = bloque reservado (nunca libre).
- **Directorio:** una entrada por checkpoint existente. Por entrada: estado (1B, 0=libre/1=ocupada), nombre (23B, terminado en '\0'), tamaño en bytes (4B), primer bloque de datos (4B). Directorio lleno → error.

### Operaciones

- **Guardar checkpoint:** si no existe, crea entrada; si existe, reemplaza contenido. Asigna tantos bloques como haga falta y libera los que sobren. Sin espacio suficiente → informa al Planificador y se finaliza el Job.
- **Cargar checkpoint:** devuelve el contenido completo; si no existe, informa al Planificador y se finaliza el Job.
- **Eliminar checkpoint:** libera su entrada de directorio y todos sus bloques en la FAT; si no existe, informa al Planificador pero **no** se finaliza el Job.

### Journaling

Garantiza consistencia ante fallas abruptas. Guardar y Eliminar checkpoint deben asentarse en el journal (archivo real, secuencial, aparte del volumen) **antes** de aplicarse. Cada operación = un "asiento" con ID único, las modificaciones (bloques de datos, entradas FAT y directorio) y marcas de completitud.

- **Paso 1:** se escribe el asiento completo en el journal (fwrite + fflush + fsync obligatorios), luego marca "Commit".
- **Paso 2:** se aplican las modificaciones al filesystem real, luego marca "Applied" (fflush/fsync acá son opcionales pero configurables).
- **Recuperación al iniciar:** recorre el journal cronológicamente — asiento sin "Commit" se descarta; con "Commit" y "Applied" se descarta; con "Commit" pero sin "Applied" se **re-aplica** completo. Las operaciones deben ser idempotentes. Se evalúa matando el proceso (`kill -9`) en medio de una operación y reiniciándolo.

### Comandos locales (consola por stdin)

`INFO` (superbloque), `LS` (entradas de directorio), `CAT <nombre> <cantBytes>` (contenido en hex), `WRITE <nombre> <hexContent>` (crea/edita, hasta 20 bytes hex), `RM <nombre>` (borra).

### Archivo de configuración del Storage

`PUERTO_ESCUCHA, LOG_LEVEL, PATH_STORAGE, CANT_BLOQUES, TAM_BLOQUE, BLOQUES_DIRECTORIO, RETARDO_ACCESO_BLOQUE, RETARDO_COMMIT`.

## Logs

Todos los módulos usan la biblioteca `so-commons-library` de la cátedra, con `LOG_LEVEL_INFO` como mínimo obligatorio (se puede extender con `LOG_LEVEL_DEBUG`). **No cumplir con los logs mínimos y obligatorios, o no guardarlos en archivo, hace que el TP se considere no evaluable y quede desaprobado** — cada módulo tiene su propio listado detallado en el enunciado (ver PDF, secciones "Logs mínimos y obligatorios" de cada módulo).

## Checks de control y entregas

| Check | Fecha | Foco |
|---|---|---|
| Check 1 | 12/09 | Conexiones y serialización: arquitectura de red, servidores multihilo, Handshake Core↔Planificador/Placa |
| Check 2 | 10/10 | Planificación y ejecución básica: ciclo de instrucción sin memoria/MMU, cola de Jobs, cambio de contexto, FIFO/RR |
| Check 3 | 31/10 | Memoria, paginación y servicios, Storage: MMU + Page Faults, tablas de páginas, paginación bajo demanda, reemplazo, Offload, servicios I/O bloqueantes, formateo del volumen |
| Check 4 | 14/11 | Storage, filesystem y persistencia: todas las operaciones de Storage + Journaling (incluye kill -9 y recuperación) |
| Entregas finales | 28/11, 12/12, 19/12 | Todos los módulos integrados, probados de forma intensiva en entorno distribuido |

## Estado de avance

El estado canónico (temas cerrados, en curso, pendientes y próximo paso) está en `.claude/memoria/01-estado-actual.md`, que se carga automáticamente al inicio. Se actualiza al final de cada sesión que avance algo.

## Documentos de referencia

- **Enunciado completo** (`TP 2C2026 - EntrenadOS.pdf`, en la raíz del repo, v1.0): fuente de verdad. Su transcripción fiel por módulo está en `.claude/memoria/1x-*.md`; consultarla antes de implementar cualquier detalle fino. Para leer el PDF en esta máquina: `pdftotext -layout "TP 2C2026 - EntrenadOS.pdf" -` (requiere `poppler-utils`).
- **readme.md**: portada del repo — orientación de dónde está cada cosa y en qué orden conviene leerla.
- **Notas de estudio.md**: notas de estudio conceptuales, se actualiza a medida que se cierran temas nuevos con el asistente.
- **.claude/memoria/**: memoria persistente del asistente (índice, estado, forma de trabajo, decisiones de diseño y specs completas por módulo).
