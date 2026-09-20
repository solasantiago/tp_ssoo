# Estado actual — dónde lo dejamos

**Última actualización:** 2026-09-20.

## Fase

Estamos en la fase de **estudio conceptual del enunciado**, tema por tema, antes de implementar. En este repo todavía **no hay código**: es un repo de notas y contexto. El código del grupo vive en un repositorio aparte (ver `02-como-trabajamos.md`).

## Temas de estudio

| Estado | Tema | Dónde quedó registrado |
|---|---|---|
| Cerrado | Visión general: los 4 módulos, los 2 problemas de diseño, temas de la materia que toca | `readme.md` |
| Cerrado | Arquitectura y conexiones: orden de arranque, quién habla con quién | `readme.md` |
| Cerrado | Niveles de planificación (largo y corto plazo) | `CLAUDE.md` |
| Cerrado | Modelo de 5 estados y sus 8 transiciones (confirmadas contra el diagrama de la pág. 10 del PDF) | `readme.md` |
| Cerrado | Nacimiento (`INIT_JOB`, Job 0) y muerte (`EXIT`) de un Job | `readme.md` |
| **En curso** | **Algoritmos de corto plazo: FIFO, RR, HRRN** | — |
| Pendiente | Page Fault y atención de syscalls (bloqueantes vs. no bloqueantes, page locking, desconexión de Core) | — |
| Pendiente | Servicios del Planificador (Loader, Labeler, Logger) y estadísticas | — |
| Pendiente | Core: registros, ciclo de instrucción, MMU, instrucciones de entrenamiento | — |
| Pendiente | Placa: paginación bajo demanda, tablas de páginas, Offload, reemplazo LRU / CLOCK-M | — |
| Pendiente | Storage: FAT32_TRAIN (superbloque, FAT, directorio), operaciones | — |
| Pendiente | Storage: journaling y recuperación ante fallas | — |
| Pendiente | Conexiones, handshake y serialización (necesario para el Check 1, aunque no es "tema" de estudio) | — |

## Próximo paso

Repasar conceptualmente **FIFO, RR y HRRN** aplicados al Planificador, y cerrar el tema en `readme.md`. Puntos a resolver en ese repaso:

- Qué configura cada parámetro: `RR_QUANTUM` (ms), `ESTIMACION_INICIAL` (ms) y `HRRN_ALFA`.
- Fórmulas de HRRN: estimación de la próxima ráfaga (aging exponencial con alfa) y cálculo de la prioridad (response ratio). El enunciado no las da; validar contra la teoría de la materia (contradecirla es motivo de desaprobación).
- Cómo interactúan con el modelo de 5 estados: qué pasa con la estimación/espera cuando un Job vuelve de BLOCK a READY; el desalojo por quantum en RR (EXEC → READY vía interrupción al Core); el hecho de que las syscalls no bloqueantes (`INIT_JOB`, `ALLOC`, `FREE`) no liberan el Core.
- El log obligatorio "Estimación" (`## (<JID>) - Prioridad HRRN calculada: <PRIORIDAD> - Estimación próxima ráfaga: <ESTIMACION>`) fija qué valores hay que poder calcular.

## Calendario relevante

Hoy (20/09) el **Check 1 (12/09, conexiones y serialización) ya pasó**. El próximo hito es el **Check 2 (10/10)**: ciclo de instrucción básico en el Core + cola de Jobs, cambio de contexto y **FIFO/RR** en el Planificador. El tema en curso es justo lo que evalúa ese check. Detalle en `15-entregas-y-checks.md`.

## Última sesión (2026-09-20)

Se creó esta carpeta de memoria a partir del PDF, `CLAUDE.md` y `readme.md`, para tener el mismo contexto en todas las computadoras del usuario. Quedó **sin commitear** para que el usuario la revise.
