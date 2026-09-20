# Teoría de la cátedra — Memoria y Memoria Virtual

Fuente: `contenido_drive/Clase 5 y 6 - Memoria y Memoria Virtual/Memoria.pptx` y `Memoria Virtual.pptx`. Teoría de base; la letra obligatoria del TP está en `13-placa.md`. La cátedra da un panorama general de gestión de memoria (asignación contigua, segmentación, paginación jerárquica, tabla invertida) — **EntrenadOS solo usa paginación simple de un nivel con paginación bajo demanda**; el resto es contexto para entender por qué se eligió ese esquema, no algo a implementar.

## MMU y traducción de direcciones

La **MMU (Memory Management Unit)** traduce direcciones lógicas (DL) a físicas (DF). La reasignación puede hacerse en tiempo de compilación o de carga (DL == DF) o en **tiempo de ejecución** (DL != DF, requiere HW de traducción) — EntrenadOS traduce en tiempo de ejecución, en cada acceso.

Requisitos que debe satisfacer un esquema de memoria: realocación, protección (un proceso no accede al espacio de otro sin permiso), compartir memoria, y una organización lógica vs. física.

## Paginación simple (el esquema que usa la Placa)

- La memoria física se divide en **frames** (marcos) y el espacio de un proceso en **páginas**, ambos del **mismo tamaño fijo**. Cualquier marco puede asignarse a cualquier página → **no genera fragmentación externa**. Sí genera **fragmentación interna** en la última página de cada proceso (si el tamaño ocupado no es múltiplo exacto del tamaño de página).
- Cada proceso tiene una **tabla de páginas**: por cada página, indica en qué marco está y un **bit de validez** (si la página pertenece al espacio de direcciones del proceso). Los marcos libres se administran típicamente con un **bitmap**.
- Traducción con potencias de 2 (caso general, coincide con la fórmula del enunciado del TP):
  ```
  nro_página    = floor(dir_lógica / tamaño_página)
  offset        = dir_lógica % tamaño_página
  dir_física    = nro_marco * tamaño_marco + offset
  ```
  Con tamaños potencia de 2 esto se puede resolver con corrimientos de bits (los N bits más altos de la DL son el número de página, los bits bajos son el offset) — el enunciado del TP permite calcularlo con las fórmulas aritméticas directamente, no exige bitwise.
- **PTBR (Page Table Base Register):** registro de la CPU con el puntero a la tabla de páginas del proceso en ejecución; se guarda en el PCB en cada cambio de contexto. *[derivado]* En EntrenadOS este rol lo cumple la Placa manteniendo una tabla de páginas por JID (no hay un registro físico, pero el concepto — "cuál tabla corresponde al proceso corriendo ahora" — es análogo).
- **TLB (Translation Look-aside Buffer):** caché de hardware de alta velocidad que guarda traducciones recientes para evitar el acceso extra a la tabla de páginas en cada instrucción. El enunciado de EntrenadOS **no pide implementar TLB** (sería una optimización fuera del alcance del TP); cada traducción pasa siempre por la Placa.
- **Protección y compartición:** con bits de permisos (r/w/x) en la tabla de páginas, y apuntando varias tablas al mismo marco para compartir memoria entre procesos. El TP no pide permisos por página, pero **sí pide un análogo de compartición inversa**: la Placa hace reemplazo **global** (elige víctima entre páginas de cualquier Job), a diferencia de un reemplazo local por proceso.

### Estructuras de tabla de páginas más avanzadas (contexto, no usadas por el TP)
Paginación jerárquica (2 niveles, para no necesitar una tabla contigua gigante) y tabla de páginas invertida (una sola tabla para todo el sistema, indexada por marco). El TP usa **una tabla de páginas simple por Job** (no jerárquica, no invertida): la limitación de tamaño de tabla no es un problema porque `TAM_MEMORIA` y `TAM_PAGINA` son chicos en las pruebas.

## Memoria virtual y paginación bajo demanda

- **Problema que resuelve:** sin memoria virtual, un proceso debe caber entero en RAM para ejecutar, lo que limita su tamaño máximo. La memoria virtual permite que un proceso tenga partes en RAM y partes en disco (swap), de forma transparente para el programador.
- **Paginación bajo demanda (demand paging):** en vez de mover el proceso entero, se mueven páginas individuales, de forma "lazy" — se cargan a memoria física **solo cuando se necesitan**. Esto es exactamente lo que pide `13-placa.md`: "los Jobs empiezan sin marcos asignados, y una página pasa a ocupar uno recién cuando el Job la accede."
- Cada entrada de tabla de páginas necesita, como mínimo, un **bit de presencia (P)**: `P=1` la página está en memoria (indica el frame); `P=0` la página no está cargada (nunca se cargó, o fue reemplazada). Acceder a una página con `P=0` dispara una interrupción de **Page Fault**.

### Atención de un Page Fault (secuencia teórica completa)
1. Comprobar si la dirección es **válida** para ese proceso (¿está dentro de su espacio de direcciones?). Si es inválida: se finaliza el proceso o se informa un error (según el SO).
2. Si es válida, hay que cargar la página a memoria: se dispara una lectura a disco. Si no hay un marco libre, primero hay que **elegir una víctima y desalojarla** (algoritmo de reemplazo).
3. Al completarse la lectura, se actualiza la tabla de páginas (`P=1`, con el marco correspondiente).
4. Se **reinicia la instrucción** que causó la interrupción (no se continúa desde la mitad).

Esto valida punto por punto lo que dice `12-core.md` sobre no actualizar el PC ante un Page Fault (para que la instrucción se reintente completa) y lo que dice `11-planificador.md` sobre bloquear el Job y pedirle la carga a la Placa.

