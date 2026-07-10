# Introducción

## Nivel 1 — ELI20

Los patrones de diseño (GoF: Gamma, Helm, Johnson, Vlissides, 1994) son
soluciones con nombre a problemas que se repiten. El problema real de este
volumen no es enseñarte qué es un Factory o un Strategy — eso lo genera
Claude Code en segundos, como se repite capítulo a capítulo en este libro.
El problema real es que la mayoría de proyectos con "muchos patrones" no
tienen más patrones porque los necesiten — tienen más patrones porque
alguien los aplicó sin que hubiera un problema real detrás. Este volumen
trata tanto de cuándo usar cada patrón como, con el mismo peso, de cuándo
*no*.

## Nivel 2 — Ingeniería

Los 23 patrones originales del GoF se agrupan en tres familias:

- **Creacionales** — cómo se construyen objetos sin acoplar el código al
  proceso exacto de construcción (Factory Method, Abstract Factory,
  Builder, Singleton, Prototype).
- **Estructurales** — cómo se componen objetos y clases en estructuras más
  grandes sin que cada pieza conozca los detalles internos de las demás
  (Adapter, Decorator, Facade, Composite, Proxy, Bridge, Flyweight).
- **De comportamiento** — cómo se distribuye responsabilidad y comunicación
  entre objetos (Strategy, Observer, Command, State, Template Method,
  Chain of Responsibility, Iterator, Mediator, Memento, Visitor,
  Interpreter).

Este volumen no cubre los 23 en profundidad — cubre los que aparecen con
frecuencia real en un backend como el de ParcelFlow (Strategy, Factory,
Observer/Domain Events ya visto en el Volumen 2, Decorator, Adapter), y
dedica tanto espacio a la sección de Anti-patrones como a la de patrones,
porque el anti-patrón — la cadena Repository → Service → Manager → Helper →
Util → Factory → Builder → Facade → Controller sin que ninguna capa
resuelva un problema real — es más común en código de producción que
cualquier patrón bien aplicado.

## Nivel 3 — Senior Engineer

Un senior nunca empieza por el patrón. Empieza por el problema, y solo
después reconoce que un patrón ya conocido lo resuelve — la secuencia
inversa (empezar por "voy a aplicar Strategy aquí") casi siempre produce una
abstracción buscando un problema, no al revés. La pregunta de una línea
para cualquier patrón candidato: *¿qué cambia con el tiempo en este punto
del código, y ese cambio justifica una indirección hoy, o puedo esperar a
la segunda vez que aparezca antes de abstraer?*

## Nivel 4 — Software Architect

El argumento frente al CTO no es "usamos patrones de diseño" — un patrón
mal aplicado es deuda técnica con nombre elegante. El argumento real es:
cada patrón tiene un costo medible (una interfaz más, una capa más de
indirección, una curva de entrada más alta para el próximo ingeniero) y un
beneficio medible (qué tan barato es el próximo cambio esperado). Un
arquitecto justifica el patrón mostrando el cambio futuro concreto que se
vuelve barato — no citando el nombre del patrón como si fuera autoridad
suficiente.

## ❌ Mitos

**"Más patrones de diseño significa mejor arquitectura."** No. El número de
patrones aplicados no correlaciona con calidad — a menudo correlaciona con
lo contrario. Un módulo con la cantidad justa de indirección, ni una capa
de más, suele tener menos patrones nombrados que uno sobrediseñado, no más.

## ❌ Anti-patrones

**La cadena de sinónimos.** `OrderRepository` → `OrderService` →
`OrderManager` → `OrderHelper` → `OrderUtil` → `OrderFactory` →
`OrderBuilder` → `OrderFacade` → `OrderController`, todos en el mismo
proyecto, ninguno resolviendo un problema que el anterior no resolvía ya.
Pasa constantemente porque cada capa nueva se siente "segura" — es fácil
agregar una capa, es incómodo cuestionar si la anterior ya bastaba.

## 🤖 Cómo cambió la IA este concepto

Antes, memorizar la estructura de un Factory o un Strategy era parte del
trabajo — abrir el libro, copiar la forma, adaptarla. Hoy Claude Code
escribe cualquiera de los 23 patrones GoF correctamente estructurado en
segundos, con solo pedirlo. Eso vuelve irrelevante memorizar la sintaxis del
patrón y vuelve crítico saber cuál — si alguno — se necesita. La habilidad
que este volumen construye no es "saber escribir un Decorator": es
reconocer el problema que un Decorator resuelve, y descartar los otros 22
patrones que no aplican ahí.

## Caso real (ParcelFlow)

A lo largo de este volumen: `Strategy` para elegir transportista de envío
según reglas de negocio variables (peso, destino, urgencia) sin un árbol de
`if/else` creciente; `Factory Method` para construir el `Package` correcto
según el tipo de mercancía declarado; `Decorator` para componer reglas de
descuento sobre un `Order` sin una clase por cada combinación posible.

## Errores que cometí (o cometería) si empezara otra vez

Aplicar `Strategy` para una regla de negocio que, en la práctica, nunca tuvo
una segunda variante en dos años de producción — un `if/else` de dos ramas
que se volvió una interfaz, dos clases y un Factory para instanciarlas.
Esperar a la segunda variante real antes de introducir el patrón habría
costado cero en velocidad de entrega y ahorrado un archivo entero de
indirección que nadie necesitó.
