# 02. Dependency Rule

## Nivel 1 — ELI20

Dibuja círculos concéntricos. En el centro, las reglas de negocio puras. En
el borde exterior, todo lo que puede cambiar por razones ajenas al negocio:
el framework web, la base de datos, el proveedor de email. La única regla
que importa: **una flecha de dependencia en el código fuente solo puede
apuntar hacia adentro**. Un círculo interior jamás puede importar, requerir
ni conocer nada de un círculo exterior.

Si tu caso de uso "registrar pedido" tiene un `import` de Express en algún
lado, esa flecha apunta hacia afuera. Ya rompiste la regla, sin importar qué
tan bonita esté la carpeta `domain/`.

## Nivel 2 — Ingeniería

El diagrama original de Robert C. Martin tiene cuatro anillos:

1. **Entities** — reglas de negocio empresariales, las más estables. No
   saben que existe una base de datos ni una API.
2. **Use Cases** — reglas de negocio de la aplicación, orquestan entidades
   para cumplir una acción concreta ("registrar pedido", "cancelar envío").
3. **Interface Adapters** — controladores, presentadores, gateways. Traducen
   entre el formato que le conviene al caso de uso y el formato que le
   conviene al mundo exterior (HTTP, SQL, JSON).
4. **Frameworks & Drivers** — Express, el driver de Postgres, el SDK del
   proveedor de email. El detalle más volátil y más reemplazable.

El número de anillos es una guía, no una ley. Lo único mecánicamente
obligatorio es la dirección de la flecha.

**¿Cómo cruza información una frontera si el anillo interior no puede
importar el exterior?** Con **Dependency Inversion**: el anillo interior
define una interfaz que necesita, y el anillo exterior la implementa.

```typescript
// use-cases/register-order.ts  (anillo interior)
interface OrderRepository {
  save(order: Order): Promise<void>;
}

class RegisterOrderUseCase {
  constructor(private readonly orders: OrderRepository) {}

  async execute(input: RegisterOrderInput): Promise<void> {
    const order = Order.create(input); // valida reglas de negocio
    await this.orders.save(order);
  }
}

// infrastructure/pg-order-repository.ts  (anillo exterior)
class PgOrderRepository implements OrderRepository {
  constructor(private readonly db: Pool) {}

  async save(order: Order): Promise<void> {
    await this.db.query(
      "INSERT INTO orders (id, merchant_id, total) VALUES ($1, $2, $3)",
      [order.id, order.merchantId, order.total]
    );
  }
}
```

`RegisterOrderUseCase` nunca importa `pg`. La flecha de dependencia de
`PgOrderRepository` apunta hacia `OrderRepository`, que vive en el anillo
interior. La implementación concreta queda afuera; el contrato, adentro.

Un segundo detalle que se pasa por alto tan seguido como la regla misma:
**qué cruza la frontera**. Deben ser estructuras simples (DTOs, objetos
planos) — nunca un modelo de ORM, nunca el `Request` de Express. Si un
controlador le pasa un objeto `Sequelize.Model` al caso de uso, o el caso de
uso retorna algo que el controlador serializa campo por campo asumiendo el
esquema de una tabla, la frontera ya es una ficción.

## Nivel 3 — Senior Engineer

Un senior no empieza preguntando "¿cuatro carpetas o tres?". Empieza
preguntando **qué frontera es real** en este sistema, hoy:

- ¿Hay una dependencia externa con probabilidad real de cambiar (proveedor
  de base de datos, pasarela de pagos, servicio de notificaciones)? Esa
  frontera se gana una interfaz.
- ¿Hay una función interna, sin volatilidad real, sin necesidad de mock en
  tests? Esa no necesita una interfaz — invertir su dependencia agrega
  indirección sin comprador.

Colapsar anillos es válido y común: en un proyecto chico, "Interface
Adapters" y "Frameworks & Drivers" pueden vivir en la misma carpeta
`infrastructure/`, siempre que la dirección de la flecha se respete. Lo que
un senior no hace es tratar la cantidad de anillos como el objetivo — el
objetivo es la dirección de las flechas donde realmente importa.

