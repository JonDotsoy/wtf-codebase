**Español** | [English](./README-EN.md)

# 🔥 wtf-codebase

> Una colección de malas prácticas reales con explicación de por qué duelen y cómo hacerlo mejor.

Este repositorio **no es una guía de buenas prácticas**. Es todo lo contrario: un catálogo honesto de los errores más comunes (y tentadores) que cometemos como developers, con el contexto suficiente para entender exactamente por qué son un problema.

Cada ejemplo viene con código real, síntomas en producción y una alternativa razonable.

---

## 📋 Índice de malas prácticas

| # | Título | Categoría | Nivel de daño |
|---|--------|-----------|---------------|
| WTFCODE-1 | [Usar un archivo JSON/YAML como base de datos](#wtfcode-1---usar-un-archivo-jsonyamltoml-como-base-de-datos) | Persistencia | 🔥🔥🔥 |
| WTFCODE-2 | [URLs concatenadas con strings](#wtfcode-2---urls-concatenadas-con-strings) | Security | 🔥🔥 |

---

## WTFCODE-1 - Usar un archivo JSON/YAML/TOML como base de datos

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

## WTFCODE-2 - URLs concatenadas con strings

### El problema

Construir URLs uniendo strings es algo que todos hemos hecho. Parece inofensivo: sabes cuál es la base, conoces la ruta, solo concatenas el ID o el parámetro y listo. El problema aparece cuando el valor que insertas viene de fuera de tu control: un input del usuario, un parámetro de query, un dato de una base de datos. Cualquier carácter especial de la especificación URL (https://url.spec.whatwg.org/) como `?`, `&`, `#`, `/` o incluso espacios puede romper la URL silenciosamente, modificar la estructura de la petición o, en el peor caso, abrir vectores de ataque para tus usuarios.

### ✅ Cuándo sí tiene sentido

- Cuando los valores son literales hardcodeados y nunca cambian (e.g., `"/api/v1/health"`).
- En scripts de un solo uso donde el control del input está garantizado y no hay usuarios involucrados.

### ❌ Cuándo no tiene sentido

- Cuando cualquier parte de la URL viene de input del usuario, parámetros de query o datos externos.
- Cuando construyes query strings con múltiples parámetros: el orden, el encoding y los caracteres especiales son tu responsabilidad (y te vas a equivocar).
- Cuando la URL resultante se usa en fetch, axios u otro cliente HTTP en producción.
- Cuando los valores pueden contener `?`, `&`, `#`, `/` o caracteres Unicode: la URL queda mal formada sin que nadie te avise.

### El código que nadie quiere ver en code review

```js
// Parece inocente...
const url = "https://my-web/user/" + userId;

// Ya empieza a doler
const url = "https://my-web/products?category=" + category + "&rank=" + rank;

// Si userId = "123/admin" → "https://my-web/user/123/admin"  ← path traversal silencioso
// Si category = "shoes&rank=0&admin=true" → inyecta parámetros extra
// Si rank = "<script>" → depende del servidor, pero ya estás rezando
fetch(url); // 🙏
```

### Las alternativas

#### Opción A — Usar la API estándar `URL`

Disponible en el navegador y en Node.js (sin imports). Maneja encoding automáticamente y hace imposible romper la estructura de la URL.

```js
// Path con valor dinámico
const url = new URL("https://my-web/");
url.pathname = `/user/${userId}`;
// Si userId = "123/admin" → pathname queda "/user/123%2Fadmin" ← seguro

// Query params con múltiples valores
const url = new URL("https://my-web/products");
url.searchParams.set("category", category);
url.searchParams.set("rank", rank);
// Si category = "shoes&rank=0" → queda "?category=shoes%26rank%3D0" ← no inyecta nada

fetch(url.toString());
```

#### Opción B — Usar `node:url` en entornos Node.js

```js
import { URL } from "node:url";

const url = new URL("/api/orders", "https://my-api.internal");
url.searchParams.set("status", status);
url.searchParams.set("page", page);

await fetch(url);
```

#### Opción C — Helper utilitario si construyes muchas URLs

Si en tu codebase hay docenas de lugares construyendo URLs, centraliza la lógica:

```js
function buildUrl(base, pathname, params = {}) {
  const url = new URL(base);
  if (pathname) url.pathname = pathname;
  for (const [key, value] of Object.entries(params)) {
    url.searchParams.set(key, value);
  }
  return url;
}

const url = buildUrl("https://my-web", `/user/${userId}`, { tab: "orders" });
```

### Resumen

- **¿Cuándo es seguro concatenar URLs?** Solo cuando todos los segmentos son literales hardcodeados, sin variables externas.
- **¿Qué pasa si el valor tiene `?` o `&`?** Se rompe la estructura de la URL: puedes inyectar parámetros extra o llegar a un endpoint distinto.
- **¿`encodeURIComponent` no es suficiente?** Parcialmente: codifica el valor, pero tienes que aplicarlo manualmente en cada lugar y es fácil olvidarlo. La API `URL` lo hace automáticamente y de forma consistente.
- **¿La API `URL` está disponible en el navegador?** Sí, es parte del estándar web y está disponible en todos los navegadores modernos y en Node.js desde v10.

---

## Contribuir

¿Tienes una mala práctica que te persigue en las noches? [Abre un PR](https://github.com/JonDotsoy/wtf-codebase/issues/new). El único requisito es que el ejemplo sea real (o muy creíble) y que la explicación sea honesta sobre por qué duele.

---

<sub>Hecho con 🔥 y cicatrices de producción.</sub>