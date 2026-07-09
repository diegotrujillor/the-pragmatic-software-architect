# 03. Comportamiento: Command, Template Method

## Nivel 1 — ELI20

**Command**: convierte "una acción a ejecutar" en un objeto que se puede
guardar, poner en cola, reintentar o deshacer — en vez de una llamada de
función directa que se ejecuta y desaparece. **Template Method**: define el
esqueleto fijo de un algoritmo en una clase base, y deja que las subclases
solo rellenen los pasos que realmente varían — el resto del proceso queda
garantizado igual para todas.

## Nivel 2 — Ingeniería

**Command** — ParcelFlow necesita reintentar el envío de una notificación de
entrega si el proveedor de email falla, y necesita poder encolar esa acción
para ejecutarla más tarde en vez de bloquear el caso de uso esperando la
respuesta del proveedor:

```typescript
interface Command {
  execute(): Promise<void>;
}

class SendDeliveryNotification implements Command {
  constructor(
    private readonly emailService: EmailService,
    private readonly orderId: string
  ) {}

  async execute(): Promise<void> {
    await this.emailService.send(this.orderId);
  }
}

class CommandQueue {
  private pending: Command[] = [];

  enqueue(cmd: Command): void {
    this.pending.push(cmd);
  }

  async processWithRetry(maxAttempts = 3): Promise<void> {
    for (const cmd of this.pending) {
      for (let attempt = 1; attempt <= maxAttempts; attempt++) {
        try {
          await cmd.execute();
          break;
        } catch (err) {
          if (attempt === maxAttempts) throw err;
        }
      }
    }
  }
}
```

El suscriptor de `OrderFulfilled` (Volumen 2, capítulo 04) no llama al
`EmailService` directamente — construye un `SendDeliveryNotification` y lo
encola. La lógica de reintento vive en un solo lugar (`CommandQueue`), no
repetida en cada punto que necesita reintentar algo.

**Template Method** — generar cualquier reporte en ParcelFlow (factura,
guía de envío, resumen de cuenta) sigue siempre la misma secuencia: obtener
datos, aplicar formato, agregar encabezado y pie de página, exportar. Solo
"aplicar formato" varía según el tipo de reporte:

```typescript
abstract class ReportGenerator {
  async generate(orderId: string): Promise<Buffer> {
    const data = await this.fetchData(orderId);      // fijo
    const body = this.formatBody(data);                // varía por subclase
    const withHeader = this.addHeaderFooter(body);      // fijo
    return this.export(withHeader);                     // fijo
  }

  protected abstract formatBody(data: OrderData): string;

  private async fetchData(orderId: string) { /* ... */ }
  private addHeaderFooter(body: string) { /* ... */ }
  private export(content: string): Buffer { /* ... */ }
}

class InvoiceReport extends ReportGenerator {
  protected formatBody(data: OrderData): string {
    return `Factura #${data.orderId}\nTotal: ${data.total}`;
  }
}
```

Cada tipo de reporte nuevo solo implementa `formatBody()` — el resto del
proceso (obtener datos, encabezado, exportar) está garantizado idéntico y
no se puede romper por accidente en una subclase.

## Nivel 3 — Senior Engineer

Con Command, la pregunta de un senior es: **¿esta acción necesita
sobrevivir más allá de la llamada de función que la originó** — reintento,
cola, log de auditoría, deshacer? Si la acción siempre se ejecuta
inmediatamente, una vez, sin necesidad de reintento ni historial, envolverla
en un objeto Command es indirección sin beneficio — una llamada de función
directa ya cumple el mismo trabajo.

Con Template Method, la señal real es: **¿el esqueleto del algoritmo es
genuinamente fijo, y solo una parte varía?** Si más de un paso varía de
forma independiente entre subclases, Template Method tiende a producir
jerarquías rígidas y frágiles — ahí Strategy (capítulo 01), con composición
en vez de herencia, suele envejecer mejor.

## Nivel 4 — Software Architect

El argumento de Command frente al CTO es **resiliencia operable**: acciones
que fallan (llamadas a proveedores externos, envío de notificaciones) se
vuelven reintentables, auditable y observables como una cola, en vez de
"si falló, se perdió". Eso convierte una falla transitoria de un proveedor
externo en un problema recuperable automáticamente, no en un ticket de
soporte.

El argumento de Template Method es **consistencia obligatoria bajo
crecimiento de equipo**: con cuatro tipos de reporte y cuatro ingenieros
distintos implementándolos con el tiempo, Template Method garantiza
mecánicamente que ningún reporte nuevo se olvide el encabezado legal
requerido o el formato de exportación — el paso fijo no se puede saltar
porque no es opcional, es parte del método base.

## ❌ Mitos

**"Template Method es simplemente herencia, y la herencia es siempre mala."**
No exactamente. La herencia se vuelve un problema cuando se usa para
reutilizar comportamiento variable de formas impredecibles (herencia
profunda, múltiples responsabilidades mezcladas). Template Method es
herencia acotada a un propósito específico y angosto — fijar una secuencia,
variar un paso — que es precisamente el caso donde la herencia es la
herramienta correcta, no un antipatrón por sí sola.

## ❌ Anti-patrones

**Command para una llamada síncrona simple sin reintento, sin cola, sin
auditoría.** Envolver `await this.emailService.send(id)` en una clase
`SendEmailCommand` con un único método `execute()` que hace exactamente eso,
sin ningún mecanismo de cola o reintento alrededor, es una capa de
indirección disfrazada de patrón — la llamada directa ya era suficiente.

## 🤖 Cómo cambió la IA este concepto

Un asistente genera `CommandQueue` con reintentos, o `ReportGenerator` con
su Template Method completo, en un prompt. Lo que no decide por ti: si la
acción realmente necesita sobrevivir a fallos (justifica Command) o si el
reporte realmente comparte un esqueleto fijo entre todas sus variantes
(justifica Template Method) — o si estás empaquetando una llamada simple en
ceremonia porque el patrón "suena robusto". Pedir "hazlo reintentable" sin
haber decidido si de verdad necesita reintento es el mismo error, más rápido.

## Caso real (ParcelFlow)

`CommandQueue` procesa hoy tres tipos de comando en producción — envío de
notificación, sincronización con el sistema de Billing, y actualización de
tracking del transportista — todos con reintento automático hasta tres
intentos antes de escalar a una alerta manual. `ReportGenerator` genera
cuatro tipos de documento (factura, guía de envío, resumen de cuenta,
certificado de entrega) compartiendo el mismo pipeline de obtención de
datos, encabezado corporativo y exportación a PDF, con solo `formatBody()`
distinto en cada subclase.

## Errores que cometí (o cometería) si empezara otra vez

Envolver la llamada de reseteo de contraseña en un objeto `Command` con cola
y reintento, cuando esa acción por diseño de negocio *debe* fallar
inmediata y visiblemente si el email no se puede enviar — no tiene sentido
reintentarla en segundo plano ni encolarla, el usuario está esperando la
respuesta en pantalla. Aplicar Command ahí no aportó resiliencia real, solo
demoró el error que el usuario necesitaba ver de inmediato.
