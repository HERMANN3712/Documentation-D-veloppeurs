# Formation — Modules Node.js

> **Public cible** : développeurs (débutants à intermédiaires) utilisant Node.js.
>
> **Prérequis** : JavaScript (ES6+), notions de CLI, compréhension de base du runtime Node.
>
> **Objectifs pédagogiques**
>
> - Comprendre le **système de modules** de Node.js et son intérêt pour structurer une application.
> - Savoir importer/exporter du code avec **CommonJS** (`require`, `module.exports`) et **ES Modules** (`import`, `export`).
> - Utiliser correctement des **modules intégrés** : `fs`, `path`, `os`, `http`.
> - Savoir choisir entre CJS et ESM, et gérer l’interopérabilité.

---

## Plan de la formation

1. **Pourquoi un système de modules ?**
   - Problèmes du code monolithique
   - Encapsulation, réutilisabilité, testabilité
2. **Deux systèmes de modules dans Node.js**
   - CommonJS (historique)
   - ES Modules (standard moderne)
3. **CommonJS en pratique**
   - `require()`, `module.exports`, `exports`
   - Résolution des chemins
   - Cache des modules
4. **ES Modules (ESM) en pratique**
   - `import`, `export` (nommés / par défaut)
   - Activation ESM (`"type": "module"` / `.mjs`)
   - Particularités (chemins, `__dirname`)
5. **Interopérabilité CJS ↔ ESM**
   - Importer du CJS en ESM
   - Importer de l’ESM en CJS (limitations)
6. **Modules intégrés (core modules)**
   - `fs` : lecture/écriture de fichiers
   - `path` : manipulation de chemins cross-platform
   - `os` : informations système
   - `http` : serveur HTTP minimal
7. **Bonnes pratiques et organisation de code**
   - Structure de projet
   - Nommer ses exports
   - Gestion des effets de bord
8. **Exercices guidés**
   - Mini-projet : serveur HTTP qui expose des infos OS et lit un fichier

---

## 1) Pourquoi un système de modules ?

Quand une base de code grandit, mettre tout dans un seul fichier pose des problèmes :

- **Lisibilité** : trop de responsabilités dans un même fichier.
- **Évolutivité** : difficile d’ajouter des fonctionnalités sans casser l’existant.
- **Réutilisation** : duplication de code.
- **Tests** : tester une fonction isolée devient compliqué.

### Ce que permettent les modules

- **Encapsuler** des fonctionnalités (une API claire).
- **Séparer** les responsabilités (I/O, logique métier, utilitaires).
- **Réutiliser** du code via import/export.
- **Composer** une application avec des briques.

---

## 2) Deux systèmes de modules dans Node.js

Node.js supporte deux systèmes :

1. **CommonJS (CJS)** — historique dans Node.
   - Chargement synchrone
   - Utilise `require()` et `module.exports`

2. **ES Modules (ESM)** — standard JavaScript moderne.
   - Utilise `import`/`export`
   - Support natif dans Node (avec configuration)

> Dans les projets modernes, ESM est souvent préféré (aligné avec le standard), mais de nombreux packages et projets existent encore en CJS.

---

## 3) CommonJS en pratique

### 3.1 Exporter avec `module.exports`

**`math.js`**

```js
function add(a, b) {
  return a + b;
}

function sub(a, b) {
  return a - b;
}

module.exports = { add, sub };
```

### 3.2 Importer avec `require()`

**`index.js`**

```js
const math = require('./math');

console.log(math.add(2, 3)); // 5
console.log(math.sub(7, 4)); // 3
```

### 3.3 Raccourci : `exports`

Node fournit `exports` comme référence vers `module.exports`.

```js
// utils.js
exports.capitalize = (s) => s.charAt(0).toUpperCase() + s.slice(1);
```

⚠️ Attention : si vous réassignez `exports = ...`, vous cassez le lien.

