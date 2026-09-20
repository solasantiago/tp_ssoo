# Memoria del proyecto EntrenadOS — índice

Esta carpeta (`.claude/memoria/`) es la memoria persistente del asistente para este TP. Vive **dentro del repo** para que viaje entre las distintas computadoras del usuario: la memoria automática de Claude Code (`~/.claude/projects/...`) es local a cada máquina y no sirve para eso.

Este archivo es la versión **operativa** del índice (la que el asistente carga en cada sesión vía `CLAUDE.md`). El `readme.md` de la raíz del repo es su versión **humana**: lo que ve cualquiera que abra el repo en GitHub, con la misma orientación de estructura y orden de lectura pero sin las instrucciones dirigidas al asistente. Si se actualiza uno, conviene revisar que el otro siga coherente.

## Cómo se carga

- **Siempre cargados** (importados desde `CLAUDE.md` al iniciar cada sesión): `00-indice.md`, `01-estado-actual.md`, `02-como-trabajamos.md`.
- **Bajo demanda** (leerlos con `Read` cuando el tema lo requiera, sin pedir permiso): los archivos `03-*` y `1x-*`. Antes de afirmar cualquier detalle fino de un módulo (formato de log, campo de config, layout de bytes, semántica de una syscall), **leer el archivo del módulo**; no responder de memoria.

## Esquema de numeración

Los prefijos numéricos agrupan los archivos por familia, con huecos dejados a propósito entre familias para poder insertar un archivo nuevo sin renumerar los existentes (dentro de cada familia sí van correlativos):

| Rango | Familia |
|---|---|
| `00–03` | Meta: índice, estado, forma de trabajo, decisiones de diseño |
| `04–09` | *(libre — reservado para una futura familia meta)* |
| `10–15` | Letra del enunciado del TP: transcripción fiel del PDF, por módulo |
| `16–19` | *(libre — reservado para más contenido del enunciado)* |
| `20–25` | Teoría de la cátedra: diapositivas de clase + complementos del libro |

Si se agrega un archivo nuevo, va al primer número libre de la familia que le corresponda (o abre una familia nueva en un rango libre) — no se renumeran los archivos existentes.

## Archivos

| Archivo | Contenido | Cuándo leerlo |
|---|---|---|
| `01-estado-actual.md` | Dónde lo dejamos: temas cerrados, en curso, pendientes, próximo paso, última sesión | siempre (auto) |
| `02-como-trabajamos.md` | Preferencias y forma de trabajo del usuario con el asistente | siempre (auto) |
| `03-decisiones-de-diseno.md` | Decisiones de diseño/implementación tomadas por el grupo y su justificación | al diseñar o implementar algo |
| `10-arquitectura-y-reglas-generales.md` | Datos del TP, evaluación, motivos de desaprobación, deployment, arquitectura, quién habla con quién, glosario, links de la cátedra | dudas generales, arranque de un módulo |
| `11-planificador.md` | Spec completa del Planificador: estados, largo/corto plazo, syscalls, servicios, estadísticas, logs obligatorios, config | todo lo que toque planificación/syscalls/servicios |
| `12-core.md` | Spec completa del Core: registros, ciclo de instrucción, MMU, instrucciones, logs obligatorios, config | todo lo que toque ejecución/MMU/instrucciones |
| `13-placa.md` | Spec completa de la Placa: memoria de instrucciones y de usuario, Offload, reemplazo, operaciones, consola, logs obligatorios, config | todo lo que toque memoria/paginación |
| `14-storage.md` | Spec completa del Storage: FAT32_TRAIN (superbloque, FAT, directorio), operaciones, journaling, consola, logs obligatorios, config | todo lo que toque checkpoints/filesystem |
| `15-entregas-y-checks.md` | Checks de control con fecha, objetivos y cómo se testean; entregas finales; calendario | planificar el trabajo, saber qué se evalúa cuándo |
| `20-teoria-planificacion.md` | Teoría de la cátedra: planificadores por plazo, eventos de replanificación, FIFO/SJF/SRT/prioridades/HRRN/RR con fórmulas, hilos | al estudiar o justificar algo de planificación de corto plazo |
| `21-teoria-sincronizacion.md` | Teoría de la cátedra: condición de carrera, sección crítica, semáforos (mutex/contador), productor-consumidor | al diseñar la concurrencia interna de un módulo (colas de servicios, page locking) |
| `22-teoria-memoria-virtual.md` | Teoría de la cátedra: MMU, paginación simple, paginación bajo demanda, atención de Page Fault, algoritmos de reemplazo (FIFO/Óptimo/LRU/CLOCK/CLOCK-M) con la regla exacta, thrashing, lockeo de páginas | al estudiar o justificar algo de la Placa/MMU |
| `23-teoria-filesystems.md` | Teoría de la cátedra: qué es un FS, estrategias de asignación de bloques, FAT en detalle, journaling en general | al estudiar o justificar algo del Storage |
| `24-teoria-deadlocks.md` | Teoría de la cátedra: deadlocks y livelock. No lo pide el enunciado de EntrenadOS; documentado por completitud (parcial/coloquio) | solo si aparece deadlock en una consulta puntual |
| `25-teoria-libro-complementos.md` | Complementos del libro de cátedra (Silberschatz), citados con página exacta: CLOCK-M completo (4 clases), otros algoritmos de reemplazo, FAT confirmado contra el libro, journaling vs. fsck | al necesitar más profundidad que la de las diapositivas en reemplazo de páginas o journaling |

