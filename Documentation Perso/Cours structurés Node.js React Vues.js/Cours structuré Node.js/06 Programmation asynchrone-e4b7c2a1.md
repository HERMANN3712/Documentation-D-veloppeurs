# 06 — Programmation asynchrone (Node.js)

> **Public** : développeurs ayant les bases de JavaScript et de Node.js.  
> **Objectif** : comprendre et maîtriser l’asynchronisme en Node.js (Event Loop, callbacks, Promises, `async/await`), savoir concevoir un code non bloquant, robuste et lisible.

---

## Plan de la formation

1. **Pourquoi l’asynchrone en Node.js ?**
   - Modèle I/O non bloquant
   - Concurrence vs parallélisme
   - Impacts sur les performances et la scalabilité
2. **Comprendre le modèle d’exécution**
   - Thread principal, libuv, pool de threads
   - Event Loop (phases)
   - Macrotâches vs microtâches
   - `setTimeout`, `setImmediate`, `process.nextTick`, Promises
3. **Les callbacks**
   - Principe et usages
   - Convention Node.js (error-first)
   - Callback hell : causes et anti-patterns
   - Convertir des callbacks en Promises
4. **Les Promises**
   - États d’une Promise
   - Chaînage, propagation d’erreurs
   - `Promise.all`, `allSettled`, `race`, `any`
   - Patterns courants (séquence, parallèle, limitation)
5. **`async/await`**
   - Syntaxe et traduction mentale en Promises
   - Gestion d’erreurs avec `try/catch/finally`
   - Exécution séquentielle vs parallèle
   - Pièges fréquents (`forEach` + `await`, oubli de `await`, etc.)
6. **Bonnes pratiques & design asynchrone**
   - Non-blocage (CPU-bound vs I/O-bound)
   - Backpressure (streams, files, HTTP)
   - Timeouts, retries, cancellation
   - Observabilité : logs, métriques, tracing
7. **Ateliers / exercices corrigés**
   - Manipulation de l’Event Loop
   - Refactor callbacks → Promises → `async/await`
   - Téléchargements parallèles avec limite de concurrence
   - Mini API HTTP : route async robuste

---

## 1) Pourquoi l’asynchrone en Node.js ?

### 1.1 Le problème : attendre l’I/O
Dans une application serveur, la majorité du temps est passée à **attendre** :
- réponses réseau (HTTP, DB)
- accès disque (read/write)
- timers

Si chaque requête bloquait le thread principal pendant ces attentes, le serveur traiterait très peu de requêtes par seconde.

### 1.2 Le choix Node.js : I/O non bloquant
Node.js repose sur :
- **un thread principal JS** (exécute votre code)
- une couche C/C++ (libuv) qui gère les I/O de manière asynchrone

Résultat : pendant qu’une opération I/O est en cours, le thread JS peut continuer à **traiter d’autres événements**.

### 1.3 Concurrence vs parallélisme
- **Concurrence** : gérer plusieurs tâches *en même temps* au niveau logique (interleaving) sur un même thread.
- **Parallélisme** : exécuter véritablement en même temps sur plusieurs cœurs (multi-threads/process).

Node.js est **très concurrent** (I/O), mais pas automatiquement parallèle pour le CPU.

---

## 2) Comprendre le modèle d’exécution

### 2.1 Les composants clés
- **JavaScript main thread** : exécution synchrone de vos fonctions.
- **Call stack** : pile d’appels.
- **Task queues** : files d’attente des callbacks à exécuter.
- **libuv** : gère la boucle d’événements, les I/O système et un **thread pool** (par défaut 4) pour certaines opérations.

### 2.2 L’Event Loop (vision pratique)
L’Event Loop orchestre quand exécuter :
- les callbacks de timers (`setTimeout`, `setInterval`)
- les callbacks I/O (sockets, fs)
- les callbacks `setImmediate`
- les microtâches (Promises) et `process.nextTick`

**Simplification utile** :
1. Exécuter le code synchrone (call stack).
2. Vider la queue des **microtâches** (Promises), puis `nextTick` (particularité Node).
3. Exécuter une macrotâche (timer/I/O/immediate), puis revenir au point 2.

### 2.3 Microtâches vs macrotâches
- **Microtâches** : `.then`, `queueMicrotask`, résolution/rejet de Promises.
- **Macrotâches** : `setTimeout`, `setImmediate`, callbacks I/O.

