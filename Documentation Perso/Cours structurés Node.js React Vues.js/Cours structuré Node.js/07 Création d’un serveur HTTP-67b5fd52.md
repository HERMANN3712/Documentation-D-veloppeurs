# Formation Node.js — Création d’un serveur HTTP

> Objectif : savoir créer, démarrer et faire évoluer un serveur HTTP avec **Node.js** (module natif `http`), comprendre le cycle **requête → réponse**, gérer les routes, les en-têtes, les erreurs et les bases de performance.

---

## Plan de la formation

1. **Introduction au HTTP et au rôle d’un serveur**
2. **Le module natif `http` de Node.js**
3. **Créer un premier serveur minimal**
4. **Comprendre `req` et `res` (Request / Response)**
5. **Écouter un port et gérer le cycle de vie du serveur**
6. **Routing simple (URL, méthodes HTTP)**
7. **Réponses : statut, headers, contenu (texte/HTML/JSON)**
8. **Lecture du corps (body) d’une requête**
9. **Servir des fichiers (statique) et gérer les erreurs**
10. **Bonnes pratiques : sécurité, performance, validation**
11. **Atelier : mini API HTTP 완 Node.js (sans framework)**
12. **Quiz + récapitulatif**

---

## 1) Introduction : HTTP et serveur web

### Qu’est-ce que HTTP ?
HTTP (HyperText Transfer Protocol) est un protocole de communication **client ↔ serveur**.

- Le **client** (navigateur, app mobile, outil comme `curl`, etc.) envoie une **requête**.
- Le **serveur** reçoit la requête, la traite et renvoie une **réponse**.

Une interaction HTTP typique :
- Requête : méthode (GET/POST…), URL, en-têtes, parfois un body.
- Réponse : statut (200/404…), en-têtes, body (HTML, JSON, fichier…).

### Notions fondamentales
- **Port** : un serveur écoute sur un port (ex. 3000). Une machine peut héberger plusieurs services via différents ports.
- **Adresse** : `http://localhost:3000` pointe vers votre machine locale.
- **Statuts HTTP** : 200 (OK), 201 (Created), 400 (Bad Request), 404 (Not Found), 500 (Server Error)…
- **Headers (en-têtes)** : métadonnées (Content-Type, Authorization, etc.).

---

## 2) Le module `http` de Node.js

Node.js propose un module **natif** `http` permettant de créer un serveur web **sans dépendance externe**.

- Avantages : léger, pédagogique, contrôle fin.
- Inconvénients : vous devez gérer vous-même routing, parsing, erreurs… (ce que des frameworks comme Express abstraient).

Import :

```js
const http = require('http');
```

---

## 3) Créer un serveur minimal

### Exemple : “Hello World”
Créez un fichier `server.js` :

```js
const http = require('http');

const server = http.createServer((req, res) => {
  res.statusCode = 200;
  res.setHeader('Content-Type', 'text/plain; charset=utf-8');
  res.end('Hello from Node.js HTTP server!');
});

server.listen(3000, () => {
  console.log('Server listening on http://localhost:3000');
});
```

Exécution :

```bash
node server.js
```

Test :

- Navigateur : ouvrir `http://localhost:3000`
- Terminal :

```bash
curl -i http://localhost:3000
```

### Ce qui se passe
- `http.createServer(handler)` crée un serveur.
- À chaque requête, Node appelle `handler(req, res)`.
- `res.end()` termine la réponse (à ne pas oublier).

---

## 4) Comprendre `req` et `res`

### L’objet `req`
`req` (IncomingMessage) contient les informations de la requête.

Propriétés utiles :
- `req.method` (GET, POST…)
- `req.url` (ex. `/api/users?limit=10`)
- `req.headers` (objet des headers)

Exemple de log :

```js
const server = http.createServer((req, res) => {
  console.log('method:', req.method);
  console.log('url:', req.url);
  console.log('headers:', req.headers);

  res.end('OK');
});
```

### L’objet `res`
`res` (ServerResponse) permet de construire la réponse.

- `res.statusCode = 200`
- `res.setHeader(name, value)`
- `res.write(chunk)` (optionnel, pour écrire en plusieurs morceaux)
- `res.end(body)` (termine la réponse)

---

## 5) Écouter un port & cycle de vie du serveur

### Choisir le port
- En local : 3000/4000 souvent
- En production : souvent 80/443 derrière un reverse proxy
- Bon réflexe : utiliser une variable d’environnement

```js
const PORT = process.env.PORT || 3000;
server.listen(PORT, () => console.log(`Listening on ${PORT}`));
```

### Gérer les erreurs de démarrage
Ex. port déjà utilisé (`EADDRINUSE`).

```js
server.on('error', (err) => {
  console.error('Server error:', err);
  process.exit(1);
});
```

### Arrêt propre (graceful shutdown)

```js
process.on('SIGINT', () => {
  console.log('Shutting down...');
  server.close(() => {
    console.log('Server closed');
    process.exit(0);
  });
});
```

---

## 6) Routing simple (URL + méthode)

