# Arquitectura, reglas generales y evaluación

Fuente: enunciado v1.0 (25/08/2026), págs. 1–9. Transcripción fiel; lo marcado como *[derivado]* es una deducción nuestra.

## Datos del TP

- **EntrenadOS** — "Diseño e Implementación de un Sistema Distribuido". Cátedra de Sistemas Operativos, UTN FRBA, TP cuatrimestral 2C2026, versión 1.0 (publicada 25/08/2026).
- Modalidad: **grupal (5 integrantes ± 0)** y obligatorio.
- Fecha de comienzo: 25/08. Entregas finales: **28/11, 12/12, 19/12**. Corrección: Laboratorio de Sistemas, sede Medrano.

## Objetivos declarados
Adquirir práctica con herramientas/APIs del SO; entender aspectos de diseño de un SO; afirmar conceptos teóricos implementándolos; familiarizarse con diseño de sistemas distribuidos, archivos de configuración y de log. Los conceptos son versiones **simplificadas o alteradas** de los reales, para resaltar diseño o simplificar implementación.

## Evaluación (2 etapas)

1. **Pruebas en laboratorio** de los programas. Antes de la evaluación final se publican pruebas para validar. El TP solo es **evaluable** si provee logs claros: cada módulo tiene un listado de **logs mínimos y obligatorios**.
2. **Coloquio individual** (solo si se aprueba la etapa 1): valida la aplicación de los conocimientos y define la nota **de cada integrante**. Se recomienda repartir el trabajo equitativamente, pero eso **no asegura la aprobación de todos**: cada uno debe defender teórica y prácticamente lo desarrollado. **No es recuperable.**

Motivos de desaprobación explícitos:
- Una implementación que **contradiga lo visto en clase o lo escrito en el enunciado**.
- Desarrollar **únicamente** conectividad, serialización o sincronización ("Aclaración importante", pág. 9): desaprobación directa.
- No cumplir los **logs mínimos** o no guardarlos en archivo: TP no evaluable → desaprobado.
- Omitir puntos de "Lineamiento e Implementación" de un módulo puede conllevar desaprobación.

## Deployment y testing
- Los módulos (procesos) deben poder ejecutarse en **diferentes computadoras**. La cantidad de máquinas y la distribución de procesos la define **cada test**, y **puede cambiar en el momento de la evaluación**.
- Es responsabilidad del grupo **automatizar el despliegue** de los procesos con sus archivos de configuración para cada test.
- Todo se detalla en el **documento de pruebas**, publicado cerca de la entrega final. Archivos y programas de ejemplo: repositorio de la cátedra.
- Lectura **mandatoria** de las Normas del TP: `https://docs.utnso.com.ar/primeros-pasos/normas-tp`.

## Convenciones del enunciado
- **Lineamiento e Implementación**: definición funcional + aspectos técnicos **obligatorios** de cada módulo.
- **Archivos de configuración**: parámetros mínimos para ajustar el comportamiento **sin recompilar**. En la evaluación debe alcanzar con detener, modificar el archivo y volver a ejecutar. El grupo **puede agregar parámetros extra**.
- **Logs**: con `so-commons-library` (`https://sisoputnfrba.github.io/so-commons-library/`, doc de log: `https://faq.utnso.com.ar/commons-docs/log_8h.html`). Los obligatorios en **`LOG_LEVEL_INFO`**; se pueden extender con `LOG_LEVEL_DEBUG`. La notación `<>` encierra valores variables; **los caracteres `<` y `>` no deben aparecer** en el log final.
- Donde el enunciado no define algo, la implementación queda a decisión del equipo; se recomienda consultar en el **foro** (`https://github.com/sisoputnfrba/foro`) porque las justificaciones se exponen en el coloquio.

## Qué simula el sistema
Un **cluster** (computadoras interconectadas que trabajan como un único equipo) dedicado a entrenar modelos de IA. Los usuarios envían **Jobs** (trabajos de entrenamiento) que quedan encolados hasta que haya recursos. Un Job repite muchas veces un ciclo: trae un lote de datos, lo procesa hacia adelante por las capas del modelo, recorre las capas hacia atrás calculando correcciones, y aplica las correcciones al modelo. Cada Job **reserva su espacio de trabajo en la memoria de la Placa** y opera sobre él toda su ejecución.

Dos condiciones que definen el diseño:
1. El espacio de trabajo de un Job suele ser **más grande que la memoria de la Placa**, y varios Jobs conviven en ella → paginación bajo demanda + Offload.
2. Un entrenamiento **puede durar días** → guardar estado y retomarlo, incluso tras una caída → checkpoints + journaling.

Metodología: **iterativa incremental** (primero ciertos módulos, luego integración total), alineada a los checks de control (ver `15-entregas-y-checks.md`). El orden es un lineamiento, no una obligación.

## Arquitectura y dependencias

| Módulo | Rol | Cantidad |
|---|---|---|
| Planificador | Administra la cola de Jobs y planifica su ejecución | 1 |
| Core | Ejecuta las instrucciones de los Jobs | **uno o más** |
| Placa | Administra la memoria donde los Jobs alojan su espacio de trabajo | 1 |
| Storage | Persiste los checkpoints | 1 |

Diagrama (pág. 8): flechas = **dependencias**. Core → Planificador, Core → Placa, Planificador → Placa, Planificador → Storage.

- Cada módulo es un programa en **C**, compilado y ejecutado en la máquina virtual de la cátedra: un **proceso real** del SO.
- **Orden de arranque** *[derivado del diagrama]*: (1) Placa y Storage; (2) Planificador (se conecta a ambos); (3) Cores (se conectan a Planificador y Placa).
- "Job" (con J mayúscula) = trabajo simulado que se planifica y ejecuta dentro del TP.

### Quién habla con quién (resumen de los cuatro módulos)
- **Core → Placa**: fetch de instrucción por PC; obtención de marco (MMU); lectura/escritura de datos; deslockeo de páginas.
- **Core → Planificador**: recibe JID + contexto; devuelve JID + contexto por Page Fault, syscall bloqueante, interrupción o EXIT; solicita syscalls.
- **Planificador → Placa**: creación de Job; asignación/liberación de espacio (ALLOC/FREE); carga de página; lectura/escritura de datos (servicios y checkpoints); deslockeo de páginas; finalización de Job.
- **Planificador → Storage**: guardar / cargar / eliminar checkpoint.
- **Planificador → Core**: interrupciones (desalojo por quantum, etc.).
- El Core **nunca** habla con el Storage.

## Glosario
- **JID**: Job ID, numérico, identifica unívocamente a cada Job. El Job inicial es el JID 0.
- **Conjunto residente**: páginas de un Job actualmente cargadas en la Placa.
- **Page locking**: marcar una página/marco temporalmente para que no pueda ser removida de memoria por un reemplazo.
- **Offload**: archivo de respaldo (swap) de las páginas que no están en la memoria de la Placa.
- **Asiento**: registro de una operación en el journal (id + modificaciones + marcas Commit/Applied).

## Links de la cátedra (extraídos del PDF)
- Normas del TP: `https://docs.utnso.com.ar/primeros-pasos/normas-tp`
- so-commons-library: `https://sisoputnfrba.github.io/so-commons-library/`
- Doc de `log.h`: `https://faq.utnso.com.ar/commons-docs/log_8h.html`
- Job de ejemplo: `https://github.com/sisoputnfrba/entrenados-pruebas/blob/main/ejemplo_job/job_entrenamiento.asm`
- Foro: `https://github.com/sisoputnfrba/foro`
