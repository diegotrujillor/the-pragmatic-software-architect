# 01. Por qué existe

## Nivel 1 — ELI20

Imagina el backend de ParcelFlow escrito de la forma más directa posible: un
controlador de Express que recibe el request de "registrar pedido", arma el
`INSERT` de Postgres ahí mismo, y devuelve el JSON. Funciona. Se siente
productivo. El problema no aparece hoy — aparece el día que:

- Quieres testear "registrar pedido rechaza cantidades negativas" sin
  levantar una base de datos real.
- Cambias de Postgres a otro motor y la lógica de negocio está entrelazada
  con `pg` en cuarenta archivos.
- Necesitas la misma regla de negocio ("un paquete no puede exceder el peso
  máximo del pedido") desde el endpoint REST y desde un job de importación
  masiva, y hoy vive pegada al controlador de un solo endpoint.

Clean Architecture existe para que ese día nunca llegue como sorpresa: separa
"qué hace el negocio" de "con qué herramienta está implementado hoy", para
que cambiar la herramienta no signifique reescribir el negocio.

## Nivel 2 — Ingeniería

No nació de una sola idea. Es la convergencia de varias corrientes que
llegaron a la misma conclusión por caminos distintos:

- **Hexagonal Architecture / Ports & Adapters** (Alistair Cockburn, 2005) —
  el sistema tiene un núcleo y "puertos"; cualquier tecnología externa entra
  por un "adaptador".
- **Onion Architecture** (Jeffrey Palermo, 2008) — capas concéntricas, el
  dominio en el centro, la infraestructura en el borde.
- **DCI** (Data, Context, Interaction — Trygve Reenskaug) — separar qué es
  un dato de qué rol cumple en un caso de uso específico.
- **Screaming Architecture** (Robert C. Martin) — la estructura de carpetas
  de nivel superior debería anunciar qué hace el sistema, no qué framework
  usa.

Robert C. Martin unificó estas ideas en 2012 (blog) y las formalizó en
*Clean Architecture* (2017). El diagrama de círculos concéntricos es solo la
ilustración; la regla real, la única que importa mecánicamente, es la
**Dependency Rule**: las dependencias del código fuente solo pueden apuntar
hacia adentro, hacia las políticas de más alto nivel. Nada en un círculo
interior puede saber nada sobre un círculo exterior.

**Ventajas:**
- Independiente de framework: Express, Fastify o lo que exista en 2030 es un
  detalle reemplazable.
- Independiente de UI: la misma lógica sirve a REST, CLI o un worker batch.
- Independiente de base de datos: Postgres, otro motor, o memoria en tests —
  el caso de uso no lo sabe.
- Testeable sin infraestructura: el núcleo se prueba con objetos en memoria.

**Desventajas:**
- Más indirección: una interfaz y su implementación en vez de una función.
- Curva de entrada: alguien que solo ha hecho CRUDs ve "ceremonia" donde hay
  una razón.
- Costo real si no hay múltiples casos de uso o múltiples adaptadores que
  justifiquen la separación.

## Nivel 3 — Senior Engineer

Un senior no aplica la regla porque "es la forma correcta" — la aplica
midiendo tres cosas:

1. **Volatilidad de la dependencia externa.** ¿Postgres, el proveedor cloud,
   o el framework HTTP tienen probabilidad real de cambiar en la vida del
   proyecto? Si la respuesta es "nunca, y lo sé con certeza", la abstracción
   cuesta más de lo que ahorra.
2. **Reutilización del caso de uso.** ¿La misma regla de negocio se invoca
   desde más de un punto de entrada (API, cron, CLI, otro servicio)? Si es
   sí, separar la regla del punto de entrada deja de ser opcional.
3. **Costo de no poder testear sin infraestructura.** Si cada test de lógica
   de negocio necesita una base de datos real levantada, el ciclo de
   feedback se degrada con el tiempo — eso es una señal de que la Dependency
   Rule ya se está violando y cobrando factura.

La decisión de un senior casi nunca es "todo o nada". Es aplicar la regla
donde el costo de no aplicarla ya es visible, y dejar el resto simple.

## Nivel 4 — Software Architect

Frente a un CTO, el argumento no es "es una buena práctica" — es riesgo y
costo total de propiedad:

- **Riesgo de vendor lock-in.** Si la lógica de negocio importa el SDK de un
  proveedor cloud directamente, migrar de proveedor deja de ser un problema
  de infraestructura y se convierte en un problema de reescribir negocio.
  Esto no es hipotético: cualquier equipo que declare como principio
  "infraestructura migratable" necesita que todo el stack pueda moverse de
  proveedor con costo acotado — y eso es imposible si el negocio depende del
  SDK de un proveedor específico.
- **Velocidad de release.** Un núcleo testeable sin infraestructura acelera
  CI; uno que no lo es, lo degrada linealmente con cada feature nueva.
- **Costo de la decisión es asimétrico en el tiempo.** Corregir la dirección
  de las dependencias en el mes 1 cuesta una tarde. Corregirla en el mes 18,
  con treinta casos de uso acoplados a un ORM específico, cuesta un
  trimestre y un freeze de features.

El argumento que convence a un CTO no es arquitectónico — es financiero:
pagar un poco de indirección ahora, o pagar una reescritura completa cuando
la dependencia externa cambie y ya no sea opcional.

## ❌ Mitos

**"Clean Architecture existe para tener 100% de cobertura de tests."** No.
Existe para que el negocio no dependa de la infraestructura. La testeabilidad
es una *consecuencia* de esa separación, no el objetivo en sí. Se puede tener
100% de cobertura sobre un desastre acoplado (con mocks agresivos) y 0%
sobre un núcleo perfectamente separado recién escrito.

## ❌ Anti-patrones

Tener la carpeta `domain/` correcta y, dentro de un caso de uso, importar
`express` "solo para leer un header, ya lo saco después" — nunca se saca.
La Dependency Rule no se mide por la estructura de carpetas; se mide por qué
importa qué, línea por línea.

## 🤖 Cómo cambió la IA este concepto

Antes, entender "por qué existe" requería leer el libro completo o vivir el
dolor de una migración fallida. Hoy, un asistente como Claude Code puede
generar el andamiaje de puertos y adaptadores en minutos — pero no puede
decidir por ti si, en *tu* sistema, la regla de negocio de "un paquete no
excede el peso del pedido" se va a invocar algún día desde un segundo punto
de entrada. Ese juicio sigue siendo humano y sigue siendo el contenido real
de este capítulo: la mecánica se automatizó, el "por qué" no.

## Caso real (ParcelFlow)

Supongamos que ParcelFlow declara, como principio de ingeniería, "no vendor
lock-in cuando sea razonable" e "infraestructura migratable con costo
acotado". Si los casos de uso del backend importaran el SDK de un proveedor
cloud directamente, o construyeran queries acopladas a una extensión
específica de Postgres, esos dos principios dejarían de cumplirse en la
práctica aunque estuvieran escritos en un documento. Clean Architecture es el
mecanismo concreto que hace cumplibles esos principios, no una capa
decorativa encima.

## Errores que cometí (o cometería) si empezara otra vez

Tratar "por qué existe" como una sección académica para saltarse, e ir
directo a copiar la estructura de carpetas de un tutorial. El resultado es
cargo-culting: se paga el costo completo de la indirección (interfaces,
inyección de dependencias, mapeos) sin entender qué problema resuelve, y por
lo tanto sin saber cuándo *no* aplicarla. Entender el "por qué" primero es lo
que permite, más adelante, decidir con criterio dónde la regla no se gana su
lugar.