Sans framework, on route souvent avec `if`/`switch` :

```js
const server = http.createServer((req, res) => {
  if (req.method === 'GET' && req.url === '/') {
    res.writeHead(200, { 'Content-Type': 'text/html; charset=utf-8' });
    return res.end('<h1>Accueil</h1>');
  }

  if (req.method === 'GET' && req.url === '/health') {
    res.writeHead(200, { 'Content-Type': 'application/json; charset=utf-8' });
    return res.end(JSON.stringify({ status: 'ok' }));
  }

  res.writeHead(404, { 'Content-Type': 'text/plain; charset=utf-8' });
  res.end('Not Found');
});
```

### Attention : query string
`req.url` inclut la query string (`/users?limit=10`). Pour séparer proprement :

```js
const { URL } = require('url');

const server = http.createServer((req, res) => {
  const requestUrl = new URL(req.url, `http://${req.headers.host}`);
  const pathname = requestUrl.pathname; // "/users"
  const limit = requestUrl.searchParams.get('limit'); // "10"

  // ...routing sur pathname
});
```

---

## 7) Construire une réponse : statut, headers, contenu

### Répondre en texte

```js
res.writeHead(200, { 'Content-Type': 'text/plain; charset=utf-8' });
res.end('Bonjour');
```

### Répondre en HTML

```js
res.writeHead(200, { 'Content-Type': 'text/html; charset=utf-8' });
res.end('<h1>Page</h1><p>Serveur Node</p>');
```

### Répondre en JSON
Toujours définir `Content-Type`.

```js
const data = { message: 'Hello', time: new Date().toISOString() };
res.writeHead(200, { 'Content-Type': 'application/json; charset=utf-8' });
res.end(JSON.stringify(data));
```

### Codes statut utiles
- **200** OK
- **201** Created (souvent après un POST)
- **204** No Content (réponse vide)
- **400** Bad Request (données invalides)
- **401** Unauthorized
- **403** Forbidden
- **404** Not Found
- **405** Method Not Allowed
- **500** Internal Server Error

---

## 8) Lire le body d’une requête (POST/PUT)

Dans Node natif, le corps est un **stream** : il faut écouter `data` puis `end`.

### Exemple : recevoir du JSON

```js
function readJsonBody(req) {
  return new Promise((resolve, reject) => {
    let raw = '';

    req.on('data', (chunk) => {
      raw += chunk;
      // Protection simple contre payload trop gros
      if (raw.length > 1e6) {
        reject(new Error('Payload too large'));
        req.destroy();
      }
    });

    req.on('end', () => {
      try {
        const parsed = raw ? JSON.parse(raw) : {};
        resolve(parsed);
      } catch (e) {
        reject(new Error('Invalid JSON'));
      }
    });

    req.on('error', reject);
  });
}
```

Usage :

```js
const server = http.createServer(async (req, res) => {
  if (req.method === 'POST' && req.url === '/echo') {
    try {
      const body = await readJsonBody(req);
      res.writeHead(200, { 'Content-Type': 'application/json; charset=utf-8' });
      return res.end(JSON.stringify({ received: body }));
    } catch (err) {
      res.writeHead(400, { 'Content-Type': 'application/json; charset=utf-8' });
      return res.end(JSON.stringify({ error: err.message }));
    }
  }

  res.writeHead(404);
  res.end();
});
```

---

## 9) Servir des fichiers (statique) & gérer les erreurs

### Lire un fichier et le renvoyer
Pour des fichiers statiques simples, on peut utiliser `fs.createReadStream`.

```js
const fs = require('fs');
const path = require('path');

function sendFile(res, filePath, contentType) {
  res.writeHead(200, { 'Content-Type': contentType });
  fs.createReadStream(filePath)
    .on('error', () => {
      res.writeHead(404, { 'Content-Type': 'text/plain; charset=utf-8' });
      res.end('File not found');
    })
    .pipe(res);
}

const server = http.createServer((req, res) => {
  if (req.method === 'GET' && req.url === '/') {
    return sendFile(res, path.join(__dirname, 'index.html'), 'text/html; charset=utf-8');
  }

  res.writeHead(404);
  res.end();
});
```

### Gestion d’erreurs générales
Encapsulez votre logique pour éviter les crash.

```js
const server = http.createServer((req, res) => {
  try {
    // logique
    res.end('OK');
  } catch (err) {
    console.error(err);
    res.writeHead(500, { 'Content-Type': 'text/plain; charset=utf-8' });
    res.end('Internal Server Error');
  }
});
```

---

## 10) Bonnes pratiques (essentiel)

### 10.1 Séparer les responsabilités
Même sans framework, organisez :
- `router(req, res)`
- `handlers/health.js`, `handlers/users.js`
- utilitaires `readBody`, `sendJson`, etc.

### 10.2 Headers importants
- `Content-Type`
- `Cache-Control` (si besoin)
- `Access-Control-Allow-Origin` (CORS) si API consommée par front

### 10.3 Sécurité minimale
- Limiter taille des payloads
- Valider l’input (schéma simple)
- Ne pas exposer d’informations sensibles dans les erreurs

### 10.4 Performance
- Préférer les streams pour fichiers
- Éviter le JSON.stringify géant si possible
- Logguer sans bloquer (éviter logs trop verbeux en prod)

---

## 11) Atelier — Mini API “Tasks” (CRUD simplifié)

Objectif : construire une API HTTP sans Express.

### Spécification
- `GET /health` → `{ "status": "ok" }`
- `GET /tasks` → liste
- `POST /tasks` avec body `{ "title": "..." }` → crée une tâche
- `GET /tasks?id=...` (simplifié) → retourne une tâche
- Codes statuts appropriés

### Implémentation complète (fichier unique)

Créez `tasks-server.js` :

```js
const http = require('http');
const { URL } = require('url');

