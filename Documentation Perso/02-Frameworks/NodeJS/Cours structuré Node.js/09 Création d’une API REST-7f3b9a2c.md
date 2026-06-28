# Formation — Création d’une API REST (Node.js)

> **Objectif** : concevoir, développer, sécuriser et documenter une API REST exposant des endpoints CRUD (**Create, Read, Update, Delete**) via les méthodes HTTP **GET, POST, PUT, DELETE**.

---

## Sommaire

1. [Public visé & prérequis](#1-public-visé--prérequis)
2. [Objectifs pédagogiques](#2-objectifs-pédagogiques)
3. [Architecture REST & principes](#3-architecture-rest--principes)
4. [Setup du projet Node.js](#4-setup-du-projet-nodejs)
5. [Concevoir les endpoints CRUD](#5-concevoir-les-endpoints-crud)
6. [Implémentation avec Express](#6-implémentation-avec-express)
7. [Validation des données](#7-validation-des-données)
8. [Gestion d’erreurs & statuts HTTP](#8-gestion-derreurs--statuts-http)
9. [Persistance (exemple : mémoire puis base de données)](#9-persistance-exemple--mémoire-puis-base-de-données)
10. [Pagination, tri, filtres](#10-pagination-tri-filtres)
11. [Sécurité de base (CORS, headers, rate limiting)](#11-sécurité-de-base-cors-headers-rate-limiting)
12. [Documentation (OpenAPI/Swagger)](#12-documentation-openapiswagger)
13. [Tests d’API (Postman & tests automatisés)](#13-tests-dapi-postman--tests-automatisés)
14. [Bonne pratiques & check-list de mise en prod](#14-bonne-pratiques--check-list-de-mise-en-prod)
15. [Exercices & corrections guidées](#15-exercices--corrections-guidées)

---

## 1) Public visé & prérequis

### Public
- Développeurs Node.js débutants/intermédiaires
- Développeurs front souhaitant construire une API

### Prérequis
- JavaScript (ES6+), notions de promesses/async-await
- Node.js installé (>= 18 recommandé)
- Connaissances de base HTTP (headers, body, codes)

---

## 2) Objectifs pédagogiques

À la fin de la formation, vous saurez :
- Expliquer ce qu’est une API REST et ses conventions
- Définir des endpoints CRUD cohérents
- Implémenter une API avec **Express**
- Gérer validation, erreurs, statuts HTTP
- Préparer l’API pour une utilisation réelle (pagination, sécurité, doc, tests)

---

## 3) Architecture REST & principes

### 3.1 Définition
Une **API REST** (Representational State Transfer) expose des **ressources** via des **URLs** et utilise les **méthodes HTTP** pour exprimer l’action.

- **Ressource** : un élément métier (ex. `users`, `products`, `posts`)
- **Représentation** : JSON le plus souvent

### 3.2 CRUD et méthodes HTTP

| Opération | HTTP | Exemple endpoint | Description |
|---|---|---|---|
| Create | POST | `POST /tasks` | Crée une ressource |
| Read (liste) | GET | `GET /tasks` | Liste les ressources |
| Read (détail) | GET | `GET /tasks/:id` | Récupère une ressource |
| Update (remplacement) | PUT | `PUT /tasks/:id` | Remplace une ressource |
| Delete | DELETE | `DELETE /tasks/:id` | Supprime une ressource |

> Remarque : on voit souvent `PATCH` pour mise à jour partielle, mais ici on se concentre sur **GET/POST/PUT/DELETE** comme demandé.

### 3.3 Conventions REST essentielles
- Utiliser des noms de ressources **au pluriel** : `/tasks`
- Les identifiants en path : `/tasks/123`
- JSON en entrée/sortie
- Codes HTTP corrects (200, 201, 400, 404, 500…)
- API **stateless** : chaque requête contient l’info nécessaire (pas de session serveur implicite)

---

## 4) Setup du projet Node.js

### 4.1 Initialisation
```bash
mkdir api-rest-tasks
cd api-rest-tasks
npm init -y
```

### 4.2 Dépendances
```bash
npm i express cors helmet morgan
npm i -D nodemon
```

- **express** : framework HTTP
- **cors** : gestion CORS
- **helmet** : headers de sécurité
- **morgan** : logs HTTP
- **nodemon** : rechargement auto en dev

### 4.3 Scripts
Dans `package.json` :
```json
{
  "scripts": {
    "dev": "nodemon src/server.js",
    "start": "node src/server.js"
  }
}
```

### 4.4 Arborescence conseillée
```
api-rest-tasks/
  src/
    server.js
    app.js
    routes/
      tasks.routes.js
    controllers/
      tasks.controller.js
    services/
      tasks.service.js
    middlewares/
      error.middleware.js
      notFound.middleware.js
    validators/
      tasks.validator.js
```

---

## 5) Concevoir les endpoints CRUD

### Ressource de démonstration : `Task`
Nous allons construire une API de gestion de tâches.

#### Modèle simple
- `id` : string (UUID ou identifiant auto)
- `title` : string (obligatoire)
- `completed` : boolean (défaut `false`)
- `createdAt` : date ISO

### Endpoints
- `GET /tasks` : liste (avec pagination/filtres ensuite)
- `GET /tasks/:id` : détail
- `POST /tasks` : création
- `PUT /tasks/:id` : mise à jour (remplacement)
- `DELETE /tasks/:id` : suppression

---

## 6) Implémentation avec Express

### 6.1 Créer l’application
**`src/app.js`**
```js
const express = require("express");
const cors = require("cors");
const helmet = require("helmet");
const morgan = require("morgan");

const tasksRoutes = require("./routes/tasks.routes");
const notFound = require("./middlewares/notFound.middleware");
const errorHandler = require("./middlewares/error.middleware");

function createApp() {
  const app = express();

  // Middlewares globaux
  app.use(helmet());
  app.use(cors());
  app.use(morgan("dev"));
  app.use(express.json());

  // Routes
  app.get("/health", (req, res) => res.json({ status: "ok" }));
  app.use("/tasks", tasksRoutes);

  // 404 + gestion d'erreurs
  app.use(notFound);
  app.use(errorHandler);

  return app;
}

module.exports = createApp;
```

### 6.2 Démarrer le serveur
**`src/server.js`**
```js
const createApp = require("./app");

const app = createApp();
const PORT = process.env.PORT || 3000;

app.listen(PORT, () => {
  console.log(`API listening on http://localhost:${PORT}`);
});
```

### 6.3 Définir les routes
**`src/routes/tasks.routes.js`**
```js
const express = require("express");
const ctrl = require("../controllers/tasks.controller");

const router = express.Router();

router.get("/", ctrl.list);
router.get("/:id", ctrl.getById);
router.post("/", ctrl.create);
router.put("/:id", ctrl.update);
router.delete("/:id", ctrl.remove);

module.exports = router;
```

### 6.4 Contrôleurs
Les contrôleurs gèrent l’interface HTTP :
- lire paramètres/headers/body
- appeler la couche service
- répondre avec statut + payload

**`src/controllers/tasks.controller.js`**
```js
const service = require("../services/tasks.service");
const { validateCreateTask, validateUpdateTask } = require("../validators/tasks.validator");

exports.list = (req, res, next) => {
  try {
    const result = service.list(req.query);
    res.json(result);
  } catch (err) {
    next(err);
  }
};

exports.getById = (req, res, next) => {
  try {
    const task = service.getById(req.params.id);
    res.json(task);
  } catch (err) {
    next(err);
  }
};

exports.create = (req, res, next) => {
  try {
    const input = validateCreateTask(req.body);
    const created = service.create(input);
    res.status(201).json(created);
  } catch (err) {
    next(err);
  }
};

exports.update = (req, res, next) => {
  try {
    const input = validateUpdateTask(req.body);
    const updated = service.update(req.params.id, input);
    res.json(updated);
  } catch (err) {
    next(err);
  }
};

exports.remove = (req, res, next) => {
  try {
    service.remove(req.params.id);
    res.status(204).send();
  } catch (err) {
    next(err);
  }
};
```

---

## 7) Validation des données

Sans validation :
- risque de données incohérentes
- erreurs difficiles à diagnostiquer

Ici une validation simple « maison » (vous pourrez remplacer par Joi/Zod ensuite).

**`src/validators/tasks.validator.js`**
```js
function badRequest(message, details) {
  const err = new Error(message);
  err.statusCode = 400;
  if (details) err.details = details;
  return err;
}

exports.validateCreateTask = (body) => {
  if (!body || typeof body !== "object") {
    throw badRequest("Body JSON invalide");
  }
  const { title, completed } = body;

  if (typeof title !== "string" || title.trim().length === 0) {
    throw badRequest("Le champ 'title' est obligatoire et doit être une chaîne non vide");
  }

  if (completed !== undefined && typeof completed !== "boolean") {
    throw badRequest("Le champ 'completed' doit être un booléen");
  }

  return {
    title: title.trim(),
    completed: completed ?? false
  };
};

exports.validateUpdateTask = (body) => {
  if (!body || typeof body !== "object") {
    throw badRequest("Body JSON invalide");
  }

  const { title, completed } = body;

  // PUT = remplacement : on exige tous les champs principaux
  if (typeof title !== "string" || title.trim().length === 0) {
    throw badRequest("Pour PUT, le champ 'title' est obligatoire et doit être une chaîne non vide");
  }
  if (typeof completed !== "boolean") {
    throw badRequest("Pour PUT, le champ 'completed' est obligatoire et doit être un booléen");
  }

  return {
    title: title.trim(),
    completed
  };
};
```

---

## 8) Gestion d’erreurs & statuts HTTP

### 8.1 Middleware 404
**`src/middlewares/notFound.middleware.js`**
```js
module.exports = (req, res) => {
  res.status(404).json({
    error: "Not Found",
    path: req.originalUrl
  });
};
```

### 8.2 Middleware d’erreurs
**`src/middlewares/error.middleware.js`**
```js
module.exports = (err, req, res, next) => {
  const statusCode = err.statusCode || 500;

  const payload = {
    error: statusCode === 500 ? "Internal Server Error" : err.message
  };

  if (err.details) payload.details = err.details;

  // En dev, on peut exposer plus d’info (à éviter en prod)
  if (process.env.NODE_ENV !== "production" && statusCode === 500) {
    payload.stack = err.stack;
  }

  res.status(statusCode).json(payload);
};
```

### 8.3 Codes HTTP recommandés
- **200 OK** : lecture / update réussis
- **201 Created** : création réussie
- **204 No Content** : suppression réussie (pas de body)
- **400 Bad Request** : validation échouée
- **404 Not Found** : ressource inexistante
- **500 Internal Server Error** : bug serveur

---

## 9) Persistance (exemple : mémoire puis base de données)

On commence en mémoire (pédagogique), puis on explique la migration vers une DB.

### 9.1 Service en mémoire
**`src/services/tasks.service.js`**
```js
const { randomUUID } = require("crypto");

function notFound(message) {
  const err = new Error(message);
  err.statusCode = 404;
  return err;
}

// "Base" en mémoire
const tasks = new Map();

exports.list = ({ page = "1", limit = "10", completed } = {}) => {
  const pageNum = Math.max(parseInt(page, 10) || 1, 1);
  const limitNum = Math.min(Math.max(parseInt(limit, 10) || 10, 1), 100);

  let items = Array.from(tasks.values());

  if (completed !== undefined) {
    const bool = completed === "true" ? true : completed === "false" ? false : null;
    if (bool !== null) items = items.filter(t => t.completed === bool);
  }

  const total = items.length;
  const start = (pageNum - 1) * limitNum;
  const paginated = items.slice(start, start + limitNum);

  return {
    page: pageNum,
    limit: limitNum,
    total,
    items: paginated
  };
};

exports.getById = (id) => {
  const task = tasks.get(id);
  if (!task) throw notFound("Task introuvable");
  return task;
};

exports.create = ({ title, completed }) => {
  const now = new Date().toISOString();
  const task = {
    id: randomUUID(),
    title,
    completed: !!completed,
    createdAt: now
  };
  tasks.set(task.id, task);
  return task;
};

exports.update = (id, { title, completed }) => {
  const existing = tasks.get(id);
  if (!existing) throw notFound("Task introuvable");

  const updated = {
    ...existing,
    title,
    completed
  };

  tasks.set(id, updated);
  return updated;
};

exports.remove = (id) => {
  const existed = tasks.delete(id);
  if (!existed) throw notFound("Task introuvable");
};
```

### 9.2 Vers une base de données (concept)
Quand passer à une DB :
- besoin de persistance
- concurrence (multi-instances)
- recherche/tri avancés

Approches :
- SQL (PostgreSQL) + ORM (Prisma/Sequelize)
- NoSQL (MongoDB) + ODM (Mongoose)

> Le mapping reste identique : contrôleur → service/repository → DB.

---

## 10) Pagination, tri, filtres

### 10.1 Pourquoi
- éviter de renvoyer 50 000 lignes
- performance & UX

### 10.2 Pagination par query params
Ex :
- `GET /tasks?page=2&limit=20`

Réponse typique :
```json
{
  "page": 2,
  "limit": 20,
  "total": 57,
  "items": []
}
```

### 10.3 Filtres
Ex :
- `GET /tasks?completed=true`

> Bonnes pratiques :
> - documenter les filtres
> - typer correctement (string → boolean/number)

---

## 11) Sécurité de base (CORS, headers, rate limiting)

### 11.1 CORS
- côté navigateur, protège contre appels cross-domain non autorisés

Dans `app.js`, `cors()` est permissif.
Pour restreindre :
```js
app.use(cors({
  origin: ["https://mon-front.com"],
  methods: ["GET", "POST", "PUT", "DELETE"],
}));
```

### 11.2 Headers de sécurité
`helmet()` ajoute des headers utiles.

### 11.3 Rate limiting
Pour protéger des abus (bruteforce, flood) :
```bash
npm i express-rate-limit
```
Puis :
```js
const rateLimit = require("express-rate-limit");

app.use(rateLimit({
  windowMs: 60_000,
  limit: 100
}));
```

---

## 12) Documentation (OpenAPI/Swagger)

### 12.1 Pourquoi
- faciliter consommation (front, mobile)
- partager un contrat
- tester via Swagger UI

### 12.2 Mise en place rapide
```bash
npm i swagger-ui-express yamljs
```

Créer `openapi.yaml` (à la racine par ex.). Exemple minimal :
```yaml
openapi: 3.0.3
info:
  title: Tasks API
  version: 1.0.0
servers:
  - url: http://localhost:3000
paths:
  /tasks:
    get:
      summary: Liste des tâches
      parameters:
        - in: query
          name: page
          schema: { type: integer, default: 1 }
        - in: query
          name: limit
          schema: { type: integer, default: 10 }
        - in: query
          name: completed
          schema: { type: boolean }
      responses:
        '200':
          description: OK
    post:
      summary: Créer une tâche
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [title]
              properties:
                title: { type: string }
                completed: { type: boolean }
      responses:
        '201':
          description: Created
  /tasks/{id}:
    get:
      summary: Obtenir une tâche
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      responses:
        '200': { description: OK }
        '404': { description: Not Found }
    put:
      summary: Remplacer une tâche
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [title, completed]
              properties:
                title: { type: string }
                completed: { type: boolean }
      responses:
        '200': { description: OK }
        '404': { description: Not Found }
    delete:
      summary: Supprimer une tâche
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string }
      responses:
        '204': { description: No Content }
        '404': { description: Not Found }
```

Dans `app.js` :
```js
const swaggerUi = require("swagger-ui-express");
const YAML = require("yamljs");

const swaggerDocument = YAML.load("openapi.yaml");
app.use("/docs", swaggerUi.serve, swaggerUi.setup(swaggerDocument));
```

Accès : `http://localhost:3000/docs`

---

## 13) Tests d’API (Postman & tests automatisés)

### 13.1 Scénarios Postman
À tester :
1. `POST /tasks` avec body valide → **201**, retourne `id`
2. `GET /tasks` → **200**, contient l’item
3. `GET /tasks/:id` → **200**
4. `PUT /tasks/:id` → **200**, modifie `completed`
5. `DELETE /tasks/:id` → **204**
6. `GET /tasks/:id` après delete → **404**

### 13.2 Tests automatisés (option)
Librairies courantes :
- **jest** + **supertest**

Installation :
```bash
npm i -D jest supertest
```
Principe : lancer l’app en mémoire et requêter.

---

## 14) Bonne pratiques & check-list de mise en prod

### 14.1 Bonnes pratiques
- Structure par couches : routes → controllers → services
- Validation systématique des entrées
- Réponses cohérentes (format d’erreur stable)
- Logs utiles (mais sans données sensibles)
- Variables d’environnement (`PORT`, `NODE_ENV`)

### 14.2 Check-list
- [ ] CORS restreint (prod)
- [ ] Helmet activé
- [ ] Rate limiting
- [ ] Documentation OpenAPI
- [ ] Tests (au moins de non-régression)
- [ ] Observabilité (logs, métriques)

---

## 15) Exercices & corrections guidées

### Exercice 1 — Créer une ressource `users`
**But** : reproduire la structure `tasks` pour `users`.

- Champs : `id`, `email`, `name`, `createdAt`
- Endpoints CRUD avec GET/POST/PUT/DELETE

**Critères de réussite** :
- validation `email` non vide
- codes HTTP corrects
- erreurs 404 si user absent

### Exercice 2 — Ajouter un filtre
Ajouter `GET /tasks?title=foo` qui renvoie les tâches dont le titre contient `foo` (insensible à la casse).

### Exercice 3 — Normaliser les erreurs
Imposer un format :
```json
{ "error": "...", "code": "...", "details": [] }
```

---

## Annexes — Exemples d’appels HTTP

### Créer une tâche
```bash
curl -X POST http://localhost:3000/tasks \
  -H "Content-Type: application/json" \
  -d '{"title":"Apprendre REST","completed":false}'
```

### Lister les tâches
```bash
curl "http://localhost:3000/tasks?page=1&limit=10"
```

### Détail
```bash
curl http://localhost:3000/tasks/<id>
```

### Remplacer (PUT)
```bash
curl -X PUT http://localhost:3000/tasks/<id> \
  -H "Content-Type: application/json" \
  -d '{"title":"Apprendre REST (update)","completed":true}'
```

### Supprimer
```bash
curl -X DELETE http://localhost:3000/tasks/<id>
```
