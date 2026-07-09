# 04. La cadena de sinónimos

## Nivel 1 — ELI20

Repository → Service → Manager → Helper → Util → Factory → Builder →
Facade → Controller. Esta cadena existe, casi idéntica, en una cantidad
enorme de proyectos reales. Cada capa tiene un nombre distinto y suena a un
patrón legítimo — pero cuando se les pregunta "¿qué problema específico
resuelve esta capa que la anterior no resolvía ya?", la respuesta casi
siempre es silencio. Este capítulo es sobre cómo reconocer esa cadena antes
de escribirla, no después de mantenerla dos años.

## Nivel 2 — Ingeniería

La cadena no aparece por mala intención — aparece por una secuencia de
decisiones locales, cada una razonable en el momento:

1. `OrderRepository` — legítimo, encapsula acceso a datos (Volumen 1).
2. Alguien necesita lógica que no es puro acceso a datos → nace
   `OrderService`. Razonable, si de verdad orquesta un caso de uso.
3. El `OrderService` crece → alguien separa "las partes complicadas" en
   `OrderManager`. ¿Qué distingue a un Manager de un Service? Casi nunca hay
   una respuesta clara — ambos terminan siendo "donde va la lógica que no
   sé dónde poner".
4. Código compartido entre `Service` y `Manager` → `OrderHelper`.
5. Código que ni siquiera es específico de `Order` → `OrderUtil` (a veces ni
   siquiera eso: un `Util` genérico para todo el proyecto).
6. Alguien necesita construir un `Order` complejo → `OrderFactory`, después
   `OrderBuilder` (¿en qué se diferencian, en este proyecto? con frecuencia,
   en nada).
7. Alguien quiere "simplificar la interfaz" de las seis capas anteriores →
   `OrderFacade`, que termina llamando a las seis.
8. Y el `OrderController`, que técnicamente es el único que tenía una razón
   real de existir desde el principio — el punto de entrada HTTP.

El costo compuesto: para seguir una sola regla de negocio ("¿por qué se
rechazó este pedido?"), un ingeniero nuevo tiene que saltar por seis a ocho
archivos, sin ninguna garantía de en cuál realmente vive la lógica que
busca.

## Nivel 3 — Senior Engineer

Un senior aplica una prueba mecánica antes de crear cualquier capa nueva
con un nombre genérico (`Service`, `Manager`, `Helper`, `Util`):
**¿puedo nombrar esta clase con un verbo específico del negocio en vez de
una palabra genérica?** `CancelOrderUseCase` sí puede nombrarse así — es un
verbo de negocio. `OrderManager` no puede — "manejar" no es una acción de
negocio, es una forma de evitar decidir qué hace realmente la clase.

Otra señal práctica: si dos capas consecutivas de la cadena podrían
fusionarse sin que ningún test se rompa y sin que ninguna responsabilidad
quede confusa, es señal de que nunca fueron dos responsabilidades — eran
una sola con dos nombres.

## Nivel 4 — Software Architect

Frente al CTO, el costo de la cadena de sinónimos no es estético — es
**velocidad de onboarding y costo de cada incidente en producción**. Cada
capa genérica adicional es una parada más que un ingeniero de guardia tiene
que atravesar a las 3 a.m. para encontrar dónde falló una regla de negocio.
Ese tiempo, multiplicado por cada incidente durante la vida del proyecto,
es el costo real que la cadena nunca muestra en ningún sprint — se paga
lentamente, uno a la vez, y por eso nadie lo relaciona con la decisión de
arquitectura original.

El argumento correcto no es "eliminemos capas por principio" — es que cada
capa debe poder justificarse con una frase de una línea, con verbo de
negocio, delante del equipo. Las que no pasan esa prueba son la cadena.

## ❌ Mitos

**"Tener más capas siempre es más profesional / más enterprise."** No. Una
arquitectura madura tiene exactamente las capas que resuelven problemas
reales del sistema — ni una más. Sistemas con menos capas, bien nombradas y
con responsabilidad clara, son más fáciles de operar en producción que
sistemas con ocho capas genéricas cuyo propósito nadie en el equipo actual
puede explicar sin adivinar.

## ❌ Anti-patrones

Este capítulo entero es, en esencia, la documentación de un solo
anti-patrón — pero vale nombrar su variante más traicionera: la cadena
**parcial**, donde solo existen tres o cuatro de las ocho capas, lo cual la
hace parecer razonable a simple vista. El síntoma es el mismo: nombres
genéricos (`Service`, `Helper`, `Manager`) sin una regla clara de qué va en
cada uno, que crecen sin límite porque no hay ninguna razón de diseño que
diga "esto no va aquí".

## 🤖 Cómo cambió la IA este concepto

Este es, de todos los anti-patrones del libro, el que la IA generativa
empeora más rápido si no se le pone freno. Pedirle a un asistente "agrega
una capa Service sobre este Repository" o "crea un Manager para esto" genera
la capa perfectamente escrita en segundos — sin que el asistente tenga
ninguna razón para cuestionar si esa capa se gana su lugar. La disciplina de
nombrar con verbo de negocio, y de preguntar "¿qué problema resuelve esto
que la capa anterior no resolvía?", tiene que venir de quien pide el
código, no de quien lo escribe. La IA acelera tanto el patrón bueno como la
cadena mala — el criterio de cuál estás construyendo sigue siendo
enteramente tuyo.

## Caso real (ParcelFlow)

ParcelFlow no tiene `OrderManager` ni `OrderHelper` ni `OrderUtil`. Tiene
`OrderRepository` (acceso a datos, Volumen 1), un Interactor por verbo de
negocio (`RegisterOrderUseCase`, `MarkPackageDeliveredUseCase`,
`CancelOrderUseCase` — Volumen 1, capítulo 03), y `OrderCheckoutFacade`
(capítulo 02 de este volumen) — que sí se justifica, porque orquesta cuatro
subsistemas distintos con una responsabilidad de negocio nombrable
("completar el checkout"). Cada capa que existe puede explicarse en una
frase con verbo de negocio; ninguna existe "por si acaso" o "para no
ensuciar el Repository".

## Errores que cometí (o cometería) si empezara otra vez

Crear `OrderHelper` para "funciones que varios lugares necesitan sobre
Order" sin definir de antemano qué tipo de función iba ahí. Seis meses
después tenía cálculo de impuestos, formateo de fechas para reportes, y
lógica de validación de negocio, las tres sin relación entre sí, en el mismo
archivo — porque `Helper` no tiene ninguna regla de pertenencia, así que
todo cabe. Nombrar cada función nueva con su propio módulo específico
(`tax-calculator.ts`, `date-formatting.ts`) desde el principio habría
costado lo mismo y evitado el archivo fantasma que terminó siendo `Helper`.
