# Teoría de la cátedra — Deadlocks

Fuente: `contenido_drive/Clase 4 - Deadlocks/Deadlock.pptx`. **No corresponde a ningún requisito del enunciado de EntrenadOS** (esta clase, a diferencia de las otras 4, no tiene una diapositiva de cierre "y esto cómo afecta al TP" — es la única de las 5 sin esa sección). Se deja documentada igual porque es parte de la teoría de la materia y puede aparecer en el parcial o en el coloquio, pero no hay ningún `1x-*` de EntrenadOS al que corresponda.

## Definición

Un conjunto de procesos está en **deadlock (interbloqueo)** cuando todos esperan un suceso que solo puede producir otro proceso del mismo conjunto. Los procesos nunca terminan y los recursos que retienen quedan ocupados indefinidamente, impidiendo que otros trabajos avancen.

## Condiciones necesarias (deben darse las 4 simultáneamente)

1. **Exclusión mutua:** al menos un recurso es de uso no compartido.
2. **Retención y espera:** un proceso retiene un recurso mientras espera adquirir otros.
3. **Sin desalojo:** un recurso solo se libera voluntariamente, nunca se lo quitan a la fuerza.
4. **Espera circular:** existe un ciclo de procesos, cada uno esperando un recurso que retiene el siguiente.

## Tratamiento

1. **Ignorar el problema** (política del avestruz — la más común en SOs de propósito general, porque las otras son costosas).
2. **Prevención / evasión:** un protocolo asegura que el sistema nunca entre en deadlock.
3. **Detección y recuperación:** se permite que ocurra, se detecta y se recupera.

### Prevención
Ataca una de las 4 condiciones necesarias para que nunca se cumplan todas a la vez: evitar exclusión mutua donde se pueda (ej. abrir en modo lectura), pedir todos los recursos por adelantado (ataca retención y espera), permitir desalojo forzado, o **establecer un orden total de pedido de recursos** (ataca espera circular: si todos piden en orden creciente de un valor F(recurso), no puede cerrarse un ciclo).

### Evasión (Algoritmo del Banquero)
Cada proceso declara de antemano el **máximo** de cada recurso que puede llegar a pedir. Ante cada solicitud, el SO simula si otorgarla deja al sistema en **estado seguro** (existe algún orden en que todos los procesos podrían terminar sin deadlock). Solo se asigna si el estado resultante sigue siendo seguro; si no, el proceso espera. Requiere: matriz de peticiones máximas, matriz de recursos asignados, vector de recursos totales. Overhead alto (se corre en cada petición).

### Detección y recuperación
Se deja que el sistema pida libremente, y periódicamente se corre un algoritmo (similar al del banquero pero con peticiones *actuales*, no máximas) para detectar si hay deadlock. Si lo hay, recuperación por:
- **Finalizar procesos:** todos los del ciclo (caro) o de a uno hasta romper el ciclo (más trabajo por iteración, pero menos procesos perdidos).
- **Desalojar recursos:** quitarle el recurso a una víctima y devolverla a un estado desde el que pueda reanudar; riesgo de starvation si siempre se elige la misma víctima.

### Comparación
Prevención: poco overhead, pero puede subutilizar recursos según la política. Evasión: nunca hay deadlock, pero alto overhead (banquero en cada pedido) y es pesimista (puede negar una asignación seguro que en la práctica no habría causado problema). Detección y recuperación: overhead intermedio (depende de la frecuencia del chequeo), pero el deadlock **sí puede llegar a ocurrir** antes de detectarse.

## Livelock (relacionado, no es deadlock)

Un conjunto de procesos no logra progresar, pero a diferencia del deadlock **siguen ejecutándose** (no están bloqueados) — consumen CPU sin avanzar. Más difícil de detectar que un deadlock porque no hay procesos "quietos" para inspeccionar. Suele darse con recursos pedidos de forma no bloqueante (p. ej. semáforos con espera activa mal usados).

## Por qué no aplica directamente a EntrenadOS

*[derivado]* El TP no pide implementar prevención/evasión/detección de deadlocks como funcionalidad del sistema simulado. Donde sí es relevante como **cuidado de implementación** (a nivel de los hilos reales de C, no de los Jobs simulados): si el grupo usa varios mutex/semáforos internos (p. ej. uno por cola de servicio, uno por tabla de páginas, uno para el hilo único del Offload), hay que pedirlos siempre en el mismo orden entre threads para no crear una espera circular real entre los hilos del propio proceso Planificador/Placa/Storage — eso sería un deadlock de implementación, no un requisito funcional del enunciado.
