# The Pragmatic Software Architect

**From CRUD Developer to Software Architect in the AI Era (2026 Edition)**

No es un resumen. Es el manual que me habría gustado tener cuando empecé a
trabajar como ingeniero de software — con el impacto de la IA en el
desarrollo moderno incorporado, no como nota al pie.

## Filosofía

El objetivo no es enseñar a programar. Es enseñar a **pensar como arquitecto**:
que al terminar la colección puedas leer un proyecto y decir con criterio
propio "esto está sobreingenierizado", "aquí falta DDD" o "aquí Clean
Architecture no aporta nada" — en vez de seguir recetas.

Cada afirmación importante está respaldada por referencias (Fowler, Evans,
Robert C. Martin, Sam Newman, documentación oficial de Microsoft/AWS/Google,
etc.) traducidas a lenguaje práctico y moderno.

## Estructura de cada capítulo

Cuatro niveles de profundidad:

| Nivel | Nombre | Pregunta que responde |
|---|---|---|
| 1 | ELI20 | "Explícamelo como si fuera un ingeniero que viene de CRUDs." |
| 2 | Ingeniería | Arquitectura, ventajas, desventajas, ejemplos. |
| 3 | Senior Engineer | ¿Cómo decidiría alguien con diez años de experiencia? |
| 4 | Software Architect | ¿Cómo se justifica frente al CTO? |

Secciones que ningún libro trae:

- **❌ Mitos** — creencias comunes que no resisten un caso real.
- **❌ Anti-patrones** — ej. Repository → Service → Manager → Helper → Util →
  Factory → Builder → Facade → Controller. Pasa muchísimo.
- **🤖 Cómo cambió la IA este concepto** — antes vs. hoy (Claude Code escribe
  un Factory en segundos; lo importante ya no es escribirlo, es saber cuándo).
- **Casos reales** — todos aterrizados en un mismo sistema ficticio de
  ejemplo (Merchant → Warehouse → Order → Package → Delivery), nunca bancos
  ni tiendas online genéricas.
- **Errores que cometí (o cometería) si empezara otra vez** — microservicios
  prematuros, 150 interfaces innecesarias, SOLID aplicado a todo, capas desde
  el día uno.

## Volúmenes

1. Clean Architecture
2. Domain-Driven Design
3. Design Patterns
4. Testing
5. Microservices

Ver [`SUMMARY.md`](SUMMARY.md) para el índice detallado y
[`REFERENCES.md`](REFERENCES.md) para el libro canónico detrás de cada
volumen.

## Flujo de trabajo del proyecto

- **Autor (contenido):** redacción, estructura, ejemplos, decisiones técnicas.
- **Editor (repo):** estructura de carpetas, MkDocs Material, CI/CD, export a
  PDF/EPUB, publicación en GitHub Pages, commits.

Avanzamos un capítulo a la vez: se revisa, se ajusta, y solo entonces se pasa
al siguiente. Nada de generar todo de una sola vez sin refinamiento.

## Herramientas

```bash
pip install mkdocs-material
mkdocs serve   # preview local
mkdocs build   # sitio estático
```

Export a PDF/EPUB vía GitHub Actions (`.github/workflows/`) al hacer push a
`main` o al crear un tag.