#### Démo : ordre d’exécution (Node)
```js
console.log('A');

setTimeout(() => console.log('timeout'), 0);
setImmediate(() => console.log('immediate'));

Promise.resolve().then(() => console.log('promise')); 
process.nextTick(() => console.log('nextTick'));

console.log('B');
```

**Attendu (généralement)** :
1. `A`
2. `B`
3. `nextTick`
4. `promise`
5. `timeout` / `immediate` (dépend du contexte d’exécution)

> Note : l’ordre exact de `setTimeout(0)` et `setImmediate()` peut varier selon le contexte (I/O callback vs top-level). Ce qui compte : **microtâches d’abord**, puis macrotâches.

### 2.4 Le thread pool libuv (à connaître)
Certaines opérations utilisent un pool de threads :
- `fs` (selon APIs)
- `crypto`
- `zlib`
- DNS (certaines résolutions)

Vous pouvez ajuster :
```bash
UV_THREADPOOL_SIZE=8 node app.js
```

---

## 3) Les callbacks

### 3.1 Principe
On passe une fonction qui sera appelée plus tard :
```js
const fs = require('fs');

fs.readFile('data.txt', 'utf8', (err, content) => {
  if (err) return console.error(err);
  console.log(content);
});

console.log('Lecture lancée');
```

### 3.2 Convention Node.js : error-first callback
Signature standard : `(err, result)`
- `err` est `null` ou `undefined` si tout va bien
- sinon `err` contient l’erreur

Cela permet une gestion uniforme de l’erreur.

### 3.3 Problèmes classiques
#### Callback hell
Imbrications profondes :
```js
getUser(id, (err, user) => {
  if (err) return cb(err);
  getOrders(user.id, (err, orders) => {
    if (err) return cb(err);
    getInvoices(orders, (err, invoices) => {
      if (err) return cb(err);
      cb(null, invoices);
    });
  });
});
```

**Risques** :
- lisibilité faible
- duplication de gestion d’erreurs
- difficulté d’évolution

#### Inversion of control
Vous déléguez le *moment* et la *garantie* d’appel à un tiers. D’où l’intérêt des Promises et `async/await`.

### 3.4 Passer des callbacks aux Promises
Beaucoup d’APIs Node proposent déjà une variante Promise (ex: `fs/promises`). Sinon on peut « promiser » :

```js
const { promisify } = require('util');
const fs = require('fs');

const readFileAsync = promisify(fs.readFile);

readFileAsync('data.txt', 'utf8')
  .then(console.log)
  .catch(console.error);
```

---

## 4) Les Promises

### 4.1 Une Promise, c’est quoi ?
Une Promise représente une valeur **future**.
- `pending` → `fulfilled(value)` ou `rejected(error)`

Créer une Promise :
```js
function wait(ms) {
  return new Promise((resolve) => setTimeout(resolve, ms));
}

wait(200).then(() => console.log('200ms plus tard'));
```

### 4.2 Chaînage et propagation
```js
fetchUser(id)
  .then(user => fetchOrders(user.id))
  .then(orders => computeTotal(orders))
  .then(total => console.log({ total }))
  .catch(err => console.error('Erreur:', err));
```

**Règles importantes** :
- retourner une valeur dans `.then` la passe au `.then` suivant
- retourner une Promise l’enchaîne (flat)
- une exception dans `.then` déclenche `.catch`

### 4.3 Promises en parallèle
#### `Promise.all`
- échoue dès le **premier rejet**
- conserve l’ordre des résultats

```js
const [a, b, c] = await Promise.all([
  fetchA(),
  fetchB(),
  fetchC(),
]);
```

#### `Promise.allSettled`
- n’échoue pas globalement
- utile pour batchs tolérants aux échecs

```js
const results = await Promise.allSettled([task1(), task2()]);
for (const r of results) {
  if (r.status === 'fulfilled') console.log(r.value);
  else console.error(r.reason);
}
```

#### `Promise.race`
- la première Promise (résolue ou rejetée) gagne

#### `Promise.any`
- la première **résolue** gagne
- si toutes échouent: `AggregateError`

### 4.4 Pattern : séquence vs parallèle
Séquence (dépendances) :
```js
const user = await fetchUser(id);
const orders = await fetchOrders(user.id);
```

