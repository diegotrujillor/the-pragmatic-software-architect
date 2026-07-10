# The Pragmatic Software Architect

**From CRUD Developer to Software Architect in the AI Era (2026 Edition)**

## Esto no es otro catálogo de patrones

Existen decenas de libros y sitios que explican qué es un Factory, qué es
un Aggregate, o qué es un Bounded Context. Este no es uno más de esos.

Hoy, pedirle a un asistente de IA que escriba un Factory Method, un
Aggregate completo o los cuatro anillos de Clean Architecture toma
segundos. La mecánica de escribir el patrón dejó de ser el problema. **El
problema real es saber cuándo un patrón se gana su lugar y cuándo es
sobreingeniería disfrazada de buena práctica.**

Este libro no enseña a programar — enseña a **decidir**:

- ¿Esto necesita una capa de abstracción, o es indirección sin comprador?
- ¿Este servicio debería ser un microservicio, o una carpeta dentro del
  monolito?
- ¿DDD aporta algo aquí, o es ceremonia porque "así se hace"?
- ¿Este `if/else` de dos ramas necesita convertirse en un Strategy, o
  puede esperar a la segunda variante real?

Ninguna de esas preguntas tiene una respuesta que un asistente de IA pueda
darte solo — porque la respuesta depende de tu dominio, tu equipo y tu
etapa de producto, no de la sintaxis del patrón. Ese es el criterio que
este libro construye, capítulo a capítulo.

## Cuatro niveles, la misma idea

Cada capítulo cubre el mismo concepto desde cuatro ángulos distintos —no
son dificultad creciente, son cuatro lentes:

| Nivel | Nombre | Pregunta que responde |
|---|---|---|
| 1 | ELI20 | "Explícamelo como si fuera un ingeniero que viene de CRUDs." |
| 2 | Ingeniería | Arquitectura, ventajas, desventajas, ejemplos de código. |
| 3 | Senior Engineer | ¿Cómo decidiría alguien con diez años de experiencia? |
| 4 | Software Architect | ¿Cómo se justifica frente al CTO? |

## Secciones que ningún libro de arquitectura trae

- **❌ Mitos** — creencias comunes que no resisten un caso real (ej. "DDD
  siempre es mejor" — no lo es).
- **❌ Anti-patrones** — las formas concretas en que cada idea se aplica
  mal, incluyendo la cadena `Repository → Service → Manager → Helper →
  Util → Factory → Builder → Facade → Controller`, que aparece en
  proyectos reales constantemente.
- **🤖 Cómo cambió la IA este concepto** — qué dejó de ser valioso
  memorizar (la sintaxis del patrón) y qué se volvió más importante que
  nunca decidir (si ese patrón se lo gana).
- **Caso real** — todo aterrizado en un mismo sistema ficticio de ejemplo
  a lo largo de las cinco volúmenes (`Merchant → Warehouse → Order →
  Package → Delivery`) — nunca bancos ni tiendas online genéricas que
  cambian de nombre cada capítulo.
- **Errores que cometí (o cometería) si empezara otra vez** — honestidad
  sobre sobreingeniería propia: microservicios prematuros, interfaces sin
  segunda implementación, capas separadas desde el primer commit.

## Los cinco volúmenes

1. **[Clean Architecture](volume-01-clean-architecture/00-introduccion.md)**
   — la Dependency Rule, y por qué es la decisión más barata de tomar
   temprano y la más cara de corregir tarde.
2. **[Domain-Driven Design](volume-02-ddd/00-introduccion.md)** —
   Entities, Value Objects, Aggregates, Bounded Contexts, Domain Events.
3. **[Design Patterns](volume-03-design-patterns/00-introduccion.md)** —
   Strategy, Factory, Decorator, Adapter, Facade, Command, Template
   Method — y la cadena de sinónimos que los reemplaza de mala manera.
4. **[Testing](volume-04-testing/00-introduccion.md)** — por qué la
   cobertura mide cantidad, no protección real contra bugs.
5. **[Microservices](volume-05-microservices/00-introduccion.md)** —
   boundaries de servicio, trade-offs reales, y sobre todo, cuándo NO
   usarlos.

Cada afirmación importante está respaldada por referencias — Robert C.
Martin, Eric Evans, Freeman & Robson, Vladimir Khorikov, Sam Newman — ver
el [mapa completo de referencias](https://github.com/diegotrujillor/the-pragmatic-software-architect/blob/main/REFERENCES.md).

Código fuente y contexto completo del proyecto en
[GitHub](https://github.com/diegotrujillor/the-pragmatic-software-architect).