## Nivel 4 — Software Architect

El argumento frente al CTO no es "aislamos las capas" — es **paralelismo de
equipo y reversibilidad de decisiones**:

- **Trabajo en paralelo real.** Con `OrderRepository` como contrato, un
  equipo puede construir `RegisterOrderUseCase` contra una implementación en
  memoria mientras otro equipo — o el mismo, después — construye
  `PgOrderRepository`. No hay bloqueo mutuo esperando que la infraestructura
  esté lista.
- **Migraciones incrementales (strangler fig).** Cambiar de proveedor de
  base de datos, o migrar un caso de uso a un microservicio, se reduce a
  escribir una nueva implementación de la interfaz y cambiar el punto de
  composición — no a reescribir la lógica de negocio.
- **El costo de la interfaz es fijo y pequeño; el costo de no tenerla crece
  con cada caso de uso nuevo acoplado a la dependencia externa.** Ese es el
  argumento financiero: pagar la indirección una vez, temprano, contra
  pagarla multiplicada por N casos de uso, tarde.

## ❌ Mitos

**"Cada anillo debe ser su propio paquete o proyecto separado."** No. La
Dependency Rule es sobre la dirección de las dependencias en el código
fuente, no sobre límites físicos de despliegue. Un monolito con cuatro
carpetas dentro de un mismo proceso puede respetar la regla perfectamente;
cuatro microservicios pueden violarla si el "de negocio" importa el SDK del
"de infraestructura".

## ❌ Anti-patrones

Crear una interfaz que nunca tendrá una segunda implementación real, "por si
acaso" — ceremonia sin comprador. Distinto es crear la interfaz porque hoy
mismo hay dos implementaciones: la real (`PgOrderRepository`) y una en
memoria para tests. Esa sí paga su costo desde el primer día.

Otro común: la interfaz existe, pero está moldeada por la tabla SQL en vez de
por lo que el caso de uso necesita — un `OrderRepository` genérico con
`findAll()`, `findById()`, `update()`, cuando el caso de uso solo necesita
`save()` y `findPendingOlderThan(date)`. Eso es una fuga de la capa de
persistencia disfrazada de abstracción.

## 🤖 Cómo cambió la IA este concepto

Generar `OrderRepository` + `PgOrderRepository` + el wiring de inyección de
dependencias es, hoy, un prompt de treinta segundos. Lo que un asistente no
sabe sin que se lo digas: si esa frontera específica es real en tu sistema, o
si estás pagando indirección sobre una dependencia que jamás vas a cambiar.
Pedirle a Claude Code "genera el repositorio para `Order`" sin haber decidido
primero qué método necesita el caso de uso reproduce exactamente el
anti-patrón del `findAll()` genérico — más rápido que nunca.

## Caso real (ParcelFlow)

`RegisterOrderUseCase` (arriba) es el ejemplo completo: el caso de uso valida
que el `Package` no exceda el peso máximo del `Order` — regla de negocio pura,
sin `import` de infraestructura — y delega el guardado a `OrderRepository`.
En tests, se instancia con un `InMemoryOrderRepository` (un array en memoria
que implementa la misma interfaz); en producción, con `PgOrderRepository`. El
caso de uso no cambia una línea entre ambos escenarios — esa es la prueba de
que la frontera es real y no decorativa.

## Errores que cometí (o cometería) si empezara otra vez

Definir `Repository<T>` genérico con CRUD completo para cada entidad del
dominio (`OrderRepository`, `PackageRepository`, `DeliveryRepository`, todos
con la misma forma `findAll/findById/save/delete`) antes de tener un solo
caso de uso que necesitara más que `save()` y una consulta específica. El
resultado: interfaces anchas, moldeadas por lo que el ORM sabe hacer, no por
lo que el negocio necesita — exactamente el anti-patrón de esta lección,
cometido por mí mismo en más de un proyecto.
