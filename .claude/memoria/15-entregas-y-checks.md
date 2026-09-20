# Entregas y checks de control

Fuente: enunciado v1.0 (25/08/2026), págs. 32–33. Los checks están diseñados para desarrollar el TP de forma iterativa incremental, en paralelo a los temas de la teoría.

## Check 1 — Conexiones y Serialización — 12/09

Objetivos:
- Arquitectura de red y ruteo de mensajes básica implementados.
- Planificador, Placa y Storage inicializan sus **servidores multihilo**.
- El Core se conecta exitosamente al Planificador y a la Placa, realiza un **Handshake inicial** para identificarse y envía/recibe un **paquete serializado** de prueba.

Cómo se testea: se levantan los 4 procesos de forma aislada y se valida por los **logs obligatorios** que los handshakes se realizan correctamente y que los paquetes se envían/reciben **sin cuelgues ni caídas de conexión**.

## Check 2 — Planificación y Ejecución Básica — 10/10

Objetivos:
- Ciclo de instrucción integrado en el Core (**aún sin memoria de datos ni MMU**).
- El Planificador gestiona la cola de Jobs y demuestra el **cambio de contexto** (context switch), evaluando los algoritmos de corto plazo **FIFO y RR**.

Cómo se testea: ejecución **concurrente** de varios scripts matemáticos o de loops contadores (`NOOP`, `SET`, `SUM`, `SUB`, `JNZ`). Se verifica en los logs del Planificador que los Jobs **respeten quantums y prioridades**, alternando correctamente entre **READY, EXEC y EXIT**.

## Check 3 — Memoria, Paginación y Servicios, Storage — 31/10

Objetivos:
- Core: habilitar la **MMU** para traducir direcciones y reportar **Page Faults**.
- Placa: **tablas de páginas**, **paginación bajo demanda**, **algoritmo de reemplazo** y **Offload** (swap).
- Planificador: **servicios I/O bloqueantes**.
- Storage: **formateo del volumen** y **comandos locales**.

Cómo se testea: script tipo **Mini Entrenamiento** que reserva memoria (`ALLOC`), lee datos simulados (`LOAD_BATCH`) y recorre ese espacio con `FORWARD`, `BACKWARD`, `UPDATE`. Fuerza acceso a múltiples páginas generando Page Faults. Se valida en los logs el envío de páginas al Offload y el pasaje de los Jobs por la cola de **BLOCKED**.

## Check 4 — Storage, Filesystem y Persistencia — 14/11

Objetivos:
- Storage: todas las solicitudes que puedan llegar del Planificador.
- Storage: **Journaling**.

Cómo se testea: un script trabaja datos en memoria y los persiste con `SAVE_CHECKPOINT`; luego un script **independiente** recupera esa sesión con `LOAD_CHECKPOINT`. Además, se fuerza la detención abrupta del Storage (**`kill -9`**) en medio de un guardado; al reiniciarlo, se valida en los logs que el journal **restaura el filesystem a un estado íntegro y consistente**.

## Entregas finales — 28/11, 12/12, 19/12

- Finalizar el desarrollo de todos los módulos.
- Probar el TP de manera **intensiva en un entorno distribuido**.
- Todos los componentes ejecutan los requerimientos de forma integral.

## Calendario (para ubicarse rápido)

| Fecha | Hito |
|---|---|
| 25/08 | Comienzo |
| 12/09 | Check 1 |
| 10/10 | Check 2 |
| 31/10 | Check 3 |
| 14/11 | Check 4 |
| 28/11 | Entrega final 1 |
| 12/12 | Entrega final 2 |
| 19/12 | Entrega final 3 |
