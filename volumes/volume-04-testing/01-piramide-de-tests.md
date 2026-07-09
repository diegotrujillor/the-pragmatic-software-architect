# 01. La pirámide de tests

## Nivel 1 — ELI20

No todos los tests cuestan lo mismo. Un test unitario (una función, sin
base de datos, sin red) corre en milisegundos. Un test de integración
(contra una base de datos real) corre en cientos de milisegundos o
segundos. Un test end-to-end (levanta la app entera, simula un usuario)
corre en segundos o minutos. La pirámide dice: escribe muchos del primer
tipo, menos del segundo, pocos del tercero — porque el costo crece por
capa, y el tipo de bug que cada capa atrapa no siempre justifica ese costo.

## Nivel 2 — Ingeniería

Cada nivel de la pirámide protege contra una clase distinta de bug:

- **Unitarios** — reglas de negocio puras. `Order.addPackage()` rechaza
  exceder el límite de peso (Volumen 2). No tocan Postgres, no tocan
  Express — corren con un `InMemoryOrderRepository` o directamente sobre
  la entidad. Cientos de estos deberían correr en segundos.

```typescript
test("rejects package that exceeds order weight limit", () => {
  // Arrange
  const order = new Order("order-1", /* weightLimitKg */ 10);
  const heavyPackage = Package.create({ weightKg: 15 });

  // Act & Assert
  expect(() => order.addPackage(heavyPackage)).toThrow(
    "Excede el límite de peso del pedido"
  );
});
```

- **Integración** — que las piezas reales se conecten correctamente:
  `PgOrderRepository` de verdad guarda y recupera un `Order` desde Postgres,
  con el mapeo de columnas correcto. Estos SÍ tocan infraestructura real
  (típicamente contra un contenedor Docker desechable), y por eso son más
  lentos y menos numerosos.

- **End-to-end** — el flujo completo desde el punto de entrada externo:
  un cliente HTTP real llama `POST /orders`, y se verifica la respuesta.
  Reservados para los flujos críticos de negocio ("un cliente puede
  registrar un pedido y recibir confirmación"), no para cada variación de
  regla — esas ya las cubrieron los unitarios.

La forma de pirámide (muchos unitarios, menos integración, pocos e2e) no es
una proporción estética — es consecuencia directa de que los unitarios son
los únicos capaces de cubrir exhaustivamente las combinaciones de reglas de
negocio a un costo que se mantiene barato incluso con cientos de ellos.

## Nivel 3 — Senior Engineer

Un senior no persigue una proporción numérica exacta ("70% unitarios, 20%
integración, 10% e2e") — persigue la pregunta detrás de la pirámide:
**¿qué nivel detecta este bug específico de la forma más rápida y barata
posible?** Una regla de negocio pura pertenece a un test unitario, sin
excepción — probarla con un test end-to-end es válido pero carísimo y
lento comparado con la alternativa que ya la cubre en milisegundos.

La inversión de la pirámide — un "cono de helado", muchos e2e lentos y
pocos unitarios — es la señal de alerta más común en proyectos que
crecieron rápido sin disciplina de testing: la suite tarda veinte minutos,
falla de forma intermitente por razones de infraestructura ajenas a la
lógica de negocio, y el equipo empieza a saltarse tests localmente antes de
hacer push.

## Nivel 4 — Software Architect

El argumento frente al CTO es **tiempo de feedback como multiplicador de
velocidad de todo el equipo**: una suite mayormente unitaria da un
veredicto en segundos, en la máquina local del ingeniero, antes incluso de
hacer commit. Una suite mayormente e2e da el mismo veredicto en CI, minutos
después, con el ingeniero ya cambiando de contexto a otra tarea — el costo
no es solo el tiempo de CI, es el costo de recuperar el contexto cuando
finalmente llega el resultado.

La proporción correcta de la pirámide, entonces, no es una regla de estilo
— es una decisión de inversión de tiempo de ingeniería medible: cada test
e2e que podría haber sido unitario es tiempo de feedback regalado, para
todo el equipo, en cada ejecución, durante toda la vida del proyecto.

## ❌ Mitos

**"Los tests end-to-end dan más confianza porque prueban 'todo junto', así
que son mejores tests."** Dan un tipo distinto de confianza — que el
sistema ensamblado funciona — no una confianza superior. Un bug en la regla
"un pedido no puede exceder el límite de peso" se detecta igual de bien (y
mucho más rápido) con un test unitario directo sobre `Order` que con un
flujo e2e completo que además depende de que la base de datos, la red y el
servidor HTTP estén sanos.

## ❌ Anti-patrones

**El cono de helado.** Una suite con pocos tests unitarios y una montaña de
tests end-to-end lentos, porque "probar por la API se siente más real".
El resultado típico: una suite de veinte minutos, resultados intermitentes
por razones de infraestructura no relacionadas con el bug real, y un equipo
que deja de confiar en la suite — que es, en la práctica, lo mismo que no
tener tests.

## 🤖 Cómo cambió la IA este concepto

Pedirle a un asistente "genera tests para esta función" con frecuencia
produce, por defecto, el nivel equivocado — un test que levanta toda la
aplicación para probar una regla que un test unitario de tres líneas ya
cubriría. La IA no conoce la pirámide de tu proyecto a menos que se lo
digas explícitamente: "este es un test unitario, sin infraestructura" tiene
que ser parte del prompt, no una suposición. El criterio de en qué nivel
pertenece cada test sigue siendo una decisión humana, aplicada
conscientemente cada vez que se pide un test nuevo.

## Caso real (ParcelFlow)

La suite de ParcelFlow tiene alrededor de 300 tests unitarios sobre
entidades y casos de uso (corren en menos de cinco segundos, en cada
guardado del editor), 40 tests de integración sobre repositorios reales
contra Postgres en Docker (corren en CI, unos 90 segundos), y 8 tests
end-to-end sobre los flujos críticos: registrar pedido, marcar entregado,
cancelar pedido, y checkout completo con descuentos aplicados. Los bugs de
regla de negocio se atrapan casi siempre en el primer nivel, antes de que
el código llegue a CI.

## Errores que cometí (o cometería) si empezara otra vez

Escribir el primer año de tests casi enteramente como end-to-end, "porque
prueban el sistema real". La suite completa tardaba doce minutos para
cubrir reglas de negocio que, individualmente, se hubieran probado en
milisegundos como tests unitarios. Migrar la mayoría a unitarios, casi un
año después, redujo el tiempo de feedback de doce minutos a ocho segundos
para el mismo conjunto de reglas — tiempo que se había estado regalando,
en cada ejecución, durante todo ese año.