Parallèle (indépendantes) :
```js
const [profile, settings] = await Promise.all([
  fetchProfile(id),
  fetchSettings(id),
]);
```

### 4.5 Limiter la concurrence (pattern essentiel)
Lancer 10 000 Promises d’un coup peut saturer :
- sockets
- DB
- mémoire

Exemple simple de limite de concurrence (sans lib) :
```js
async function mapWithConcurrency(items, limit, mapper) {
  const results = new Array(items.length);
  let i = 0;

  async function worker() {
    while (true) {
      const current = i++;
      if (current >= items.length) return;
      results[current] = await mapper(items[current], current);
    }
  }

  const workers = Array.from({ length: limit }, worker);
  await Promise.all(workers);
  return results;
}

// usage
const urls = ['https://a', 'https://b', 'https://c'];
const pages = await mapWithConcurrency(urls, 2, fetchPage);
```

---

## 5) `async/await`

### 5.1 Ce que `async/await` apporte
- une syntaxe plus proche du synchrone
- une gestion d’erreur via `try/catch`
- lisibilité + maintenabilité

`async` retourne toujours une Promise :
```js
async function f() {
  return 42;
}

f().then(console.log); // 42
```

### 5.2 `await` : suspendre sans bloquer le processus
`await` suspend **la fonction async**, pas l’Event Loop :
```js
async function handler(req, res) {
  const user = await fetchUser(req.params.id);
  res.end(JSON.stringify(user));
}
```

### 5.3 Gestion d’erreurs
```js
async function load() {
  try {
    const data = await fetchData();
    return data;
  } catch (err) {
    // Ajouter contexte + rethrow
    throw new Error('load() failed: ' + err.message);
  } finally {
    // nettoyage : release ressources
  }
}
```

### 5.4 Paralléliser avec `await`
Erreur fréquente : séquentialiser involontairement.

Séquentiel (lent si indépendant) :
```js
const a = await taskA();
const b = await taskB();
```

Parallèle (souvent préférable) :
```js
const pa = taskA();
const pb = taskB();
const [a, b] = await Promise.all([pa, pb]);
```

### 5.5 Pièges fréquents
#### `forEach` + `await`
`forEach` n’attend pas l’async callback.

À éviter :
```js
items.forEach(async (item) => {
  await doWork(item);
});
```

Préférer :
- séquentiel :
```js
for (const item of items) {
  await doWork(item);
}
```
- parallèle :
```js
await Promise.all(items.map(doWork));
```

#### Oublier `await`
Vous retournez une Promise au lieu de la valeur attendue, causant des bugs subtils.

#### Mélanger callbacks et Promises
Évitez de « wrapper » en callback à l’extérieur si votre API interne est Promise : exposez un contrat cohérent.

---

## 6) Bonnes pratiques & design asynchrone

### 6.1 Ne pas bloquer l’Event Loop (CPU-bound)
Tout calcul lourd en JS bloque :
- hashing intensif
- compression custom
- boucles massives

Solutions :
- **Worker Threads** (`node:worker_threads`)
- **Child processes** (isolation)
- déléguer à un service externe

### 6.2 Backpressure (pression retour)
Quand une source produit plus vite que le consommateur.

En Node.js, les **Streams** gèrent nativement la backpressure.
Exemple (pipe) :
```js
const fs = require('fs');

fs.createReadStream('big.bin')
  .pipe(fs.createWriteStream('copy.bin'));
```

### 6.3 Timeouts
Toujours éviter les attentes infinies sur réseau.
- `AbortController` (fetch, certains clients HTTP)

Ex :
```js
const ac = new AbortController();
const t = setTimeout(() => ac.abort(), 2000);

try {
  const res = await fetch(url, { signal: ac.signal });
  return await res.json();
} finally {
  clearTimeout(t);
}
```

### 6.4 Retries avec stratégie
Un retry aveugle peut empirer la situation.
- backoff exponentiel
- jitter
- limite max
- retries seulement sur erreurs transitoires

### 6.5 Observabilité
- loggez avec corrélation (request id)
- mesurez latences
- tracez les erreurs avec stack + contexte

---

## 7) Ateliers (avec propositions de corrigés)

### Atelier 1 — Explorer l’Event Loop
**Objectif** : prédire l’ordre d’exécution.

