# Formation 01 — Introduction à Node.js

> **Public** : développeurs web (JS) souhaitant comprendre Node.js et démarrer côté serveur.  
> **Pré-requis** : JavaScript (ES6+), notions HTTP (requête/réponse), bases terminal/CLI.  
> **Durée suggérée** : 3h–6h (selon exercices).  

---

## Objectifs pédagogiques

À l’issue de cette formation, vous saurez :

- Expliquer ce qu’est Node.js, son lien avec **V8** et ce que signifie « JavaScript côté serveur ».
- Comprendre le **modèle d’exécution** (event loop, I/O non bloquantes) à un niveau introductif.
- Installer Node.js et utiliser **npm** pour gérer des dépendances.
- Écrire et exécuter des scripts, lire des fichiers, manipuler des modules.
- Créer un **serveur HTTP** simple et comprendre les bases d’un backend Node.
- Identifier les cas d’usage, les avantages et les limites de Node.js.

---

## Plan de la formation

1. [Qu’est-ce que Node.js ?](#1-quest-ce-que-nodejs)
2. [Pourquoi Node.js : usages, forces et limites](#2-pourquoi-nodejs--usages-forces-et-limites)
3. [Installation & environnement de travail](#3-installation--environnement-de-travail)
4. [Premier pas : exécuter du JavaScript hors navigateur](#4-premier-pas--exécuter-du-javascript-hors-navigateur)
5. [Modules : CommonJS vs ES Modules](#5-modules--commonjs-vs-es-modules)
6. [npm : gestion des dépendances et scripts](#6-npm--gestion-des-dépendances-et-scripts)
7. [Asynchronisme : callbacks, Promises, async/await](#7-asynchronisme--callbacks-promises-asyncawait)
8. [Event Loop & I/O non bloquantes (intro)](#8-event-loop--io-non-bloquantes-intro)
9. [Le module `fs` : lecture/écriture de fichiers (intro)](#9-le-module-fs--lectureécriture-de-fichiers-intro)
10. [Créer un serveur HTTP minimal](#10-créer-un-serveur-http-minimal)
11. [Structurer un mini-backend (bonne pratiques de base)](#11-structurer-un-mini-backend-bonne-pratiques-de-base)
12. [Exercices & corrections guidées](#12-exercices--corrections-guidées)
13. [Conclusion & prochaines étapes](#13-conclusion--prochaines-étapes)

---

## 1. Qu’est-ce que Node.js ?

### Définition

**Node.js** est un **environnement d’exécution JavaScript côté serveur** basé sur le moteur **V8** de Google Chrome.

- **V8** : moteur qui compile et exécute JavaScript (très performant).
- **Node.js** : ajoute autour de V8 un ensemble d’API (fichiers, réseau, processus…) pour permettre d’exécuter JavaScript **en dehors du navigateur**.

### Ce que Node.js permet

Node.js permet de construire :

- des **API REST** et backends web,
- des serveurs temps réel (WebSocket),
- des outils CLI (linters, bundlers, scripts),
- des services d’intégration, workers, jobs, etc.

### Ce que Node.js n’est pas

- Ce n’est pas « un framework » : Node est une **plateforme**. Express, Fastify, NestJS sont des frameworks.
- Ce n’est pas « du JS dans le navigateur » : Node n’a pas de DOM, pas de `window`, pas de `document`.

---

## 2. Pourquoi Node.js : usages, forces et limites

### Cas d’usage typiques

- **Backends I/O-bound** : API qui attend des BDD, services externes, fichiers.
- **Temps réel** : chat, notifications, collaboration (WebSocket).
- **Prototypage** rapide : même langage front & back.
- **Microservices** : services légers, facilement déployables.

### Forces

- **Performance en I/O** grâce au modèle non bloquant.
- **Écosystème npm** immense.
- **Unification du langage** entre frontend et backend.
- **Scalabilité** : bonnes pratiques + clustering / multiples instances.

### Limites (honnêtement)

- Moins idéal pour les tâches **CPU-bound** (calcul intensif) si elles bloquent l’event loop.
  - Solutions : workers (Worker Threads), offload vers des services, file de jobs, langages dédiés.
- Les performances dépendent beaucoup de la **qualité du code asynchrone** et de la gestion des erreurs.

---

## 3. Installation & environnement de travail

### Installer Node.js

Recommandation : installer via un gestionnaire de versions (ex. **nvm**) ou installer la version LTS.

Vérifier :

```bash
node -v
npm -v
```

### Créer un dossier de projet

```bash
mkdir intro-node
cd intro-node
npm init -y
```

Vous obtenez un fichier `package.json` (métadonnées projet + scripts + dépendances).

---

## 4. Premier pas : exécuter du JavaScript hors navigateur

### Hello Node

Créer `index.js` :

```js
console.log('Hello from Node.js');
```

Exécuter :

```bash
node index.js
```

### Comprendre `process`

Node expose des infos runtime via `process` :

```js
console.log('PID:', process.pid);
console.log('Platform:', process.platform);
console.log('Node version:', process.version);
```

Arguments CLI :

```js
// node index.js --name=Alice
console.log(process.argv);
```

---

## 5. Modules : CommonJS vs ES Modules

Node supporte deux systèmes :

### 5.1 CommonJS (historique)

- Import : `require()`
- Export : `module.exports`

Exemple `math.cjs` :

```js
function add(a, b) {
  return a + b;
}

module.exports = { add };
```

Exemple `index.cjs` :

```js
const { add } = require('./math.cjs');
console.log(add(2, 3));
```

### 5.2 ES Modules (standard moderne)

- Import : `import ... from ...`
- Export : `export`

Pour activer ES Modules :

- soit utiliser l’extension `.mjs`,
- soit mettre dans `package.json` :

```json
{
  "type": "module"
}
```

Exemple `math.js` :

```js
export function add(a, b) {
  return a + b;
}
```

Exemple `index.js` :

```js
import { add } from './math.js';
console.log(add(2, 3));
```

### Bonnes pratiques (intro)

- Ne mélangez pas CommonJS et ESM au hasard.
- Pour un nouveau projet, choisissez **ESM** sauf contrainte.

---

## 6. npm : gestion des dépendances et scripts

### Installer une dépendance

```bash
npm install express
```

Cela ajoute :

- `node_modules/` (dépendances)
- `package-lock.json` (verrouillage de versions)
- `dependencies` dans `package.json`

### Installer une dépendance de dev

```bash
npm install -D nodemon
```

### Scripts npm

Dans `package.json` :

```json
{
  "scripts": {
    "start": "node index.js",
    "dev": "nodemon index.js"
  }
}
```

Exécution :

```bash
npm run dev
npm start
```

---

## 7. Asynchronisme : callbacks, Promises, async/await

Node est conçu pour gérer de nombreuses opérations I/O sans bloquer.

### 7.1 Callbacks (style historique)

Exemple conceptuel :

```js
function doAsync(cb) {
  setTimeout(() => {
    cb(null, 'done');
  }, 200);
}

doAsync((err, result) => {
  if (err) return console.error(err);
  console.log(result);
});
```

Limite : les callbacks imbriqués deviennent difficiles à lire (« callback hell »).

### 7.2 Promises

```js
function doAsync() {
  return new Promise((resolve) => {
    setTimeout(() => resolve('done'), 200);
  });
}

doAsync().then(console.log).catch(console.error);
```

### 7.3 async/await

```js
async function main() {
  try {
    const result = await doAsync();
    console.log(result);
  } catch (err) {
    console.error(err);
  }
}

main();
```

**À retenir** : `async/await` est une syntaxe au-dessus des Promises, souvent la plus lisible.

---

## 8. Event Loop & I/O non bloquantes (intro)

### Idée générale

Node exécute votre JS sur un **thread principal**. Pour rester performant, il évite de bloquer ce thread avec des opérations lentes.

- Les opérations I/O (disque, réseau) sont déléguées.
- Lorsque l’opération se termine, un callback (ou la résolution d’une Promise) est planifié.

### Conséquence pratique

- Node peut gérer **beaucoup de connexions** simultanées.
- Mais si vous faites un calcul CPU lourd dans le thread principal, tout le serveur peut ralentir.

### Exemple de blocage (à éviter)

```js
// Exemple volontairement mauvais
function heavy() {
  let sum = 0;
  for (let i = 0; i < 2e9; i++) sum += i;
  return sum;
}

console.log('start');
console.log(heavy());
console.log('end');
```

Pendant `heavy()`, le process ne répond plus.

---

## 9. Le module `fs` : lecture/écriture de fichiers (intro)

Node fournit des modules « core » (inclus) : `fs`, `path`, `http`, etc.

### Lire un fichier en async (Promises)

Créez `read-file.js` :

```js
import { readFile } from 'node:fs/promises';

async function main() {
  const content = await readFile('./data.txt', 'utf-8');
  console.log(content);
}

main().catch(console.error);
```

### Récap : chemins avec `path`

```js
import path from 'node:path';

console.log(path.join('a', 'b', 'c.txt')); // a/b/c.txt
```

**Bon réflexe** : évitez de concaténer les chemins à la main.

---

## 10. Créer un serveur HTTP minimal

### 10.1 Avec le module `http` (sans framework)

Créer `server.js` :

```js
import http from 'node:http';

const server = http.createServer((req, res) => {
  // req : infos requête (url, headers, method)
  // res : réponse

  if (req.url === '/health') {
    res.writeHead(200, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({ status: 'ok' }));
    return;
  }

  res.writeHead(200, { 'Content-Type': 'text/plain; charset=utf-8' });
  res.end('Hello HTTP from Node');
});

server.listen(3000, () => {
  console.log('Server listening on http://localhost:3000');
});
```

Lancer :

```bash
node server.js
```

Tester :

- Navigateur : `http://localhost:3000`
- Ou curl :

```bash
curl http://localhost:3000/health
```

### 10.2 Pourquoi utiliser un framework ensuite ?

Le module `http` est bas niveau. Un framework (Express/Fastify) aide pour :

- Routing plus clair
- Middlewares
- Gestion du JSON, erreurs, validation
- Logs, sécurité, etc.

---

## 11. Structurer un mini-backend (bonne pratiques de base)

Même pour une intro, prenez de bonnes habitudes :

### Structure minimale

```txt
intro-node/
  package.json
  src/
    server.js
    routes/
      health.js
```

### Exemple simple (découpage)

`src/routes/health.js` :

```js
export function handleHealth(req, res) {
  res.writeHead(200, { 'Content-Type': 'application/json' });
  res.end(JSON.stringify({ status: 'ok' }));
}
```

`src/server.js` :

```js
import http from 'node:http';
import { handleHealth } from './routes/health.js';

const server = http.createServer((req, res) => {
  if (req.url === '/health') return handleHealth(req, res);

  res.writeHead(404, { 'Content-Type': 'text/plain; charset=utf-8' });
  res.end('Not Found');
});

server.listen(3000, () => {
  console.log('Server listening on http://localhost:3000');
});
```

### Erreurs & robustesse (bases)

- Renvoyer des codes HTTP cohérents (200, 404, 500…)
- Logger les erreurs
- Ne pas exposer des infos sensibles

---

## 12. Exercices & corrections guidées

### Exercice 1 — Créer un script CLI

**Objectif** : exécuter `node greet.js Alice` et afficher `Bonjour Alice`.

**Consignes** :
- lire `process.argv`
- gérer le cas où le nom n’est pas fourni

**Proposition de correction** (`greet.js`) :

```js
const name = process.argv[2];

if (!name) {
  console.error('Usage: node greet.js <name>');
  process.exit(1);
}

console.log(`Bonjour ${name}`);
```

---

### Exercice 2 — Lire un fichier et afficher le nombre de lignes

**Objectif** : lire `data.txt` et afficher le nombre de lignes.

**Indices** :
- `readFile(..., 'utf-8')`
- `split('\n')`

**Correction (ESM)** :

```js
import { readFile } from 'node:fs/promises';

async function main() {
  const content = await readFile('./data.txt', 'utf-8');
  const lines = content.split(/\r?\n/).filter(Boolean);
  console.log('Nombre de lignes:', lines.length);
}

main().catch(console.error);
```

---

### Exercice 3 — Endpoint `/time`

**Objectif** : ajouter un endpoint HTTP `GET /time` qui renvoie la date serveur en JSON.

**Correction** (extrait) :

```js
if (req.url === '/time') {
  res.writeHead(200, { 'Content-Type': 'application/json' });
  res.end(JSON.stringify({ now: new Date().toISOString() }));
  return;
}
```

---

### Exercice 4 — Notion de concurrence (petit test)

**Objectif** : démontrer que Node sert plusieurs requêtes sans bloquer quand les traitements sont non bloquants.

**Idée** : créer un endpoint `/wait?ms=200` qui utilise `setTimeout` avant de répondre.

**Correction** (extrait) :

```js
import { URL } from 'node:url';

const url = new URL(req.url, 'http://localhost');

if (url.pathname === '/wait') {
  const ms = Number(url.searchParams.get('ms') ?? 200);
  setTimeout(() => {
    res.writeHead(200, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({ waited: ms }));
  }, ms);
  return;
}
```

Test : lancez plusieurs `curl` en parallèle et observez que ça répond correctement.

---

## 13. Conclusion & prochaines étapes

Vous avez vu que :

- **Node.js** exécute JavaScript **côté serveur**, basé sur **V8**.
- Son modèle **asynchrone** est très adapté aux backends I/O.
- Vous savez créer un projet, gérer des dépendances, écrire des modules, lire des fichiers et servir du HTTP.

### Pour aller plus loin

- Framework web : **Express**, **Fastify**, ou **NestJS**
- Accès BDD : PostgreSQL (node-postgres), MongoDB, Prisma
- Qualité : ESLint, Prettier, tests (Vitest/Jest)
- Production : logs structurés, variables d’env, monitoring, Docker
- Performance : profiling, worker threads pour CPU-bound

---

## Annexe — Mémo commandes

```bash
node -v
npm -v

npm init -y
npm i <package>
npm i -D <package>

npm run dev
npm start
```
