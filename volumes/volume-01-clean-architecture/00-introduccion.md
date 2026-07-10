# Introducción

Este libro no enseña a programar. Enseña a **decidir**.

Si ya sabes escribir un CRUD, ya sabes programar. Lo que falta — lo que
ningún bootcamp ni curso de "patrones de diseño" te da — es el criterio para
responder preguntas como: ¿esto necesita una capa de abstracción o es
sobreingeniería? ¿Este servicio debería ser un microservicio o una carpeta
dentro del monolito? ¿DDD aporta algo aquí o es ceremonia?

Ese criterio es lo que este libro intenta construir, volumen a volumen,
usando un solo sistema ficticio como hilo conductor: **ParcelFlow**, una
plataforma de seguimiento de pedidos y envíos. Nada de ejemplos que cambian
de nombre cada dos páginas — cada capítulo aterriza la idea en las mismas
cinco entidades: `Merchant`, `Warehouse`, `Order`, `Package`, `Delivery`. Si
el ejemplo no sirve para decidir algo real en ese dominio, no entra al
libro.

## Cómo leer cada capítulo

Cada capítulo tiene cuatro niveles de profundidad. No son dificultad
creciente porcentual — son **cuatro lentes distintos** sobre la misma idea:

- **Nivel 1 — ELI20**: la explicación que le darías a un ingeniero que viene
  de hacer CRUDs y nunca ha oído el término.
- **Nivel 2 — Ingeniería**: arquitectura, ventajas, desventajas, ejemplos de
  código.
- **Nivel 3 — Senior Engineer**: cómo decide alguien con diez años de
  experiencia — cuándo aplica el patrón y cuándo lo descarta.
- **Nivel 4 — Software Architect**: cómo se justifica la decisión frente al
  CTO, en términos de costo, riesgo y tiempo.

Además, cada capítulo trae secciones que ningún libro de arquitectura
incluye:

- **❌ Mitos** — afirmaciones populares que no resisten un caso real.
- **❌ Anti-patrones** — las formas concretas en que la idea se aplica mal.
- **🤖 Cómo cambió la IA este concepto** — qué dejó de ser valioso memorizar
  y qué se volvió más importante decidir, ahora que un asistente como Claude
  Code escribe el boilerplate en segundos.
- **Caso real (ParcelFlow)** — la idea aplicada al dominio del libro.
- **Errores que cometí (o cometería) si empezara otra vez** — honestidad
  sobre sobre-ingeniería, no solo sobre lo que salió bien.

## Por qué Clean Architecture es el Volumen 1

Porque es la decisión más barata de tomar temprano y la más cara de
corregir tarde. Un sistema con las dependencias apuntando en la dirección
correcta se puede migrar de base de datos, envolver en microservicios o
cubrir de tests sin reescribirlo. Uno con la dirección invertida — lógica de
negocio importando el ORM, casos de uso llamando directamente al SDK de un
proveedor cloud — arrastra ese error a cada decisión posterior.

Este volumen no es un resumen de *Clean Architecture* (Robert C. Martin,
Pearson) — ver [`REFERENCES.md`](../../REFERENCES.md). Es esa idea, filtrada
por diez años de verla funcionar y fallar en proyectos reales, y aterrizada
en ParcelFlow, un stack Node/Express/PG típico, con restricciones realistas
de presupuesto reducido y de independencia de proveedor — las mismas
presiones que enfrenta cualquier equipo pequeño en producción.

## ❌ Mitos

**"Clean Architecture significa carpetas `domain/`, `application/`,
`infrastructure/` desde el día uno."** No. La estructura de carpetas es el
síntoma más visible y el menos importante. La regla real es la dirección de
las dependencias en el código fuente, no los nombres de las carpetas. Puedes
violar la Dependency Rule con una carpeta `domain/` perfecta, y puedes
respetarla en un solo archivo.

## ❌ Anti-patrones

Empezar un proyecto de cero copiando la estructura de capas de un tutorial,
sin tener todavía un caso de uso real que la necesite. El resultado más
común: cuatro capas y una interfaz por cada clase, para una app que hace tres
`SELECT` y un `INSERT`.

## 🤖 Cómo cambió la IA este concepto

Antes, la barrera de entrada era mecánica: escribir la interfaz, la
implementación, el mapeo, la inyección de dependencias — mucho tipeo para
llegar al principio. Hoy, Claude Code genera esa mecánica en segundos.

Eso no vuelve la Dependency Rule irrelevante — la vuelve **más importante**,
no menos. Cuando generar diez interfaces cuesta un prompt, la tentación de
generarlas sin necesitarlas es mayor, no menor. El trabajo humano se movió de
"escribir la abstracción" a "decidir si esta abstracción se gana su lugar".
Ese es exactamente el criterio que este libro quiere construir.

## Caso real (ParcelFlow)

El backend de ParcelFlow expone casos de uso como "registrar un pedido" o
"generar guía de envío de un paquete". La regla de Clean Architecture
aplicada ahí, en su forma más simple: el caso de uso no sabe que Postgres
existe. No importa `pg`, no arma `SQL`, no conoce el nombre de la tabla.
Recibe un repositorio abstracto, hace la operación de negocio, y delega la
persistencia. Eso es lo que permite que "migrar de proveedor con costo
acotado" sea real y no aspiracional.

## Errores que cometí (o cometería) si empezara otra vez

Separar en capas (`domain/`, `application/`, `infrastructure/`) desde el
primer commit, para un MVP de un solo desarrollador. La estructura por capas
se gana su lugar cuando hay más de un caso de uso compitiendo por la misma
lógica de negocio — no antes. Empezar así solo agrega navegación entre
carpetas sin ningún beneficio todavía real.