1) Reprendre la démo du chapitre 2.3 et expliquer la sortie.
2) Ajouter un I/O (lecture fichier) et comparer `setImmediate` vs `setTimeout(0)` dans un callback I/O.

Corrigé (piste) : dans un callback d’I/O, `setImmediate` est souvent exécuté avant `setTimeout(0)`.

---

### Atelier 2 — Refactor callback hell → Promises → async/await
**Énoncé** : transformer un enchaînement callback en code moderne.

Version callback (exemple) :
```js
function getInvoiceTotal(userId, cb) {
  getUser(userId, (err, user) => {
    if (err) return cb(err);
    getOrders(user.id, (err, orders) => {
      if (err) return cb(err);
      getInvoices(orders, (err, invoices) => {
        if (err) return cb(err);
        cb(null, invoices.reduce((s, i) => s + i.amount, 0));
      });
    });
  });
}
```

Corrigé (async/await) :
```js
async function getInvoiceTotal(userId) {
  const user = await getUser(userId);
  const orders = await getOrders(user.id);
  const invoices = await getInvoices(orders);
  return invoices.reduce((s, i) => s + i.amount, 0);
}

// usage
try {
  const total = await getInvoiceTotal('u1');
  console.log({ total });
} catch (err) {
  console.error(err);
}
```

> Prérequis : `getUser/getOrders/getInvoices` doivent retourner des Promises.

---

### Atelier 3 — Téléchargements parallèles avec limite de concurrence
**Énoncé** : télécharger N URLs en limitant à 3 requêtes simultanées.

Corrigé (réutiliser `mapWithConcurrency`) :
```js
async function fetchText(url) {
  const res = await fetch(url);
  if (!res.ok) throw new Error('HTTP ' + res.status);
  return res.text();
}

const pages = await mapWithConcurrency(urls, 3, fetchText);
console.log(pages.map(p => p.length));
```

---

### Atelier 4 — Mini API HTTP robuste (async)
**Énoncé** : créer une route `/users/:id` qui :
- récupère l’utilisateur
- renvoie 404 si absent
- gère les erreurs avec code 500

Corrigé (Node http natif, minimal) :
```js
const http = require('http');

async function getUserById(id) {
  // Simuler I/O
  await new Promise(r => setTimeout(r, 50));
  if (id === '42') return { id, name: 'Ada' };
  return null;
}

const server = http.createServer(async (req, res) => {
  try {
    const match = req.url.match(/^\/users\/(\w+)$/);
    if (!match) {
      res.statusCode = 404;
      return res.end('Not found');
    }

    const user = await getUserById(match[1]);
    if (!user) {
      res.statusCode = 404;
      return res.end('User not found');
    }

    res.setHeader('Content-Type', 'application/json');
    res.end(JSON.stringify(user));
  } catch (err) {
    res.statusCode = 500;
    res.end('Internal error');
    console.error(err);
  }
});

server.listen(3000, () => console.log('http://localhost:3000'));
```

---

## Annexes

### A. Checklist de décision (callbacks vs Promises vs async/await)
- **API existante callback** : envisager `util.promisify` ou `fs/promises`.
- **Code applicatif moderne** : privilégier Promises + `async/await`.
- **Exécution parallèle** : `Promise.all` / `allSettled`.
- **Contrôle fin (streams/backpressure)** : utiliser Streams.

### B. Glossaire
- **Event Loop** : boucle qui planifie l’exécution des callbacks.
- **Non-bloquant** : l’application continue à traiter d’autres événements pendant une attente.
- **Microtask** : tâches à haute priorité (Promises).
- **Macrotask** : tâches planifiées (timers, I/O callbacks).
- **Backpressure** : mécanisme pour ne pas saturer un consommateur plus lent.

---

## Résumé
- Node.js excelle en I/O grâce à un modèle **non bloquant** piloté par l’**Event Loop**.
- Les callbacks sont historiques mais peuvent devenir illisibles.
- Les **Promises** structurent l’asynchrone (chaînage, parallélisme, utilitaires).
- `**async/await**` améliore fortement la lisibilité, tout en gardant les avantages des Promises.
- Les vraies difficultés sont souvent : **limiter la concurrence**, gérer **timeouts/retries**, éviter de **bloquer** l’Event Loop.
