# Teoría de la cátedra — File Systems, FAT y Journaling

Fuente: `contenido_drive/Clase 7 y 8 - File Systems/File Systems.pptx` y `FAT y EXT2.pptx`. Teoría de base; la letra obligatoria del TP está en `14-storage.md`. La cátedra da un panorama general de filesystems (asignación contigua/enlazada/indexada, inodos, links) — **EntrenadOS implementa específicamente una variante de FAT**, así que el resto (EXT2, inodos, links) es contexto para entender por qué FAT se diseña como se diseña, no algo a implementar.

## Qué es un filesystem y qué garantiza

Da servicio de uso de archivos a las aplicaciones, ubicado como capa entre el SO y el hardware de almacenamiento. Objetivos: almacenar y operar datos, garantizar su integridad, optimizar desempeño, soportar múltiples tipos de almacenamiento y de usuarios, minimizar pérdida de datos, y dar una interfaz estandarizada.

Un **archivo** es un conjunto de datos etiquetado con un nombre, de existencia duradera, compartible entre procesos. El **directorio** es en sí mismo un archivo que lista nombres de otros archivos (y sus atributos), mapeando nombres a los archivos reales.

## Organización de datos en disco (visión general)

- **MBR / sector de partida:** bloque de control de arranque del volumen.
- **Volumen:** control general del filesystem (metadata siempre en memoria al montar) + estructuras para espacio libre y para administrar archivos (parcialmente en memoria) + los archivos en sí.
- La **unidad mínima de asignación** de un FS es el **bloque lógico** (`tam_bloque = N * tam_sector`). El TP fija esto directamente con `TAM_BLOQUE` y `CANT_BLOQUES`.

## Estrategias de asignación de bloques a un archivo (contexto — FAT usa la "enlazada")

- **Contigua:** ventajas de acceso directo y secuencial rápido, pero sufre fragmentación externa y es difícil de agrandar un archivo ya creado.
- **Enlazada/encadenada:** cada bloque apunta al siguiente. No sufre fragmentación externa ni limita el crecimiento, pero es mala para acceso directo (hay que recorrer la cadena) y es sensible a que se corrompa un puntero.
- **Indexada:** cada archivo tiene un bloque de índice con punteros a sus bloques de datos (eventualmente multinivel, para archivos grandes). Buena para acceso directo y secuencial, pero gasta espacio en punteros.

## FAT (File Allocation Table) — la familia a la que pertenece FAT32_TRAIN

**FAT es una variación de la asignación enlazada**, optimizada así: en vez de guardar el puntero "al siguiente bloque" dentro de cada bloque de datos (lo que obligaría a leer el bloque completo solo para saber cuál sigue), **todos los punteros se agrupan en una tabla aparte** — la FAT — que se carga entera a memoria al iniciar el sistema, para que seguir la cadena sea rápido (acceso a la tabla en RAM en vez de ir a disco por cada eslabón). Por su importancia, en un FS FAT real se guarda además una copia de respaldo de la tabla (el TP no pide esto).

- Hay **una entrada de FAT por cada bloque (cluster)** de disco. La entrada indica cuál es el bloque que sigue al de ese índice.
- **No usa FCB por archivo**: la información administrativa de cada archivo (nombre, tamaño, primer bloque) se guarda **directamente en la entrada de directorio**. Esto es exactamente la estructura de directorio que pide `14-storage.md` (estado + nombre + tamaño + primer bloque de datos, sin una estructura de FCB separada).
- Directorios de **tamaño fijo**, con **entradas de tamaño fijo**.
- Para conseguir un bloque libre, hay que **recorrer la tabla** buscando una entrada en 0 (o el equivalente a "libre").

### Tamaño de las entradas (FAT12 / FAT16 / FAT32 reales)

```
Tam FAT           = cant_entradas * tam_entrada
Tam máx teórico FS = 2^tam_entrada * tam_cluster
```

