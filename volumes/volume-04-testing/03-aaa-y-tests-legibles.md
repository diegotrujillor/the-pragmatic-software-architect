# 03. AAA y tests legibles

## Nivel 1 — ELI20

Arrange-Act-Assert: prepara el escenario, ejecuta la acción, verifica el
resultado. Tres bloques, en ese orden, separados visualmente. No es una
regla burocrática — es la diferencia entre un test que se lee en cinco
segundos y uno que hay que rastrear línea por línea para entender qué
demonios está probando.

```typescript
test("cancels a pending order", () => {
  // Arrange
  const order = new Order("order-1", 10);

  // Act
  order.cancel();

  // Assert
  expect(order.status).toBe("cancelled");
});
```

## Nivel 2 — Ingeniería

Cada bloque tiene una responsabilidad única, y mezclarlos es lo que produce
tests ilegibles:

- **Arrange** — construye el estado inicial. Si este bloque necesita más de
  cinco o seis líneas, casi siempre es una señal de que falta un test
  builder o una función helper de construcción (`buildOrder({ weightLimitKg: 10 })`),
  no que el test esté "bien detallado".
- **Act** — idealmente una sola línea, una sola llamada. Si el bloque Act
  tiene múltiples pasos, el test probablemente está verificando más de un
  comportamiento a la vez — candidato a dividirse en varios tests más
  angostos.
- **Assert** — verifica el resultado observable, no la implementación
  interna (ver capítulo 02, Mocks vs. Stubs). Múltiples `expect()` están
  bien si verifican distintas facetas del *mismo* resultado — no si
  verifican comportamientos no relacionados que deberían ser tests
  separados.

El nombre del test importa tanto como su estructura. `test("order cancel works")`
no dice nada; `test("cancels a pending order")` o, mejor,
`test("rejects cancelling an already-fulfilled order")` describe el
comportamiento exacto sin necesidad de leer el cuerpo del test — el nombre
mismo funciona como documentación viva de una regla de negocio.

## Nivel 3 — Senior Engineer

Un senior trata el bloque Arrange como la primera señal de un diseño
problemático. Si construir el escenario para un test unitario de una sola
regla de negocio requiere quince líneas de configuración, encadenar cuatro
mocks, y levantar tres colaboradores — el problema casi nunca es el test.
Es que la clase bajo prueba tiene demasiadas dependencias, o mezcla
demasiadas responsabilidades (recordar el God Service del Volumen 1,
capítulo 03) — el test simplemente hizo visible un problema de diseño que
ya existía.

Otra decisión de senior: **un test, un comportamiento**. Un solo `test()`
que verifica cinco cosas distintas con nombres genéricos (`test("order works")`)
es difícil de leer cuando falla — no queda claro cuál de las cinco
aserciones fue la que falló ni por qué, sin abrir el código del test.

## Nivel 4 — Software Architect

El argumento frente al CTO: **los tests son la documentación que nunca se
desactualiza**, porque si se desactualiza, falla — a diferencia de un
comentario o un documento de diseño que puede mentir silenciosamente
durante años. Un nombre de test bien escrito (`rejects cancelling an
already-fulfilled order`) es, en la práctica, una especificación ejecutable
de una regla de negocio. Un ingeniero nuevo puede leer los nombres de la
suite completa de `Order` y entender las reglas de negocio del sistema sin
abrir el código de producción — eso reduce el costo de onboarding de forma
medible, y es un activo que se mantiene correcto por construcción, no por
disciplina de mantenimiento de documentación.

## ❌ Mitos

**"Los tests no necesitan ser 'código limpio', solo necesitan pasar."** No.
Un test es código de producción con la misma vida útil que el código que
prueba — se lee, se modifica, se debuggea bajo presión cuando algo falla en
CI a las 6 p.m. Un test desordenado, sin AAA claro, con nombres genéricos,
cuesta tiempo real cada vez que alguien necesita entenderlo — que es
exactamente el momento en que menos tiempo sobra.

## ❌ Anti-patrones

**El test "kitchen sink".** Un solo `test()` gigante que arma un escenario
complejo y hace quince aserciones sobre cinco comportamientos distintos sin
relación directa entre sí. Cuando falla, el mensaje de error dice "el test
X falló" — sin indicar cuál de los quince comportamientos realmente se
rompió, obligando a leer el test completo, línea por línea, bajo presión,
para diagnosticar algo que un test bien dividido habría señalado en el
nombre mismo del test que falló.

## 🤖 Cómo cambió la IA este concepto

Un asistente genera tests con estructura AAA correcta casi por defecto, si
se le pide — la mecánica de formato ya no es fricción real. Lo que sigue
siendo trabajo humano: decidir dónde termina un test y empieza el
siguiente. Es común que un asistente, al generar tests para una función,
los agrupe todos en un solo `test()` largo en vez de dividir por
comportamiento — hay que pedir explícitamente "un test por comportamiento,
con nombre descriptivo de la regla" para obtener la granularidad que
realmente ayuda cuando algo falla en producción meses después.

## Caso real (ParcelFlow)

La suite de `Order` tiene test builders (`buildOrder({ weightLimitKg: 10 })`)
que reducen cualquier bloque Arrange a una línea, incluso para escenarios
complejos con múltiples paquetes ya agregados. Cada regla de negocio tiene
su propio test con nombre explícito: `rejects package exceeding weight
limit`, `allows cancelling a pending order`, `rejects cancelling a
fulfilled order` — la lista completa de nombres de test de `Order.test.ts`,
leída de corrido, es prácticamente la especificación de las reglas de
negocio del Aggregate, sin necesidad de abrir el código.

## Errores que cometí (o cometería) si empezara otra vez

Escribir un solo test gigante `test("Order behaves correctly")` que armaba
un pedido con paquetes, lo cancelaba, verificaba el peso, y probaba tres
casos límite distintos en el mismo bloque Assert. Cuando algo se rompió en
CI, el reporte solo decía que "Order behaves correctly" había fallado, sin
ninguna pista de cuál de los seis comportamientos era el culpable —
perdí más tiempo diagnosticando el test que arreglando el bug real. Dividir
ese único test en seis, con nombres específicos, habría señalado el
problema exacto en el reporte de CI, sin tener que abrir el archivo.
