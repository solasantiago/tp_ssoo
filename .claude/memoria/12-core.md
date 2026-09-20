# Módulo: Core — especificación completa

Fuente: enunciado v1.0 (25/08/2026), págs. 15–20. Transcripción fiel; lo marcado como *[derivado]* es una deducción nuestra, no texto del enunciado.

## Rol y arranque

Interpreta y ejecuta las instrucciones de los Jobs mediante un ciclo de instrucción simplificado: **Fetch → Decode → Execute → Check Interrupt**.

```
./bin/core [Archivo Config] [Identificador]
```

- Cada Core tiene un **identificador propio**, recibido por parámetro.
- Cada Core debe tener **archivo de log y archivo de configuración independientes**, y ambos deben incluir el identificador en su nombre para saber a qué Core corresponden.
- Se conecta al **Planificador** y a la **Placa**. Una vez establecidas ambas conexiones, queda **a la espera de recibir del Planificador un JID junto con su contexto de ejecución** para comenzar el primer ciclo de instrucción.
- Para instrucciones que interactúan con memoria, traduce direcciones lógicas a físicas mediante la **MMU**.

## Registros del Core

Todos de 4 bytes, tipo `uint32_t`:

| Registro | Descripción |
|---|---|
| `PC` | Program Counter: próxima instrucción a ejecutar. **Si una operación lo modifica, se omite el paso de sumarle 1 al final del ciclo.** |
| `P0`…`P5` | Contienen una dirección lógica de memoria |
| `AX`, `BX`, `CX`, `DX` | Registros numéricos de propósito general |
| `E1`, `E2`, `E3` | Operandos de las instrucciones de entrenamiento |

Total: 14 registros. *[derivado]* El contexto de ejecución que guarda el Planificador es una copia de estos 14 registros (56 bytes), inicializados en 0.

## Ciclo de instrucción

### Fetch
Se pide la próxima instrucción a la **Placa** usando el **PC**, que representa el **número de instrucción relativo al Job en ejecución**.

### Decode
Interpretar qué instrucción se va a ejecutar y si requiere traducción de dirección lógica a física.

### MMU (traducción lógica → física)

Direcciones lógicas y físicas **siempre en decimal**. Esquema de **paginación**; la dirección lógica se interpreta como `[N° Página | Desplazamiento]`:

```
nro_pagina     = floor(dir_logica / tamaño_pagina)
desplazamiento = dir_logica % tamaño_pagina
```

- Una petición de lectura/escritura **puede abarcar más de una página**; es responsabilidad del grupo **dividir** la petición según las páginas involucradas.
- **Todas** las traducciones que requiera una instrucción deben resolverse **antes** de modificar la memoria o los registros del Job.
- Si la página no está en la memoria de la Placa: se devuelve el Job al Planificador con motivo **Page Fault**, **sin actualizar el PC** (así la instrucción se reintenta al volver a EXEC).
- Las páginas obtenidas quedan **lockeadas** en la Placa. El Core debe **informar su liberación** al finalizar la instrucción o al devolver el Job al Planificador. **Excepción:** syscalls que operan sobre memoria del Job, cuyas páginas quedan lockeadas hasta que el **Planificador** informe su liberación.
- *[derivado]* El tamaño de página no está en la config del Core; hay que obtenerlo de la Placa (p. ej. en el handshake).

### Listado de instrucciones a interpretar (ejemplo del enunciado)

```
NOOP
EXIT
SET AX 6
SET BX 256
SET P0 1024
SUM P0 BX
SUB AX 1
JNZ AX 5
FORWARD P0 P1 P2 BX
BACKWARD P0 P1 P2 BX
UPDATE P2 P3 P0
ALLOC P0 3072
FREE P0
LOAD_BATCH lote1 P1 CX
LABEL P1 CX
REPORT P1 CX
SAVE_CHECKPOINT modelo7 P0 BX
LOAD_CHECKPOINT modelo7 P0 BX
DELETE_CHECKPOINT modelo7
INIT_JOB finetune
```

El listado es solo ilustrativo (no sigue lógica alguna). En las pruebas, **ninguna instrucción tendrá errores sintácticos ni semánticos**.

### Execute

#### Instrucciones básicas
- `NOOP`: No Operation; solo consume el tiempo del ciclo de instrucción.
- `SET (Registro, Valor)`: asigna al registro el valor, que puede ser **un número o el contenido de otro registro**.
- `SUM (Registro, Valor)`: suma al registro el valor pasado.
- `SUB (Registro, Valor)`: resta al registro el valor pasado.
- `JNZ (Registro, Instrucción)`: si el valor del registro es **distinto de cero**, actualiza el PC al número de instrucción pasado por parámetro.

#### Instrucciones de entrenamiento
Se representan con aritmética simple sobre enteros de 4 bytes sin signo (`uint32_t`). Los registros **E1, E2 y E3 se corresponden con los tres primeros parámetros** de la instrucción (registros de dirección lógica), y el Core carga en ellos el **contenido de las direcciones apuntadas** antes de ejecutar. **Solo debe cargar los registros que la instrucción vaya a consumir (leer).**

