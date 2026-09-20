# EntrenadOS — Notas de estudio

## Visión general

EntrenadOS es un sistema distribuido que simula el funcionamiento de un sistema operativo. El escenario es un **cluster de entrenamiento de modelos de IA**: los usuarios envían trabajos de entrenamiento (**Jobs**) que el sistema planifica, ejecuta en una CPU simulada y administra en memoria.

### Los cuatro módulos

| Módulo | Rol |
|---|---|
| **Planificador** | Administra la cola de Jobs y decide cuándo y en qué orden se ejecutan |
| **Core** | Ejecuta las instrucciones de un Job, como si fuera una CPU |
| **Placa** | Administra la memoria: dónde viven los datos de cada Job |
| **Storage** | Persiste los checkpoints del modelo en un filesystem propio |

Todos pueden correr en máquinas distintas y comunicarse por red. Cada módulo es un proceso real del SO, escrito en C.

### Los dos problemas que definen el diseño

El enunciado impone dos condiciones que generan casi toda la complejidad:

1. **La memoria de la Placa es más chica que el espacio que necesitan los Jobs** → hay que implementar paginación bajo demanda y un mecanismo de swap llamado **Offload**.
2. **Un entrenamiento puede durar días** → los Jobs tienen que poder guardar su estado en disco (**checkpoints**) y retomarlo más adelante, incluso después de una caída del sistema.

### Temas de la materia que toca

- **Planificación de procesos:** largo plazo (cuántos Jobs entran al sistema) y corto plazo (FIFO, Round Robin, HRRN)
- **Memoria virtual y paginación:** MMU, page faults, reemplazo de páginas (LRU / Clock Modificado), page locking
- **Sincronización y concurrencia:** servidores multihilo, servicios con colas propias
- **Filesystem:** FAT32 simplificado implementado desde cero, con journaling para recuperarse ante fallas

---

## Arquitectura y conexiones

### Orden de arranque

El orden de inicio es obligatorio por dependencias:

1. **Placa** y **Storage** (no dependen de nadie)
2. **Planificador** (se conecta a Placa y Storage al iniciar)
3. **Core** (se conecta a Planificador y Placa al iniciar)

### Quién le habla a quién

- **Core → Placa:** fetch de instrucciones y traducción de páginas (MMU)
- **Core → Planificador:** reportar page faults y syscalls
- **Planificador → Placa:** cargar páginas, leer/escribir memoria, desbloquear páginas
- **Planificador → Storage:** operaciones de checkpoint

> El Core **nunca habla con el Storage**. Cuando encuentra una syscall de checkpoint, se la delega al Planificador, que es quien resuelve con Storage.

---

## Módulo: Planificador

El Planificador administra la cola de Jobs del sistema y decide cuándo y en qué orden se ejecutan.

### Modelo de 5 estados

Los Jobs no avanzan secuencialmente de un estado al siguiente: hay transiciones definidas en direcciones específicas, y no todas las combinaciones son válidas.

Los cinco estados son **NEW, READY, EXEC, BLOCK y EXIT**. Las transiciones posibles:

- `NEW → READY`: el Job ingresa al sistema y queda listo para ejecutar
- `READY → EXEC`: el Planificador le asigna un Core
- `READY → EXIT`: el Job es cancelado antes de ejecutar
- `EXEC → READY`: el Job es desalojado (ej: se acabó el quantum)
- `EXEC → BLOCK`: el Job necesita esperar algo externo (I/O, carga de página)
- `EXEC → EXIT`: el Job terminó o fue cancelado
- `BLOCK → READY`: lo que esperaba terminó
- `BLOCK → EXIT`: el Job es cancelado mientras espera

> "Son pasos que no ocurren secuencialmente en un orden definido, pero sí hay direcciones explícitas en las que deben fluir." No existe, por ejemplo, ir de BLOCK directo a EXEC.

**¿Por qué BLOCK y READY son estados distintos si los dos son "espera"?**

> "BLOCK depende de alguna instrucción para avanzar, y READY no."

READY significa *tengo todo lo que necesito, solo espero que me asignen un Core*. BLOCK significa *no puedo avanzar aunque me den un Core, porque estoy esperando algo externo*. La distinción importa porque solo los Jobs en READY son candidatos para ejecutar — mezclarlos con los de BLOCK sería ineficiente.

### Cómo nace y muere un Job

**Nacimiento**

Un Job nace cuando el Core ejecuta la syscall `INIT_JOB`. El Core le pasa al Planificador el nombre de un archivo de pseudocódigo, y el Planificador crea el Job en estado NEW. Esta syscall no bloquea al Job que la solicitó: el Job que pidió crear otro sigue ejecutando con normalidad.

> Caso especial: el **Job 0** no nace por `INIT_JOB`. Es el Job inicial del sistema, y su archivo de pseudocódigo se le pasa al Planificador directamente como parámetro al arrancarlo.

**Muerte**

Un Job muere cuando el Core ejecuta la syscall `EXIT`. El Planificador lo pasa a estado EXIT, libera todas las estructuras asociadas en todo el sistema (memoria en la Placa, estructuras administrativas) y, si el grado de multiprogramación lo permite, deja entrar a un nuevo Job de NEW a READY.

> En resumen: los Jobs nacen y mueren por instrucciones que ejecuta el Core, pero quien efectivamente los crea y destruye en el sistema es el Planificador.

---

*Próximo tema: algoritmos de planificación de corto plazo (FIFO, Round Robin, HRRN).*
