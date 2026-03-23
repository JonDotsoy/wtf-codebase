# CLAUDE.md — Contribution guide for wtf-codebase

This file describes how to add new anti-patterns to the repository and how to keep the documentation files up to date.

---

## Project structure

- `README.md` — Main catalog in Spanish
- `README-EN.md` — English catalog (must be kept in sync with README.md)
- `CLAUDE.md` — This file: instructions for contributing

---

## How to add a new case

Each new scenario or bad practice must use the format `WTFCODE-<number>`.

The number must be consecutive to the last one in the `README.md` index.

### Steps

1. Add a row to the index table in both `README.md` and `README-EN.md`
2. Add the full section at the end of the catalog (before the "Contributing" section) in both files
3. The section title must follow the format `## WTFCODE-<number> - <title>`

---

## Format of each case

Each entry must have exactly these sections, in this order:

### `## WTFCODE-<number> - <title>`

Main title of the anti-pattern.

### `### El problema` / `### The problem`

Narrative description of the problem. Explain the context, why the pattern is tempting, and when it starts to hurt.

### `### ✅ Cuándo sí tiene sentido` / `### ✅ When it makes sense`

List of conditions under which the pattern is acceptable or even correct. Be honest: not every practice is bad in every context.

### `### ❌ Cuándo no tiene sentido` / `### ❌ When it doesn't make sense`

List of conditions or signals that indicate the pattern is a problem. Include concrete consequences.

### `### El código que nadie quiere ver en code review` / `### The code nobody wants to see in code review`

A real (or very believable) code block illustrating the bad practice. Add inline comments explaining exactly what fails.

### `### Las alternativas` / `### The alternatives`

One or more concrete alternatives with code. Each alternative has a subheading `#### Option A — ...`.

### `### Resumen` / `### Summary`

A list of common questions with short answers. Format:

```markdown
### Summary

- **When is it OK to use X?** When Y and Z conditions are met.
- **What happens if X occurs?** The system does W.
- **Is there a simpler alternative?** Yes, consider P or Q.
```

---

## Index in README.md and README-EN.md

Every time a new case is added, update the index table in both files:

```markdown
| WTFCODE-N | [Case title](#anchor) | Category | 🔥🔥🔥 |
```

Suggested categories (reuse existing ones before creating new ones):

- `Persistence`
- `Concurrency`
- `Security`
- `Performance`
- `Architecture`
- `Testing`
- `Dependencies`

Damage levels:

- `🔥` — Annoying
- `🔥🔥` — Real problem
- `🔥🔥🔥` — Catastrophic in production

---

## Sync rules for README.md / README-EN.md

- Every section in Spanish must have its English equivalent.
- Code blocks are not translated (inline code comments should be).
- Section headings (`### ✅ When it makes sense`, `### ❌ When it doesn't make sense`, `### Summary`) must be translated.
- The order and numbering `WTFCODE-<number>` must be identical in both files.

---

## Minimal example of a new entry

```markdown
## WTFCODE-2 - <Title>

### El problema

<Description of the problem>

### ✅ Cuándo sí tiene sentido

- <Condition 1>
- <Condition 2>

### ❌ Cuándo no tiene sentido

- <Signal 1>
- <Signal 2>

### El código que nadie quiere ver en code review

\`\`\`js
// Example of the bad practice
\`\`\`

### Las alternativas

#### Opción A — <Alternative name>

\`\`\`js
// Example of the alternative
\`\`\`

### Resumen

- **¿Common question 1?** Short answer.
- **¿Common question 2?** Short answer.
- **¿Common question 3?** Short answer.
```

---

## Additional notes

- `WTFCODE-1` already exists: using a JSON/YAML file as a database.
- All examples must be real or very believable; do not invent artificial scenarios.
- Keep a direct, non-judgmental tone: the goal is to explain, not to mock.