```js
// 🚫 À éviter
exports = function () {};

// ✅ Correct
module.exports = function () {};
```

### 3.4 Résolution des modules (idée générale)

- `require('fs')` → module **intégré** (core)
- `require('./math')` → fichier local (relatif)
- `require('lodash')` → package dans `node_modules`

### 3.5 Cache des modules

CommonJS met en cache les modules après le premier chargement.

```js
// counter.js
let count = 0;

module.exports = () => {
  count += 1;
  return count;
};
```

```js
// index.js
const counter = require('./counter');

console.log(counter()); // 1
console.log(counter()); // 2

// require('./counter') renverra le même module (cache)
```

Conséquence :
- Les **effets de bord** au chargement (ex: connexion DB) peuvent se produire une seule fois.
- Bien pour les singletons, mais attention au test (isolation).

---

## 4) ES Modules (ESM) en pratique

### 4.1 Activer ESM

Deux approches :

1. Dans `package.json` :

```json
{
  "type": "module"
}
```

2. Utiliser l’extension `.mjs`.

> Recommandation : utiliser `"type": "module"` pour un projet ESM cohérent.

### 4.2 Export nommé

**`math.js`** (en ESM)

```js
export function add(a, b) {
  return a + b;
}

export function sub(a, b) {
  return a - b;
}
```

**`index.js`**

```js
import { add, sub } from './math.js';

console.log(add(2, 3));
console.log(sub(7, 4));
```

> En ESM, l’extension `.js` doit généralement être explicitement indiquée dans les imports de fichiers locaux.

### 4.3 Export par défaut

**`logger.js`**

```js
export default function logger(message) {
  console.log(`[LOG] ${message}`);
}
```

**`index.js`**

```js
import logger from './logger.js';

logger('Hello');
```

### 4.4 Importer tout un module

```js
import * as math from './math.js';

console.log(math.add(1, 2));
```

### 4.5 Particularités ESM : `__dirname` et `__filename`

En CommonJS, on dispose de `__dirname` et `__filename`. En ESM, il faut les reconstruire :

```js
import { fileURLToPath } from 'node:url';
import path from 'node:path';

const __filename = fileURLToPath(import.meta.url);
const __dirname = path.dirname(__filename);

console.log(__dirname);
```

---

## 5) Interopérabilité CJS ↔ ESM

### 5.1 Importer un module CommonJS depuis ESM

Si un module CJS exporte un objet via `module.exports = ...`, alors en ESM vous pouvez souvent faire :

```js
import pkg from './cjs-module.cjs';
// pkg correspond à module.exports
```

### 5.2 Importer un module ESM depuis CommonJS

C’est plus contraint. On ne peut pas `require()` un ESM directement. Il faut utiliser un import dynamique :

```js
// index.cjs
(async () => {
  const mod = await import('./esm-module.js');
  console.log(mod);
})();
```

### 5.3 Choisir entre CJS et ESM

- **ESM** : recommandé pour nouveaux projets, cohérent avec l’écosystème moderne.
- **CJS** : utile si vous dépendez d’un outillage/legacy ou si votre environnement impose CJS.

---

## 6) Modules intégrés (core modules)

Node.js fournit des modules « intégrés » disponibles sans installation. Dans cette formation :

- `fs`
- `path`
- `os`
- `http`

> Convention moderne : préfixer avec `node:` (ex: `node:fs`) pour indiquer explicitement un core module.

---

## 6.1 `fs` — File System

### Lire un fichier (promesses)

```js
import { readFile } from 'node:fs/promises';

const content = await readFile('./data.txt', 'utf8');
console.log(content);
```

### Écrire un fichier

```js
import { writeFile } from 'node:fs/promises';

await writeFile('./out.txt', 'Bonjour', 'utf8');
```

### Lire un fichier (callback — héritage)

```js
const fs = require('node:fs');

fs.readFile('./data.txt', 'utf8', (err, content) => {
  if (err) return console.error(err);
  console.log(content);
});
```