## Orden de lectura recomendado (para estudiar)

Si el usuario quiere leer los `.md` de esta carpeta directamente (no solo que el asistente los use de contexto), el orden sugerido es:

1. **Panorama general:** `10-arquitectura-y-reglas-generales.md` + `Notas de estudio.md` (raíz del repo) — qué es el TP, los 4 módulos, evaluación, y los temas que ya se cerraron con las palabras del usuario.
2. **Por bloque temático, teoría primero y después la letra del TP** (para entender el concepto general antes de ver cómo el enunciado lo simplifica):

   | Teoría (concepto) | Enunciado (implementación) |
   |---|---|
   | `20-teoria-planificacion.md` | `11-planificador.md` |
   | `21-teoria-sincronizacion.md` | (transversal a Planificador/Placa/Storage multihilo) |
   | — | `12-core.md` (no tiene teoría propia dedicada) |
   | `22-teoria-memoria-virtual.md` | `13-placa.md` |
   | `23-teoria-filesystems.md` + `25-teoria-libro-complementos.md` | `14-storage.md` |

3. **Cierre:** `15-entregas-y-checks.md` (qué se evalúa en cada check) y `03-decisiones-de-diseno.md` (a medida que el grupo resuelva los puntos abiertos).

`24-teoria-deadlocks.md` queda al margen de este recorrido — no lo pide el enunciado de EntrenadOS.

**Advertencia sobre estas fuentes:** los `2x-teoria-*.md` son resúmenes del asistente (citados con página/diapositiva exacta), no las fuentes primarias, y tienen dos límites conocidos: (a) se extrajo solo texto de los `.pptx`, así que los diagramas e imágenes originales (diagramas de estados, gráficos de fallos de página, layouts de memoria, etc.) no están reproducidos — hay que abrir el `.pptx` en `contenido_drive/` para verlos; (b) del libro (`Libro - Fundamentos de Sistemas Operativos.pdf`) solo se hizo OCR selectivo de un puñado de secciones puntuales (ver `25-teoria-libro-complementos.md`), no de los capítulos completos. Para el parcial o el coloquio, donde hay que defender el tema, conviene complementar con una lectura directa de esas fuentes en los puntos que lo requieran — pedirle al asistente que extienda el OCR o el resumen de una sección puntual es válido en cualquier momento.

## Jerarquía de fuentes

1. El **PDF del enunciado** (`TP 2C2026 - EntrenadOS.pdf`, en la raíz del repo, v1.0) es la fuente de verdad de **qué hay que construir**.
2. Los archivos `1x-*` de esta carpeta son una **transcripción fiel** del PDF, con las deducciones propias marcadas como *[derivado]*. Ante duda entre ambos, gana el PDF.
3. Los archivos `2x-teoria-*` son la **teoría de la cátedra** (diapositivas de clase en `contenido_drive/`, no versionado — ver `02-como-trabajamos.md`), usada para **justificar y completar** lo que el enunciado deja implícito o sin fórmula. Si un `2x-teoria-*` contradice al enunciado, **gana el enunciado**: es una simplificación académica del TP, no un error de la teoría.
4. `CLAUDE.md` es un **resumen orientativo**; si contradice a `1x-*` o `2x-*`, ganan estos.
5. `Notas de estudio.md` (raíz del repo) son las **notas de estudio** del usuario, escritas tema por tema a medida que se cierran.

## Regla de mantenimiento

Al final de cualquier sesión en la que se avance algo (se cierra un tema, se toma una decisión, cambia el próximo paso), **actualizar `01-estado-actual.md`** (y `03-decisiones-de-diseno.md` si corresponde) y avisarle al usuario que quedó pendiente de commit. Si la cátedra publica una nueva versión del enunciado, actualizar los `1x-*` y anotar la versión.
