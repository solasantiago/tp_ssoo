# EntrenadOS — TP Sistemas Operativos (UTN FRBA, 2C2026)

Este repositorio contiene el **contexto y las notas de estudio** para el trabajo práctico cuatrimestral de la Cátedra de Sistemas Operativos: diseñar e implementar, en C, un sistema distribuido que simula un sistema operativo para un cluster de entrenamiento de modelos de IA. **No contiene el código del TP** — eso vive en un repositorio aparte (ver más abajo).

## Qué hay acá

| Archivo / carpeta | Qué es | ¿Viaja con `git clone`? |
|---|---|---|
| `TP 2C2026 - EntrenadOS.pdf` | El enunciado oficial del TP (v1.0) — fuente de verdad | Sí |
| `CLAUDE.md` | Resumen orientativo del TP para el asistente (Claude Code) | Sí |
| `Notas de estudio.md` | Notas de estudio conceptuales, tema por tema, con citas de las discusiones | Sí |
| `.claude/memoria/` | Base de conocimiento estructurada: transcripción fiel del enunciado + teoría de la cátedra, con citas exactas | Sí |
| `contenido_drive/` | Diapositivas y ejercicios originales de la cátedra | No (`.gitignore` — material con derechos de la cátedra) |
| `Libro - Fundamentos de Sistemas Operativos.pdf` | Libro de referencia (Silberschatz) | No (`.gitignore`) |
| `tp-2026-2c-sigma/` | Repo de código del grupo (la implementación en C que se entrega) | No, es un git aparte |

## Por dónde empezar

- **¿Querés el panorama general del TP?** Leé `Notas de estudio.md` — está escrito en lenguaje llano, tema por tema, a medida que se van cerrando.
- **¿Buscás una spec exacta** (formato de un log obligatorio, un campo de config, el layout de bytes del filesystem)? Andá directo a `.claude/memoria/1x-*.md` (uno por módulo: Planificador, Core, Placa, Storage) — son transcripción fiel del enunciado.
- **¿Querés entender la teoría detrás de una decisión de diseño** (por qué CLOCK-M prefiere páginas no modificadas, de dónde sale la fórmula de HRRN, cómo funciona el journaling de un filesystem real)? Mirá `.claude/memoria/2x-teoria-*.md` — resume las diapositivas de la cátedra y extractos puntuales del libro, citados con página/diapositiva exacta.
- **¿Vas a usar Claude Code para seguir trabajando en esto?** No hace falta que leas nada más: `CLAUDE.md` carga automáticamente el índice y el estado actual al iniciar sesión.

## Orden de lectura recomendado (para estudiar)

1. **Panorama general:** `.claude/memoria/10-arquitectura-y-reglas-generales.md` + `Notas de estudio.md` — qué es el TP, los 4 módulos, evaluación, y los temas ya cerrados.
2. **Por bloque temático, teoría primero y después la letra del TP** (para entender el concepto general antes de ver cómo el enunciado lo simplifica):

   | Teoría (concepto) | Enunciado (implementación) |
   |---|---|
   | `20-teoria-planificacion.md` | `11-planificador.md` |
   | `21-teoria-sincronizacion.md` | (transversal a Planificador/Placa/Storage multihilo) |
   | — | `12-core.md` (no tiene teoría propia dedicada) |
   | `22-teoria-memoria-virtual.md` | `13-placa.md` |
   | `23-teoria-filesystems.md` + `25-teoria-libro-complementos.md` | `14-storage.md` |

   (todos dentro de `.claude/memoria/`)
3. **Cierre:** `15-entregas-y-checks.md` (qué se evalúa en cada check) y `03-decisiones-de-diseno.md` (a medida que el grupo resuelva los puntos que el enunciado deja abiertos).

`24-teoria-deadlocks.md` queda al margen de este recorrido — no lo pide el enunciado de EntrenadOS, está documentado por completitud.

## Advertencia sobre las fuentes de `.claude/memoria/`

Los archivos `2x-teoria-*.md` son resúmenes armados con el asistente (citados con página/diapositiva exacta), no las fuentes primarias, y tienen dos límites conocidos:

- Se extrajo solo **texto** de las diapositivas (`.pptx`): los diagramas e imágenes originales (diagramas de estados, gráficos de fallos de página, layouts de memoria, etc.) no están reproducidos — hay que abrir el `.pptx` en `contenido_drive/` para verlos.
- Del libro solo se hizo **OCR selectivo** de un puñado de secciones puntuales (ver `25-teoria-libro-complementos.md`), no de los capítulos completos.

Para el parcial o el coloquio, donde hay que defender el tema, conviene complementar con una lectura directa de esas fuentes en los puntos que lo requieran.

## Jerarquía de fuentes (ante cualquier contradicción)

1. El **PDF del enunciado** manda sobre todo lo demás: es la fuente de verdad de qué hay que construir.
2. Los `.claude/memoria/1x-*.md` son transcripción fiel de ese PDF.
3. Los `.claude/memoria/2x-teoria-*.md` son teoría de la cátedra, usada para completar lo que el enunciado deja implícito — si contradicen al enunciado, gana el enunciado (es una simplificación académica adrede).
4. `CLAUDE.md` es un resumen orientativo; si contradice a los anteriores, ganan ellos.

---

*Este archivo es la versión humana del índice; su equivalente operativo (el que carga el asistente en cada sesión) es `.claude/memoria/00-indice.md`.*
