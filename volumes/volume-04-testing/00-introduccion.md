# 00. Introducción

## Nivel 1 — ELI20

Un desarrollador CRUD mide sus tests por cobertura: "tengo 90% de cobertura,
mi código está probado." Vladimir Khorikov (*Unit Testing: Principles,
Practices, and Patterns*, 2020 — ver [`REFERENCES.md`](../../REFERENCES.md))
plantea la pregunta que este volumen adopta como propia: cobertura mide
*cuánto* código se ejecuta durante los tests, no si esos tests **detectarían
un bug real** si alguien introdujera uno. Se puede tener 100% de cobertura
con tests que no fallarían nunca, pase lo que pase con la lógica de negocio
— y 60% de cobertura con una red de seguridad genuinamente sólida sobre las
partes que importan.

## Nivel 2 — Ingeniería

Este volumen organiza los tests por lo que Khorikov llama las **cuatro
propiedades de un test valioso** — cualquier test que falle en una de
ellas es, en algún grado, un test que cuesta más de lo que aporta:

1. **Protección contra regresiones** — ¿el test realmente falla si se
   rompe la lógica de negocio?
2. **Resistencia a refactors** — ¿el test sigue pasando si cambia la
   implementación interna pero el comportamiento observable no cambió?
   (Este es el criterio que más tests mal escritos violan.)
3. **Feedback rápido** — ¿cuánto tarda en decirme que algo se rompió?
4. **Mantenibilidad** — ¿qué tan fácil es leer y modificar el test cuando
   el requisito cambia?

Sobre esa base, el volumen cubre: la pirámide de tests (unitarios,
integración, end-to-end — y por qué la proporción importa más que el
volumen absoluto), mocks vs. stubs (y por qué mockear todo rompe la
propiedad #2), y el patrón AAA (Arrange-Act-Assert) como estructura mínima
legible.

## Nivel 3 — Senior Engineer

Un senior no pregunta "¿está esto testeado?" — pregunta **"¿este test
sobreviviría un refactor legítimo que no cambia el comportamiento?"**. Un
test que verifica que se llamó a un método interno específico, en vez de
verificar el resultado observable, falla ante cualquier refactor —
entrenando al equipo, con el tiempo, a tratar los tests como un obstáculo
en vez de una red de seguridad. Esa es la causa raíz más común de
"deshabilitamos ese test, molestaba" en proyectos reales.

## Nivel 4 — Software Architect

El argumento frente al CTO no es "tenemos 80% de cobertura" — es **el
tiempo que un test suite tarda en darle confianza al equipo para desplegar
sin miedo**. Una suite lenta, frágil ante refactors, o llena de falsos
positivos no acelera al equipo — lo entrena a ignorarla, que es
funcionalmente idéntico a no tenerla, con el costo adicional del tiempo
gastado en mantenerla. El ROI real de testing se mide en velocidad de
release sostenida, no en un porcentaje de cobertura en un dashboard.

## ❌ Mitos

**"Más cobertura siempre es mejor."** No. Cobertura es una métrica de
cantidad, no de calidad de la protección. Perseguir 100% de cobertura lleva,
con frecuencia, a testear getters triviales y código sin lógica de negocio
real — inflando el número sin agregar ninguna red de seguridad genuina.

## ❌ Anti-patrones

**Mockear todo por defecto.** Un test que mockea cada colaborador de la
clase bajo prueba, incluyendo objetos de valor simples sin comportamiento
externo, termina verificando que el código llama a los métodos que el
propio test le dijo que llamara — no que el comportamiento de negocio sea
correcto. Pasa siempre, incluso si la lógica real está rota, porque nunca
ejecutó la lógica real.

## 🤖 Cómo cambió la IA este concepto

Un asistente puede generar un test para casi cualquier función en segundos
— y ahí está el riesgo nuevo: genera tests que *pasan*, no necesariamente
tests que *protegen*. Es trivial pedir "genera un test para esta función" y
recibir un test que solo verifica que la función no lance una excepción,
sin verificar ningún caso límite de negocio real. La habilidad que este
volumen construye no es "escribir un test" — es reconocer, leyendo un test
generado, si de verdad cumple las cuatro propiedades o solo tiene la forma
de un test.

## Caso real (ParcelFlow)

A lo largo de este volumen: tests unitarios sobre las reglas de negocio de
`Order.addPackage()` (Volumen 2) sin tocar Postgres, usando el
`InMemoryOrderRepository` ya mencionado en el Volumen 1; tests de
integración sobre `PgOrderRepository` contra una base de datos real
levantada en Docker; y la pregunta de por qué `CarrierSelector` (Volumen 3)
se testea distinto según si sus `CarrierStrategy` tienen lógica propia o
son simples mapeos de datos.

## Errores que cometí (o cometería) si empezara otra vez

Escribir tests que verifican la implementación interna de un caso de uso
— "que se llamó a `repository.save()` exactamente una vez, con estos
argumentos exactos" — en vez de verificar el resultado observable. El
primer refactor legítimo (cambiar de un solo `save()` a un `saveMany()` por
performance, sin cambiar el comportamiento de negocio) rompió quince tests
que no protegían nada real, y el equipo aprendió a ignorarlos en vez de
arreglarlos.
