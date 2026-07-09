# 01. Strategy y Factory Method

## Nivel 1 — ELI20

**Strategy**: en vez de un `if/else` (o peor, un `switch`) que crece cada vez
que aparece una variante nueva de una regla, cada variante se convierte en
su propia clase intercambiable. **Factory Method**: en vez de que el código
que *usa* un objeto también sepa *cómo construirlo* con todos sus detalles,
esa responsabilidad se aísla en un solo lugar. Los dos patrones existen por
la misma razón: cuando algo varía, aislar esa variación en su propio sitio
evita que el resto del código tenga que conocerla.

## Nivel 2 — Ingeniería

**Strategy** — ParcelFlow necesita elegir transportista según peso, destino
y urgencia. Sin el patrón, esa lógica crece como un árbol de condicionales
dentro del caso de uso:

```typescript
// Sin Strategy: crece con cada transportista nuevo
function selectCarrier(pkg: Package): Carrier {
  if (pkg.weightKg > 20) return new HeavyCargoCarrier();
  if (pkg.destination.isInternational()) return new InternationalCarrier();
  if (pkg.isUrgent) return new ExpressCarrier();
  return new StandardCarrier();
}
```

Con Strategy, cada regla es su propia clase que implementa un contrato
común, y la selección se delega a algo que sabe evaluarlas:

```typescript
interface CarrierStrategy {
  appliesTo(pkg: Package): boolean;
  carrier(): Carrier;
}

class HeavyCargoStrategy implements CarrierStrategy {
  appliesTo(pkg: Package) { return pkg.weightKg > 20; }
  carrier() { return new HeavyCargoCarrier(); }
}

class CarrierSelector {
  constructor(private readonly strategies: CarrierStrategy[]) {}

  select(pkg: Package): Carrier {
    const match = this.strategies.find((s) => s.appliesTo(pkg));
    return (match ?? this.defaultStrategy).carrier();
  }
}
```

Agregar un transportista nuevo significa agregar una clase — no editar una
función central que ya usan todos los transportistas existentes.

**Factory Method** — construir un `Package` correcto depende del tipo de
mercancía declarado (`fragile`, `perishable`, `standard`), cada uno con
reglas de empaque distintas. En vez de que el caso de uso conozca esas
reglas, un Factory las encapsula:

```typescript
class PackageFactory {
  create(type: PackageType, data: PackageData): Package {
    switch (type) {
      case "fragile": return Package.withFragileHandling(data);
      case "perishable": return Package.withColdChain(data);
      default: return Package.standard(data);
    }
  }
}
```

El `switch` no desaparece — se mueve a un único lugar responsable de
construcción, en vez de repetirse en cada punto del código que necesita
crear un `Package`.

## Nivel 3 — Senior Engineer

Un senior aplica Strategy cuando **ya existen dos o más variantes reales**
de la misma regla — no antes. Un `if/else` de dos ramas que nunca tuvo una
tercera en la vida del proyecto no necesita convertirse en una interfaz y
dos clases; eso es indirección pagada sin comprador (ver capítulo 04, La
cadena de sinónimos).

Con Factory Method la pregunta de un senior es distinta: **¿la lógica de
construcción es lo suficientemente compleja o dispersa como para que
centralizarla ahorre más de lo que cuesta la capa extra?** Si construir un
`Package` es literalmente `new Package(data)` sin ramas, un Factory es
ceremonia — se usa el constructor directo. El Factory se gana su lugar
cuando construir el objeto correctamente implica decisiones, no solo
asignar campos.

## Nivel 4 — Software Architect

Frente al CTO, el argumento de Strategy es **costo marginal de agregar una
variante de negocio**. Sin el patrón, cada transportista nuevo es un cambio
de riesgo alto: se edita una función que ya usan todos los transportistas
existentes, con posibilidad de romper alguno por accidente. Con el patrón,
agregar un transportista es una clase nueva, aislada, testeable sola, sin
tocar código que ya funciona en producción — el mismo argumento del
Open/Closed Principle que ya apareció con Domain Events (Volumen 2).

El argumento de Factory Method es **consistencia bajo crecimiento de
equipo**: sin un punto único de construcción, cada ingeniero nuevo que
necesita crear un `Package` reinventa las reglas de empaque a su manera, y
la lógica de construcción termina duplicada e inconsistente entre
controladores, jobs y scripts.

## ❌ Mitos

**"Cualquier `if/else` debería ser un Strategy."** No. Strategy paga su
costo cuando las variantes son varias, cambian con frecuencia independiente
entre sí, o necesitan probarse por separado. Un `if/else` de dos ramas
estables durante años es exactamente eso — dos ramas — y convertirlo en un
patrón solo agrega archivos que hay que abrir para entender una decisión de
tres líneas.

## ❌ Anti-patrones

**Factory que no fabrica nada.** Una clase `PackageFactory` con un único
método `create()` que simplemente llama `new Package(data)` sin ninguna
decisión adentro — el patrón sin el problema que lo justifica. Es
indistinguible, en costo, de haber llamado al constructor directamente, pero
con una capa extra que hay que atravesar para entenderlo.

## 🤖 Cómo cambió la IA este concepto

Pedirle a un asistente "convierte este `if/else` en Strategy" produce el
código de arriba en segundos, sintácticamente perfecto. Lo que la IA no
puede decidir por ti es si *deberías* pedirlo — si las variantes que ves hoy
son suficientes y suficientemente inestables como para justificar la
indirección, o si dos ramas de negocio estables no necesitan convertirse en
tres archivos. La tentación de "ya que es gratis, refactorízalo a Strategy"
es real y hay que resistirla activamente — sigue siendo la misma pregunta
del Nivel 3.

## Caso real (ParcelFlow)

`CarrierSelector` (arriba) es el ejemplo completo en producción: cinco
transportistas activos, cada uno con su propia clase `CarrierStrategy`,
agregados incrementalmente a lo largo de un año sin tocar el código de los
transportistas anteriores ni una sola vez. `PackageFactory` centraliza las
tres variantes de empaque (`fragile`, `perishable`, `standard`) en un solo
archivo que cualquier ingeniero nuevo revisa en dos minutos para entender
todas las reglas de construcción del sistema.

## Errores que cometí (o cometería) si empezara otra vez

Construir `CarrierSelector` con Strategy desde el primer transportista —
cuando solo existía `StandardCarrier` y ninguna variante más en el
horizonte. Una función de tres líneas se convirtió en una interfaz, una
clase, y un selector, para un caso que durante los primeros seis meses del
proyecto tuvo exactamente una sola rama. El patrón se ganó su lugar después,
con el segundo transportista real — debí esperar a esa señal en vez de
anticiparla.
