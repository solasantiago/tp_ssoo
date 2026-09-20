# Complementos del libro de cátedra (Silberschatz, "Fundamentos de Sistemas Operativos")

Fuente: `Libro - Fundamentos de Sistemas Operativos.pdf` (Silberschatz, Galvin y Gagne, 7ª edición en español; PDF escaneado sin texto — se hizo **OCR selectivo con `tesseract`** de las secciones citadas abajo, no del libro completo). Es el mismo libro de referencia de la cátedra, complementario a las diapositivas de clase (`2x-teoria-*.md`).

**Alcance de este archivo:** el libro tiene 844 páginas; se hizo OCR únicamente de las secciones con contenido que **agrega valor real** sobre lo ya cubierto en las diapositivas de clase (algoritmos de reemplazo de páginas más detallados, métodos de asignación de bloques tipo FAT, y recuperación/journaling de filesystems), no de los 6 capítulos completos que tocan temas del TP (Cap. 5, 6, 8, 9, 10, 11). Si en el futuro hace falta más detalle de algún capítulo, se puede repetir el procedimiento: `pdftoppm -r 150 -f <N> -l <M> "Libro - Fundamentos de Sistemas Operativos.pdf" salida` + `tesseract salida-NNN.png - -l spa`. El desfase entre página impresa y página del PDF es de **+17** (página impresa `p` = página de PDF `p+17`), verificado visualmente contra varias páginas.

## Algoritmo mejorado de segunda oportunidad = CLOCK-M (Cap. 9.4.5.3, pág. 299–300)

Esta es la fuente original y más precisa del algoritmo que el TP llama `CLOCK-M`, más detallada que la diapositiva de la cátedra (que ya lo resumía bien, pero acá está la razón completa):

Con el par ordenado (bit de referencia/uso, bit de modificación), toda página cae en una de 4 clases:

1. **(0, 0)** — no usada ni modificada recientemente: **la mejor víctima** (no hace falta escribirla a disco).
2. **(0, 1)** — no usada recientemente pero sí modificada: peor que la anterior, porque hay que escribirla antes de reemplazarla.
3. **(1, 0)** — usada recientemente pero limpia: probablemente se la vuelva a usar pronto.
4. **(1, 1)** — usada y modificada recientemente: probablemente se la use pronto **y además** habría que escribirla a disco.

El algoritmo recorre la cola circular (como CLOCK simple) pero, en vez de mirar solo el bit de uso, busca la **primera página de la clase no vacía más baja** (es decir, prioriza 1 sobre 2 sobre 3 sobre 4), pudiendo dar varias vueltas completas a la cola si hace falta. **Esto confirma exactamente** lo que se dedujo en `13-placa.md` y `22-teoria-memoria-virtual.md`: CLOCK-M prioriza páginas no modificadas como víctima para minimizar E/S al Offload, y la diferencia con el CLOCK simple es justamente esa preferencia por páginas limpias.

*[derivado]* La versión de 2 pasadas que dan las diapositivas de la cátedra (buscar primero `(0,0)`, si no hay buscar `(0,1)` limpiando bits U en el camino) es una simplificación práctica de estas 4 clases teóricas — en la práctica, con solo 2 bits, las clases 3 y 4 no se distinguen bien de 1 y 2 sin una segunda pasada que ya haya limpiado bits de uso, así que ambas descripciones son consistentes entre sí.

## Otros algoritmos de reemplazo (Cap. 9.4, pág. 291–301) — contexto adicional a lo ya visto en clase

