# Cómo trabajamos

Preferencias y forma de trabajo del usuario con el asistente en este proyecto. Actualizar cuando el usuario corrija o fije una preferencia nueva.

## Contexto del usuario

- Alumno de Sistemas Operativos (UTN FRBA), integrante de un grupo de 5 que hace el TP EntrenadOS 2C2026.
- Trabaja desde **varias computadoras**. Por eso todo el contexto persistente vive en el repo (`CLAUDE.md`, `readme.md`, `Notas de estudio.md`, `.claude/memoria/`) y no en la memoria local de Claude Code.
- Idioma: **español rioplatense**, trato de vos, tono directo e informal.

## Los dos repositorios

- **Este repo (`tp_ssoo`)**: notas de estudio y contexto para el asistente. No tiene código.
- **`tp-2026-2c-sigma/`**: el repo de código del grupo (el que se entrega a la cátedra). Es un git aparte; puede estar clonado dentro de `tp_ssoo/` y está en `.gitignore`. Cuando se empiece a implementar, el código va ahí, no acá.

Dentro de `tp_ssoo`, `readme.md` y `Notas de estudio.md` tienen roles distintos y no hay que confundirlos ni fusionarlos: `readme.md` es la portada del repo (orientación de estructura y orden de lectura, lo que ve cualquiera en GitHub — espejo humano de `.claude/memoria/00-indice.md`), y `Notas de estudio.md` es el diario de estudio conceptual tema por tema (antes se llamaba `readme.md`, se renombró para liberar ese nombre para la portada).

## Material de la cátedra (no versionado)

En `contenido_drive/` (carpeta local, en `.gitignore`) están las diapositivas y ejercicios de las clases de la cátedra, y en la raíz hay un `Libro - Fundamentos de Sistemas Operativos.pdf` (también en `.gitignore`). Ninguno de los dos se sube al repo porque son material con derechos de la cátedra y pesan mucho (82MB + 26MB). Su contenido relevante para el TP ya está procesado y resumido, con citas, en `.claude/memoria/2x-teoria-*.md` (que sí viaja con el repo). Si en otra computadora hace falta reprocesar algo del material original, primero hay que copiar esas carpetas/archivos a esa máquina (no viajan solos con `git pull`).

El libro es un **PDF escaneado sin capa de texto**: hace falta `tesseract` (instalado, `tesseract-ocr` + `tesseract-ocr-spa`) para leer contenido nuevo de ahí — `pdftoppm -r 150 -f <N> -l <M> archivo.pdf salida` y después `tesseract salida-NNN.png - -l spa`. El desfase entre página impresa y página del PDF es **+17** (verificado). Es el libro "Fundamentos de Sistemas Operativos" de Silberschatz, Galvin y Gagne (7ª ed. en español, 844 páginas); su índice completo ya se relevó (ver `25-teoria-libro-complementos.md` para cómo se referencian capítulos y páginas). Las diapositivas (`.pptx`) sí tienen texto nativo extraíble con `python3` + `zipfile` (sin necesidad de `pip` ni LibreOffice).

## Modalidad de trabajo

- **Estudio conceptual guiado, tema por tema**, siguiendo el orden del enunciado. El usuario explica el concepto con sus palabras y el asistente valida o corrige contra el enunciado y la teoría de la materia. Las notas en `Notas de estudio.md` citan frases textuales del usuario cuando resumen bien la idea.
- Al **cerrar un tema**: se agrega la sección a `Notas de estudio.md` (mismo estilo: explicación + citas + "¿por qué?"), se actualiza `01-estado-actual.md` y se anota el "Próximo tema" al final de `Notas de estudio.md`.
- Cuando el enunciado no define algo, no inventar: marcarlo como decisión del grupo, registrar la opción elegida y su justificación en `03-decisiones-de-diseno.md` (se defiende en el coloquio).
- Cualquier propuesta de implementación debe ser **coherente con la teoría vista en clase**: contradecirla es motivo de desaprobación directa. Si una simplificación conveniente contradice la teoría, avisar.
- Los **logs obligatorios** se transcriben literalmente desde `1x-*`; nunca reconstruirlos de memoria.

## Git y commits

- El usuario **revisa y commitea**. El asistente prepara los cambios y avisa qué quedó pendiente de commit; solo commitea si se lo piden explícitamente.
- Estilo de mensajes de commit (según el historial): **español, infinitivo/imperativo, una línea descriptiva**, sin prefijos tipo `feat:`. Ejemplos reales: "Trackear .claude/settings.local.json para que viaje con el clone", "Aclarar preconceptos teóricos: paginación bajo demanda, offload y checkpointing".

## Entorno

- Máquina actual: WSL2 (Linux) con `bash`. Hay `poppler-utils` instalado (`pdftotext`, `pdftoppm`) para leer el PDF; no hay `pip`. `sudo` pide contraseña y no funciona desde el `!` de Claude Code: los comandos con `sudo` los corre el usuario en otra terminal.
- Antes de hacer tareas grandes, el usuario prefiere que el asistente **repregunte lo necesario** y luego avance sin interrupciones.
