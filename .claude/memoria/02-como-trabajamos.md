# Cómo trabajamos

Preferencias y forma de trabajo del usuario con el asistente en este proyecto. Actualizar cuando el usuario corrija o fije una preferencia nueva.

## Contexto del usuario

- Alumno de Sistemas Operativos (UTN FRBA), integrante de un grupo de 5 que hace el TP EntrenadOS 2C2026.
- Trabaja desde **varias computadoras**. Por eso todo el contexto persistente vive en el repo (`CLAUDE.md`, `readme.md`, `.claude/memoria/`) y no en la memoria local de Claude Code.
- Idioma: **español rioplatense**, trato de vos, tono directo e informal.

## Los dos repositorios

- **Este repo (`tp_ssoo`)**: notas de estudio y contexto para el asistente. No tiene código.
- **`tp-2026-2c-sigma/`**: el repo de código del grupo (el que se entrega a la cátedra). Es un git aparte; puede estar clonado dentro de `tp_ssoo/` y está en `.gitignore`. Cuando se empiece a implementar, el código va ahí, no acá.

## Modalidad de trabajo

- **Estudio conceptual guiado, tema por tema**, siguiendo el orden del enunciado. El usuario explica el concepto con sus palabras y el asistente valida o corrige contra el enunciado y la teoría de la materia. Las notas en `readme.md` citan frases textuales del usuario cuando resumen bien la idea.
- Al **cerrar un tema**: se agrega la sección a `readme.md` (mismo estilo: explicación + citas + "¿por qué?"), se actualiza `01-estado-actual.md` y se anota el "Próximo tema" al final de `readme.md`.
- Cuando el enunciado no define algo, no inventar: marcarlo como decisión del grupo, registrar la opción elegida y su justificación en `03-decisiones-de-diseno.md` (se defiende en el coloquio).
- Cualquier propuesta de implementación debe ser **coherente con la teoría vista en clase**: contradecirla es motivo de desaprobación directa. Si una simplificación conveniente contradice la teoría, avisar.
- Los **logs obligatorios** se transcriben literalmente desde `1x-*`; nunca reconstruirlos de memoria.

## Git y commits

- El usuario **revisa y commitea**. El asistente prepara los cambios y avisa qué quedó pendiente de commit; solo commitea si se lo piden explícitamente.
- Estilo de mensajes de commit (según el historial): **español, infinitivo/imperativo, una línea descriptiva**, sin prefijos tipo `feat:`. Ejemplos reales: "Trackear .claude/settings.local.json para que viaje con el clone", "Aclarar preconceptos teóricos: paginación bajo demanda, offload y checkpointing".

## Entorno

- Máquina actual: WSL2 (Linux) con `bash`. Hay `poppler-utils` instalado (`pdftotext`, `pdftoppm`) para leer el PDF; no hay `pip`. `sudo` pide contraseña y no funciona desde el `!` de Claude Code: los comandos con `sudo` los corre el usuario en otra terminal.
- Antes de hacer tareas grandes, el usuario prefiere que el asistente **repregunte lo necesario** y luego avance sin interrupciones.