- **Sustitución básica:** ante un Page Fault sin marcos libres, el flujo es (1) localizar la página en el respaldo, (2) elegir marco víctima con el algoritmo de reemplazo, (3) escribir la víctima al respaldo (si hace falta) y actualizar tablas, (4) leer la página deseada al marco liberado, (5) reiniciar la instrucción. Esto duplica el trabajo de E/S (una descarga + una carga) — de ahí el valor de evitar la escritura cuando la página no está modificada (ver CLOCK-M arriba).
- **Algoritmo óptimo (OPT/MIN):** reemplaza la página que no se va a usar durante más tiempo hacia adelante. Da la tasa de fallos más baja posible, pero requiere conocer el futuro — solo sirve como cota de comparación (igual que SJF en planificación de CPU, que requiere conocer la próxima ráfaga).
- **LRU — dos formas de implementarlo en la teoría "pura"** (sin la aproximación de bits de CLOCK): (a) **contadores**: un reloj lógico que se copia al acceder a una página, y se busca el mínimo cuando hace falta reemplazar; (b) **pila**: cada acceso mueve la página referenciada al tope de una pila (lista doblemente enlazada), la víctima es siempre la del fondo. Ambas requieren soporte de hardware caro para actualizar en cada acceso a memoria — por eso en la práctica se usan aproximaciones como CLOCK/CLOCK-M (bit de referencia con soporte de hardware mucho más barato).
- **LRU y OPT son "algoritmos de pila"**: nunca sufren la Anomalía de Belady (el conjunto de páginas en memoria con `n` marcos es siempre subconjunto del conjunto con `n+1` marcos). **FIFO no es un algoritmo de pila** y sí puede sufrirla.
- **LFU / MFU** (por contador de frecuencia de referencias, no de recencia): existen en la teoría pero **la propia cátedra los descarta** ("ni LFU ni MFU se utilizan de forma común... son bastante caros de implementar y tampoco son buenas aproximaciones a OPT"). El TP no los pide — coherente.
- **Búfer de páginas / pool de marcos libres:** en vez de esperar a escribir la víctima antes de poder usar el marco, se mantiene un pool de marcos ya libres: la página nueva se carga ahí de inmediato (el proceso se reanuda antes) y la víctima se escribe al respaldo después, en segundo plano. *[derivado]* Es una optimización de implementación, no un requisito del enunciado — el TP no exige este comportamiento, pero es válido implementarlo así siempre que los logs reflejen correctamente el orden real de "Bajada al Offload" / "Subida a Memoria".
- **Número mínimo de marcos por proceso:** está acotado por cuántas páginas distintas puede llegar a tocar una sola instrucción (para poder reiniciarla completa tras un Page Fault). *[derivado]* En EntrenadOS esto es relevante para las instrucciones de entrenamiento (`FORWARD`/`BACKWARD` tocan hasta 3 direcciones lógicas + registros, más el fetch de la instrucción en sí) y sobre todo para las syscalls que operan sobre un rango de memoria que puede abarcar varias páginas (`LOAD_BATCH`, `REPORT`, etc., que el enunciado ya resuelve pidiendo que el Core parta la petición en las páginas involucradas y las lockee todas antes de ejecutar).

## Asignación de bloques y FAT (Cap. 11.4, pág. 378–383) — confirma el diseño de FAT32_TRAIN

- Los tres métodos clásicos de asignación de bloques a un archivo son **contigua, enlazada e indexada** (ya vistos en las diapositivas de File Systems). El libro agrega el detalle de la **variante FAT de la asignación enlazada**:

  > "Una variante importante del mecanismo de asignación enlazada es la que se basa en el uso de una tabla de asignación de archivos (FAT). [...] Una sección del disco al principio de cada volumen se reserva para almacenar esa tabla, que tiene una entrada por cada bloque del disco y está indexada según el número de bloque. [...] Cada entrada de directorio contiene el número de bloque del primer bloque del archivo. La entrada de la tabla indexada según ese número de bloque contiene el número de bloque del siguiente bloque del archivo. Esta cadena continúa hasta el último bloque, que tiene un valor especial de fin de archivo como entrada de la tabla. Los bloques no utilizados se indican mediante un valor de tabla igual a 0."

  Esto es **exactamente** la estructura de `14-storage.md`: FAT como tabla de "siguiente bloque", `0` = libre, un valor especial de fin de cadena (`0x0FFFFFFF` en el TP). Confirma que el diseño de FAT32_TRAIN no es una invención del enunciado sino el mecanismo FAT real y estándar.
