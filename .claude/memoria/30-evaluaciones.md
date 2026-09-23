# Evaluaciones de estudio

Registro de las evaluaciones tipo mesa de examen hechas para cada tema: la consigna exacta, la respuesta textual del usuario, y la corrección del asistente. Se agrega una sección nueva por tema evaluado; si una evaluación queda a mitad de camino, se retoma en la misma sección la próxima vez.

## Tema: FIFO, RR y HRRN aplicados al Planificador

**Fecha:** 2026-09-23. **Fuentes de estudio indicadas:** `20-teoria-planificacion.md`, `11-planificador.md`.

### Tanda 1 — conceptos base

**Pregunta 1:** ¿Por qué FIFO y HRRN son algoritmos sin desalojo, pero RR siempre es con desalojo? Explicá la diferencia en términos de qué evento dispara la replanificación en cada caso.

> **Respuesta:** "No sé como explicarlo en términos de qué evento dispara la replanificación en cada caso, pero FIFO y HRRN ejecutan instrucciones desde su estado anterior (inicio o interrupción) hasta su próximo estado (interrupción o finalización); en cambio RR ejecuta fracciones de instrucciones lo cual amerita almacenar el estado al final de cada quantum."

**Corrección:** Parcialmente correcto, con una imprecisión importante: RR **no** ejecuta "fracciones de instrucciones" — el chequeo de interrupción (Check Interrupt) ocurre siempre **entre instrucciones completas**, nunca a mitad de una. La distinción real está en **qué eventos escucha cada uno**: FIFO/HRRN (sin desalojo) solo reaccionan a eventos que el propio proceso genera al soltar la CPU por su cuenta (termina, se bloquea, cede voluntariamente) — nunca le quitan la CPU a uno que sigue en condiciones de correr. RR (con desalojo) además reacciona a un evento que **no** libera la CPU por sí solo — la interrupción de timer del quantum — y ahí sí se la arrebata activamente.

**Pregunta 2:** En el modelo de 5 estados de EntrenadOS, ¿qué transición exacta corresponde al "desalojo por fin de quantum" de RR? ¿Y qué transición corresponde a que un Job termine su ráfaga voluntariamente (por ejemplo, por una syscall bloqueante)?

> **Respuesta:** "La transición que indica el desalojo por fin de quantum sería de EXEC→READY, dado que el resto de transiciones desde exec indican su finalización la interrupción por espera de acción externa."

**Corrección:** Correcto en la primera parte (**EXEC → READY**). Faltó decir explícitamente la segunda parte: una syscall bloqueante corresponde a **EXEC → BLOCK**. Para el coloquio conviene contestar ambos costados de una pregunta doble, aunque el segundo parezca implícito en la respuesta.

**Pregunta 3:** Escribí de memoria la fórmula de estimación de la próxima ráfaga (aging exponencial) y decime qué representa cada símbolo, incluyendo qué es `ESTIMACION_INICIAL` en esa fórmula.

> **Respuesta:** "Est(n+1)=alfa\*R(n)+(1-alfa)\*Est(n) con a entre 0 y 1. La estimación inicial es Est(0) y es una estimación antes de la primera ráfaga real, esta es necesaria debido a la recurrencia de la fórmula que necesita un paso base. Alfa es un parámetro que a mayor valor, mayor peso le da a lo último ejecutado, y a menor valor más peso tiene el historial (esto lo hace más estable). R(n) indica lo que realmente ejecutó la ráfaga anterior en la CPU."

**Corrección:** Correcto, sin errores. Fórmula bien, cada símbolo bien explicado, y el rol de `ESTIMACION_INICIAL` como caso base de la recurrencia quedó claro.

**Pregunta 4:** Escribí la fórmula del Response Ratio de HRRN y explicá con tus palabras por qué evita la starvation que puede sufrir SJF puro.

> **Respuesta:** *(pendiente)*

### Estado de esta evaluación

3 de 4 preguntas de la tanda 1 respondidas y corregidas. **Pendiente:** pregunta 4, y la tanda 2 completa (aplicación a EntrenadOS: qué cuenta como tiempo de espera para un Job que vuelve de BLOCK a READY, interacción con las syscalls no bloqueantes `INIT_JOB`/`ALLOC`/`FREE`).
