# 04. Domain Events

## Nivel 1 — ELI20

Un Domain Event es un hecho del negocio que **ya ocurrió** — se nombra en
tiempo pasado, no como un comando: `OrderFulfilled`, no `FulfillOrder`. La
diferencia importa: un comando puede rechazarse (¿tienes permiso para
cancelar este pedido?); un evento ya pasó, no se puede "rechazar" que un
pedido se haya cumplido, solo reaccionar a que se cumplió.

Sirve para una cosa concreta: cuando una acción de negocio en un Aggregate
necesita que *otras partes del sistema se enteren y reaccionen*, sin que el
Aggregate original tenga que conocer, importar ni llamar directamente a
esas otras partes.

## Nivel 2 — Ingeniería

El evento se emite desde dentro del Aggregate cuando su invariante decide
que algo relevante pasó, y el caso de uso (Volumen 1, capítulo 03) es quien
lo publica después de persistir con éxito:

```typescript
interface DomainEvent {
  readonly occurredAt: Date;
}

class OrderFulfilled implements DomainEvent {
  readonly occurredAt = new Date();
  constructor(readonly orderId: string) {}
}

class Order {
  private packages: Package[] = [];
  private pendingEvents: DomainEvent[] = [];

  markPackageDelivered(packageId: string): void {
    // ... lógica de marcar el paquete ...
    if (this.allPackagesDelivered()) {
      this.pendingEvents.push(new OrderFulfilled(this.id));
    }
  }

  pullEvents(): DomainEvent[] {
    const events = [...this.pendingEvents];
    this.pendingEvents = [];
    return events;
  }
}
```

El Aggregate acumula el evento internamente — no lo publica él mismo, no
conoce ningún bus de mensajes. El caso de uso, después de guardar
exitosamente, extrae los eventos pendientes y los publica:

```typescript
async execute(req: MarkPackageDeliveredRequest) {
  const order = await this.orders.findById(req.orderId);
  order.markPackageDelivered(req.packageId);
  await this.orders.save(order);

  for (const event of order.pullEvents()) {
    await this.eventBus.publish(event); // recién aquí sale del Aggregate
  }
}
```

Esta separación — el Aggregate decide *qué* pasó, el caso de uso decide
*cuándo* anunciarlo, después de que la persistencia tuvo éxito — evita el
bug clásico de publicar un evento sobre un cambio que después falló al
guardarse.

Los suscriptores de `OrderFulfilled` (enviar email de confirmación, avisar
al sistema de Billing, actualizar un dashboard) viven completamente
desacoplados de `Order` — ni `Order` ni `MarkPackageDeliveredUseCase`
conocen su existencia.

## Nivel 3 — Senior Engineer

La decisión que un senior enfrenta con cada evento: **¿consistencia
inmediata o eventual?** Si "enviar email de confirmación" falla, ¿debe
fallar también "marcar el pedido como cumplido"? Casi nunca. Esa es
precisamente la señal de que el email pertenece a un manejador de evento
asíncrono, desacoplado, y no a una llamada directa dentro de la misma
transacción del caso de uso.

La contraparte de esa libertad es un costo real que un senior sí reconoce:
consistencia eventual significa que, por un instante, el sistema tiene
estados que un observador externo podría ver como "inconsistentes" (el
pedido ya está `fulfilled` pero el email todavía no se envió). Para algunas
invariantes de negocio eso es perfectamente aceptable; para otras — como
"el saldo debitado debe coincidir exactamente con el saldo acreditado en
todo momento" — no lo es, y ahí la consistencia debe ser inmediata, dentro
del mismo Aggregate o la misma transacción.

## Nivel 4 — Software Architect

Frente al CTO, los Domain Events son el mecanismo que permite agregar
funcionalidad nueva **sin tocar el código existente**: el día que Marketing
pide "cuando un pedido se cumpla, agrega puntos de fidelidad", la respuesta
es un nuevo suscriptor de `OrderFulfilled` — cero cambios a
`MarkPackageDeliveredUseCase`, cero riesgo de romper el flujo de
cumplimiento de pedidos al construir una feature no relacionada. Eso es el
Open/Closed Principle aplicado a nivel de arquitectura, no solo de clase.

También son el puente natural hacia un Bounded Context distinto (capítulo
anterior): Billing puede suscribirse a `OrderFulfilled` sin que Fulfillment
sepa que Billing existe — el evento es la Anti-Corruption Layer en sí misma,
en forma de contrato de datos plano y estable.

## ❌ Mitos

**"Domain Event es lo mismo que un mensaje de Kafka o RabbitMQ."** No. El
Domain Event es un concepto de modelado — un hecho de negocio nombrado en
pasado. Puede implementarse con un `EventEmitter` en memoria dentro de un
monolito, sin cola de mensajes de por medio. La infraestructura de mensajería
es un detalle de implementación del `DomainEventBus`, intercambiable sin
tocar el modelo — exactamente la Dependency Rule del Volumen 1 aplicada
aquí también.

## ❌ Anti-patrones

Publicar el evento **antes** de confirmar que la persistencia tuvo éxito. Si
`OrderFulfilled` se publica y un suscriptor ya envió el email de
confirmación, pero el `save()` del pedido falla después por timeout de base
de datos, el sistema queda mintiendo: el email dice que el pedido se cumplió
y la base de datos dice que no. El orden correcto siempre es: persistir
primero, publicar después.

## 🤖 Cómo cambió la IA este concepto

Generar la clase `OrderFulfilled`, el mecanismo de `pendingEvents` y el
wiring del bus es, hoy, trabajo de un prompt. La decisión que sigue siendo
enteramente humana: **qué hechos del negocio merecen ser un evento de
dominio nombrado**, versus cuáles son simplemente detalles de
implementación que no le importan a nadie más. Convertir cada cambio de
campo en un evento — "por si acaso alguien lo necesita" — reproduce, a nivel
de eventos, el mismo anti-patrón del Aggregate genérico: ceremonia sin
suscriptor real.

## Caso real (ParcelFlow)

`OrderFulfilled` (arriba) es el evento formal detrás de lo que el Volumen 1,
capítulo 03, ya mostró de forma más simple. Hoy tiene dos suscriptores
reales en el sistema: uno en el Bounded Context de Billing (genera la línea
de factura final) y uno en un servicio de notificaciones (envía el email al
`Merchant`). Ninguno de los dos fue tocado ni conocido por
`MarkPackageDeliveredUseCase` al agregarse — se registraron como
suscriptores del evento, meses después de que el caso de uso ya estaba en
producción.

## Errores que cometí (o cometería) si empezara otra vez

Llamar directamente al servicio de email desde dentro del caso de uso
(`await this.emailService.send(...)`) en vez de publicar un evento, "porque
total solo hay un suscriptor por ahora". Cuando llegó el segundo suscriptor
(Billing), la única forma de agregarlo fue editar el caso de uso ya
existente y probado, en vez de simplemente registrar un nuevo manejador.
Publicar el evento desde el día uno — aunque exista un solo suscriptor —
cuesta lo mismo que la llamada directa y deja la puerta abierta sin
reescribir nada después.
