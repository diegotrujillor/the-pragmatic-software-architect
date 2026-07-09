# 03. Use Cases

## Nivel 1 — ELI20

Un caso de uso es **un verbo del negocio, con nombre y apellido**:
`MarkPackageDelivered`, `CancelOrder`, `RegisterOrder`. No es
`OrderService.handle(action, payload)` con un `switch` adentro. Si tuvieras
que explicarle a alguien de producto qué hace tu sistema, la lista de casos
de uso debería sonar exactamente igual a la lista de cosas que el producto
dice que hace.

## Nivel 2 — Ingeniería

Un caso de uso (Interactor, en el vocabulario de Robert C. Martin) recibe un
**Request** (datos planos de entrada), orquesta una o más entidades y
repositorios, y produce una **Response** (datos planos de salida). No
contiene las reglas de negocio más profundas — esas viven en las entidades —
pero sí decide *el orden* y *las condiciones* en que se invocan para cumplir
una acción concreta.

```typescript
// use-cases/mark-package-delivered.ts
interface MarkPackageDeliveredRequest {
  packageId: string;
  deliveredAt: Date;
}

interface MarkPackageDeliveredResponse {
  packageId: string;
  orderFulfilled: boolean;
}

class MarkPackageDeliveredUseCase {
  constructor(
    private readonly packages: PackageRepository,
    private readonly orders: OrderRepository,
    private readonly events: DomainEventBus
  ) {}

  async execute(
    req: MarkPackageDeliveredRequest
  ): Promise<MarkPackageDeliveredResponse> {
    const pkg = await this.packages.findById(req.packageId);
    pkg.markDelivered(req.deliveredAt); // regla de negocio vive en Package

    const order = await this.orders.findById(pkg.orderId);
    const fulfilled = order.checkFulfillment([pkg]); // regla vive en Order

    await this.packages.save(pkg);
    if (fulfilled) {
      await this.orders.save(order);
      this.events.publish("OrderFulfilled", { orderId: order.id });
    }

    return { packageId: pkg.id, orderFulfilled: fulfilled };
  }
}
```

Nótese qué **no** hace esta clase: no arma SQL, no sabe qué es Express, no
decide el formato JSON de la respuesta HTTP. Solo orquesta.

Un detalle de diseño que separa un caso de uso bien hecho de uno mediocre:
**una clase por verbo**, no una clase `OrderService` con quince métodos
(`create`, `update`, `cancel`, `refund`, `markDelivered`...). El costo de la
alternativa: el constructor de `OrderService` termina necesitando *todas*
las dependencias de *todos* los métodos, aunque cada llamador solo use una.
Testear un método implica cargar (o mockear) las quince dependencias.
Desplegar un cambio a `refund` arriesga romper `cancel` por accidente, porque
comparten archivo.

## Nivel 3 — Senior Engineer

Un senior decide la granularidad con dos preguntas:

1. **¿Este verbo tiene reglas de orquestación propias** (coordina más de una
   entidad, decide side effects, publica eventos)? Si sí, se gana su propio
   Interactor.
2. **¿Dónde vive la lógica que no es orquestación pura?** Una regla como
   "una entrega no puede marcarse dos veces" pertenece a `Package`, no al
   caso de uso. Si el caso de uso empieza a acumular `if` sobre el estado
   interno de una entidad, es señal de que esa entidad tiene una regla que
   debería encapsular ella misma (ver [Mitos](#-mitos)).

El caso de uso también es, casi siempre, el **límite natural de
transacción**: todo lo que debe persistir atómicamente para que la acción de
negocio sea consistente (guardar el paquete Y el pedido, o ninguno) se
envuelve ahí — no en el controlador, no repartido entre dos llamadas HTTP.

## Nivel 4 — Software Architect

Frente al CTO, un caso de uso por verbo no es una preferencia estética — es
una unidad de:

- **Autorización.** Cada Interactor es el punto natural para preguntar "¿este
  usuario puede ejecutar `CancelOrder`?" — sin ese límite, los permisos
  terminan dispersos entre rutas HTTP y validaciones ad hoc.
- **Observabilidad con significado de negocio.** Loguear y medir por nombre
  de caso de uso (`MarkPackageDelivered`) da métricas que hablan el idioma
  del producto, no el idioma de rutas HTTP (`POST /packages/:id/deliver`).
- **Reutilización entre entradas.** El mismo `MarkPackageDeliveredUseCase`
  se invoca igual desde un controlador REST, un consumidor de cola, o un
  script de reproceso — sin duplicar lógica en cada punto de entrada.
- **Screaming Architecture real.** Una carpeta `use-cases/` con un archivo
  por verbo es, literalmente, el índice de lo que el producto hace. Eso
  reduce el tiempo de onboarding de un ingeniero nuevo de días a minutos.

## ❌ Mitos

**"Todo caso de uso necesita su propia interfaz de Input Boundary y su
propio Presenter/Output Boundary."** No siempre. Ese nivel de indirección
extra se paga cuando de verdad hay múltiples formas de presentar el mismo
resultado (web y CLI, por ejemplo). Para una sola API REST, devolver
directamente el objeto `Response` del caso de uso, sin una capa de Presenter
adicional, no rompe la Dependency Rule — solo evita ceremonia sin comprador.

## ❌ Anti-patrones

**God Service.** Un `OrderService` con quince métodos no relacionados,
forzando a cada consumidor a depender (y en tests, mockear) de toda la
superficie aunque solo use un método. Ver el costo detallado en el Nivel 2.

**Caso de uso anémico.** Un Interactor que es solo
`return this.repo.save(input)`, sin una sola regla de negocio adentro. No es
un error tener casos de uso simples — es una señal de alerta cuando *todos*
lo son, porque usualmente significa que la lógica real se fue a filtrar al
controlador, o que la entidad tiene setters públicos sin invariantes.

## 🤖 Cómo cambió la IA este concepto

Generar un Interactor nuevo — clase, Request, Response, wiring — es hoy un
prompt de un párrafo. Eso quita la única excusa razonable que existía antes
para construir un God Service ("no quiero repetir tanto boilerplate por cada
método"). Lo que la IA no resuelve por ti: nombrar el caso de uso en el
idioma correcto del negocio (`MarkPackageDelivered`, no `UpdatePackage`), y
decidir dónde termina la orquestación y empieza la regla que le pertenece a
la entidad. Ese es el criterio que sigue siendo trabajo humano.

## Caso real (ParcelFlow)

`MarkPackageDeliveredUseCase` (arriba) es el ejemplo completo: orquesta
`Package.markDelivered()` y `Order.checkFulfillment()` — dos entidades
distintas — decide si ambas requieren persistirse, y publica un evento de
dominio solo si el pedido completo quedó cumplido. Ninguna de esas tres
decisiones (marcar entregado, verificar cumplimiento del pedido, publicar el
evento) vive en el controlador HTTP ni en el repositorio — vive exactamente
en el caso de uso, que es el único lugar responsable de orquestar esa acción
de negocio de principio a fin.

## Errores que cometí (o cometería) si empezara otra vez

Construir un `OrderService` único "para no fragmentar la lógica de pedidos"
apenas al segundo o tercer caso de uso relacionado con `Order`. Terminó, con
el tiempo, en un archivo de cientos de líneas mezclando doce acciones de
negocio no relacionadas, con un constructor que acumulaba dependencias de
todas ellas. Separar por verbo desde el segundo caso de uso — no esperar a
que el archivo duela — habría costado cero y ahorrado meses de fricción en
cada revisión de código.
