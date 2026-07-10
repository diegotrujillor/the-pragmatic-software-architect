# 00. Introducción

## Nivel 1 — ELI20

Un desarrollador CRUD suele pensar los microservicios como "la forma
correcta de construir a escala" — algo que se adopta desde el día uno
porque "así lo hacen las empresas grandes". Sam Newman (*Building
Microservices*, 2ª ed., 2021 — ver [`REFERENCES.md`](../../REFERENCES.md))
plantea la pregunta que este volumen adopta como propia: un microservicio
no es una unidad de código, es una **unidad de despliegue independiente con
su propio ciclo de release** — y esa independencia tiene un costo real de
coordinación, observabilidad y consistencia de datos que un monolito bien
diseñado simplemente no paga. Este volumen no enseña a construir
microservicios primero — enseña a reconocer cuándo un monolito ya duele lo
suficiente como para justificarlo.

## Nivel 2 — Ingeniería

Los microservicios resuelven un problema específico: **equipos que ya no
pueden desplegar de forma independiente** porque su código comparte un
proceso, un deploy y un ciclo de release con el código de otros equipos.
Ese problema es organizacional antes que técnico — la Ley de Conway,
mencionada ya en el Volumen 2 (Bounded Contexts), es la razón real por la
que "extraer un servicio" casi siempre funciona mejor cuando ya existe un
Bounded Context bien delimitado detrás.

Este volumen cubre: **boundaries de servicio** (por qué el límite correcto
es casi siempre el mismo que el Bounded Context, no una división arbitraria
por capa técnica), los **trade-offs reales** (consistencia eventual en vez
de transaccional, latencia de red donde antes había una llamada de función,
observabilidad distribuida donde antes había un stack trace), y, con el
mismo peso que el resto, **cuándo NO usarlos**.

## Nivel 3 — Senior Engineer

Un senior no pregunta "¿deberíamos migrar a microservicios?" en abstracto —
pregunta **"¿qué dolor específico y medible tenemos hoy que un monolito
bien modularizado no puede resolver?"**. Los dolores legítimos son
concretos: un equipo no puede desplegar sin coordinar con otros tres, una
parte del sistema necesita escalar independientemente por carga
desproporcionada, o dos subdominios tienen requisitos de disponibilidad tan
distintos que compartir el mismo proceso arriesga a uno por el otro. Sin
uno de esos dolores presentes, la respuesta por defecto es un monolito
modular — con los Bounded Contexts del Volumen 2 ya trazados como límites
internos claros, listos para extraerse el día que el dolor aparezca.

## Nivel 4 — Software Architect

El argumento frente al CTO no es "microservicios escalan mejor" — es un
balance de costos explícito: microservicios cambian complejidad de
*dentro del código* (fácil de depurar con un debugger local) a *entre
servicios en producción* (llamadas de red que fallan parcialmente,
consistencia eventual, necesidad real de tracing distribuido, versionado de
contratos entre equipos). Ese balance solo es rentable cuando el costo
organizacional de *no* separar servicios — releases bloqueados, equipos
pisándose el código, un solo servicio cayéndose y arrastrando todo el
sistema — ya supera el costo operativo nuevo que la separación introduce.

## ❌ Mitos

**"Microservicios son necesarios para escalar."** No necesariamente. Un
monolito bien modularizado, con procesos replicados horizontalmente detrás
de un balanceador de carga, escala en tráfico igual de bien que muchos
sistemas de microservicios mal diseñados — y sin la complejidad operativa
adicional. El problema que los microservicios resuelven primero es
organizacional (equipos, releases independientes), no de capacidad de
cómputo.

## ❌ Anti-patrones

**Microservicios prematuros.** Dividir un MVP de un solo equipo y unos
pocos usuarios en ocho servicios desde el primer sprint, antes de que exista
un solo Bounded Context probado en producción. El resultado típico: la
misma cantidad de complejidad de negocio, ahora repartida en ocho
despliegues, ocho bases de código, y llamadas de red allí donde antes había
una llamada de función — sin ningún equipo real que se beneficie de
desplegar independientemente, porque sigue siendo la misma persona o el
mismo equipo tocando los ocho.

## 🤖 Cómo cambió la IA este concepto

Un asistente puede generar el andamiaje de ocho microservicios — Docker
Compose, contratos de API, clientes HTTP entre ellos — en una fracción del
tiempo que tomaba hace unos años. Eso reduce el costo mecánico de dividir,
pero no reduce el costo real que Newman describe: consistencia eventual,
observabilidad distribuida, versionado de contratos, y sobre todo, la
pregunta de si existe un dolor organizacional real detrás. La IA hace más
fácil construir la arquitectura equivocada más rápido — la responsabilidad
de decidir si esa arquitectura se necesita sigue siendo, más que nunca,
enteramente humana.

## Caso real (ParcelFlow)

ParcelFlow opera como monolito modular, con los Bounded Contexts
Fulfillment y Billing (Volumen 2) ya como módulos separados dentro del
mismo proceso, comunicándose por Domain Events internos. Este volumen
explora la pregunta hipotética: el día que Billing necesite su propio ciclo
de release porque un segundo equipo se hace cargo de él exclusivamente —
¿qué cambia, y qué no cambia, al extraerlo a su propio servicio?

## Errores que cometí (o cometería) si empezara otra vez

Diseñar la arquitectura de un MVP de un solo desarrollador como seis
microservicios desde el día uno, "para no tener que migrar después". El
resultado: seis `docker-compose` que levantar en desarrollo local, seis
pipelines de CI, y llamadas HTTP internas entre servicios que un monolito
modular habría resuelto con una llamada de función — para un sistema que,
durante su primer año entero, tuvo exactamente un equipo de una persona
tocando los seis servicios.
