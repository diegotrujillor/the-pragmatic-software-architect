# 01. Boundaries de servicio

## Nivel 1 — ELI20

La pregunta equivocada al dividir un sistema en servicios es "¿cómo lo
separo por capa técnica?" — un servicio de "frontend API", otro de
"lógica de negocio", otro de "acceso a datos". Esa división obliga a que
casi cualquier cambio de negocio toque los tres servicios a la vez, con
tres despliegues coordinados. La pregunta correcta es "¿qué partes del
negocio ya cambian juntas, y cuáles cambian de forma independiente?" — y
esa pregunta ya tiene respuesta si el Volumen 2 (Bounded Contexts) se hizo
bien: el límite de servicio correcto es, casi siempre, el mismo límite que
ya trazó el Bounded Context.

## Nivel 2 — Ingeniería

Extraer Billing (Volumen 2, capítulo 03) de ParcelFlow como su propio
servicio no es una decisión nueva de diseño — es ejecutar un límite que ya
existía como módulo dentro del monolito:

```typescript
// Antes: Billing es un módulo dentro del mismo proceso,
// se suscribe a OrderFulfilled directamente en memoria
class BillingEventHandler {
  constructor(private readonly invoicing: InvoicingService) {}

  onOrderFulfilled(event: OrderFulfilled): void {
    this.invoicing.generateInvoice(event.orderId);
  }
}

// Después: Billing es su propio servicio. El evento cruza la red,
// no la memoria — pero la interfaz de negocio (qué hace, no cómo
// se entera) permanece exactamente igual.
class BillingEventConsumer {
  constructor(
    private readonly invoicing: InvoicingService,
    private readonly queue: MessageQueue
  ) {}

  async start(): Promise<void> {
    this.queue.subscribe("OrderFulfilled", async (event) => {
      await this.invoicing.generateInvoice(event.orderId);
    });
  }
}
```

Lo que cambió no es la lógica de negocio — es el mecanismo de entrega del
evento (en memoria → cola de mensajes) y todo lo que eso implica: el
evento ahora puede llegar duplicado (necesita idempotencia), puede llegar
tarde (consistencia eventual real, no solo teórica), y puede no llegar
nunca si la cola falla (necesita monitoreo de dead-letter queue). El
Bounded Context ya delimitaba *qué* pertenece a Billing; extraerlo a
servicio solo cambia *cómo* Billing se entera de lo que pasa afuera.

Cada servicio extraído necesita, además, **su propia base de datos** — un
servicio que sigue leyendo directamente la tabla de otro servicio no ganó
independencia de despliegue real, solo movió el acoplamiento de la capa de
código a la capa de base de datos, que es más difícil de detectar y más
caro de deshacer.

## Nivel 3 — Senior Engineer

Un senior valida el límite de un servicio candidato con la misma prueba que
ya aplicó al Aggregate (Volumen 2, capítulo 02) y al Bounded Context
(Volumen 2, capítulo 03): **¿qué invariante o qué frecuencia de cambio
independiente justifica esta frontera?** Si Billing y Fulfillment siguen
cambiando juntos, en el mismo pull request, por el mismo equipo, no hay
beneficio real en separarlos — solo el costo nuevo de coordinar dos
despliegues para un solo cambio de negocio.

La señal más confiable de que un límite está listo para extraerse: el
módulo ya se comunica con el resto del sistema exclusivamente por eventos o
por una interfaz explícita (nunca importando clases internas de otro
módulo directamente) — si esa disciplina ya existe dentro del monolito,
extraerlo a proceso separado es un cambio de infraestructura, no un
rediseño.

## Nivel 4 — Software Architect

Frente al CTO, el argumento no es "separamos Billing en un servicio" — es
**qué gana el negocio con esa separación específica, medido en tiempo de
release y en radio de impacto de una falla**. Si el equipo de Billing puede
desplegar un cambio de cálculo de impuestos sin coordinar con el equipo de
Fulfillment, y sin arriesgar que un bug de Billing tumbe el flujo de
registro de pedidos — esas dos ganancias son medibles y reales. Si ninguna
de las dos aplica todavía, la separación es costo puro sin el beneficio que
la justifica.

## ❌ Mitos

**"El límite correcto de un microservicio es una tabla de base de datos."**
No. Ese es exactamente el error de dividir por capa técnica en vez de por
negocio. Un servicio bien delimitado casi siempre necesita varias tablas
relacionadas (todo lo que participa en sus invariantes de Aggregate,
Volumen 2) — dividir por tabla individual produce servicios que no pueden
completar una sola operación de negocio sin llamar a otros tres servicios.

## ❌ Anti-patrones

**El monolito distribuido.** Servicios separados en el despliegue pero
compartiendo la misma base de datos, o peor, un servicio leyendo
directamente las tablas internas de otro. Tiene todo el costo operativo de
microservicios (red, despliegues múltiples, observabilidad distribuida) sin
ninguno de sus beneficios reales — un cambio de esquema en un servicio
sigue rompiendo silenciosamente al otro, exactamente como en un monolito,
pero ahora es más difícil de diagnosticar.

## 🤖 Cómo cambió la IA este concepto

Un asistente puede generar el `BillingEventConsumer`, la configuración de
la cola de mensajes, y hasta el `docker-compose.yml` completo para separar
un servicio, en minutos. Lo que no puede decidir por ti: si el límite que le
pediste extraer es el correcto — si de verdad corresponde a un Bounded
Context maduro con cambio independiente real, o si es una división
arbitraria por capa técnica que reproduce el monolito distribuido más
rápido que nunca. Pedir "separa esto en microservicios" sin haber validado
el límite primero es delegarle a la IA una decisión que solo el
conocimiento del negocio puede tomar.

## Caso real (ParcelFlow)

`BillingEventHandler` (arriba) es el módulo interno tal como opera hoy,
dentro del monolito. `BillingEventConsumer` es el diseño ya preparado para
el día de la extracción — mismo comportamiento de negocio, mecanismo de
entrega distinto. La existencia de ese segundo archivo, sin que Billing se
haya extraído todavía, es intencional: demuestra que el límite ya es lo
bastante limpio como para cambiar solo el mecanismo de comunicación sin
tocar la lógica de facturación en sí.

## Errores que cometí (o cometería) si empezara otra vez

Diseñar el límite de un servicio candidato alrededor de "todo lo que toca
la tabla `invoices`" en vez de "todo lo que participa en la invariante de
facturación de un pedido". El primer diseño terminó fragmentando una sola
operación de negocio (generar una factura completa) en llamadas a tres
servicios distintos porque cada tabla relacionada vivía en un servicio
diferente — el segundo diseño, alineado al Bounded Context real, la
mantiene en una sola operación dentro de un solo servicio.
