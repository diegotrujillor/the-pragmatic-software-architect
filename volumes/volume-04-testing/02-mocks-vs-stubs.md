# 02. Mocks vs. Stubs

## Nivel 1 — ELI20

Dos formas de reemplazar un colaborador real en un test, y confundirlas es
la causa más común de tests frágiles:

- **Stub**: le da al código bajo prueba una respuesta falsa cuando le
  pregunta algo. "Si me preguntas el precio, te digo 100." No te importa
  si le preguntaron una vez o diez.
- **Mock**: verifica que el código bajo prueba *hizo algo* — que llamó a un
  método específico, con argumentos específicos. "Comprueba que sí se
  llamó a `enviar email`."

La regla práctica: usa Stub cuando te importa el resultado del test (qué
devolvió la función). Usa Mock solo cuando lo único que puedes observar es
que una acción ocurrió — típicamente una llamada a un sistema externo que
no tiene un resultado observable propio.

## Nivel 2 — Ingeniería

```typescript
// Stub: reemplaza una consulta, el test verifica el RESULTADO
class StubOrderRepository implements OrderRepository {
  async findById(id: string): Promise<Order> {
    return new Order(id, /* weightLimitKg */ 10); // respuesta fija
  }
  async save(order: Order): Promise<void> {} // no verificado, solo no debe fallar
}

test("rejects delivering an already-cancelled order", async () => {
  // Arrange
  const useCase = new MarkPackageDeliveredUseCase(
    new StubOrderRepository(),
    /* ... */
  );

  // Act & Assert — se verifica el RESULTADO devuelto, no cómo se obtuvo
  await expect(useCase.execute({ orderId: "cancelled-order" }))
    .rejects.toThrow("Pedido cancelado");
});
```

```typescript
// Mock: el test verifica que la ACCIÓN ocurrió, porque no hay resultado
// observable de otra forma — enviar un email no devuelve nada útil.
test("sends delivery notification when order is fulfilled", async () => {
  // Arrange
  const emailService = { send: jest.fn() }; // mock
  const useCase = new MarkPackageDeliveredUseCase(/* ... */, emailService);

  // Act
  await useCase.execute({ orderId: "order-1", packageId: "pkg-1" });

  // Assert — aquí sí importa que se llamó, es la única forma de saberlo
  expect(emailService.send).toHaveBeenCalledWith("order-1");
});
```

El error que rompe la propiedad #2 de Khorikov (resistencia a refactors,
Vol. 4 cap. 00) es usar Mock donde bastaba un Stub: verificar *cómo*
`OrderRepository.findById()` fue llamado, cuando lo único que le importa al
test es *qué* devolvió. Si mañana el caso de uso llama `findById()` dos
veces por una optimización interna que no cambia el resultado, un test con
Mock sobre esa llamada se rompe sin que ningún comportamiento de negocio
haya cambiado.

## Nivel 3 — Senior Engineer

La pregunta de un senior antes de escribir `expect(x).toHaveBeenCalledWith(...)`:
**¿esta llamada tiene un efecto observable de otra forma, o la llamada
misma ES el comportamiento que me importa verificar?** Guardar un pedido en
un repositorio tiene un efecto observable — se puede volver a leer y
comparar. Enviar un email a un proveedor externo no tiene ese lujo dentro
del test — la llamada misma, con sus argumentos, es la única evidencia
disponible de que el comportamiento correcto ocurrió.

Un senior reserva Mocks casi exclusivamente para **colaboradores en el
borde del sistema** — servicios externos sin estado observable desde el
test (email, SMS, webhooks). Para colaboradores internos (repositorios,
otras entidades), casi siempre Stub o una implementación real en memoria es
la opción más resistente a refactors.

## Nivel 4 — Software Architect

El argumento frente al CTO: una suite con exceso de Mocks es una suite que
**documenta la implementación de hoy, no el comportamiento del negocio**.
Cada refactor legítimo de implementación interna — cambiar el orden de dos
llamadas, combinar dos consultas en una, cambiar de repositorio — rompe
tests con Mocks aunque el comportamiento observable no haya cambiado en
absoluto. Eso convierte cada refactor en una sesión de "arreglar tests" que
no protegen nada, un costo recurrente que crece con cada sprint y que el
equipo termina evitando — congelando la calidad del código porque refactorizar
"cuesta demasiado" en tiempo de tests roto.

## ❌ Mitos

**"Mockear todas las dependencias es una buena práctica de aislamiento."**
No necesariamente. Aislamiento total suena riguroso pero, cuando se aplica
sin distinguir Mock de Stub, produce tests que verifican la cadena exacta
de llamadas internas en vez del comportamiento de negocio — el
anti-patrón central de este capítulo, ver Volumen 4, capítulo 00.

## ❌ Anti-patrones

**Mock donde bastaba un Stub.** `expect(orderRepository.findById).toHaveBeenCalledWith("order-1")`
cuando lo único relevante para el test es qué devolvió `findById()`, no
cuántas veces ni con qué argumentos exactos se le llamó. Este único hábito,
aplicado sistemáticamente en un proyecto, es la causa más común de "cada
refactor rompe cincuenta tests que no tienen nada que ver con lo que
cambié."

## 🤖 Cómo cambió la IA este concepto

Un asistente, al generar un test, por defecto tiende a mockear *todos* los
colaboradores por igual — es el patrón más común en el código de
entrenamiento, no necesariamente el más correcto. Pedirle explícitamente
"usa un Stub para el repositorio, verifica solo el resultado; usa Mock solo
para el email, que no tiene resultado observable" produce una suite mucho
más resistente a refactors que pedir genéricamente "agrega tests para esto".
La distinción Mock/Stub sigue siendo una decisión de diseño que hay que
comunicarle a la IA, no una que ella infiera sola.

## Caso real (ParcelFlow)

`MarkPackageDeliveredUseCase` se testea con `StubOrderRepository` y
`StubPackageRepository` (verificando el `Response` devuelto), pero con un
Mock real sobre `DomainEventBus.publish()` — porque la única forma de
verificar "se publicó `OrderFulfilled`" es comprobar que la llamada
ocurrió, no hay un estado alternativo que leer después. Esa distinción,
aplicada consistentemente en toda la suite, es la razón por la que un
refactor grande de `PgOrderRepository` (cambiar de queries individuales a
batch) no rompió un solo test de caso de uso — el comportamiento observado
no cambió, solo la implementación interna.

## Errores que cometí (o cometería) si empezara otra vez

Mockear `OrderRepository` en cada test de caso de uso, verificando llamadas
exactas a `findById` y `save`, en vez de usar un Stub o una implementación
en memoria real. El día que optimizamos `RegisterOrderUseCase` para
validar inventario antes de consultar el repositorio — cambiando el orden
de dos llamadas sin cambiar ningún resultado de negocio — se rompieron
catorce tests, ninguno de los cuales detectaba un bug real. Migrar esos
tests a Stubs, verificando solo el `Response` final, los hizo sobrevivir
ese mismo refactor sin ningún cambio.
