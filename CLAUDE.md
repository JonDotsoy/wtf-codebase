# CLAUDE.md — Guía de contribución para wtf-codebase

Este archivo describe cómo agregar nuevas malas prácticas al repositorio y cómo mantener los archivos de documentación actualizados.

---

## Estructura del proyecto

- `README.md` — Catálogo principal en español
- `README-EN.md` — Catálogo en inglés (debe mantenerse sincronizado con README.md)
- `CLAUDE.md` — Este archivo: instrucciones para contribuir

---

## Cómo agregar un nuevo caso

Cada nuevo escenario o mala práctica debe usar el formato `WTFCODE-<number>`.

El número debe ser consecutivo al último existente en el índice de `README.md`.

### Pasos

1. Agregar una fila al índice en `README.md` y `README-EN.md`
2. Agregar la sección completa al final del catálogo (antes de la sección "Contribuir") en ambos archivos
3. El título de la sección debe seguir el formato `## WTFCODE-<number> - <title>`

---

## Formato de cada caso

Cada entrada debe tener exactamente estas secciones, en este orden:

### `## WTFCODE-<number> - <title>`

Título principal de la mala práctica.

### `### El problema`

Descripción narrativa del problema. Explica el contexto, por qué es tentador caer en este patrón y cuándo empieza a doler.

### `### ✅ Cuándo sí tiene sentido`

Lista de condiciones bajo las cuales el patrón es aceptable o incluso correcto. Ser honesto: no toda práctica es mala en todos los contextos.

### `### ❌ Cuándo no tiene sentido`

Lista de condiciones o señales que indican que el patrón es un problema. Incluir consecuencias concretas.

### `### El código que nadie quiere ver en code review`

Bloque de código real (o muy creíble) que ilustra la mala práctica. Agregar comentarios en el código que expliquen exactamente qué falla.

### `### Las alternativas`

Una o más alternativas concretas con código. Cada alternativa lleva un subtítulo `#### Opción A — ...`.

### `### Resumen`

Una lista de preguntas frecuentes con sus respuestas cortas. Formato:

```markdown
### Resumen

- **¿Cuándo está bien usar X?** Cuando Y y Z se cumplen.
- **¿Qué pasa si X ocurre?** El sistema hace W.
- **¿Hay alguna alternativa más simple?** Sí, considera P o Q.
```

---

## Índice en README.md y README-EN.md

Cada vez que se agrega un nuevo caso, se debe actualizar la tabla del índice en ambos archivos:

```markdown
| WTFCODE-N | [Título del caso](#ancla) | Categoría | 🔥🔥🔥 |
```

Categorías sugeridas (usar las existentes antes de crear nuevas):

- `Persistencia`
- `Concurrencia`
- `Seguridad`
- `Rendimiento`
- `Arquitectura`
- `Testing`
- `Dependencias`

Niveles de daño:

- `🔥` — Molesto
- `🔥🔥` — Problema real
- `🔥🔥🔥` — Catastrófico en producción

---

## Reglas de sincronización README.md / README-EN.md

- Cada sección en español debe tener su equivalente en inglés.
- Las secciones de código no se traducen (los comentarios del código sí).
- Los títulos de sección (`### ✅ When it makes sense`, `### ❌ When it doesn't make sense`, `### Summary`) deben traducirse.
- El orden y la numeración `WTFCODE-<number>` es idéntica en ambos archivos.

---

## Ejemplo mínimo de una nueva entrada

```markdown
## WTFCODE-2 - <Título>

### El problema

<Descripción del problema>

### ✅ Cuándo sí tiene sentido

- <Condición 1>
- <Condición 2>

### ❌ Cuándo no tiene sentido

- <Señal 1>
- <Señal 2>

### El código que nadie quiere ver en code review

\`\`\`js
// Ejemplo de la mala práctica
\`\`\`

### Las alternativas

#### Opción A — <Nombre de la alternativa>

\`\`\`js
// Ejemplo de la alternativa
\`\`\`

### Resumen

- **¿Pregunta común 1?** Respuesta corta.
- **¿Pregunta común 2?** Respuesta corta.
- **¿Pregunta común 3?** Respuesta corta.
```

---

## Notas adicionales

- `WTFCODE-1` ya existe: usar un archivo JSON/YAML como base de datos.
- Todos los ejemplos deben ser reales o muy creíbles; no inventar escenarios artificiales.
- Mantener un tono directo y sin juicio: el objetivo es explicar, no ridiculizar.
