# 02. Aggregates

## Nivel 1 — ELI20

Un Aggregate es un grupo de Entities y Value Objects que **siempre debe
guardarse y modificarse como una sola unidad consistente**. Tiene una
puerta de entrada única — el **Aggregate Root** — y nadie de afuera puede
tocar las piezas internas directamente. Si `Order` es el Aggregate Root y
`Package` vive dentro de su límite, nadie llama
`packageRepository.update(pkg)` a mano por fuera de `Order`. Todo pasa por
`Order`.

## Nivel 2 — Ingeniería

El Aggregate existe para proteger una **invariante** — una regla que debe
ser verdadera siempre, sin excepción, incluso a mitad de operaciones
concurrentes. Ejemplo: "la suma de los pesos de los `Package` de un `Order`
no puede exceder el límite contratado por el `Merchant`". Esa regla
atraviesa varias entidades (`Order`, `Package`) — por eso necesitan vivir
dentro del mismo límite transaccional.

```typescript
class Order {
  private packages: Package[] = [];

  constructor(
    readonly id: string,
    private readonly weightLimitKg: number
  ) {}

  addPackage(pkg: Package): void {
    const totalWeight = this.totalWeight() + pkg.weightKg;
    if (totalWeight > this.weightLimitKg) {
      throw new Error("Excede el límite de peso del pedido");
    }
    this.packages.push(pkg);
  }

  private totalWeight(): number {
    return this.packages.reduce((sum, p) => sum + p.weightKg, 0);
  }

  // Única forma de leer los paquetes desde afuera: una copia, no la referencia interna
  listPackages(): ReadonlyArray<Package> {
    return [...this.packages];
  }
}
```

Reglas mecánicas que se derivan de esto:

- **Un repositorio por Aggregate Root**, no uno por cada Entity interna. No
  existe `PackageRepository` independiente si `Package` vive dentro del
  límite de `Order` — se guarda a través de `orderRepository.save(order)`.
- **Referencias entre Aggregates distintos son por `id`, no por objeto.**
  `Order` no sostiene una referencia directa al objeto `Merchant` completo —
  sostiene `merchantId: string`. Cargar el `Merchant` completo, si hace
  falta, es una operación explícita y separada.
- **Un Aggregate por transacción**, como regla general. Si una operación
  necesita modificar dos Aggregates a la vez de forma atómica, es una señal
  para revisar si de verdad son dos Aggregates o deberían ser uno — o si la
  consistencia entre ambos puede ser eventual en vez de inmediata (ver
  Domain Events, capítulo 04).

## Nivel 3 — Senior Engineer

La pregunta que decide el límite de un Aggregate no es "¿estos objetos están
relacionados?" — casi todo en un dominio está relacionado con algo. La
pregunta es: **¿qué invariante necesito que sea verdadera de forma
inmediata y atómica, sin excepción?** Solo lo que participa directamente en
esa invariante entra al Aggregate. Todo lo demás se referencia por `id` y se
carga aparte.

El error más común de un ingeniero con experiencia media es diseñar
Aggregates demasiado grandes — "total, `Order` tiene `Merchant`, `Package`,
`Delivery`, todo relacionado, mejor lo cargo todo junto" — lo cual produce
transacciones lentas, bloqueos de fila innecesarios, y conflictos de
concurrencia entre operaciones que en realidad no compiten por la misma
invariante. Un senior por defecto empieza con Aggregates pequeños y solo los
agranda cuando encuentra una invariante real que lo exige.

## Nivel 4 — Software Architect

Frente al CTO, el límite del Aggregate es directamente el límite de
**consistencia transaccional versus escalabilidad**:

- Aggregates chicos → transacciones cortas, menos bloqueo de fila, mejor
  throughput bajo concurrencia, y un candidato natural para particionar o
  incluso extraer a su propio servicio más adelante.
- Aggregates grandes → consistencia inmediata más amplia, pero más
  contención: dos operaciones de negocio no relacionadas terminan peleando
  por el mismo lock de fila porque "viven en el mismo Aggregate" aunque no
  compartan ninguna invariante real.

El límite del Aggregate, bien trazado, es también el mapa más honesto de
por dónde el sistema *podría* dividirse en servicios el día que haga falta —
mucho antes de decidir si conviene microservicios (Volumen 5), ya se decidió
dónde están las costuras reales.

## ❌ Mitos

**"Un Aggregate debe mapear uno a uno con una tabla, o con todo el árbol de
relaciones de la base de datos."** No. El límite del Aggregate lo define la
invariante de negocio, no el modelo entidad-relación. Es común que un
Aggregate persista en varias tablas (`orders` + `order_packages`), y
también es común que dos "tablas relacionadas" en SQL no deban vivir en el
mismo Aggregate si no comparten una invariante real.

## ❌ Anti-patrones

**God Aggregate.** Un `Order` que carga, dentro de su propio límite,
`Merchant` completo, historial de `Delivery`, y datos de facturación —
porque "todo está relacionado con el pedido". Cada operación sobre
cualquiera de esas partes bloquea el Aggregate entero, y cualquier cambio en
el esquema de una parte no relacionada obliga a tocar la clase raíz.

## 🤖 Cómo cambió la IA este concepto

Un asistente puede generar el Aggregate `Order` completo, con su método
`addPackage()` validando la invariante, en un prompt. Lo que no puede
decidir por ti: **cuál es la invariante real** que justifica que `Package`
viva dentro del límite de `Order` y no sea su propio Aggregate independiente
referenciado por `id`. Esa decisión depende de una conversación de negocio
específica — "¿alguna vez necesitamos modificar un `Package` sin pasar por
su `Order`?" — que la IA no puede tener por ti.

## Caso real (ParcelFlow)

`Order` es el Aggregate Root; `Package` vive dentro de su límite porque la
invariante de peso máximo los conecta directamente. `Delivery`, en cambio,
**no** vive dentro de `Order` — es su propio Aggregate, referenciado por
`orderId`, porque una entrega tiene su propio ciclo de vida (reintentos,
cambios de transportista) que no necesita bloquear ni ser bloqueado por
cambios al pedido. Esa asimetría — `Package` adentro, `Delivery` afuera — es
exactamente el tipo de decisión que este capítulo enseña a tomar con
criterio, no por simetría estética.

## Errores que cometí (o cometería) si empezara otra vez

Meter `Delivery` dentro del Aggregate `Order` "porque un pedido tiene una
entrega, tiene sentido que vivan juntos". El resultado: cada actualización
de estado de tracking de la transportista —que llega con frecuencia y no
tiene nada que ver con el contenido del pedido— terminaba bloqueando el
mismo registro que usaban las operaciones de negocio reales sobre el
pedido. Separarlos en dos Aggregates conectados por `id`, desde el
principio, habría evitado meses de contención de escritura bajo carga.
