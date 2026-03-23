# 🔥 wtf-codebase

> Una colección de malas prácticas reales con explicación de por qué duelen y cómo hacerlo mejor.

Este repositorio **no es una guía de buenas prácticas**. Es todo lo contrario: un catálogo honesto de los errores más comunes (y tentadores) que cometemos como developers, con el contexto suficiente para entender exactamente por qué son un problema.

Cada ejemplo viene con código real, síntomas en producción y una alternativa razonable.

---

## 📋 Índice de malas prácticas

| # | Título | Categoría | Nivel de daño |
|---|--------|-----------|---------------|
| 01 | [Usar un archivo JSON/YAML como base de datos](#01---usar-un-archivo-jsonyamltoml-como-base-de-datos) | Persistencia | 🔥🔥🔥 |

---

## 01 - Usar un archivo JSON/YAML/TOML como base de datos

### El problema

Cuando necesitas persistencia en tu app, lo más natural del mundo es crear un archivo. Es simple, no requiere dependencias, cualquier editor lo puede abrir y en cinco minutos ya estás escribiendo estado. Muy tentador. Y en algunos casos, perfectamente razonable.

El problema aparece cuando ese archivo empieza a mutar con frecuencia.

### ✅ Cuándo sí tiene sentido

Un archivo plano es una opción válida si se cumplen estas condiciones:

- El estado **no cambia frecuentemente** (o casi nunca en tiempo de ejecución)
- El archivo es **relativamente pequeño**
- El ciclo de vida del dato es **largo**: dependencias, versiones, descripciones, títulos, fecha de creación

En ese escenario, el patrón es limpio: **1 lectura al iniciar** y **1 escritura al finalizar**. Así funciona un `package.json`, un `config.yaml`, un `.env`. Nadie se queja.

### ❌ Cuándo no tiene sentido

El problema real empieza cuando el estado se actualiza recurrentemente:

```
1 lectura al iniciar → N escrituras durante la vida de la app
```

Cada escritura en un archivo plano implica reescribir **todo el archivo completo** en disco, incluso si solo cambió un campo. Esto tiene consecuencias directas:

- 🧊 **Congela tu app**: en Node.js (y en general en entornos de un solo hilo), `fs.writeFileSync` bloquea el event loop. Tu servidor deja de responder durante la escritura.
- 💾 **Consume I/O innecesario**: estás escribiendo 50KB cuando solo cambió un número.
- ⚠️ **Riesgo de corrupción**: si el proceso muere a mitad de una escritura, el archivo queda en estado inválido. No hay transacciones, no hay rollback.
- 🐛 **Race conditions**: si dos partes de tu código escriben "al mismo tiempo", una sobrescribe a la otra silenciosamente.

### El código que nadie quiere ver en code review

```js
// state.json: { "counter": 0, "lastSeen": {} }

import fs from "fs";

const STATE_FILE = "./state.json";

function getState() {
  return JSON.parse(fs.readFileSync(STATE_FILE, "utf-8"));
}

function saveState(state) {
  // 🔥 Esto bloquea el event loop cada vez que se llama
  fs.writeFileSync(STATE_FILE, JSON.stringify(state, null, 2));
}

// Y si esto se llama en cada request HTTP...
app.post("/visit", (req, res) => {
  const state = getState();         // Lee TODO el archivo
  state.counter++;
  state.lastSeen[req.ip] = Date.now();
  saveState(state);                 // Escribe TODO el archivo
  res.json({ ok: true });
});
```

Con 10 usuarios concurrentes esto ya empieza a temblar. Con 100, tu app está congelada.

### Las alternativas

#### Opción A — SQLite

SQLite no reescribe el archivo completo. Escribe solo las páginas del disco que deben cambiar, lo que lo hace órdenes de magnitud más eficiente para escrituras frecuentes. Además soporta transacciones, índices y queries.

```js
import Database from "better-sqlite3";

const db = new Database("state.db");

db.exec(`
  CREATE TABLE IF NOT EXISTS visits (
    ip TEXT PRIMARY KEY,
    last_seen INTEGER,
    count INTEGER DEFAULT 0
  )
`);

const upsert = db.prepare(`
  INSERT INTO visits (ip, last_seen, count) VALUES (?, ?, 1)
  ON CONFLICT(ip) DO UPDATE SET
    last_seen = excluded.last_seen,
    count = count + 1
`);

app.post("/visit", (req, res) => {
  upsert.run(req.ip, Date.now()); // Escritura eficiente, sin bloquear
  res.json({ ok: true });
});
```

#### Opción B — Worker para escrituras en background

Si quieres seguir usando un archivo plano (por simplicidad o portabilidad), mueve las escrituras a un Worker. La idea es que **otra CPU escriba los datos**, y así tu hilo principal no se congela esperando I/O.

```js
// writer.worker.js
import fs from "fs";
import { workerData } from "worker_threads";

fs.writeFileSync("./state.json", JSON.stringify(workerData, null, 2));
```

```js
// app.js
import { Worker } from "worker_threads";

function saveStateAsync(state) {
  new Worker("./writer.worker.js", { workerData: state });
  // El hilo principal sigue libre
}
```

> ⚠️ Esta opción resuelve el bloqueo del hilo, pero **no resuelve las race conditions** ni la ineficiencia de reescribir el archivo completo. Úsala solo si SQLite no es una opción viable.

### Resumen

| Situación | Archivo plano | SQLite | Worker |
|-----------|:---:|:---:|:---:|
| Config que casi no cambia | ✅ | — | — |
| Estado que cambia frecuentemente | ❌ | ✅ | ⚠️ |
| Necesitas queries / índices | ❌ | ✅ | — |
| Quieres evitar bloquear el hilo | ❌ | ✅ | ✅ |
| Riesgo de corrupción | Alto | Bajo | Medio |

---

## Contribuir

¿Tienes una mala práctica que te persigue en las noches? Abre un PR. El único requisito es que el ejemplo sea real (o muy creíble) y que la explicación sea honesta sobre por qué duele.

---

<sub>Hecho con 🔥 y cicatrices de producción.</sub>