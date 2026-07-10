# 02. Trade-offs reales

## Nivel 1 — ELI20

Cuando `RegisterOrderUseCase` y `BillingEventHandler` (capítulo anterior)
viven en el mismo proceso, "guardar el pedido y generar la factura" puede
ser una sola transacción: o pasan las dos cosas, o no pasa ninguna. Cuando
Billing se vuelve un servicio separado, esa garantía desaparece — ya no
existe una transacción que abarque los dos. Ahora es posible que el pedido
se guarde y la factura tarde un minuto en generarse, o que falle y necesite
reintentarse. Ese cambio de reglas — de "inmediato y garantizado" a
"eventual y reintentable" — es el trade-off central de este capítulo, y no
es opcional: es la naturaleza de cualquier comunicación que cruza una red.

## Nivel 2 — Ingeniería

Tres costos concretos aparecen al extraer un servicio, todos ausentes en un
monolito bien diseñado:

**Consistencia eventual.** La transacción atómica del Volumen 2 (Aggregate,
capítulo 02) solo protege lo que vive dentro de un mismo proceso y una
misma base de datos. Cruzar a otro servicio significa aceptar una ventana
de tiempo donde el sistema está, técnicamente, en un estado intermedio —
pedido cumplido, factura todavía no generada — y diseñar el sistema para
que ese estado intermedio sea seguro de observar, no un bug.

**Latencia y fallos parciales de red.** Una llamada de función nunca falla
por timeout. Una llamada HTTP o un mensaje de cola sí — y peor, puede fallar
*después* de que el otro lado ya procesó la petición, dejando ambigüedad
sobre si la operación ocurrió o no. Esto obliga a diseñar cada operación
entre servicios como **idempotente** — ejecutarla dos veces por error de
reintento debe producir el mismo resultado que ejecutarla una vez:

```typescript
class BillingEventConsumer {
  async handleOrderFulfilled(event: OrderFulfilled): Promise<void> {
    const existing = await this.invoices.findByOrderId(event.orderId);
    if (existing) return; // ya procesado, el reintento es seguro

    await this.invoicing.generateInvoice(event.orderId);
  }
}
```

**Observabilidad distribuida.** Un stack trace ya no alcanza para explicar
un bug que atraviesa dos servicios — se necesita un `traceId` que viaje en
cada llamada y cada evento, y herramientas (tracing distribuido, logs
centralizados) que antes eran innecesarias dentro de un solo proceso.
Depurar "por qué esta factura no se generó" pasa de "leer el stack trace" a
"correlacionar logs de dos servicios y una cola de mensajes por `traceId`".

## Nivel 3 — Senior Engineer

Un senior no trata estos costos como generalidades abstractas — los mapea
uno a uno contra la operación específica que se está separando. La
pregunta antes de aceptar consistencia eventual en un caso concreto: **¿qué
pasa si un usuario observa el estado intermedio?** Para "pedido cumplido,
factura pendiente" — un retraso de segundos es invisible para el negocio.
Para "saldo debitado, saldo acreditado pendiente" en un sistema financiero,
esa misma ventana puede ser inaceptable, y es una señal de que esa
operación específica *no* debería cruzar el límite de servicio, sin importar
qué tan limpio esté el Bounded Context detrás.

## Nivel 4 — Software Architect

Frente al CTO, estos trade-offs no son notas técnicas — son partidas de
costo operativo permanentes: tracing distribuido no es gratis, ni en
infraestructura ni en el tiempo de ingeniería que toma instrumentarlo
correctamente. Idempotencia en cada operación entre servicios no es gratis
— cada endpoint nuevo necesita diseñarse con esa disciplina desde el
principio, no agregarse después. El argumento correcto frente a un CTO
nunca es "los microservicios tienen trade-offs" en abstracto — es el costo
operativo específico, medido en tiempo de ingeniería y en infraestructura
nueva, contrastado línea por línea contra el beneficio de despliegue
independiente del capítulo anterior.

## ❌ Mitos

**"La consistencia eventual es un detalle de implementación que no afecta
al negocio."** No. Afecta directamente lo que un usuario puede observar y
lo que el negocio puede prometer. Si Soporte le dice a un cliente "tu
pedido está cumplido" mientras la factura todavía no existe, eso es una
decisión de producto — no un detalle técnico invisible — y alguien del
negocio, no solo ingeniería, debe estar de acuerdo con esa ventana antes de
aceptarla.

## ❌ Anti-patrones

**Tratar una llamada entre servicios como si fuera una llamada de función.**
Código que llama a otro servicio por HTTP sin manejar timeout, sin
reintento, y sin idempotencia — asumiendo implícitamente que "va a
responder, como siempre respondía cuando estaba en el mismo proceso". El
primer fallo parcial de red en producción (el servicio de Billing procesó
la factura pero la respuesta se perdió por timeout) genera una factura
duplicada, porque nadie diseñó para esa posibilidad.

## 🤖 Cómo cambió la IA este concepto

Un asistente puede generar el código de reintento con backoff exponencial,
el manejo de idempotencia, y la instrumentación de tracing distribuido
correctamente estructurados, en minutos — la mecánica de resiliencia ya no
es el obstáculo que era hace años. Lo que sigue siendo trabajo humano:
decidir, operación por operación, si la ventana de consistencia eventual
resultante es aceptable para el negocio. Ese es un juicio de producto, no
uno técnico, y ningún asistente puede tomarlo sin que alguien del negocio
lo valide primero.

## Caso real (ParcelFlow)

`BillingEventConsumer.handleOrderFulfilled()` (arriba) es idempotente por
diseño — verifica si la factura ya existe antes de generar una nueva,
protegiendo contra reintentos duplicados de la cola de mensajes. El
`traceId` generado en `RegisterOrderUseCase` viaja en cada evento publicado
y en cada llamada HTTP subsecuente, permitiendo reconstruir el flujo
completo de un pedido específico a través de Fulfillment y Billing en una
sola consulta de logs, en vez de correlacionar manualmente por timestamp
aproximado.

## Errores que cometí (o cometería) si empezara otra vez

Extraer el primer servicio (Billing) sin diseñar idempotencia desde el
principio, "porque las colas raramente duplican mensajes en la práctica".
La primera vez que la cola sí reintentó un mensaje — por un timeout
temporal del broker, no por ningún bug en el código — generó dos facturas
para el mismo pedido. Diseñar la idempotencia desde el primer día de la
extracción, no después del primer incidente, habría costado lo mismo en
tiempo de desarrollo y evitado un problema de datos real en producción.
