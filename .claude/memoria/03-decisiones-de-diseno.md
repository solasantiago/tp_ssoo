# Decisiones de diseño e implementación

Registro de las decisiones que el enunciado deja a criterio del grupo, con la opción elegida y su justificación (se defienden en el coloquio). Una entrada por decisión, con fecha. Si una decisión se revierte, no borrar: agregar la nueva entrada y marcar la anterior como reemplazada.

Formato sugerido:

```
## <fecha> — <tema>
**Decisión:** ...
**Alternativas descartadas:** ...
**Justificación (teoría):** ...
**Impacta a:** <módulos>
```

## Puntos abiertos que el enunciado deja al grupo (a resolver cuando toque)

- **Planificador:** fórmulas de estimación y prioridad de HRRN; si el tiempo en BLOCK cuenta para la espera de HRRN; cómo se implementa la interrupción de fin de quantum (hilo temporizador por Core, etc.); estructura de colas por estado; política si un Core se desconecta con un Job en EXEC (el enunciado solo dice que vuelve a READY).
- **Core:** cómo obtiene el tamaño de página (no está en su config); formato del handshake; cómo se reporta el "motivo" al devolver el contexto.
- **Placa:** estructura de la tabla de páginas (bits de presencia, uso, modificado, lock, marco, posición en Offload); qué hace CLOCK-M con víctimas no modificadas; cómo se representa el "espacio asignado" para que `FREE` sepa cuántas páginas liberar; asignación de direcciones lógicas (siempre al inicio de página).
- **Storage:** formato concreto del journal (texto); cómo se garantiza la idempotencia; si se usan `fflush`/`fsync` en el paso 2 y el nombre del parámetro de config que los desactiva; política de asignación de bloques (primer libre, etc.).
- **Comunes:** protocolo de mensajes (códigos de operación, serialización), biblioteca compartida entre módulos, scripts de deployment.

## Decisiones tomadas

_(todavía ninguna: el grupo está en la fase de estudio conceptual)_
