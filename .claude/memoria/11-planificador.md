# Módulo: Planificador — especificación completa

Fuente: enunciado v1.0 (25/08/2026), págs. 9–14. Transcripción fiel; lo marcado como *[derivado]* es una deducción nuestra, no texto del enunciado.

## Rol y arranque

Administra la cola de Jobs del cluster y gestiona su planificación a lo largo de las pruebas.

```
./bin/planificador [Archivo Config] [Path Job Inicial]
```

- 1er parámetro: archivo de configuración.
- 2do parámetro: path al **Job Inicial**, que será el **JID 0** del sistema y a partir del cual se crean los demás Jobs. (JID = "Job ID", valor numérico que identifica unívocamente a cada Job.)
- Al iniciar: se conecta con la **Placa** y con el **Storage**. Una vez establecidas esas conexiones, crea un **servidor multihilo** para atender concurrentemente las peticiones de los Cores.
- Los Cores pueden conectarse y desconectarse durante toda la ejecución → la escucha de nuevas conexiones debe estar **siempre activa**.

## Modelo de 5 estados

Estados: **NEW, READY, EXEC, BLOCK, EXIT**. La implementación de la planificación es decisión de diseño del grupo, pero debe gestionar este modelo.

Transiciones que muestra el diagrama oficial (pág. 10), confirmadas visualmente:

| Desde | Hacia |
|---|---|
| NEW | READY |
| READY | EXEC, EXIT |
| EXEC | READY, BLOCK, EXIT |
| BLOCK | READY, EXIT |

No existe BLOCK → EXEC ni ninguna otra transición fuera de esa tabla.

### Contexto de ejecución

Por cada Job se almacena un contexto de ejecución que consiste, **como mínimo**, en una copia de todos los registros del Core, **inicializados en 0** al crear el Job. Se le envía al Core al ponerlo a ejecutar, y el Core lo devuelve actualizado al liberarlo.

### Planificación de largo plazo

- Ingresa Jobs de **NEW a READY**.
- Si el **grado máximo de multiprogramación** (config `GRADO_MULTIPROGRAMACION`) lo permite, los Jobs pasan a READY mediante **FIFO**.
- Como los Jobs inician **sin memoria asignada**, este pasaje **no requiere verificar** espacio disponible en la Placa.
- Cuando un Job finaliza, pasa a **EXIT**, liberando las estructuras asociadas al mismo **en todo el sistema**.

### Planificación de corto plazo

- Algoritmos posibles: **FIFO, RR y HRRN**.
- Elegible por config (`ALGORITMO_PLANIFICACION`); **no cambia** a lo largo de una prueba.
- HRRN: tiene un valor estimado inicial por config (`ESTIMACION_INICIAL`) y un alfa (`HRRN_ALFA`) para el cálculo de la estimación de las ráfagas. **Confirmado por la teoría de la cátedra** (`20-teoria-planificacion.md`): estimación exponencial `Est(n+1) = α·R(n) + (1-α)·Est(n)`, con `ESTIMACION_INICIAL = Est(0)`; prioridad HRRN (Response Ratio) `RR = (S + W) / S = 1 + W/S`, con S = ráfaga estimada y W = tiempo de espera en READY. El enunciado no trae las fórmulas explícitamente, pero coinciden con las diapositivas de la cátedra, así que no son una invención nuestra.
- RR: quantum en milisegundos por config (`RR_QUANTUM`). Al vencer, el Job es desalojado (log "Desalojado por fin de quantum") y vuelve a READY (transición EXEC → READY). El desalojo se materializa enviando una interrupción al Core, que la detecta en Check Interrupt y devuelve JID + contexto con el motivo (mecanismo confirmado por la teoría de RR en `20-teoria-planificacion.md`: siempre con desalojo, vía interrupción de timer).

### Page Fault

Cuando un Core informa un Page Fault: se **bloquea** al Job (EXEC → BLOCK) y se le solicita a la Placa la **carga de la página** indicada. Una vez cargada, el Job pasa a **READY**.

### Atención de syscalls

Durante su ejecución en un Core, un Job puede pedirle al Planificador que ejecute syscalls. Según cuál sea, la syscall **bloquea o no** al Job que la solicitó.

