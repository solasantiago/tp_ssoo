# Módulo: Storage — especificación completa

Fuente: enunciado v1.0 (25/08/2026), págs. 26–31. Transcripción fiel; lo marcado como *[derivado]* es una deducción nuestra, no texto del enunciado.

## Rol y arranque

Persiste los **Checkpoints** de los Jobs. Los checkpoints **sobreviven a la finalización del Job** que los generó → un Job puede cargar un checkpoint creado por otro.

```
./bin/storage [Archivo Config]
```

- Al iniciar crea un **servidor multihilo** capaz de atender peticiones del **Planificador** de manera concurrente.
- Toda la información que administra se persiste dentro del directorio `PATH_STORAGE`, de modo que **sobreviva entre ejecuciones** del módulo.

## Filesystem: FAT32_TRAIN

Variante de FAT32. Junto al volumen vive el **journal** (ver Journaling).

### Estructura interna
- El volumen se persiste en **un archivo real** del SO anfitrión (Linux), dividido en **bloques de igual tamaño numerados desde 0**. Cantidad (`CANT_BLOQUES`) y tamaño (`TAM_BLOQUE`) por config.
- Cuatro regiones, en este orden:
  1. **Superbloque**: bloque 0.
  2. **Tabla FAT**: su tamaño depende de la cantidad de bloques y del tamaño del puntero.
  3. **Directorio raíz** (único): ocupa `BLOQUES_DIRECTORIO` bloques.
  4. **Bloques de datos**: todos los restantes.
- Si al iniciar el archivo del filesystem **no existe**, debe **crearse y formatearse**: escribir el superbloque, la FAT con **todos sus bloques de datos libres** y el directorio con **todas sus entradas libres**.

### Superbloque (bloque 0)
Estructura de datos **contigua y exacta**:

| Campo | Tamaño | Valor |
|---|---|---|
| Firma (Magic Number) | 11 bytes | cadena estática `"FAT32_TRAIN"` |
| Tamaño de bloque | 4 bytes (`uint32_t`) | |
| Cantidad de bloques del filesystem | 4 bytes (`uint32_t`) | |
| Cantidad de bloques del directorio | 4 bytes (`uint32_t`) | |

El resto del bloque 0 (hasta completar `TAM_BLOQUE`) se llena con **padding (ceros)**, reservado para uso futuro. *[derivado]* Contenido útil: 23 bytes.

### File Allocation Table (FAT)
- **Una entrada por cada bloque del filesystem**, de tipo `uint32_t` → tamaño total = `CANT_BLOQUES * sizeof(uint32_t)`.
- Cada entrada indica el **bloque siguiente** del checkpoint al que pertenece. De los 32 bits, **solo los 28 menos significativos** forman el número de bloque; los **4 más significativos están reservados**.
- Valores especiales:
  - `0` → bloque **libre**.
  - `0x0FFFFFFF` → **fin de checkpoint** (último bloque de la cadena).
  - `0x0FFFFFFE` → **bloque reservado** (superbloque, FAT, directorio): nunca se considera libre.
- La tabla ocupa **tantos bloques como haga falta** para sus entradas, completando con **ceros** el resto del último bloque si no lo llena.
- *[derivado]* Bloques de la FAT = `ceil(CANT_BLOQUES * 4 / TAM_BLOQUE)`. Primer bloque de datos = `1 + bloques_FAT + BLOQUES_DIRECTORIO`.

### Directorio
- Una entrada por cada checkpoint (archivo) existente, ocupando la región **de manera contigua desde su primer byte**.
- Estructura **exacta** de cada entrada:

| Campo | Tamaño | Valor |
|---|---|---|
| Estado de la entrada | 1 byte (`uint8_t`) | `0` libre, `1` ocupada |
| Nombre del checkpoint | 23 bytes | cadena terminada en `'\0'` |
| Tamaño del checkpoint en bytes | 4 bytes (`uint32_t`) | |
| Primer bloque de datos | 4 bytes (`uint32_t`) | |

