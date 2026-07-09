# 01. Entities y Value Objects

## Nivel 1 — ELI20

Dos cosas que a simple vista parecen iguales — ambas son "clases con
campos" — pero responden preguntas distintas:

- **Entity**: ¿tiene identidad que persiste en el tiempo, aunque sus datos
  cambien? Un `Order` sigue siendo *el mismo pedido* aunque cambie su
  estado de `pending` a `fulfilled`. Se compara por identidad (`id`), no
  por sus campos.
- **Value Object**: ¿le importa *qué es*, no *quién es*? Un `Address` con
  calle "123 Main St" es intercambiable con cualquier otro `Address` que
  tenga exactamente los mismos valores. No tiene `id`. Se compara por
  igualdad estructural.

La pregunta de una línea para decidir cuál es cuál: *si cambio un campo,
¿sigue siendo "lo mismo" o es "otra cosa"?* Si sigue siendo lo mismo, es
Entity. Si es otra cosa, es Value Object.

## Nivel 2 — Ingeniería

**Value Objects** son inmutables por diseño: cualquier "cambio" crea una
instancia nueva, nunca muta la existente (la misma regla de inmutabilidad
que aplica en cualquier código bien escrito, aquí es estructural, no solo
estilo). Además se auto-validan en el constructor — un `Address` inválido no
debería poder existir ni un microsegundo.

```typescript
class Address {
  private constructor(
    readonly street: string,
    readonly city: string,
    readonly zip: string
  ) {}

  static create(street: string, city: string, zip: string): Address {
    if (!street || !city || !/^\d{5}$/.test(zip)) {
      throw new Error("Address inválida");
    }
    return new Address(street, city, zip);
  }

  equals(other: Address): boolean {
    return (
      this.street === other.street &&
      this.city === other.city &&
      this.zip === other.zip
    );
  }
}
```

**Entities** sí tienen identidad y sí cambian de estado a lo largo de su
vida — pero eso no significa setters públicos libres. El cambio de estado
pasa por un método con nombre de negocio que valida la transición:

```typescript
class Order {
  private status: "pending" | "fulfilled" | "cancelled" = "pending";

  constructor(
    readonly id: string,
    readonly merchantId: string,
    private deliveryAddress: Address
  ) {}

  cancel(): void {
    if (this.status === "fulfilled") {
      throw new Error("No se puede cancelar un pedido ya cumplido");
    }
    this.status = "cancelled";
  }

  equals(other: Order): boolean {
    return this.id === other.id; // identidad, no estructura
  }
}
```

`cancel()` — no `setStatus("cancelled")` — porque el nombre del método debe
ser el verbo del negocio, y la validación de la transición vive ahí, no
afuera.

## Nivel 3 — Senior Engineer

Un senior aplica una prueba práctica antes de escribir la clase: **¿este
concepto necesita rastrearse a través del tiempo, o solo necesita
compararse por su contenido en este instante?** Un `Package` necesita
rastreo (¿dónde está, fue entregado) → Entity. Un rango de peso permitido
(`min: 0kg, max: 30kg`) no necesita historia, solo necesita ser válido → Value
Object.

Error común incluso en seniors: hacer Entity algo que no necesita identidad
solo porque "tiene varios campos". Tener varios campos no es el criterio —
necesitar identidad estable en el tiempo sí lo es. La mayoría de los
conceptos de un dominio, contados con honestidad, son Value Objects: dinero,
direcciones, rangos, coordenadas, períodos de tiempo. Las Entities genuinas
suelen ser menos de las que el modelo inicial sugiere.

## Nivel 4 — Software Architect

El argumento frente al CTO: los bugs de concurrencia e inconsistencia más
caros de depurar en producción casi siempre vienen de mutación compartida —
dos partes del sistema sosteniendo una referencia al mismo objeto mutable y
modificándolo por caminos distintos. Los Value Objects inmutables eliminan
esa clase entera de bug por construcción: no hay "quién lo cambió último"
porque nadie lo cambia, se reemplaza.

Además, Value Objects auto-validados mueven la validación del borde del
sistema (un middleware de Express, fácil de olvidar en un segundo punto de
entrada) al propio tipo — imposible de sortear, porque no existe forma de
construir un `Address` inválido, sin importar desde qué controlador, cron o
script se intente.

## ❌ Mitos

**"Si tiene más de un campo, es una Entity."** No. El número de campos no
importa. `Money { amount, currency }` tiene dos campos y es Value Object
puro — dos instancias con los mismos valores son intercambiables, no
"diferentes montos de dinero que coinciden".

## ❌ Anti-patrones

Un Value Object con setters. Si `Address` expone `setStreet()`, dejó de ser
un Value Object — es una Entity disfrazada, con el costo de inmutabilidad
sin el beneficio de la seguridad que la inmutabilidad da. La mutación in
situ de un objeto que se pasa por referencia a media docena de lugares es
exactamente el bug de "quién lo cambió último" que los Value Objects existen
para eliminar.

## 🤖 Cómo cambió la IA este concepto

Generar la clase `Address` de arriba, con validación y `equals()`, es
trabajo de segundos para un asistente. Lo que sigue siendo criterio humano:
decidir si `Address` en tu dominio es realmente un Value Object o si, por
ejemplo, necesitas rastrear *qué dirección específica* se usó en un envío
histórico incluso si el cliente después la edita — en cuyo caso sí necesita
convertirse en una Entity con su propio `id`. Pedirle a la IA "genera un
Value Object Address" sin haber resuelto esa pregunta primero solo mueve el
error de diseño más rápido a producción.

## Caso real (ParcelFlow)

`Order.deliveryAddress` es un `Address` — Value Object inmutable, uno de los
campos del Aggregate `Order` (ver siguiente capítulo). Si el cliente edita
la dirección de entrega antes del despacho, no se "muta" el `Address`
existente: se crea uno nuevo y se reemplaza la referencia dentro de `Order`.
El objeto viejo simplemente deja de estar referenciado — nunca existió el
riesgo de que otra parte del sistema, sosteniendo la referencia anterior,
viera un cambio a mitad de una operación.

## Errores que cometí (o cometería) si empezara otra vez

Modelar `Address` como Entity con su propio `id` y su propia tabla desde el
día uno, "por si acaso se necesita reutilizar". Eso agrega una relación
foránea, un join, y una pregunta de sincronización (¿qué pasa si dos pedidos
apuntan al mismo `id` de dirección y uno la "actualiza"?) que un Value Object
embebido — inmutable, copiado, sin `id` — nunca tiene que responder.
