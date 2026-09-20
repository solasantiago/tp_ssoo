# Módulo: Placa — especificación completa

Fuente: enunciado v1.0 (25/08/2026), págs. 21–25. Transcripción fiel; lo marcado como *[derivado]* es una deducción nuestra, no texto del enunciado.

## Rol y arranque

Administra la memoria del sistema: en ella se almacenan las **instrucciones** de los Jobs y el **espacio de trabajo** (memoria de usuario) sobre el que operan.

```
./bin/placa [Archivo Config]
```

- Al iniciar crea un **servidor multihilo** que atiende concurrentemente peticiones del **Planificador** y de los **Cores**.
- Los Cores pueden conectarse en cualquier momento → escucha de nuevas conexiones **siempre activa**.

## Memoria de instrucciones

- Contiene las instrucciones de los Jobs, que le piden los Cores. Capacidad **ilimitada** (a diferencia de la memoria de usuario).
- Los archivos de pseudocódigo son **archivos de texto** con las instrucciones separadas por `'\n'`. Hay **uno por Job**; el grupo debe implementar una estructura que asocie **qué Job tiene qué archivo**.
- Sin restricción sobre cómo obtener las instrucciones: la Placa solo debe poder enviar la instrucción correspondiente a cada pedido del Core.

## Memoria de usuario

### Esquema y estructuras
- **Paginación con memoria virtual**, con **asignación dinámica** y **alcance global**. Tamaño total (`TAM_MEMORIA`) y tamaño de página (`TAM_PAGINA`) por config, ambos potencia de 2.
- **Paginación bajo demanda:** los Jobs empiezan **sin marcos asignados**; una página pasa a ocupar un marco **recién cuando el Job la accede**.
- Estructuras mínimas:
  1. Un **espacio contiguo de memoria** (`void*`): la memoria de datos del sistema.
  2. Una **tabla de páginas por cada Job**.
- Las tablas de páginas son estructuras administrativas y **no deben guardarse dentro de la memoria de datos**.
- El Core le pide a la Placa el **marco** correspondiente a cada página que necesite acceder.

### El Offload
- Archivo administrado por la Placa; path (`PATH_OFFLOAD`) y tamaño (`TAM_OFFLOAD`) por config.
- Guarda las páginas de los Jobs que **no están** en la memoria de la Placa.
- Imita accesos a disco: debe implementarse como si fuera **un único hilo de ejecución** — **nunca** dos o más lecturas/escrituras en paralelo.
- Cada operación (leer/escribir) tiene un retardo (`RETARDO_OFFLOAD`).

### Reemplazo de páginas
- Cuando hay que cargar una página y **no hay marcos libres**: se elige una **página víctima** entre **todas** las páginas del **conjunto residente** (páginas de un Job actualmente cargadas en la Placa) que **no estén lockeadas**, **sin importar a qué Job pertenezcan** (reemplazo global).
- Algoritmo por config (`ALGORITMO_REEMPLAZO`): **LRU** o **Clock Modificado** (`CLOCK-M`).
- *[derivado]* Reemplazar implica bajar la víctima al Offload (log "Bajada al Offload") y subir la nueva página desde el Offload al marco liberado (log "Subida a Memoria"). Con CLOCK-M, si la víctima no está modificada podría evitarse la escritura al Offload; el enunciado no lo aclara, decisión del grupo.

## Operaciones

### Creación de Job
Recibe **JID** y **nombre de archivo de pseudocódigo**, relativo al path base `PATH_INSTRUCCIONES`. Crea las estructuras administrativas del Job.

### Asignación de espacio
- Recibe **JID** y **tamaño en bytes**. Responde la **dirección lógica** en la que comienza el espacio asignado.
- **Cada asignación debe comenzar al inicio de una página.**
- Se debe **reservar en el Offload** el lugar necesario para respaldar las páginas asignadas. Si **no hay espacio suficiente**, la operación no se realiza y se informa al Planificador **para que finalice el Job**.
- *[derivado]* El límite de memoria por Job no es la memoria física sino el Offload: un Job puede tener mucho más espacio asignado que `TAM_MEMORIA` mientras entre en `TAM_OFFLOAD`.

### Liberación de espacio
Recibe **JID** y **dirección lógica** donde comienza un espacio previamente asignado. Se liberan los **marcos** ocupados por sus páginas y el lugar que tenían **reservado en el Offload**.

### Obtención de marco (pedida por el Core en la traducción de la MMU)
Recibe **JID** y **número de página**. Consulta la tabla de páginas del Job:
- Página **residente** → responde el **número de marco**; la página pasa a estar **lockeada**.
- Página **no residente** → responde indicando **Page Fault**.