- **Syscall bloqueante:** el Core devuelve al Planificador el **JID y el contexto de ejecución actualizado**. El Planificador guarda ese contexto en las estructuras administrativas del Job y, como se liberó un Core, **invoca al planificador de corto plazo** para asignarle un nuevo Job de la cola de READY.
- **Syscall no bloqueante:** **no libera el Core**. Una vez finalizada, se le responde al Core que hizo la llamada para que continúe su ejecución.
- **Syscalls que operan sobre memoria del Job** (`LOAD_BATCH`, `LABEL`, `REPORT`, `SAVE_CHECKPOINT`, `LOAD_CHECKPOINT`): llegan con las **direcciones físicas ya traducidas por el Core** — un par `[DireccionFisica; Tamaño]` **por cada página involucrada** — y con sus páginas **lockeadas** en la Placa (*page locking*: marcar una página/marco temporalmente para que no pueda ser removida de memoria por un reemplazo). Al terminar la operación, el Planificador debe **solicitar a la Placa el deslockeo** de las páginas del Job **antes** de pasarlo a READY.
- **Desconexión de un Core:** el Job que estuviera ejecutando en él **vuelve a READY** y se solicita a la Placa el deslockeo de sus páginas.

#### Syscalls de Jobs
- `INIT_JOB`: recibe el nombre de un archivo de pseudocódigo y crea un nuevo Job en **NEW**, **sin bloquear** al Job que la solicitó.
- `EXIT`: finaliza el Job que la solicitó, según lo definido en planificación de largo plazo.

#### Syscalls de memoria
- `ALLOC` y `FREE`: se **derivan a la Placa** y **no bloquean** al Job. En `ALLOC` se responde al Core la **dirección lógica** informada por la Placa.

#### Syscalls de servicios
- `LOAD_BATCH`, `LABEL`, `REPORT`: se atienden según la sección Servicios, **bloqueando** al Job hasta finalizar la operación.

#### Syscalls de checkpoints (todas bloquean al Job hasta finalizar)
- `SAVE_CHECKPOINT`: el Planificador recibe del Core el **nombre** del checkpoint, un **tamaño** y las **direcciones físicas** del rango. Lee de la memoria del Job esa cantidad de bytes y los envía al Storage para que los persista con ese nombre.
- `LOAD_CHECKPOINT`: recibe nombre, tamaño y direcciones físicas. Solicita el contenido al Storage y escribe esa cantidad de bytes en la memoria del Job.
- `DELETE_CHECKPOINT`: recibe el nombre y solicita su eliminación al Storage.

Errores que informa el Storage (ver `14-storage.md`): sin espacio al guardar → se finaliza el Job; checkpoint inexistente al cargar → se finaliza el Job; checkpoint inexistente al eliminar → **no** se finaliza el Job. Error de la Placa: `ALLOC` sin espacio suficiente en Offload → la Placa informa al Planificador para que finalice el Job.

## Servicios (propios del Planificador)

Tres servicios: **Loader, Labeler y Logger**. Cada uno atiende **una única petición por vez**, bajo **FIFO**, con **una cola de espera por servicio**. En todos los casos se bloquea al Job solicitante.

- **Loader** (`LOAD_BATCH`): recibe del Core el **nombre de un archivo**, un **tamaño** y las **direcciones físicas** del rango. Lee esa cantidad de bytes del archivo, ubicado en el directorio `PATH_DATOS`, y los escribe en la memoria del Job. Tiene un **retardo** por config (`RETARDO_LOADER`). El archivo está en el filesystem de la máquina donde corre el Planificador (footnote 4).
- **Labeler** (`LABEL`): recibe un **tamaño** y las direcciones físicas. Le pide al **usuario por teclado** esa cantidad de caracteres y, una vez recibidos, los escribe en la memoria del Job.
- **Logger** (`REPORT`): recibe un **tamaño** y las direcciones físicas. Lee esa cantidad de bytes de la memoria del Job y los escribe en formato **ASCII Hexadecimal** en el archivo de reportes (`PATH_REPORTES`), que debe ser **distinto** del archivo de log del módulo.

## Estadísticas (por Job, impresas en el log al finalizar)