const PORT = process.env.PORT || 3000;

/** In-memory store */
const tasks = new Map();

function sendJson(res, statusCode, payload) {
  const body = JSON.stringify(payload);
  res.writeHead(statusCode, {
    'Content-Type': 'application/json; charset=utf-8',
    'Content-Length': Buffer.byteLength(body),
  });
  res.end(body);
}

function readJsonBody(req) {
  return new Promise((resolve, reject) => {
    let raw = '';

    req.on('data', (chunk) => {
      raw += chunk;
      if (raw.length > 1e6) {
        reject(new Error('Payload too large'));
        req.destroy();
      }
    });

    req.on('end', () => {
      try {
        resolve(raw ? JSON.parse(raw) : {});
      } catch {
        reject(new Error('Invalid JSON'));
      }
    });

    req.on('error', reject);
  });
}

function createId() {
  return Math.random().toString(16).slice(2, 10);
}

const server = http.createServer(async (req, res) => {
  const requestUrl = new URL(req.url, `http://${req.headers.host}`);
  const pathname = requestUrl.pathname;

  // Route: health
  if (req.method === 'GET' && pathname === '/health') {
    return sendJson(res, 200, { status: 'ok' });
  }

  // Route: list tasks
  if (req.method === 'GET' && pathname === '/tasks' && !requestUrl.searchParams.get('id')) {
    const list = Array.from(tasks.values());
    return sendJson(res, 200, { tasks: list });
  }

  // Route: get task by id (simplifié via query param)
  if (req.method === 'GET' && pathname === '/tasks' && requestUrl.searchParams.get('id')) {
    const id = requestUrl.searchParams.get('id');
    if (!tasks.has(id)) return sendJson(res, 404, { error: 'Task not found' });
    return sendJson(res, 200, { task: tasks.get(id) });
  }

  // Route: create task
  if (req.method === 'POST' && pathname === '/tasks') {
    try {
      const body = await readJsonBody(req);
      if (!body.title || typeof body.title !== 'string') {
        return sendJson(res, 400, { error: 'title is required (string)' });
      }

      const id = createId();
      const task = { id, title: body.title, done: false, createdAt: new Date().toISOString() };
      tasks.set(id, task);
      return sendJson(res, 201, { task });
    } catch (err) {
      return sendJson(res, 400, { error: err.message });
    }
  }

  // Méthode non autorisée sur routes connues
  if (pathname === '/health' || pathname === '/tasks') {
    return sendJson(res, 405, { error: 'Method Not Allowed' });
  }

  // Fallback
  sendJson(res, 404, { error: 'Not Found' });
});

server.listen(PORT, () => {
  console.log(`Tasks server listening on http://localhost:${PORT}`);
});

server.on('error', (err) => {
  console.error('Server error:', err);
  process.exit(1);
});
```

### Tests rapides (curl)

```bash
curl -i http://localhost:3000/health

curl -i http://localhost:3000/tasks

curl -i -X POST http://localhost:3000/tasks \
  -H "Content-Type: application/json" \
  -d '{"title":"Apprendre le module http"}'

curl -i "http://localhost:3000/tasks?id=VOTRE_ID"
```

---

## 12) Quiz & récapitulatif

### Quiz
1. Quel est le rôle de `res.end()` ?
2. Quelle propriété contient la méthode HTTP (`GET/POST`) ?
3. Quelle différence entre `req.url` et `pathname` ?
4. Pourquoi faut-il définir `Content-Type: application/json` ?
5. Comment lire un body en Node natif ?

### Récapitulatif
- Le module `http` permet de créer un serveur web natif.
- Un serveur **écoute un port** et répond aux **requêtes HTTP**.
- `req` décrit la requête, `res` construit la réponse.
- Routing, parsing et erreurs : à gérer explicitement.

---

## Annexes — Snippets utiles

### A) Helper : réponse JSON

```js
function sendJson(res, statusCode, payload) {
  const body = JSON.stringify(payload);
  res.writeHead(statusCode, { 'Content-Type': 'application/json; charset=utf-8' });
  res.end(body);
}
```

### B) Helper : 404

```js
function notFound(res) {
  res.writeHead(404, { 'Content-Type': 'text/plain; charset=utf-8' });
  res.end('Not Found');
}
```