FAT32 real usa entradas de 32 bits pero solo **28 bits utilizables** para direccionar bloques (2^28 entradas como máximo), reservando los 4 bits más significativos — **esto es exactamente lo que dice el enunciado de FAT32_TRAIN**: "de sus 32 bits, solo los 28 menos significativos forman el número de bloque; los 4 más significativos están reservados." Confirma que el diseño del TP es fiel al FAT32 real en este punto específico, no una invención de la cátedra para el TP.

### Ejemplo minimalista de la cátedra (mismo razonamiento que aplica al TP)

Con una FAT hipotética de 4 bits por entrada: `cant_entradas = 2^4 = 16`, `tam_FAT = 16 * 4 bits = 8 bytes`, `cant_bloques = cant_entradas = 16`, `tam_FS = 16 * tam_bloque`. *[derivado]* El mismo razonamiento es el que se usó en `14-storage.md` para calcular, con el ejemplo de config oficial del TP, que la FAT ocupa 64 bloques y hay 895 bloques de datos.

## Gestión de espacio libre (visión general — el TP ya fija esto vía la FAT)

Estrategias posibles: bit vector (bitmap), bloques libres enlazados entre sí, bloques indexados, lista de bloques contiguos libres. **En FAT, el valor `0` en una entrada ya cumple el rol de "bit de libre" para ese bloque** — no hace falta una estructura de espacio libre separada, la propia FAT la contiene implícitamente. Esto es consistente con `14-storage.md`, donde no hay un bitmap adicional: buscar espacio es recorrer la FAT buscando entradas en `0`.

## Journaling (fundamento teórico, antes de la sección específica del enunciado)

> "La información de las estructuras del FS por lo general está más actualizada en memoria que en disco. Un fallo en el sistema o en el hardware puede generar inconsistencias."

La solución es registrar las modificaciones (agrupadas en una **transacción**) en un **journal** (registro secuencial aparte) **antes** de aplicarlas al filesystem real, marcando un **Commit** cuando la transacción quedó completamente escrita en el journal. Si el sistema se cae después del Commit pero antes de aplicar los cambios reales, al reiniciar se puede **re-aplicar** la transacción completa desde el journal, sin depender de en qué estado intermedio haya quedado el filesystem.

Esto es la base teórica exacta del mecanismo de `14-storage.md`: el "asiento" del TP es la "transacción" de la teoría, la marca `Commit` es la misma, y la marca `Applied` que agrega el TP es una extensión propia del enunciado (marca *cuándo* además ya se aplicó, para distinguir "hay que re-aplicar" de "ya está aplicado, se puede descartar" durante la recuperación).

## Archivos mapeados a memoria, ACL, locks de archivo (contexto, no usado por el TP)

La clase también cubre *memory-mapped files*, listas de control de acceso (ACL) y permisos estilo Unix, y locks de archivo (exclusivo/compartido, obligatorio/sugerido). **Nada de esto aparece en el enunciado de Storage**: FAT32_TRAIN no tiene permisos por checkpoint ni locks explícitos de archivo — la concurrencia la resuelve el servidor multihilo del Storage a nivel de sus propias estructuras internas (FAT, directorio), no a nivel de "lockear un checkpoint" como concepto de filesystem.

## Y esto cómo afecta al TP (síntesis de la propia cátedra, diapositivas de cierre de "File Systems")

La cátedra da dos variantes posibles de trabajo práctico según el cuatrimestre — la relevante para EntrenadOS es la de **asignación enlazada (FAT)**:

> "Módulo File System — Asignación de bloques enlazada (FAT). FCB: [en FAT, la propia entrada de directorio]. Operaciones sobre archivos: Crear, Leer/Escribir, Borrar (liberar bloques). Administración de bloques libres: a través de la propia tabla de asignación."

*[derivado]* La otra variante que menciona la cátedra (asignación indexada pura, con bloque de índice y bitmap de libres) **no es la que usa EntrenadOS** — se incluye en las diapositivas porque la cátedra reutiliza el mismo material para cuatrimestres donde el TP pide una implementación distinta. Para este TP (2C2026, EntrenadOS), la especificación aplicable es la de `14-storage.md`, que es FAT.
