# Memoria del proyecto EntrenadOS — índice

Esta carpeta (`.claude/memoria/`) es la memoria persistente del asistente para este TP. Vive **dentro del repo** para que viaje entre las distintas computadoras del usuario: la memoria automática de Claude Code (`~/.claude/projects/...`) es local a cada máquina y no sirve para eso.

## Cómo se carga

- **Siempre cargados** (importados desde `CLAUDE.md` al iniciar cada sesión): `00-indice.md`, `01-estado-actual.md`, `02-como-trabajamos.md`.
- **Bajo demanda** (leerlos con `Read` cuando el tema lo requiera, sin pedir permiso): los archivos `03-*` y `1x-*`. Antes de afirmar cualquier detalle fino de un módulo (formato de log, campo de config, layout de bytes, semántica de una syscall), **leer el archivo del módulo**; no responder de memoria.

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

## Jerarquía de fuentes

1. El **PDF del enunciado** (`TP 2C2026 - EntrenadOS.pdf`, en la raíz del repo, v1.0) es la fuente de verdad.
2. Los archivos `1x-*` de esta carpeta son una **transcripción fiel** del PDF, con las deducciones propias marcadas como *[derivado]*. Ante duda entre ambos, gana el PDF.
3. `CLAUDE.md` es un **resumen orientativo**; si contradice a `1x-*`, ganan estos.
4. `readme.md` (raíz del repo) son las **notas de estudio** del usuario, escritas tema por tema a medida que se cierran.

## Regla de mantenimiento

Al final de cualquier sesión en la que se avance algo (se cierra un tema, se toma una decisión, cambia el próximo paso), **actualizar `01-estado-actual.md`** (y `03-decisiones-de-diseno.md` si corresponde) y avisarle al usuario que quedó pendiente de commit. Si la cátedra publica una nueva versión del enunciado, actualizar los `1x-*` y anotar la versión.
