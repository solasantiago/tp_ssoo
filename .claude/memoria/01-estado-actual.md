# Estado actual — dónde lo dejamos

**Última actualización:** 2026-09-20.

## Fase

Estamos en la fase de **estudio conceptual del enunciado**, tema por tema, antes de implementar. En este repo todavía **no hay código**: es un repo de notas y contexto. El código del grupo vive en un repositorio aparte (ver `02-como-trabajamos.md`).

## Temas de estudio

| Estado | Tema | Dónde quedó registrado |
|---|---|---|
| Cerrado | Visión general: los 4 módulos, los 2 problemas de diseño, temas de la materia que toca | `Notas de estudio.md` |
| Cerrado | Arquitectura y conexiones: orden de arranque, quién habla con quién | `Notas de estudio.md` |
| Cerrado | Niveles de planificación (largo y corto plazo) | `CLAUDE.md` |
| Cerrado | Modelo de 5 estados y sus 8 transiciones (confirmadas contra el diagrama de la pág. 10 del PDF) | `Notas de estudio.md` |
| Cerrado | Nacimiento (`INIT_JOB`, Job 0) y muerte (`EXIT`) de un Job | `Notas de estudio.md` |
| **En curso** | **Algoritmos de corto plazo: FIFO, RR, HRRN** | — |
| Pendiente | Page Fault y atención de syscalls (bloqueantes vs. no bloqueantes, page locking, desconexión de Core) | — |
| Pendiente | Servicios del Planificador (Loader, Labeler, Logger) y estadísticas | — |
| Pendiente | Core: registros, ciclo de instrucción, MMU, instrucciones de entrenamiento | — |
| Pendiente | Placa: paginación bajo demanda, tablas de páginas, Offload, reemplazo LRU / CLOCK-M | — |
| Pendiente | Storage: FAT32_TRAIN (superbloque, FAT, directorio), operaciones | — |
| Pendiente | Storage: journaling y recuperación ante fallas | — |
| Pendiente | Conexiones, handshake y serialización (necesario para el Check 1, aunque no es "tema" de estudio) | — |

## Próximo paso

Repasar conceptualmente **FIFO, RR y HRRN** aplicados al Planificador, y cerrar el tema en `Notas de estudio.md`. Las fórmulas y la teoría de fondo ya están relevadas y confirmadas en `20-teoria-planificacion.md` (estimación exponencial de ráfaga, response ratio de HRRN, por qué RR es siempre con desalojo). Falta el repaso conversacional con el usuario para cerrarlo y decidir los puntos que el enunciado deja abiertos:

- Qué configura cada parámetro: `RR_QUANTUM` (ms), `ESTIMACION_INICIAL` (ms) y `HRRN_ALFA`. Ya resuelto conceptualmente en `20-teoria-planificacion.md`.
- Cómo interactúan con el modelo de 5 estados: qué pasa con la estimación/espera cuando un Job vuelve de BLOCK a READY (punto abierto, ver `03-decisiones-de-diseno.md`); el desalojo por quantum en RR (EXEC → READY vía interrupción al Core); el hecho de que las syscalls no bloqueantes (`INIT_JOB`, `ALLOC`, `FREE`) no liberan el Core.
- El log obligatorio "Estimación" (`## (<JID>) - Prioridad HRRN calculada: <PRIORIDAD> - Estimación próxima ráfaga: <ESTIMACION>`) fija qué valores hay que poder calcular.

## Calendario relevante

Hoy (20/09) el **Check 1 (12/09, conexiones y serialización) ya pasó**. El próximo hito es el **Check 2 (10/10)**: ciclo de instrucción básico en el Core + cola de Jobs, cambio de contexto y **FIFO/RR** en el Planificador. El tema en curso es justo lo que evalúa ese check. Detalle en `15-entregas-y-checks.md`.

## Última sesión (2026-09-20)

Se creó esta carpeta de memoria a partir del PDF, `CLAUDE.md` y `Notas de estudio.md` (commiteada y pusheada). En una segunda parte de la misma sesión se incorporó también la teoría de la cátedra: diapositivas de las 5 clases relevantes (`contenido_drive/`, no versionado) y OCR selectivo del libro de texto (Silberschatz, tampoco versionado — ver `02-como-trabajamos.md`), resumidos con citas en `20-teoria-planificacion.md` a `25-teoria-libro-complementos.md`. Varios puntos que antes estaban marcados como *[derivado]* en `11-planificador.md` y `13-placa.md` (fórmulas de HRRN, preferencia de CLOCK-M por páginas no modificadas) quedaron confirmados contra la teoría oficial. Después, el antiguo `readme.md` (notas de estudio) se renombró a `Notas de estudio.md`, y `readme.md` pasó a ser la portada del repo (índice humano, espejo de este archivo). Todo esto está commiteado o pendiente de commit según lo que el usuario haya revisado — chequear `git status` al retomar.
