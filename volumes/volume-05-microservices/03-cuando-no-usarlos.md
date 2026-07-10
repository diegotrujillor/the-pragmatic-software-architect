# 03. Cuándo NO usarlos

## Nivel 1 — ELI20

Este capítulo cierra el libro con la pregunta que debió estar presente
desde la primera línea del Volumen 1: **¿este problema realmente necesita
esta solución?** Microservicios resuelven un problema de coordinación entre
equipos independientes. Si ese problema no existe todavía — si eres un
equipo de una o dos personas, o un equipo que despliega todo junto sin
fricción real — adoptarlos no acelera nada. Agrega exactamente los costos
del capítulo anterior (consistencia eventual, latencia, observabilidad
distribuida) a cambio de un beneficio que nadie está cobrando todavía.

## Nivel 2 — Ingeniería

Señales concretas de que un sistema **no** debería dividirse en
microservicios todavía:

- **Un solo equipo despliega todo el sistema.** El problema organizacional
  que los microservicios resuelven — equipos bloqueándose entre sí en el
  release — no existe si solo hay un equipo. La independencia de
  despliegue entre "servicios" que en la práctica siempre libera la misma
  persona no es independencia real.
- **El dominio todavía no tiene Bounded Contexts maduros y probados.**
  Dividir antes de que el Volumen 2 esté resuelto en la práctica —no solo
  en un diagrama— casi garantiza límites de servicio incorrectos, porque
  el conocimiento de dónde está la verdadera frontera de negocio todavía
  se está formando.
- **La carga de tráfico no justifica escalado independiente.** Si ningún
  subdominio tiene un patrón de uso desproporcionadamente distinto al
  resto, replicar el monolito completo horizontalmente resuelve la
  capacidad sin ninguno de los costos de coordinación entre servicios.
- **El equipo no tiene, todavía, la disciplina operativa de observabilidad
  distribuida, manejo de fallos parciales e idempotencia** (capítulo
  anterior). Adoptar microservicios sin esa disciplina ya presente produce
  el monolito distribuido — todo el costo, ninguno de los beneficios.

## Nivel 3 — Senior Engineer

Un senior trata "seguimos siendo un monolito modular" como una decisión de
arquitectura tan legítima y defendible como cualquier otra — no como una
etapa temporal vergonzosa en el camino "hacia" microservicios. La
alternativa real y madura a microservicios prematuros no es "sin
arquitectura" — es un **monolito modular**, con los límites del Volumen 2 ya
respetados en el código (módulos que se comunican por eventos o interfaces
explícitas, nunca importando clases internas de otro módulo), listo para
extraerse el día que el dolor organizacional real aparezca — sin haber
pagado, mientras tanto, ningún costo de red, consistencia eventual ni
observabilidad distribuida que ese dolor todavía no justificaba.

## Nivel 4 — Software Architect

El argumento frente al CTO es, quizás, el más importante de todo el libro:
**la arquitectura correcta es la que resuelve el problema que el negocio
tiene hoy, medido con evidencia, no la que anticipa un problema que el
negocio podría tener en un futuro hipotético.** Adoptar microservicios sin
el dolor organizacional que los justifica no es "prepararse para escalar"
— es gastar presupuesto de ingeniería real, hoy, en un problema que no
existe, mientras el problema que sí existe (llegar a producto-mercado,
iterar rápido con un equipo chico) recibe menos atención de la que
necesita. Un arquitecto senior defiende activamente la decisión de *no*
migrar cuando la evidencia no lo justifica — esa defensa es tan parte del
trabajo como diseñar la migración cuando sí corresponde.

## ❌ Mitos

**"Si no empezamos con microservicios, migrar después será imposible o
carísimo."** No, si el monolito se construyó modular desde el principio —
exactamente lo que este volumen y el Volumen 2 enseñan a hacer. Un
monolito con Bounded Contexts limpios, comunicándose por eventos internos
sin importar clases de otros módulos directamente, se extrae servicio por
servicio con un costo acotado y medible — la migración de
`BillingEventHandler` a `BillingEventConsumer` (capítulo 01) es
exactamente esa extracción, y no requirió rediseñar nada, solo cambiar el
mecanismo de entrega.

## ❌ Anti-patrones

**Migrar a microservicios por presión externa** — un artículo, una charla,
una entrevista de trabajo donde preguntaron "¿usan microservicios?" — sin
un dolor organizacional propio, medido y presente, que lo justifique. Es la
versión arquitectónica de cargo-culting mencionada en el Volumen 1,
capítulo 01: copiar la forma de una decisión ajena sin haber vivido el
problema que la originó.

## 🤖 Cómo cambió la IA este concepto

Un asistente hace que construir microservicios sea mecánicamente más
barato que nunca — lo cual, paradójicamente, hace este capítulo más
importante, no menos. Cuando generar ocho servicios con su `docker-compose`,
sus contratos de API y su infraestructura de mensajería toma una tarde en
vez de un mes, la fricción que antes obligaba a pensarlo dos veces
desaparece. La disciplina de este capítulo — exigir evidencia real del
dolor organizacional antes de dividir — es, más que nunca, el único freno
que queda entre un monolito modular saludable y un monolito distribuido
construido en un fin de semana porque era técnicamente fácil hacerlo.

## Caso real (ParcelFlow)

ParcelFlow, a lo largo de todo este libro, permanece como monolito modular
en producción. Fulfillment y Billing son módulos con límites limpios
(Volumen 2), comunicados por eventos (Volumen 2, capítulo 04), con el
diseño de extracción ya preparado y documentado (capítulo 01 de este
volumen) — pero sin extraerse, porque el mismo equipo de dos personas sigue
desplegando todo junto sin fricción real. El día que un segundo equipo se
haga cargo exclusivamente de Billing, la extracción está lista para
ejecutarse en días, no meses — esa es la ganancia real de haber modelado
los límites correctamente desde el Volumen 2, sin haber pagado un solo
costo de red mientras tanto.

## Errores que cometí (o cometería) si empezara otra vez

Sentir presión, en las primeras semanas de un proyecto nuevo, de "al menos
separar Billing en su propio servicio para no quedar mal en la próxima
entrevista técnica o due diligence de inversionistas". Ese impulso — diseñar
para una audiencia externa hipotética en vez de para el problema real del
negocio — es exactamente el anti-patrón central de este capítulo, y
reconocerlo a tiempo evitó meses de complejidad operativa que el proyecto,
en esa etapa, no tenía ningún motivo real para cargar.
