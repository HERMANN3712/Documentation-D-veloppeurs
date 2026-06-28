# Formation — Framework Express.js (Node.js)

> **Public** : développeurs web ayant des bases JavaScript / Node.js
>
> **Objectif** : construire des API REST et applications web avec **Express.js**, en maîtrisant le routage, les middlewares, la gestion des requêtes/réponses, les erreurs, la sécurité et les bonnes pratiques de structuration.

---

## Sommaire

1. [Introduction à Express.js](#1-introduction-à-expressjs)
2. [Mise en place du projet](#2-mise-en-place-du-projet)
3. [Anatomie d’une application Express](#3-anatomie-dune-application-express)
4. [Routage (Routes) — fondations](#4-routage-routes--fondations)
5. [Middlewares — le cœur d’Express](#5-middlewares--le-cœur-dexpress)
6. [Gestion des requêtes et réponses](#6-gestion-des-requêtes-et-réponses)
7. [Validation des entrées](#7-validation-des-entrées)
8. [Gestion des erreurs](#8-gestion-des-erreurs)
9. [Structurer son API (architecture recommandée)](#9-structurer-son-api-architecture-recommandée)
10. [API REST — conventions et bonnes pratiques](#10-api-rest--conventions-et-bonnes-pratiques)
11. [Sécurité de base](#11-sécurité-de-base)
12. [Authentification (JWT) — optionnel mais courant](#12-authentification-jwt--optionnel-mais-courant)
13. [Fichiers statiques, vues et templating](#13-fichiers-statiques-vues-et-templating)
14. [CORS, logs, performance et production](#14-cors-logs-performance-et-production)
15. [Tests (supertest) et qualité](#15-tests-supertest-et-qualité)
16. [Atelier final — mini API complète](#16-atelier-final--mini-api-complète)
17. [Annexes (checklists & snippets)](#17-annexes-checklists--snippets)

---

## 1. Introduction à Express.js

### 1.1. Qu’est-ce qu’Express ?

**Express.js** est un framework **minimaliste** pour **Node.js**, conçu pour créer rapidement des **API** et **applications web**. Son rôle est de fournir une couche simple au-dessus du serveur HTTP Node natif, avec :

- un **routage** clair (`app.get`, `router.post`, etc.),
- un système de **middlewares** extrêmement flexible,
- des utilitaires pour manipuler **requêtes HTTP** (headers, body, query, params…),
- une approche modulaire permettant d’organiser votre code.

### 1.2. Quand utiliser Express ?

Express est très adapté :

- aux **API REST** (CRUD, microservices),
- aux applications web simples (servir des pages, SSR),
- comme fondation d’un back-end Node flexible.

Alternatives (à connaître) : Fastify (performance), NestJS (architecture/DI), Koa (plus bas niveau).

### 1.3. Ce qu’Express *n’est pas*

- Un framework « batteries included » : Express ne vous impose pas d’ORM, de validation ou d’architecture.
- Un système de sécurité complet : il faut ajouter des middlewares (helmet, rate limit, etc.).

---

## 2. Mise en place du projet

### 2.1. Prérequis

- Node.js LTS
- npm / pnpm / yarn
- Connaissance de base de HTTP et JavaScript

### 2.2. Initialisation

```bash
mkdir express-training
cd express-training
npm init -y
npm i express
npm i -D nodemon
```

Ajoutez un script dans `package.json` :

```json
{
  "scripts": {
    "dev": "nodemon src/server.js",
    "start": "node src/server.js"
  }
}
```

### 2.3. Premier serveur

Créez `src/server.js` :

```js
const express = require("express");

const app = express();

app.get("/", (req, res) => {
  res.send("Hello Express");
});

const port = process.env.PORT ?? 3000;
app.listen(port, () => {
  console.log(`Server listening on http://localhost:${port}`);
});
```

Lancez :

```bash
npm run dev
```

---

## 3. Anatomie d’une application Express

Express est basé sur l’enchaînement :

**Request** → (middlewares) → **route handler** → (middlewares d’erreur) → **Response**

Concepts clés :

- `app`: instance Express
- `req` (Request): représente la requête entrante
- `res` (Response): représente la réponse à renvoyer
- `next()`: passe au middleware suivant

---

## 4. Routage (Routes) — fondations

### 4.1. Méthodes HTTP

Express mappe les méthodes HTTP :

- `GET` : lecture
- `POST` : création
- `PUT/PATCH` : mise à jour
- `DELETE` : suppression

Exemples :

```js
app.get("/api/users", (req, res) => res.json([]));
app.post("/api/users", (req, res) => res.status(201).json({ id: 1 }));
app.delete("/api/users/:id", (req, res) => res.sendStatus(204));
```

### 4.2. Paramètres de route (`req.params`)

```js
app.get("/api/users/:id", (req, res) => {
  const { id } = req.params;
  res.json({ id, name: "Ada" });
});
```

### 4.3. Query string (`req.query`)

```js
app.get("/api/users", (req, res) => {
  const { page = "1", q } = req.query;
  res.json({ page: Number(page), q: q ?? null });
});
```

### 4.4. Router pour modulariser

Créez `src/routes/users.routes.js` :

```js
const express = require("express");
const router = express.Router();

router.get("/", (req, res) => res.json([]));
router.get("/:id", (req, res) => res.json({ id: req.params.id }));

module.exports = router;
```

Puis dans `src/server.js` :

```js
const usersRouter = require("./routes/users.routes");

app.use("/api/users", usersRouter);
```

### 4.5. Ordre des routes

Express exécute la **première route/middleware qui match**. Attention aux routes génériques (`/:id`) qui peuvent masquer d’autres endpoints.

---

## 5. Middlewares — le cœur d’Express

### 5.1. Définition

Un middleware est une fonction :

```js
(req, res, next) => { /* ... */ }
```

Rôles typiques :

- parsing du body (`express.json()`)
- authentification
- logging
- validation
- gestion d’erreurs

### 5.2. Middleware global

```js
app.use((req, res, next) => {
  console.log(`${req.method} ${req.url}`);
  next();
});
```

### 5.3. Middleware de route

```js
function requireApiKey(req, res, next) {
  const apiKey = req.header("x-api-key");
  if (apiKey !== process.env.API_KEY) {
    return res.status(401).json({ error: "Unauthorized" });
  }
  next();
}

app.get("/api/secure", requireApiKey, (req, res) => {
  res.json({ ok: true });
});
```

### 5.4. Middlewares intégrés utiles

- `express.json()` : parse JSON
- `express.urlencoded({ extended: true })` : parse form urlencoded
- `express.static()` : fichiers statiques

Exemple :

```js
app.use(express.json());
app.use(express.urlencoded({ extended: true }));
```

---

## 6. Gestion des requêtes et réponses

### 6.1. Le body (`req.body`)

Activez le parsing JSON :

```js
app.use(express.json());
```

Puis :

```js
app.post("/api/users", (req, res) => {
  const { name } = req.body;
  res.status(201).json({ id: 123, name });
});
```

### 6.2. Réponses : `res.send`, `res.json`, `res.status`

- `res.send("text")` : texte ou buffer
- `res.json(obj)` : JSON + content-type
- `res.status(201).json(...)` : code HTTP

### 6.3. Headers et cookies

```js
res.set("x-powered-by", "express-training");
```

Pour cookies, on utilise souvent `cookie-parser` (outil externe).

### 6.4. Codes HTTP usuels

- `200 OK` : succès
- `201 Created` : création
- `204 No Content` : suppression sans body
- `400 Bad Request` : validation/entrée invalide
- `401 Unauthorized` : non authentifié
- `403 Forbidden` : authentifié mais interdit
- `404 Not Found`
- `500 Internal Server Error`

---

## 7. Validation des entrées

Express ne valide pas nativement. Options courantes :

- `zod`
- `joi`
- `express-validator`

Exemple avec **zod** :

```bash
npm i zod
```

```js
const { z } = require("zod");

const createUserSchema = z.object({
  name: z.string().min(2),
  email: z.string().email(),
});

app.post("/api/users", (req, res) => {
  const parsed = createUserSchema.safeParse(req.body);
  if (!parsed.success) {
    return res.status(400).json({
      error: "ValidationError",
      details: parsed.error.flatten(),
    });
  }

  res.status(201).json({ id: 1, ...parsed.data });
});
```

Bonne pratique : centraliser la validation dans un middleware.

---

## 8. Gestion des erreurs

### 8.1. Middleware d’erreur

Signature spéciale :

```js
(err, req, res, next) => { }
```

Exemple :

```js
app.use((err, req, res, next) => {
  console.error(err);
  res.status(500).json({ error: "InternalServerError" });
});
```

### 8.2. Erreurs async (promesses)

En async/await, captez les erreurs :

```js
app.get("/api/fail", async (req, res, next) => {
  try {
    throw new Error("Boom");
  } catch (e) {
    next(e);
  }
});
```

Alternative : wrapper `asyncHandler(fn)`.

```js
const asyncHandler = (fn) => (req, res, next) =>
  Promise.resolve(fn(req, res, next)).catch(next);

app.get(
  "/api/fail",
  asyncHandler(async () => {
    throw new Error("Boom");
  })
);
```

### 8.3. 404 (route non trouvée)

À placer **après** les routes :

```js
app.use((req, res) => {
  res.status(404).json({ error: "NotFound" });
});
```

---

## 9. Structurer son API (architecture recommandée)

Objectif : éviter `server.js` géant.

### 9.1. Structure de dossier (exemple)

```
src/
  server.js
  app.js
  routes/
    users.routes.js
  controllers/
    users.controller.js
  services/
    users.service.js
  middlewares/
    error.middleware.js
    auth.middleware.js
  validators/
    users.validator.js
```

### 9.2. Séparer `app` et `server`

`src/app.js` :

```js
const express = require("express");
const usersRouter = require("./routes/users.routes");
const { notFound } = require("./middlewares/notFound.middleware");
const { errorHandler } = require("./middlewares/error.middleware");

function createApp() {
  const app = express();

  app.use(express.json());

  app.use("/api/users", usersRouter);

  app.use(notFound);
  app.use(errorHandler);

  return app;
}

module.exports = { createApp };
```

`src/server.js` :

```js
const { createApp } = require("./app");

const app = createApp();
const port = process.env.PORT ?? 3000;

app.listen(port, () => console.log(`Listening on ${port}`));
```

Bénéfice : testabilité et clarté.

---

## 10. API REST — conventions et bonnes pratiques

### 10.1. Conventions d’URL

- `/api/users` (collection)
- `/api/users/:id` (ressource)
- éviter les verbes : préférez `POST /api/users` à `/api/createUser`

### 10.2. Pagination, tri et filtrage

Exemple :

`GET /api/users?page=2&limit=20&sort=-createdAt&q=ada`

Réponse typique :

```json
{
  "data": [/* ... */],
  "meta": { "page": 2, "limit": 20, "total": 125 }
}
```

### 10.3. Versioning

- `/api/v1/...`
- ou header `Accept: application/vnd.myapi.v1+json`

### 10.4. Format d’erreurs cohérent

Exemple :

```json
{
  "error": "ValidationError",
  "message": "Invalid input",
  "details": { "email": "Invalid email" }
}
```

---

## 11. Sécurité de base

### 11.1. Helmet

```bash
npm i helmet
```

```js
const helmet = require("helmet");
app.use(helmet());
```

### 11.2. Rate limiting

```bash
npm i express-rate-limit
```

```js
const rateLimit = require("express-rate-limit");

app.use(
  rateLimit({
    windowMs: 60_000,
    limit: 100,
    standardHeaders: true,
    legacyHeaders: false,
  })
);
```

### 11.3. Principes

- ne jamais faire confiance au client
- valider/sanitizer toutes entrées
- limiter la taille du body
- logs sans informations sensibles

Limiter la taille JSON :

```js
app.use(express.json({ limit: "100kb" }));
```

---

## 12. Authentification (JWT) — optionnel mais courant

> Objectif : protéger des routes via un token bearer.

Librairies courantes : `jsonwebtoken`.

```bash
npm i jsonwebtoken
```

### 12.1. Générer un token

```js
const jwt = require("jsonwebtoken");

app.post("/api/auth/login", (req, res) => {
  const user = { id: "u1", role: "admin" }; // exemple
  const token = jwt.sign(user, process.env.JWT_SECRET, { expiresIn: "1h" });
  res.json({ token });
});
```

### 12.2. Middleware d’auth

```js
function requireAuth(req, res, next) {
  const auth = req.header("authorization");
  if (!auth?.startsWith("Bearer ")) {
    return res.status(401).json({ error: "Unauthorized" });
  }

  const token = auth.slice("Bearer ".length);

  try {
    req.user = jwt.verify(token, process.env.JWT_SECRET);
    next();
  } catch {
    res.status(401).json({ error: "Unauthorized" });
  }
}

app.get("/api/me", requireAuth, (req, res) => {
  res.json({ user: req.user });
});
```

---

## 13. Fichiers statiques, vues et templating

### 13.1. Servir des assets

```js
app.use(express.static("public"));
```

Structure :

```
public/
  index.html
  styles.css
```

### 13.2. Moteurs de templates (optionnel)

Express supporte Pug/EJS/Handlebars. Pour une formation API, c’est souvent secondaire.

---

## 14. CORS, logs, performance et production

### 14.1. CORS

```bash
npm i cors
```

```js
const cors = require("cors");

app.use(
  cors({
    origin: ["http://localhost:5173"],
    credentials: true,
  })
);
```

### 14.2. Logs HTTP

```bash
npm i morgan
```

```js
const morgan = require("morgan");
app.use(morgan("combined"));
```

### 14.3. Reverse proxy (prod)

Si vous êtes derrière Nginx/Proxy (render, heroku like), activez :

```js
app.set("trust proxy", 1);
```

### 14.4. Variables d’environnement

```bash
npm i dotenv
```

```js
require("dotenv").config();
```

---

## 15. Tests (supertest) et qualité

### 15.1. Installer

```bash
npm i -D jest supertest
```

### 15.2. Exemple de test d’une route

`src/app.test.js` :

```js
const request = require("supertest");
const { createApp } = require("./app");

describe("GET /", () => {
  it("should return 404 if not defined", async () => {
    const app = createApp();
    const res = await request(app).get("/");
    expect(res.status).toBe(404);
  });
});
```

> En pratique vous testerez des routes réelles (`/api/users`).

---

## 16. Atelier final — mini API complète

Objectif : construire une API `todos` simple, avec validation, erreurs, et structure modulaire.

### 16.1. Endpoints

- `GET /api/todos` : liste
- `POST /api/todos` : créer
- `PATCH /api/todos/:id` : toggler/éditer
- `DELETE /api/todos/:id` : supprimer

### 16.2. Données (in-memory)

`src/services/todos.service.js`

```js
const { randomUUID } = require("crypto");

const todos = [];

function listTodos() {
  return todos;
}

function createTodo({ title }) {
  const todo = { id: randomUUID(), title, done: false, createdAt: new Date().toISOString() };
  todos.push(todo);
  return todo;
}

function updateTodo(id, patch) {
  const idx = todos.findIndex((t) => t.id === id);
  if (idx === -1) return null;
  todos[idx] = { ...todos[idx], ...patch };
  return todos[idx];
}

function deleteTodo(id) {
  const idx = todos.findIndex((t) => t.id === id);
  if (idx === -1) return false;
  todos.splice(idx, 1);
  return true;
}

module.exports = { listTodos, createTodo, updateTodo, deleteTodo };
```

### 16.3. Validation

`src/validators/todos.validator.js`

```js
const { z } = require("zod");

const createTodoSchema = z.object({
  title: z.string().min(1).max(200),
});

const patchTodoSchema = z.object({
  title: z.string().min(1).max(200).optional(),
  done: z.boolean().optional(),
});

module.exports = { createTodoSchema, patchTodoSchema };
```

### 16.4. Controller

`src/controllers/todos.controller.js`

```js
const svc = require("../services/todos.service");
const { createTodoSchema, patchTodoSchema } = require("../validators/todos.validator");

function list(req, res) {
  res.json({ data: svc.listTodos() });
}

function create(req, res) {
  const parsed = createTodoSchema.safeParse(req.body);
  if (!parsed.success) {
    return res.status(400).json({ error: "ValidationError", details: parsed.error.flatten() });
  }
  const todo = svc.createTodo(parsed.data);
  res.status(201).json({ data: todo });
}

function patch(req, res) {
  const parsed = patchTodoSchema.safeParse(req.body);
  if (!parsed.success) {
    return res.status(400).json({ error: "ValidationError", details: parsed.error.flatten() });
  }

  const updated = svc.updateTodo(req.params.id, parsed.data);
  if (!updated) return res.status(404).json({ error: "NotFound" });

  res.json({ data: updated });
}

function remove(req, res) {
  const ok = svc.deleteTodo(req.params.id);
  if (!ok) return res.status(404).json({ error: "NotFound" });
  res.sendStatus(204);
}

module.exports = { list, create, patch, remove };
```

### 16.5. Routes

`src/routes/todos.routes.js`

```js
const express = require("express");
const ctrl = require("../controllers/todos.controller");

const router = express.Router();

router.get("/", ctrl.list);
router.post("/", ctrl.create);
router.patch("/:id", ctrl.patch);
router.delete("/:id", ctrl.remove);

module.exports = router;
```

### 16.6. Middlewares notFound + errorHandler

`src/middlewares/notFound.middleware.js`

```js
function notFound(req, res) {
  res.status(404).json({ error: "NotFound" });
}

module.exports = { notFound };
```

`src/middlewares/error.middleware.js`

```js
function errorHandler(err, req, res, next) {
  console.error(err);
  res.status(500).json({ error: "InternalServerError" });
}

module.exports = { errorHandler };
```

### 16.7. Brancher dans l’app

`src/app.js` (extrait) :

```js
const todosRouter = require("./routes/todos.routes");

app.use("/api/todos", todosRouter);
```

### 16.8. Vérifications avec curl

Créer :

```bash
curl -X POST http://localhost:3000/api/todos \
  -H 'content-type: application/json' \
  -d '{"title":"Apprendre Express"}'
```

Lister :

```bash
curl http://localhost:3000/api/todos
```

Patch :

```bash
curl -X PATCH http://localhost:3000/api/todos/<id> \
  -H 'content-type: application/json' \
  -d '{"done": true}'
```

Supprimer :

```bash
curl -X DELETE http://localhost:3000/api/todos/<id>
```

---

## 17. Annexes (checklists & snippets)

### 17.1. Checklist API Express

- [ ] `express.json()` activé
- [ ] routes via `Router()`
- [ ] validation (zod/joi)
- [ ] 404 handler après les routes
- [ ] error handler global en dernier
- [ ] logs HTTP (morgan)
- [ ] sécurité (helmet, rate limit)
- [ ] CORS configuré
- [ ] variables d’environnement (dotenv)
- [ ] tests (supertest)

### 17.2. Snippet : asyncHandler

```js
const asyncHandler = (fn) => (req, res, next) =>
  Promise.resolve(fn(req, res, next)).catch(next);
```

### 17.3. Pièges fréquents

- Oublier `return res.status(...).json(...)` et continuer le handler
- Mettre le middleware 404 **avant** les routes
- Ne pas appeler `next()` dans un middleware non terminal
- Ne pas gérer les erreurs async

---

## Conclusion

Express.js fournit une base simple mais puissante pour créer des serveurs HTTP Node : vous combinez **routes** et **middlewares** pour construire une API propre, testable et sécurisée. La qualité du projet repose surtout sur votre **structure**, vos **conventions REST**, la **validation**, et une **gestion d’erreurs** systématique.
