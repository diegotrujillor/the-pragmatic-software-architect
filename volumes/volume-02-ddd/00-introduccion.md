# 00. Introducción

## Nivel 1 — ELI20

Un desarrollador CRUD típico diseña así: primero la tabla SQL, después la
clase que la refleja columna por columna, después los endpoints que exponen
`create/read/update/delete` sobre esa clase. El "modelo" es, en realidad, el
esquema de la base de datos disfrazado de objeto — sin comportamiento, solo
getters y setters.

Domain-Driven Design invierte el orden: primero entiendes cómo habla el
experto del negocio — no el ingeniero, el experto — y modelas el software en
ese vocabulario. La tabla SQL es una consecuencia del modelo, no al revés.
Si el experto en logística dice "un pedido no puede despacharse si algún
paquete no tiene guía asignada", esa frase debería poder leerse casi textual
en el código, no reconstruirse a partir de tres `JOIN` y un `WHERE`.

## Nivel 2 — Ingeniería

DDD (Eric Evans, *Domain-Driven Design*, 2003 — ver
[`REFERENCES.md`](../../REFERENCES.md)) se divide en dos mitades que este
volumen cubre en orden ascendente:

- **Tactical DDD** (patrones de código, capítulos 01–02 de este volumen):
  **Entities**, **Value Objects**, **Aggregates**, **Domain Events**,
  **Repositories**, **Domain Services**. Herramientas concretas que se usan
  dentro de un modelo ya delimitado.
- **Strategic DDD** (patrones de organización, capítulo 03): **Bounded
  Context**, **Ubiquitous Language**, **Context Mapping**. Decide dónde
  termina un modelo y empieza otro, y cómo se comunican modelos distintos
  entre sí.

Este volumen empieza por lo táctico — lo que se toca en el código todos los
días — antes de subir a lo estratégico, porque los conceptos estratégicos
solo tienen sentido una vez que se ha sentido el dolor de un modelo sin
límites claros.

## Nivel 3 — Senior Engineer

Un senior rara vez aplica "DDD completo". Aplica **DDD táctico** — Value
Objects, Aggregates bien delimitados, un lenguaje ubicuo consistente en el
código — en cualquier subdominio con reglas de negocio no triviales, sin
necesidad de ceremonia estratégica completa (workshops de Context Mapping,
Event Storming de varios días). Reserva lo estratégico para el momento en
que de verdad hay más de un equipo, o más de un modelo mental del mismo
sustantivo, compitiendo por el mismo código.

## Nivel 4 — Software Architect

El argumento frente al CTO no es "el código va a ser más elegante" — es
**reducir la pérdida de traducción entre negocio y código**, que es donde
vive el costo real de mantenimiento a largo plazo. Cada vez que un experto
de negocio dice una regla y un ingeniero la traduce a un `boolean` o un
`if` disperso en tres archivos, esa traducción tiene fuga. DDD táctico hace
que la regla viva en un solo lugar, con el mismo nombre que usa el negocio —
lo cual baja el costo de cada futura conversación entre producto e
ingeniería, no solo el costo de escribir el código una vez.

El costo real que hay que reconocer: DDD exige tiempo de modelado y acceso
genuino a expertos de dominio. Sin eso, se construye una imitación de DDD
—clases con nombres de sustantivos de negocio, sin las invariantes reales
adentro— que es peor que no haberlo intentado.

## ❌ Mitos

**"DDD siempre es mejor."** No. DDD tiene sentido en subdominios con reglas
de negocio complejas y cambiantes. Aplicado a un CRUD genuino — un catálogo
de categorías de producto que casi nunca cambia de reglas — solo agrega
Value Objects y Aggregates alrededor de datos que nunca tuvieron
comportamiento real que proteger.

## ❌ Anti-patrones

**Modelo anémico disfrazado de DDD.** Clases llamadas `Order`, `Package`,
`Delivery` — nombres correctos — pero con setters públicos en cada campo y
cero validación de invariantes adentro. Toda la lógica de negocio real vive
afuera, en un `OrderService`. Tener sustantivos de negocio como nombres de
clase no es DDD; es CRUD con vocabulario prestado.

## 🤖 Cómo cambió la IA este concepto

Pedirle a un asistente "genera un Value Object `Money`" o "genera el
Aggregate `Order`" produce, hoy, código sintácticamente correcto en
segundos. Lo que ningún asistente puede inventar por ti: cuál es la regla de
negocio real que ese Value Object debe proteger, o dónde traza el experto de
dominio la frontera entre un Aggregate y otro. Esa información solo existe
en la conversación con quien conoce el negocio — la IA formaliza el modelo
que tú le describes, no lo descubre por ti.

## Caso real (ParcelFlow)

A lo largo de este volumen, el mismo dominio de los capítulos anteriores se
profundiza: `Order` se convierte en el Aggregate Root que protege la
invariante "la suma de pesos de sus `Package` no excede el límite
contratado"; una dirección de entrega deja de ser tres campos sueltos
(`street`, `city`, `zip`) y se convierte en el Value Object `Address`,
inmutable y auto-validado; y el evento `OrderFulfilled` — ya mencionado en
el Volumen 1 — se formaliza como Domain Event de primera clase.

## Errores que cometí (o cometería) si empezara otra vez

Diseñar Bounded Contexts y un mapa de contexto completo para un MVP de un
solo equipo y una sola base de datos compartida. La complejidad estratégica
de DDD se gana su lugar cuando hay fricción real entre modelos — dos equipos
que usan la palabra "cliente" para significar cosas distintas, por ejemplo —
no antes. Aplicarla antes de que esa fricción exista es el mismo error que
separar en capas desde el commit inicial: pagar el costo de una frontera
que todavía no protege nada.