Bonnes pratiques :
- Préférer `node:fs/promises` avec `async/await`.
- Toujours gérer les erreurs (fichier absent, permissions).

---

## 6.2 `path` — Manipulation des chemins

`path` évite les problèmes de séparateurs (`/` vs `\\`) selon l’OS.

### Construire un chemin

```js
import path from 'node:path';

const filePath = path.join('logs', 'app.log');
console.log(filePath);
```

### Extraire des infos

```js
import path from 'node:path';

console.log(path.extname('report.pdf')); // .pdf
console.log(path.basename('/tmp/test.txt')); // test.txt
```

### Résoudre un chemin absolu

```js
import path from 'node:path';

console.log(path.resolve('data', 'input.json'));
```

---

## 6.3 `os` — Informations système

```js
import os from 'node:os';

console.log('Plateforme:', os.platform());
console.log('Arch:', os.arch());
console.log('CPUs:', os.cpus().length);
console.log('Mémoire libre (MB):', Math.round(os.freemem() / 1024 / 1024));
```

Cas d’usage :
- Instrumentation, diagnostic
- Paramétrage du nombre de workers (CPU)

---

## 6.4 `http` — Serveur HTTP minimal

### Serveur simple

```js
import http from 'node:http';

const server = http.createServer((req, res) => {
  res.statusCode = 200;
  res.setHeader('Content-Type', 'text/plain; charset=utf-8');
  res.end('Hello depuis Node HTTP');
});

server.listen(3000, () => {
  console.log('Serveur démarré sur http://localhost:3000');
});
```

### Router minimal (selon `req.url`)

```js
import http from 'node:http';

const server = http.createServer((req, res) => {
  if (req.url === '/health') {
    res.writeHead(200, { 'Content-Type': 'application/json; charset=utf-8' });
    return res.end(JSON.stringify({ status: 'ok' }));
  }

  res.writeHead(404, { 'Content-Type': 'text/plain; charset=utf-8' });
  res.end('Not found');
});

server.listen(3000);
```

> Dans la vraie vie, on utilise souvent des frameworks (Express, Fastify). Mais comprendre `http` aide à saisir les fondamentaux (requêtes/réponses, headers, status codes).

---

## 7) Bonnes pratiques et organisation de code

### 7.1 Structurer un projet

Exemple (ESM) :

```txt
src/
  server/
    httpServer.js
  services/
    systemService.js
  utils/
    fileUtils.js
  index.js
package.json
```

- `utils/` : fonctions pures et réutilisables.
- `services/` : logique métier.
- `server/` : couche transport (HTTP).

### 7.2 Limiter les effets de bord au chargement

À éviter :

```js
// db.js
connectToDb(); // effet de bord immédiat
```

Préférer :

```js
// db.js
export function connect() {
  // ...
}
```

### 7.3 Nommer clairement les exports

- Exports nommés : meilleure auto-complétion, refactoring facilité.
- Export par défaut : utile pour une « principale » fonction/classe par fichier.

---

## 8) Exercices guidés (avec corrigés)

### Exercice 1 — Passer de CJS à ESM

**Énoncé** :
- Créez un module `math` en CommonJS.
- Faites une version ESM.
- Vérifiez l’usage de l’extension `.js` en ESM.

**Corrigé (CJS)**

```js
// math.cjs
module.exports = {
  add: (a, b) => a + b,
};
```

```js
// index.cjs
const { add } = require('./math.cjs');
console.log(add(1, 2));
```

**Corrigé (ESM)**

```js
// math.js
export const add = (a, b) => a + b;
```

```js
// index.js
import { add } from './math.js';
console.log(add(1, 2));
```

---

### Exercice 2 — Mini-projet : serveur HTTP + FS + OS

**Objectif** : créer un serveur HTTP qui expose :