- *[derivado]* Cada entrada ocupa **32 bytes**; entradas por bloque = `TAM_BLOQUE / 32`; capacidad total = `BLOQUES_DIRECTORIO * TAM_BLOQUE / 32`. Nombre útil máximo: 22 caracteres + `'\0'`.
- Directorio **lleno** → debe informarse con un **error**.

## Operaciones (solicitadas por el Planificador)

### Guardar un checkpoint
- Recibe **nombre** y **contenido** a persistir.
- Si **no existe**: se crea su entrada en el directorio. Si **ya existe**: su contenido anterior se **reemplaza**.
- Se asignan **tantos bloques como haga falta** para el tamaño recibido y se **liberan los que dejen de ser necesarios**.
- Sin espacio suficiente → se informa al Planificador y **se finaliza el Job**.

### Cargar un checkpoint
- Recibe **nombre**; responde su **contenido completo**.
- Si no existe → se informa al Planificador y **se finaliza el Job**.

### Eliminar un checkpoint
- Recibe **nombre**; libera el espacio: la entrada de directorio se marca **libre** y **todos los bloques de su cadena** se marcan libres en la FAT.
- Si no existe → se informa al Planificador, pero **NO se finaliza el Job**.

## Journaling

Objetivo: preservar coherencia ante fallas. **Guardar** y **Eliminar** deben quedar asentadas en el journal **antes** de aplicarse, para poder recuperar un estado consistente tras una finalización abrupta.

### El journal
- **Archivo real, distinto** del archivo del volumen, de **escritura secuencial**, donde se describen **anticipadamente** todas las operaciones a realizar en el filesystem.
- Debe respetar el formato de **archivo de texto**. La estructura definitiva queda a criterio del grupo.
- Por cada operación se registran todas las modificaciones que la integran, agrupadas en un **asiento**.
- Un asiento contiene:
  - un **identificador único**;
  - las **modificaciones concretas** (bloques de datos, entradas de la FAT y del directorio) con el **contenido que queda** en cada una;
  - **marcas** que indican el grado de completitud de la operación.

### Procedimiento
**Paso 1 — escritura en el journal:** se escriben en el journal todas las modificaciones de la operación. Completado el asiento, se agrega una marca **"Commit"**. El journal debe escribirse con **`fwrite` seguido de `fflush` y `fsync`**, garantizando que el asiento esté efectivamente en disco antes de continuar. Antes de marcar COMMITED se espera `RETARDO_COMMIT` ms.

**Paso 2 — escritura en el filesystem:** se aplican efectivamente todas las modificaciones del asiento al filesystem. Aplicada toda la operación, se agrega una marca **"Applied"**. Acá `fflush`/`fsync` son **opcionales**, pero si se usan deben poder **desactivarse por configuración** (parámetro extra del grupo).

**Recuperación ante fallas (al iniciar):** recorrer el journal **cronológicamente** buscando operaciones inconclusas; por cada asiento:
- **sin "commit"** → se **descarta**;
- **con "commit" y "applied"** → se **descarta**;
- **con "commit" pero sin "applied"** → se **re-aplica en el filesystem en su totalidad**.

Las operaciones deben ser **idempotentes**: volver a aplicar un asiento ya aplicado no puede alterar el resultado. Se evalúa **finalizando el módulo abruptamente (`kill -9`) en medio de una operación y volviéndolo a iniciar**.

### Ejemplo de asiento (del enunciado)
`SAVE_CHECKPOINT modelo7`, con bloques de datos `0x0000000F` y `0x00000010`, usando la tercera entrada de directorio:

```
RECORD_STARTED: 1
OP_DIR: Directorio 3 - Checkpoint "modelo7" - Tamaño 60 - Bloque Inicial 0x0000000F
OP_FAT: Entrada 0x0000000F -> Valor 0x00000010
OP_FAT: Entrada 0x00000010 -> Valor 0x0FFFFFFF
OP_WRITE: Bloque 0x0000000F - Contenido: <...>
OP_WRITE: Bloque 0x00000010 - Contenido: <...>
RECORD_COMMITED 1
RECORD_APPLIED 1
```