- **"El método FAT incorpora el control de los bloques libres dentro de la estructura de datos utilizada para el algoritmo de asignación; en este caso, no hace falta utilizar ningún método separado."** (Cap. 11.5.2, pág. 386) — confirma lo que se dedujo en `23-teoria-filesystems.md`: en FAT no hace falta un bitmap de libres aparte, porque el valor `0` en la FAT ya cumple ese rol.
- Ventaja de FAT sobre la asignación enlazada "pura": mejora el acceso aleatorio, porque el cabezal puede encontrar la ubicación de cualquier bloque leyendo la FAT (que se carga entera a memoria), sin tener que recorrer bloque por bloque en disco como en una lista enlazada ingenua.
- Comparación de rendimiento de los 3 métodos (contigua/enlazada/indexada): la contigua es la más rápida para cualquier tipo de acceso (un solo cálculo, sin punteros que seguir) pero sufre fragmentación externa; la enlazada resuelve la fragmentación pero es mala para acceso aleatorio si no se usa una FAT; la indexada agrupa los punteros en un bloque de índice aparte, buena para acceso directo pero gasta un bloque entero de índice incluso para archivos chicos.

## Recuperación de filesystems: fsck vs. journaling (Cap. 11.7, pág. 392–394)

El libro contrasta dos enfoques, y el segundo es la base teórica directa del journaling que pide el enunciado (`14-storage.md`):

### Comprobador de coherencia (fsck-style) — el enfoque que el TP NO pide
Un programa que corre al reiniciar, compara la estructura de directorios contra los bloques de datos reales del disco e intenta corregir inconsistencias. Problemas que señala el libro: puede requerir intervención humana, puede no lograr recuperar todo (pérdida de archivos), y es **lento** ("para comprobar varios terabytes de datos pueden hacer falta horas"). El TP evita explícitamente este enfoque reactivo/posterior a favor del journaling preventivo.

### Sistemas de archivos basados en registro/diario (journaling) — el enfoque que sí pide el TP

> "Fundamentalmente, todos los cambios de los metadatos se escriben secuencialmente en un registro. Cada conjunto de operaciones necesario para realizar una tarea específica es una **transacción**. Una vez que los cambios se han escrito en este registro, se consideran **confirmados** y la llamada al sistema puede volver al proceso de usuario [...]. Mientras tanto, estas entradas de registro se aplican en las estructuras reales del sistema de archivos. A medida que se realizan los cambios, se actualiza un puntero para indicar qué acciones se han completado y cuáles no. Una vez que se ha completado una transacción confirmada, se la elimina del archivo de registro [...]. Si el sistema sufre un fallo catastrófico, el archivo de registro contendrá cero o más transacciones [ya confirmadas pero no aplicadas], por lo que será necesario aplicarlas [de nuevo, en su totalidad]."

Esta es la **fuente teórica directa** del mecanismo de `14-storage.md`:

| Concepto del libro | Concepto del enunciado del TP |
|---|---|
| Transacción | Asiento |
| Escribir secuencialmente en el registro | Paso 1: escritura en el journal (`fwrite`+`fflush`+`fsync`) |
| "Se consideran confirmados" | Marca `Commit` |
| Se aplican en las estructuras reales | Paso 2: escritura en el filesystem real |
| Se elimina del registro tras aplicarse | *(el TP agrega la marca `Applied` en vez de borrar del journal — variante propia del enunciado, más simple de implementar que un buffer circular real, pero mismo propósito: distinguir "confirmado" de "ya aplicado")* |
| Transacción confirmada pero no aplicada al reiniciar → se re-aplica | Asiento con `Commit` sin `Applied` → se re-aplica en su totalidad |
| Transacción **no confirmada** (abortada antes del fallo) → sus cambios parciales deben deshacerse | Asiento sin `Commit` → se descarta *(el enunciado no pide siquiera aplicar parcialmente antes del Commit, así que no hay nada que deshacer: es más simple que el caso general del libro)* |

Esto **confirma por completo** que el diseño de journaling de EntrenadOS es una simplificación fiel del mecanismo real de journaling de filesystems (usado en NTFS, ext4, Veritas, etc., según el propio libro), no una invención ad-hoc de la cátedra.