## Políticas de asignación y sustitución de marcos

- **Asignación:** fija (un proceso siempre tiene N marcos) vs. dinámica (puede variar).
- **Sustitución:** local (la víctima debe salir del propio conjunto de marcos del proceso) vs. **global** (la víctima puede ser de cualquier proceso).

**EntrenadOS usa asignación dinámica + sustitución global** (confirmado también textualmente en la diapositiva de cierre de esta clase, ver más abajo), tal como especifica el enunciado: "se deberá seleccionar una página víctima entre todas las páginas del conjunto residente [...] sin importar a qué Job pertenezcan."

## Algoritmos de reemplazo de páginas (vistos en la materia)

Se evalúan sobre una secuencia de referencias, contando cuántos Page Faults genera cada uno:

- **FIFO:** víctima = la página cargada hace más tiempo (la más "vieja" en memoria, no la menos usada). Simple (cola o timestamp de carga), pero sufre la **Anomalía de Belady**: en ciertas secuencias, aumentar la cantidad de marcos puede generar *más* Page Faults, no menos (contraintuitivo).
- **Óptimo:** víctima = la página que no se va a referenciar por más tiempo hacia adelante. Da la mínima cantidad posible de Page Faults, pero requiere conocer el futuro — solo sirve como cota de comparación teórica, no es implementable en la realidad.
- **LRU (Least Recently Used):** víctima = la página menos recientemente **usada** (hace más tiempo que no se referencia), usando el pasado como aproximación del futuro. No sufre la Anomalía de Belady. Implementaciones: guardar el timestamp de última referencia de cada página (se elige el menor), o una lista donde cada referencia mueve la página al final (se elige la primera). **Es uno de los dos algoritmos que pide el TP** (`ALGORITMO_REEMPLAZO=LRU`).
- **CLOCK (aproximación barata de LRU):** basado en FIFO + un **bit de Uso (U)**. Un puntero circular recorre los marcos: si `U=0`, esa página es la víctima (se reemplaza y avanza el puntero); si `U=1`, se le da "otra oportunidad" (se pone `U=0` y avanza el puntero sin reemplazar). Evita llevar timestamps exactos.
- **CLOCK Modificado (CLOCK-M):** además del bit de Uso, considera el bit de **Modificado (M)**, para minimizar las **descargas a disco** (no todo swap-out es igual de caro: una página no modificada no hace falta escribirla de nuevo si ya existe su copia en el respaldo). Con el par `(U, M)`, el algoritmo recorre el buffer circular en hasta dos pasadas:
  1. Buscar una página `(U=0, M=0)` avanzando el puntero **sin** tocar los bits de uso.
  2. Si no se encontró, buscar una `(U=0, M=1)` avanzando el puntero y esta vez **poniendo `U=0`** a las que se van revisando.
  3. Si tampoco se encontró, se repite desde el paso 1 (ahora ya hay páginas con `U=0` gracias al paso 2).

  **Este es el otro algoritmo que pide el TP** (`ALGORITMO_REEMPLAZO=CLOCK-M`) y **confirma** lo que en `13-placa.md` estaba marcado como *[derivado]*: CLOCK-M efectivamente prioriza como víctima a una página **no modificada** antes que a una modificada, precisamente para evitar la escritura al Offload cuando es posible.

## Thrashing (sobrepaginación)

Si a un proceso se le dan **menos marcos de los que necesita** para su conjunto de páginas activas (su **localidad**), empieza a generar Page Faults constantemente sin avanzar trabajo útil — **thrashing**. Con sustitución global y asignación dinámica, esto puede escalar: el SO ve poco uso de CPU, sube el grado de multiprogramación, los procesos existentes empiezan a necesitar más marcos y se los "roban" entre sí, generando más Page Faults todavía. La solución teórica es bajar el grado de multiprogramación o usar el **conjunto de trabajo (working set)**: darle a cada proceso al menos tantos marcos como su localidad activa, cuidando que la suma de todos los conjuntos de trabajo no supere el total de marcos disponibles.

*[derivado]* `GRADO_MULTIPROGRAMACION` en el TP es exactamente el parámetro que evita este escenario: al limitar cuántos Jobs conviven, se limita cuánta presión hay sobre `TAM_MEMORIA` / cantidad de marcos.

## Lockeo de páginas (fundamento teórico del page locking)

Cuando una página está en medio de una operación de I/O (por ejemplo, una lectura de disco para resolver un Page Fault), se puede habilitar un **bit de lockeo** sobre su marco para que no pueda ser elegida como víctima **mientras la operación está en curso**. Al terminar la I/O, se destraba y vuelve a ser candidata a reemplazo. Esto es exactamente el "page locking" del enunciado (`13-placa.md`): evita una condición de carrera entre el hilo que hace reemplazo y el hilo que está sirviendo esa página a un Core o a una syscall.

## Y esto cómo afecta al TP (síntesis de la propia cátedra, diapositiva de cierre de "Memoria Virtual")

> "Módulo CPU: fundamentos de Memoria Virtual — paginación bajo demanda, estructura de tablas de páginas (bits de presencia, uso y modificado), traducción de direcciones. Módulo Memoria: accesos a memoria + SWAP, asignación dinámica y reemplazo global, algoritmos de reemplazo, thrashing. TL;DR: todo hasta thrashing inclusive."

Nota: la diapositiva de cierre de "Memoria" (la clase anterior, sobre asignación contigua/segmentación) menciona segmentación y tablas de segmentos — **eso corresponde a una edición anterior de la materia o a contenido de contexto general, no a EntrenadOS**: el enunciado del TP actual no tiene segmentos, solo paginación simple. Si en el estudio aparece una discrepancia entre esa diapositiva y el enunciado, gana el enunciado.