Notas al pie: un puntero a bloque debe representarse en **hexadecimal**; como los bloques contienen bytes, se sugiere persistir el contenido en **caracteres hexadecimales**.

## Comandos locales (consola por stdin, salida "human-readable")

- `INFO`: contenido actual del superbloque.
- `LS`: todas las entradas de directorio con todos sus atributos.
- `CAT <CheckpointName> <cantBytes>`: primeros `cantBytes` del contenido, en hexadecimal.
- `WRITE <CheckpointName> <hexaContent>`: escribe o crea un archivo con el contenido indicado, **hasta 20 bytes hexadecimales**.
- `RM <CheckpointName>`: borra el archivo.

Se pueden agregar otros comandos mientras no modifiquen ni contradigan el funcionamiento del módulo.

## Logs mínimos y obligatorios (literales, nivel INFO, sin los `< >`)

| Evento | Formato |
|---|---|
| Conexión recibida | `## Módulo: <NOMBRE_MODULO_CONECTADO>` |
| Solicitud recibida | `## Solicitud [Guardar\|Cargar\|Eliminacion] Checkpoint: <NOMBRE>` |
| Solicitud completada | `## [Guardado\|Carga\|Eliminacion] Checkpoint: <NOMBRE> - Tamaño: <TAMAÑO> completada` |
| Errores | `## ERROR - <Falta de espacio en FAT / Directorio lleno> al intentar guardar: <NOMBRE>` |
| Acceso a FAT | `## Acceso FAT - Entrada: <NRO_ENTRADA> - Valor: 0x<VALOR>` |
| Acceso a bloque de datos | `## Acceso Bloque - Checkpoint: <NOMBRE> - Bloque: <NRO_BLOQUE>` |
| Journal (inicio) | `## Asiento iniciado: <ID_ASIENTO>` |
| Journal (modificación) | `## Asiento: <ID_ASIENTO> - Modificacion: <MODIFICACION>` |
| Journal (marca) | `## Nueva marca en asiento: <ID_ASIENTO>: - [COMMIT\|APPLIED]` |
| Recuperación | `## Recuperación - Asiento: <ID_ASIENTO> - <APLICADO / DESCARTADO>` |

(En los formatos con `[A|B]` se escribe una de las alternativas; "Eliminacion" y "Modificacion" van **sin tilde** tal como aparecen en el enunciado.)

## Archivo de configuración

| Campo | Tipo | Descripción |
|---|---|---|
| `PUERTO_ESCUCHA` | Número | Puerto del servidor |
| `LOG_LEVEL` | String | Nivel máximo de detalle; compatible con `log_level_from_string()` |
| `PATH_STORAGE` | String | Carpeta con los archivos del filesystem y del journal |
| `CANT_BLOQUES` | Número | Cantidad de bloques del filesystem |
| `TAM_BLOQUE` | Número | Tamaño de bloque en bytes (potencia de 2, **comenzando en 32**) |
| `BLOQUES_DIRECTORIO` | Número | Cantidad de bloques que ocupa el directorio |
| `RETARDO_ACCESO_BLOQUE` | Número | Milisegundos a esperar ante **cada acceso a cualquier bloque** del volumen (datos, directorio o FAT) |
| `RETARDO_COMMIT` | Número | Milisegundos a esperar antes de marcar cada asiento como "COMMITED" |

Ejemplo oficial:

```
PUERTO_ESCUCHA=8080
LOG_LEVEL=INFO
PATH_STORAGE=/home/utnso/storage/
CANT_BLOQUES=1024
TAM_BLOQUE=64
BLOQUES_DIRECTORIO=64
RETARDO_ACCESO_BLOQUE=100
RETARDO_COMMIT=100
```

*[derivado]* Con el ejemplo: FAT = 1024 × 4 = 4096 bytes = **64 bloques** (bloques 1–64); directorio = bloques 65–128, con 64 × 64 / 32 = **128 entradas**; bloques de datos = 129–1023 (**895 bloques**, ~57 KB).