1. Tiempo total de ejecución (desde el inicio hasta su finalización).
2. Tiempo acumulado de espera (considerando **solo** el estado READY).
3. Tiempo acumulado para ingresar (considerando **solo** el estado NEW).
4. Tamaño máximo de memoria dinámica ocupada en un momento dado.
5. Cantidad acumulada de syscalls solicitadas.
6. Cantidad acumulada de page faults generados.
7. Cantidad acumulada de transiciones READY ⇒ EXEC.

## Logs mínimos y obligatorios (literales, nivel INFO, sin los `< >`)

| Evento | Formato |
|---|---|
| Conexión recibida | `## Módulo: <NOMBRE_MODULO_CONECTADO>` |
| Creación de Job | `## (<JID>) Se crea el Job - Estado: NEW` |
| Estimación | `## (<JID>) - Prioridad HRRN calculada: <PRIORIDAD> - Estimación próxima ráfaga: <ESTIMACION>` |
| Syscall recibida | `## (<JID>) - Solicitó syscall: <NOMBRE_SYSCALL>` |
| Cambio de estado | `## (<JID>) Pasa del estado <ESTADO_ANTERIOR> al estado <ESTADO_ACTUAL>` |
| Inicio de servicio | `## (<JID>) inicia <NOMBRE_SERVICIO>` |
| Fin de servicio | `## (<JID>) finalizó <NOMBRE_SERVICIO> y pasa a READY` |
| Acceso a memoria | `## (<JID>) - <Lectura/Escritura> - Dirección Física: <DIRECCION_FISICA> - Tamaño: <TAMAÑO>` |
| Deslockeo de páginas | `## (<JID>) - Deslockeo de páginas` |
| Solicitud del Labeler | `## (<JID>) - Ingrese <CANTIDAD> caracteres:` |
| Reporte del Logger | `## (<JID>) - <CONTENIDO>` — **en el archivo de reportes**, no en el log |
| Desalojo por fin de quantum | `## (<JID>) - Desalojado por fin de quantum` |
| Fin de Job | `## (<JID>) finalizó su ejecución con motivo de <MOTIVO>. Estadísticas: <...>` |

## Archivo de configuración

| Campo | Tipo | Descripción |
|---|---|---|
| `PUERTO_ESCUCHA` | Número | Puerto del servidor |
| `IP_PLACA` | String | IP de la Placa |
| `PUERTO_PLACA` | Número | Puerto de la Placa |
| `IP_STORAGE` | String | IP del Storage |
| `PUERTO_STORAGE` | Número | Puerto del Storage |
| `LOG_LEVEL` | String | Nivel máximo de detalle; compatible con `log_level_from_string()` |
| `ALGORITMO_PLANIFICACION` | String | Corto plazo: `FIFO`, `RR` o `HRRN` |
| `RR_QUANTUM` | Número | Milisegundos a esperar antes de finalizar el quantum de un Job |
| `ESTIMACION_INICIAL` | Número | Milisegundos de estimación inicial para la primera ráfaga en HRRN |
| `HRRN_ALFA` | Número | Alfa para la estimación de ráfagas en HRRN |
| `GRADO_MULTIPROGRAMACION` | Número | Cantidad máxima de Jobs que pueden estar residentes en la Placa simultáneamente |
| `PATH_DATOS` | String | Directorio con los archivos que lee el Loader |
| `RETARDO_LOADER` | Número | Milisegundos que demora el Loader en atender una petición |
| `PATH_REPORTES` | String | Archivo donde el Logger escribe los reportes |

Ejemplo oficial:

```
PUERTO_ESCUCHA=8080
IP_PLACA=127.0.0.1
PUERTO_PLACA=8081
IP_STORAGE=127.0.0.1
PUERTO_STORAGE=8082
LOG_LEVEL=INFO
ALGORITMO_PLANIFICACION=HRRN
RR_QUANTUM=1500
ESTIMACION_INICIAL=10000
HRRN_ALFA=0.5
GRADO_MULTIPROGRAMACION=4
PATH_DATOS=/home/utnso/datos/
RETARDO_LOADER=500
PATH_REPORTES=/home/utnso/reportes.log
```