- `FORWARD (Registro Pesos, Registro Entrada, Registro Salida, Registro Acumulador)`
  ```
  acumulador = acumulador + E1 * E2
  E3 = acumulador
  ```
- `BACKWARD (Registro Pesos, Registro Activaciones, Registro Gradientes, Registro Acumulador)`
  ```
  acumulador = acumulador - E1 * E2
  E3 = acumulador
  ```
- `UPDATE (Registro Gradientes, Registro Estado, Registro Pesos)`
  ```
  E2 = (E2 + E1) / 2
  E3 = E3 - E2
  ```

Finalizada la ejecución, los registros de entrenamiento que hayan sido **modificados (escritos)** en esa instrucción deben **escribirse en la memoria del Job** (en la dirección del parámetro correspondiente).

*[derivado]* Lecturas/escrituras por instrucción: FORWARD y BACKWARD leen E1 y E2 (no leen E3) y escriben E3; el "Acumulador" es un registro de propósito general (p. ej. BX) que se modifica en el Core. UPDATE lee E1, E2 y E3, y escribe E2 y E3.

#### Instrucciones de tipo syscall
No pueden resolverse en el Core; dependen del Planificador. Cada syscall tiene un nombre distinto para simplificar los scripts. Su semántica se detalla en `11-planificador.md`.

- Se solicitan al Planificador enviándole el **JID y los parámetros**.
- Las que **bloquean** al Job deben además **liberar el Core** y **devolver el contexto de ejecución**.
- Cuando la syscall **opera sobre memoria del Job**, el Core debe **traducir el rango** antes de devolver el Job y enviar al Planificador las **direcciones físicas resultantes** (un par `[DireccionFisica; Tamaño]` por página, según el Planificador).
- `INIT_JOB`, `ALLOC` y `FREE` **no bloquean**: el Core **espera la respuesta** del Planificador y continúa con el siguiente ciclo. En `ALLOC` debe guardar en el Registro Dirección Lógica la dirección recibida.

Firmas:
- `INIT_JOB (Archivo de instrucciones)`
- `ALLOC (Registro Dirección Lógica, Tamaño)`
- `FREE (Registro Dirección Lógica)`
- `LOAD_BATCH (Nombre de archivo, Registro Dirección Lógica, Registro Tamaño)`
- `LABEL (Registro Dirección Lógica, Registro Tamaño)`
- `REPORT (Registro Dirección Lógica, Registro Tamaño)`
- `SAVE_CHECKPOINT (Nombre, Registro Dirección Lógica, Registro Tamaño)`
- `LOAD_CHECKPOINT (Nombre, Registro Dirección Lógica, Registro Tamaño)`
- `DELETE_CHECKPOINT (Nombre)`
- `EXIT`

**Al finalizar el ciclo, el PC se incrementa en 1, siempre y cuando no haya sido modificado por la instrucción ejecutada.**

### Check Interrupt
Chequear si el Planificador envió una **interrupción** al Job en ejecución. Si sí: se devuelve al Planificador el **JID y el contexto** con el **motivo de la interrupción**. Si no: continúa con el siguiente ciclo.

### Ejemplo de Job
Job de ejemplo que realiza un paso de entrenamiento: `https://github.com/sisoputnfrba/entrenados-pruebas/blob/main/ejemplo_job/job_entrenamiento.asm`

## Logs mínimos y obligatorios (literales, nivel INFO, sin los `< >`)

| Evento | Formato |
|---|---|
| Conexión establecida | `## Conectado a <Planificador/Placa> exitosamente` |
| Fetch instrucción | `## JID: <JID> - FETCH - Program Counter: <PROGRAM_COUNTER>` |
| Interrupción recibida | `## Interrupción recibida` |
| Instrucción ejecutada | `## JID: <JID> - Ejecutando: <INSTRUCCION> - <PARAMETROS>` |
| Page Fault | `## JID: <JID> - Page Fault - Página: <NRO_PAGINA>` |
| Lectura/Escritura memoria | `## JID: <JID> - Acción: <Lectura/Escritura> - Dirección Física: <DIRECCION_FISICA> - Tamaño: <TAMAÑO>` |

## Archivo de configuración

| Campo | Tipo | Descripción |
|---|---|---|
| `LOG_LEVEL` | String | Nivel máximo de detalle; compatible con `log_level_from_string()` |
| `IP_PLANIFICADOR` | String | IP del Planificador |
| `PUERTO_PLANIFICADOR` | Número | Puerto del Planificador |
| `IP_PLACA` | String | IP de la Placa |
| `PUERTO_PLACA` | Número | Puerto de la Placa |

Ejemplo oficial:

```
LOG_LEVEL=INFO
IP_PLANIFICADOR=127.0.0.1
PUERTO_PLANIFICADOR=8080
IP_PLACA=127.0.0.1
PUERTO_PLACA=8081
```
