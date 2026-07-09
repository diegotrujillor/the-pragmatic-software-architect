# 02. Estructurales: Decorator, Adapter, Facade

## Nivel 1 — ELI20

Tres formas distintas de componer objetos sin que unos conozcan los
detalles internos de otros:

- **Decorator**: envuelve un objeto para agregarle comportamiento extra,
  capa sobre capa, sin tocar la clase original ni crear una subclase por
  cada combinación posible.
- **Adapter**: traduce la interfaz de algo externo (una librería, una API de
  terceros) a la interfaz que tu código espera, sin que tu código sepa que
  el traductor existe.
- **Facade**: esconde un subsistema complicado — con varias piezas que
  colaboran entre sí — detrás de una interfaz simple de un solo punto de
  entrada.

## Nivel 2 — Ingeniería

**Decorator** — ParcelFlow aplica descuentos a un `Order`: por volumen, por
cliente frecuente, por temporada. Sin el patrón, cada combinación posible de
descuentos necesitaría su propia clase o un método con banderas
(`applyDiscount(order, isVolume, isFrequent, isSeasonal)`). Con Decorator,
cada regla envuelve la anterior:

```typescript
interface PricedOrder {
  total(): number;
}

class BaseOrder implements PricedOrder {
  constructor(private readonly order: Order) {}
  total() { return this.order.subtotal(); }
}

class VolumeDiscount implements PricedOrder {
  constructor(private readonly inner: PricedOrder, private readonly pct: number) {}
  total() { return this.inner.total() * (1 - this.pct); }
}

class SeasonalDiscount implements PricedOrder {
  constructor(private readonly inner: PricedOrder, private readonly pct: number) {}
  total() { return this.inner.total() * (1 - this.pct); }
}

// Composición: solo se envuelve con lo que aplica a este pedido
let priced: PricedOrder = new BaseOrder(order);
if (order.qualifiesForVolume()) priced = new VolumeDiscount(priced, 0.1);
if (isHolidaySeason()) priced = new SeasonalDiscount(priced, 0.05);
```

Ninguna clase nueva de descuento requiere tocar `BaseOrder` ni las demás
capas — cada una solo conoce la interfaz `PricedOrder`.

**Adapter** — ya apareció, con otro nombre, en el Volumen 2 (Anti-Corruption
Layer es un Adapter aplicado a nivel de Bounded Context). La forma es la
misma: traducir el esquema de un proveedor externo de transportistas al
vocabulario interno de `Package`, sin que el resto del sistema conozca el
formato externo.

**Facade** — registrar un pedido completo en ParcelFlow implica coordinar
validación de inventario, cálculo de precio con descuentos, selección de
transportista y persistencia. Un controlador HTTP no debería orquestar los
cuatro subsistemas directamente:

```typescript
class OrderCheckoutFacade {
  constructor(
    private readonly inventory: InventoryChecker,
    private readonly pricing: PricingEngine,
    private readonly carriers: CarrierSelector,
    private readonly useCase: RegisterOrderUseCase
  ) {}

  async checkout(input: CheckoutInput): Promise<CheckoutResult> {
    await this.inventory.reserve(input.items);
    const total = this.pricing.calculate(input);
    const carrier = this.carriers.select(input.packages);
    return this.useCase.execute({ ...input, total, carrier });
  }
}
```

## Nivel 3 — Senior Engineer

Con Decorator, un senior pregunta: **¿las capas se combinan de formas
distintas en distintos pedidos, o siempre se aplican en el mismo orden
fijo?** Si el orden y la combinación son siempre iguales, un método con
varios pasos secuenciales es más simple y más legible que envolver objetos
— Decorator se gana su lugar cuando la combinación es genuinamente variable
en tiempo de ejecución, no fija en el código.

Con Facade, la pregunta es distinta: **¿esta Facade reduce complejidad real,
o solo mueve el mismo desorden a un archivo con nombre más bonito?** Una
Facade que sigue exponiendo los detalles de los cuatro subsistemas a quien
la llama no simplificó nada — solo agregó un salto de función.

## Nivel 4 — Software Architect

El argumento de Decorator frente al CTO es el mismo Open/Closed Principle
que ya apareció con Strategy: agregar una regla de descuento nueva es una
clase nueva, testeable sola, sin riesgo de romper las reglas de descuento
existentes en producción durante la temporada alta de ventas — momento en
que ese riesgo es exactamente el que menos se puede permitir el negocio.

El argumento de Facade es **superficie de cambio controlada**: si mañana el
proceso de checkout gana un quinto paso (por ejemplo, verificación de
fraude), ese cambio se hace dentro de `OrderCheckoutFacade`, sin tocar el
controlador HTTP, el consumidor de cola, ni cualquier otro punto de entrada
que ya use la Facade. El punto de entrada externo permanece estable aunque
la orquestación interna crezca.

## ❌ Mitos

**"Facade y Service son la misma cosa con otro nombre."** Parcialmente
cierto y parcialmente el problema: un `OrderService` con quince métodos no
relacionados (visto en el Volumen 1, capítulo 03) no es una Facade — es un
God Service. Una Facade legítima tiene una responsabilidad de orquestación
única y bien definida (aquí, "completar el checkout"), no una colección de
métodos sin relación agrupados por conveniencia.

## ❌ Anti-patrones

**Decorator innecesario para una sola combinación fija.** Envolver
`BaseOrder` en `VolumeDiscount` cuando *todos* los pedidos del sistema,
siempre, reciben ese mismo descuento — sin variación real — es una capa de
indirección para representar una regla que un método directo expresaría en
una línea.

## 🤖 Cómo cambió la IA este concepto

Pedirle a un asistente "aplica Decorator a este sistema de descuentos" o
"crea un Adapter para esta API externa" produce el andamiaje correcto en
minutos. La decisión que sigue siendo humana con Decorator: si las
combinaciones son de verdad dinámicas (justifican el patrón) o siempre las
mismas (no lo justifican). Y con Facade: dónde trazar exactamente el límite
de lo que orquesta — una Facade demasiado ambiciosa termina siendo el mismo
God Service de otro capítulo, con un nombre distinto.

## Caso real (ParcelFlow)

El sistema de descuentos (arriba) usa Decorator en producción con hasta
cuatro capas combinables (`VolumeDiscount`, `SeasonalDiscount`,
`LoyaltyDiscount`, `FirstOrderDiscount`) que se aplican condicionalmente
según el pedido — la combinación real varía pedido a pedido, la razón por
la que el patrón se gana su lugar aquí. `OrderCheckoutFacade` es el único
punto de entrada tanto del controlador REST como de un job de checkout
diferido para carritos abandonados — dos entradas, una sola orquestación.

## Errores que cometí (o cometería) si empezara otra vez

Construir el sistema de descuentos con Decorator desde el día uno, cuando
ParcelFlow solo tenía una regla de descuento (`VolumeDiscount`) sin ninguna
combinación posible todavía. Una función que multiplicaba el subtotal por
un porcentaje se convirtió en una interfaz y dos clases para un caso que,
durante meses, nunca tuvo una segunda capa que combinar. El patrón se ganó
su lugar real cuando apareció la segunda regla de descuento simultánea —
no antes.
