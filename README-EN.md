[Español](./README.md) | **English**

# 🔥 wtf-codebase

> A collection of real anti-patterns with an explanation of why they hurt and how to do it better.

This repository **is not a best-practices guide**. It's the opposite: an honest catalog of the most common (and tempting) mistakes we make as developers, with enough context to understand exactly why they are a problem.

Each example comes with real code, production symptoms, and a reasonable alternative.

---

## 📋 Anti-patterns Index

| # | Title | Category | Damage Level |
|---|-------|----------|--------------|
| WTFCODE-1 | [Using a JSON/YAML file as a database](#wtfcode-1---using-a-jsonyamltoml-file-as-a-database) | Persistence | 🔥🔥🔥 |

---

## WTFCODE-1 - Using a JSON/YAML/TOML file as a database

### The problem

When you need persistence in your app, the most natural thing in the world is to create a file. It's simple, requires no dependencies, any editor can open it, and in five minutes you're already writing state. Very tempting. And in some cases, perfectly reasonable.

The problem appears when that file starts to mutate frequently.

### ✅ When it makes sense

A flat file is a valid option if these conditions are met:

- The state **does not change frequently** (or almost never at runtime)
- The file is **relatively small**
- The data lifecycle is **long**: dependencies, versions, descriptions, titles, creation date

In that scenario, the pattern is clean: **1 read on startup** and **1 write on shutdown**. That's how `package.json`, `config.yaml`, `.env` work. Nobody complains.

### ❌ When it doesn't make sense

The real problem starts when the state is updated recurrently:

```
1 read on startup → N writes during the app's lifetime
```

Each write to a flat file involves rewriting **the entire file** to disk, even if only one field changed. This has direct consequences:

- 🧊 **Freezes your app**: in Node.js (and generally in single-threaded environments), `fs.writeFileSync` blocks the event loop. Your server stops responding during the write.
- 💾 **Wastes I/O**: you're writing 50KB when only a number changed.
- ⚠️ **Corruption risk**: if the process dies mid-write, the file is left in an invalid state. There are no transactions, no rollback.
- 🐛 **Race conditions**: if two parts of your code write "at the same time", one silently overwrites the other.

### The code nobody wants to see in code review

```js
// state.json: { "counter": 0, "lastSeen": {} }

import fs from "fs";

const STATE_FILE = "./state.json";

function getState() {
  return JSON.parse(fs.readFileSync(STATE_FILE, "utf-8"));
}

function saveState(state) {
  // 🔥 This blocks the event loop every time it's called
  fs.writeFileSync(STATE_FILE, JSON.stringify(state, null, 2));
}

// And if this is called on every HTTP request...
app.post("/visit", (req, res) => {
  const state = getState();         // Reads the ENTIRE file
  state.counter++;
  state.lastSeen[req.ip] = Date.now();
  saveState(state);                 // Writes the ENTIRE file
  res.json({ ok: true });
});
```

With 10 concurrent users this already starts to shake. With 100, your app is frozen.

### The alternatives

#### Option A — SQLite

SQLite doesn't rewrite the entire file. It only writes the disk pages that need to change, making it orders of magnitude more efficient for frequent writes. It also supports transactions, indexes, and queries.

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
  upsert.run(req.ip, Date.now()); // Efficient write, non-blocking
  res.json({ ok: true });
});
```

#### Option B — Worker for background writes

If you want to keep using a flat file (for simplicity or portability), move the writes to a Worker. The idea is that **another CPU writes the data**, so your main thread doesn't freeze waiting for I/O.

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
  // The main thread remains free
}
```

> ⚠️ This option solves the thread-blocking issue, but **does not solve race conditions** or the inefficiency of rewriting the entire file. Use it only if SQLite is not a viable option.

### Summary

| Situation | Flat file | SQLite | Worker |
|-----------|:---------:|:------:|:------:|
| Config that rarely changes | ✅ | — | — |
| State that changes frequently | ❌ | ✅ | ⚠️ |
| Need queries / indexes | ❌ | ✅ | — |
| Want to avoid blocking the thread | ❌ | ✅ | ✅ |
| Corruption risk | High | Low | Medium |

---

## Contributing

Do you have an anti-pattern that haunts you at night? [Open a PR](https://github.com/JonDotsoy/wtf-codebase/issues/new). The only requirement is that the example is real (or very believable) and that the explanation is honest about why it hurts.

Read [CLAUDE.md](./CLAUDE.md) for the exact format of each entry and how to keep files in sync.

---

<sub>Made with 🔥 and production scars.</sub>