### Carga de página (pedida por el Planificador tras un Page Fault)
Recibe **JID** y **número de página**; debe **cargarla en la memoria** de la Placa (buscando marco libre o aplicando reemplazo).

### Lectura de datos
Puede venir de un **Core o del Planificador**. Recibe **JID, dirección física y tamaño**. Responde el contenido del rango.

### Escritura de datos
Mismos parámetros que lectura, más el **contenido a escribir**.

### Deslockeo de páginas
- La envía el **Core** cada vez que finaliza con éxito un ciclo de instrucción, o cuando devuelve el contexto al Planificador (p. ej. al sufrir un Page Fault).
- En syscalls que operan sobre memoria del Job, la envía el **Planificador** al finalizar la operación.
- Recibe el **JID** y libera el estado de lockeo de **todas** sus páginas, permitiendo que sus marcos vuelvan a ser elegibles como víctimas.

### Finalización de Job
Recibe solo el **JID**. Libera **todos los marcos** que ocupaba, el lugar reservado en el **Offload** y **todas las estructuras** asociadas.

## Comandos locales (consola por stdin, salida "human-readable")

- `INFO`: porcentaje de memoria/offload ocupada, cantidad de frames libres/lockeados/totales, cantidad de Jobs actuales.
- `TLS`: lista todos los Jobs registrados; para cada uno: ID, cantidad de páginas del conjunto residente y cantidad de páginas totales.

Se pueden agregar otros comandos mientras no modifiquen ni contradigan el funcionamiento del módulo.

## Logs mínimos y obligatorios (literales, nivel INFO, sin los `< >`)

| Evento | Formato |
|---|---|
| Conexión recibida | `## Módulo: <NOMBRE_MODULO_CONECTADO>` |
| Creación de Job | `## JID: <JID> - Job Creado` |
| Obtener instrucción | `## JID: <JID> - Obtener instrucción: <PC> - Instrucción: <INSTRUCCIÓN> <...ARGS>` |
| Asignación / Liberación de espacio | `## JID: <JID> - <Asignación/Liberación> - Dirección Lógica: <DIRECCIÓN_LÓGICA> - Tamaño: <TAMAÑO>` |
| Subida de página a memoria | `## JID: <JID> - Subida a Memoria - Página: <NRO_PÁGINA> - Marco: <NRO_MARCO>` |
| Bajada de página al Offload | `## JID: <JID> - Bajada al Offload - Página: <NRO_PÁGINA>` |
| Reemplazo de página | `## Reemplazo - Marco: <NRO_MARCO> - JID Víctima: <JID> - Página Víctima: <NRO_PÁGINA>` |
| Lectura/Escritura en espacio de usuario | `## JID: <JID> - <Lectura/Escritura> - Dir. Física: <DIRECCION_FISICA> - Tamaño: <TAMAÑO>` |

## Archivo de configuración

| Campo | Tipo | Descripción |
|---|---|---|
| `PUERTO_ESCUCHA` | Número | Puerto del servidor |
| `LOG_LEVEL` | String | Nivel máximo de detalle; compatible con `log_level_from_string()` |
| `TAM_MEMORIA` | Número | Tamaño en bytes de la memoria de datos (potencia de 2) |
| `TAM_PAGINA` | Número | Tamaño de las páginas en bytes (potencia de 2) |
| `RETARDO_MEMORIA` | Número | Milisegundos a esperar para dar una respuesta |
| `ALGORITMO_REEMPLAZO` | String | `LRU` o `CLOCK-M` |
| `PATH_INSTRUCCIONES` | String | Carpeta con los archivos de pseudocódigo de los Jobs |
| `PATH_OFFLOAD` | String | Path del archivo de Offload |
| `TAM_OFFLOAD` | Número | Tamaño en bytes del archivo de Offload |
| `RETARDO_OFFLOAD` | Número | Milisegundos a esperar por cada operación del Offload (leer/escribir) |

Ejemplo oficial:

```
PUERTO_ESCUCHA=8080
LOG_LEVEL=INFO
TAM_MEMORIA=4096
TAM_PAGINA=64
RETARDO_MEMORIA=10
ALGORITMO_REEMPLAZO=CLOCK-M
PATH_INSTRUCCIONES=/home/utnso/scripts/
PATH_OFFLOAD=/home/utnso/offload.dat
TAM_OFFLOAD=65536
RETARDO_OFFLOAD=100
```

*[derivado]* Con el ejemplo: 4096 / 64 = **64 marcos** en memoria y 65536 / 64 = **1024 páginas** de capacidad en el Offload.