- `GET /` → message de bienvenue
- `GET /os` → infos basiques (platform, arch, cpus)
- `GET /file?name=data.txt` → lit un fichier et renvoie son contenu

#### Implémentation (ESM)

**`src/services/systemService.js`**

```js
import os from 'node:os';

export function getSystemInfo() {
  return {
    platform: os.platform(),
    arch: os.arch(),
    cpuCount: os.cpus().length,
  };
}
```

**`src/services/fileService.js`**

```js
import { readFile } from 'node:fs/promises';

export async function readTextFile(fileName) {
  // Minimal : dans un vrai projet, valider/sécuriser le chemin (anti path traversal)
  const content = await readFile(`./${fileName}`, 'utf8');
  return content;
}
```

**`src/server/httpServer.js`**

```js
import http from 'node:http';
import { URL } from 'node:url';

import { getSystemInfo } from '../services/systemService.js';
import { readTextFile } from '../services/fileService.js';

export function startServer(port = 3000) {
  const server = http.createServer(async (req, res) => {
    try {
      const url = new URL(req.url, `http://${req.headers.host}`);

      if (url.pathname === '/') {
        res.writeHead(200, { 'Content-Type': 'text/plain; charset=utf-8' });
        return res.end('Bienvenue sur le mini serveur Node.js');
      }

      if (url.pathname === '/os') {
        res.writeHead(200, { 'Content-Type': 'application/json; charset=utf-8' });
        return res.end(JSON.stringify(getSystemInfo()));
      }

      if (url.pathname === '/file') {
        const name = url.searchParams.get('name');
        if (!name) {
          res.writeHead(400, { 'Content-Type': 'application/json; charset=utf-8' });
          return res.end(JSON.stringify({ error: 'Missing query param: name' }));
        }

        const content = await readTextFile(name);
        res.writeHead(200, { 'Content-Type': 'text/plain; charset=utf-8' });
        return res.end(content);
      }

      res.writeHead(404, { 'Content-Type': 'text/plain; charset=utf-8' });
      res.end('Not found');
    } catch (err) {
      // Gestion d'erreur minimaliste
      res.writeHead(500, { 'Content-Type': 'application/json; charset=utf-8' });
      res.end(JSON.stringify({ error: 'Internal Server Error', message: err.message }));
    }
  });

  server.listen(port, () => {
    console.log(`Server listening on http://localhost:${port}`);
  });

  return server;
}
```

**`src/index.js`**

```js
import { startServer } from './server/httpServer.js';

startServer(3000);
```

#### Tests manuels

- `GET http://localhost:3000/`
- `GET http://localhost:3000/os`
- `GET http://localhost:3000/file?name=data.txt`

#### Points d’attention / discussion

- **Sécurité** : `readTextFile` est vulnérable au *path traversal* (ex: `name=../../etc/passwd`).
  - Correctif : autoriser seulement un répertoire, normaliser avec `path.resolve`, refuser les `..`.
- **Encodage** : renvoyer `charset=utf-8`.
- **Gestion d’erreurs** : distinguer 404 fichier vs 500.

---

## Annexes

### A) Pense-bête — syntaxe CJS vs ESM

| Besoin | CommonJS | ES Modules |
|---|---|---|
| Import | `const x = require('x')` | `import x from 'x'` |
| Export objet | `module.exports = { ... }` | `export { ... }` |
| Export par défaut | `module.exports = fn` | `export default fn` |
| `__dirname` | disponible | via `import.meta.url` |

### B) Modules intégrés cités

- `fs` : opérations fichiers
- `path` : manipulation chemins
- `os` : infos système
- `http` : serveur HTTP

---

## Conclusion

Le système de modules est la base de la structuration d’une application Node.js. Savoir manipuler **CommonJS** et **ESM**, comprendre la résolution et le cache, et exploiter les modules intégrés (`fs`, `path`, `os`, `http`) permet de construire des applications plus lisibles, robustes et maintenables.
