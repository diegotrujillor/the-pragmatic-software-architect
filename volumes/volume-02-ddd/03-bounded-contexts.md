# 03. Bounded Contexts

## Nivel 1 — ELI20

La misma palabra significa cosas distintas en distintas partes de una
empresa, y eso está bien — el problema es fingir que significa una sola
cosa. En ParcelFlow, "Cliente" para el equipo de Ventas es una empresa con
un contrato y un límite de crédito. Para el equipo de Soporte, "Cliente" es
una persona con un historial de tickets y un número de teléfono. Ambos
tienen razón dentro de su contexto — el error es construir una única clase
`Customer` que intenta cargar los campos de los dos significados a la vez.

Un **Bounded Context** es la frontera explícita donde una palabra tiene un
significado — y solo uno. Cruzar esa frontera significa traducir, no
reutilizar la misma clase.

## Nivel 2 — Ingeniería

Dentro de un Bounded Context, todo el equipo — código, conversación,
tickets, diagramas — usa el mismo vocabulario exacto: el **Ubiquitous
Language**. Si el experto de dominio dice "despachar", el método se llama
`dispatch()`, no `process()` ni `handle()`. Esa disciplina es lo que hace
que el código sea legible para alguien de negocio sin traducción mental.

Cuando dos Bounded Contexts necesitan comunicarse, DDD define patrones
explícitos de **Context Mapping** en vez de dejarlo implícito:

- **Shared Kernel** — ambos contextos comparten literalmente un subconjunto
  de modelo (arriesgado: cualquier cambio requiere coordinar dos equipos).
- **Customer/Supplier** — un contexto depende de otro, y el downstream
  puede influir en el roadmap del upstream (relación con negociación).
- **Conformist** — el downstream se adapta al modelo del upstream tal cual
  es, sin negociar cambios (típico al integrar con un proveedor externo).
- **Anti-Corruption Layer (ACL)** — el downstream traduce el modelo del
  upstream a su propio vocabulario en la frontera, para que un cambio en el
  sistema externo no contamine el modelo interno.

```typescript
// Anti-Corruption Layer: el proveedor de envíos externo llama a los
// paquetes "shipment_items" con su propio esquema — ParcelFlow no adopta
// ese vocabulario, lo traduce en la frontera.
class CarrierApiAdapter {
  toPackage(externalShipmentItem: CarrierShipmentItemDto): Package {
    return Package.create({
      id: externalShipmentItem.item_ref,
      weightKg: externalShipmentItem.weight_grams / 1000,
    });
  }
}
```

## Nivel 3 — Senior Engineer

Un senior no dibuja Bounded Contexts basándose en la estructura de carpetas
que "se ve bien". Los identifica buscando **fricción real de vocabulario**:
lugares donde el mismo sustantivo, en dos partes del sistema, ya está
generando confusión, bugs de mapeo, o reuniones donde dos equipos descubren
que hablaban de cosas distintas usando la misma palabra. Esa fricción,
cuando ya existe, es la señal — no una sospecha teórica de que "podría"
existir.

Trazar un Bounded Context demasiado pronto, sin esa fricción observada, es
el mismo error que separar capas antes de tener un segundo caso de uso: se
paga el costo de la traducción (ACL, DTOs, mapeos) sin tener todavía un
segundo significado real que proteger.

## Nivel 4 — Software Architect

Frente al CTO, los Bounded Contexts son el argumento estratégico para
**cómo se organizan los equipos**, no solo el código — esta es la
observación central de la Ley de Conway: la estructura de un sistema
termina reflejando la estructura de comunicación de la organización que lo
construye, lo decidas explícitamente o no. Trazar los Bounded Contexts a
propósito es decidir esa estructura con intención, en vez de heredarla por
accidente del organigrama.

También es el argumento correcto para **cuándo un Bounded Context se
convierte en un servicio separado**: cuando el contexto tiene su propio
ciclo de release, su propio equipo dueño, y una Anti-Corruption Layer ya
demostrada en el monolito. Extraerlo a esa altura es mover un límite que ya
existe y ya funciona — no inventar uno nuevo bajo presión de un rediseño de
infraestructura (ver Volumen 5, Microservices).

## ❌ Mitos

**"Un Bounded Context es lo mismo que un microservicio."** No. Un Bounded
Context es un límite de *modelo y lenguaje* — puede vivir perfectamente
como un módulo dentro de un monolito, con su propia carpeta y su propio
vocabulario interno, sin necesitar su propio proceso, su propio deploy ni
su propia base de datos. La decisión de separarlo en un servicio es
independiente y posterior (Volumen 5).

## ❌ Anti-patrones

**Un solo modelo `Customer` compartido entre todos los subsistemas**,
acumulando campos de cada contexto que alguna vez lo tocó (`creditLimit`
para Ventas, `supportTier` para Soporte, `shippingPreference` para
Logística) hasta convertirse en una clase de cincuenta campos donde nadie
sabe qué parte es segura de modificar sin romper otro subsistema. Es el
resultado inevitable de negarse a trazar fronteras explícitas.

## 🤖 Cómo cambió la IA este concepto

Un asistente puede generar una Anti-Corruption Layer completa —
`CarrierApiAdapter` de arriba, con su mapeo — en minutos, una vez que le
describes los dos vocabularios a traducir. Lo que no puede hacer por ti: **
detectar que dos equipos están usando la misma palabra para cosas distintas
**. Esa fricción vive en conversaciones humanas, en tickets confusos, en
reuniones donde alguien dice "espera, ¿tu 'cliente' es igual al mío?" — y
solo se identifica participando de esas conversaciones, no leyendo el
código.

## Caso real (ParcelFlow)

ParcelFlow tiene, al menos, dos Bounded Contexts reales: **Fulfillment**
(donde vive todo lo visto en este volumen: `Order`, `Package`, `Delivery`)
y **Billing** (donde un "pedido" solo importa como línea de una factura, con
campos de impuestos y moneda que a Fulfillment no le interesan). Ambos
comparten el concepto "pedido" — pero cada uno lo modela distinto, y se
comunican por `orderId` más una Anti-Corruption Layer, nunca compartiendo la
clase `Order` directamente entre los dos contextos.

## Errores que cometí (o cometería) si empezara otra vez

Compartir la misma clase `Order` entre el módulo de Fulfillment y el de
Billing "para no duplicar código", en vez de tener dos modelos más chicos y
una traducción explícita entre ambos. Cada requisito nuevo de facturación
terminaba agregando campos a una clase que Fulfillment no necesitaba y no
entendía, hasta que ningún cambio era seguro sin revisar ambos equipos.
Duplicar el concepto en dos modelos pequeños, conectados por `id`, habría
sido más código pero muchísimo menos acoplamiento.
