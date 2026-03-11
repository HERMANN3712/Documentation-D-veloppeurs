# 10 — Connexion aux bases de données (Node.js)

> **Public** : développeurs ayant des bases en JavaScript/Node.js
>
> **Objectif** : comprendre et mettre en œuvre des connexions robustes à plusieurs types de bases (MongoDB, MySQL, PostgreSQL) avec les bibliothèques usuelles (Mongoose, Sequelize), en adoptant les bonnes pratiques (sécurité, pooling, migrations, gestion d’erreurs, tests).

---

## Sommaire

1. [Objectifs pédagogiques](#1-objectifs-pédagogiques)
2. [Pré-requis et environnement](#2-pré-requis-et-environnement)
3. [Panorama des bases et stratégies d’accès](#3-panorama-des-bases-et-stratégies-daccès)
4. [Concepts communs (URI, credentials, pool, transactions)](#4-concepts-communs-uri-credentials-pool-transactions)
5. [MongoDB avec Mongoose](#5-mongodb-avec-mongoose)
6. [MySQL / PostgreSQL avec Sequelize](#6-mysql--postgresql-avec-sequelize)
7. [Gestion des migrations et seed](#7-gestion-des-migrations-et-seed)
8. [Sécurité et bonnes pratiques](#8-sécurité-et-bonnes-pratiques)
9. [Patterns d’architecture (Repository, Service, DI)](#9-patterns-darchitecture-repository-service-di)
10. [Tests & stratégie de validation](#10-tests--stratégie-de-validation)
11. [Exercices guidés](#11-exercices-guidés)
12. [Récapitulatif & check-list de prod](#12-récapitulatif--check-list-de-prod)

---

## 1. Objectifs pédagogiques

À la fin de ce module, vous saurez :

- Expliquer les différences entre **bases relationnelles** (MySQL, PostgreSQL) et **NoSQL** (MongoDB) et choisir un driver/ORM adapté.
- Configurer une application Node.js pour se connecter à une base via variables d’environnement.
- Mettre en place une connexion **robuste** (pool, retry, timeouts, logs, shutdown propre).
- Créer des modèles et exécuter des opérations CRUD avec **Mongoose** (MongoDB).
- Créer des modèles et exécuter des opérations CRUD avec **Sequelize** (MySQL/PostgreSQL).
- Gérer des **migrations** et **seed**.
- Appliquer les bonnes pratiques de **sécurité**, de performance et de qualité (tests).

---

## 2. Pré-requis et environnement

### Compétences attendues

- JavaScript moderne (ES Modules ou CommonJS)
- Node.js + npm
- Notions HTTP/REST (Express recommandé)

### Outils recommandés

- **Node.js** : 18+ (idéalement 20+)
- **Docker** et **Docker Compose** (pour lancer les DB facilement)
- Un client DB :
  - MongoDB Compass, TablePlus, DBeaver, pgAdmin, etc.

### Dépendances (selon stack)

- MongoDB : `mongoose`
- SQL : `sequelize` + `mysql2` ou `pg` + `pg-hstore`
- Variables d’environnement : `dotenv`
- Logs (optionnel) : `pino` ou `winston`

---

## 3. Panorama des bases et stratégies d’accès

### 3.1 Bases relationnelles (MySQL, PostgreSQL)

**Modèle** : tables, lignes, colonnes, contraintes, relations (FK), schéma strict.

**Points forts** :

- Intégrité référentielle (contraintes)
- Requêtes complexes (JOIN, agrégations)
- Transactions solides

**Accès depuis Node.js** :

- **Driver** (niveau bas) : `pg`, `mysql2`…
- **ORM** (niveau haut) : Sequelize, TypeORM, Prisma…

### 3.2 Bases NoSQL (MongoDB)

**Modèle** : documents JSON (BSON), collections, schéma flexible.

**Points forts** :

- Schéma flexible, itération rapide
- Stockage naturel de structures imbriquées
- Très bonne ergonomie pour certains cas (logs, profils, contenu)

**Accès depuis Node.js** :

- Driver officiel `mongodb`
- ODM : **Mongoose** (modèles, validation, middleware)

### 3.3 Choisir un outil

- MongoDB + besoin de validation & modèles : **Mongoose**
- SQL + modèle relationnel : **Sequelize** (ou autre ORM)
- Performance / requêtes SQL fines : parfois driver + SQL « à la main »

---

## 4. Concepts communs (URI, credentials, pool, transactions)

### 4.1 Configuration via variables d’environnement

Ne mettez jamais vos secrets en dur.

Exemple `.env` :

```env
NODE_ENV=development
PORT=3000

# Mongo
MONGO_URI=mongodb://localhost:27017/training_db

# SQL (exemple Postgres)
DB_DIALECT=postgres
DB_HOST=localhost
DB_PORT=5432
DB_NAME=training_db
DB_USER=training_user
DB_PASSWORD=training_password
```

Chargement côté Node :

```js
import 'dotenv/config';

const mongoUri = process.env.MONGO_URI;
```

### 4.2 Pool de connexions

- En SQL, le pool est essentiel : plutôt qu’ouvrir/fermer une connexion à chaque requête.
- En Mongo, le driver gère aussi un pool sous-jacent.

**À retenir** : une application Node gère peu de processus, donc un pool bien calibré est critique.

### 4.3 Timeouts & retries

- Timeout de connexion
- Timeout de requête
- Stratégies de retry (avec backoff) selon criticité

### 4.4 Transactions

- SQL : transactions très classiques (ACID)
- Mongo : transactions possibles (replica set), mais pas toujours nécessaires

---

## 5. MongoDB avec Mongoose

### 5.1 Installation

```bash
npm i mongoose dotenv
```

### 5.2 Connexion centralisée

Créez un fichier `src/db/mongo.js` :

```js
import mongoose from 'mongoose';

export async function connectMongo() {
  const uri = process.env.MONGO_URI;
  if (!uri) throw new Error('MONGO_URI manquant');

  // Bonnes pratiques de configuration
  mongoose.set('strictQuery', true);

  await mongoose.connect(uri, {
    // options modernes (selon version)
    serverSelectionTimeoutMS: 5000,
  });

  console.log('[mongo] connected');
}

export async function disconnectMongo() {
  await mongoose.disconnect();
  console.log('[mongo] disconnected');
}
```

Dans `src/index.js` :

```js
import 'dotenv/config';
import express from 'express';
import { connectMongo, disconnectMongo } from './db/mongo.js';

const app = express();
app.use(express.json());

app.get('/health', (req, res) => res.json({ ok: true }));

const port = process.env.PORT ?? 3000;

await connectMongo();

const server = app.listen(port, () => {
  console.log(`API listening on :${port}`);
});

// arrêt propre
process.on('SIGINT', shutdown);
process.on('SIGTERM', shutdown);

let shuttingDown = false;
async function shutdown() {
  if (shuttingDown) return;
  shuttingDown = true;

  console.log('Shutting down...');
  server.close(async () => {
    await disconnectMongo();
    process.exit(0);
  });
}
```

### 5.3 Définir un schéma & un modèle

Exemple `src/models/User.js` :

```js
import mongoose from 'mongoose';

const userSchema = new mongoose.Schema(
  {
    email: {
      type: String,
      required: true,
      unique: true,
      lowercase: true,
      trim: true,
    },
    name: {
      type: String,
      required: true,
      trim: true,
    },
    roles: {
      type: [String],
      default: ['user'],
    },
  },
  {
    timestamps: true,
  }
);

export const User = mongoose.model('User', userSchema);
```

### 5.4 CRUD (Create, Read, Update, Delete)

Exemples de fonctions repository `src/repositories/userRepository.js` :

```js
import { User } from '../models/User.js';

export function createUser(data) {
  return User.create(data);
}

export function findUserByEmail(email) {
  return User.findOne({ email }).exec();
}

export function listUsers({ limit = 20, offset = 0 } = {}) {
  return User.find()
    .sort({ createdAt: -1 })
    .skip(offset)
    .limit(limit)
    .exec();
}

export function updateUserName(userId, name) {
  return User.findByIdAndUpdate(userId, { name }, { new: true }).exec();
}

export function deleteUser(userId) {
  return User.findByIdAndDelete(userId).exec();
}
```

### 5.5 Validation & gestion d’erreurs

- Les erreurs Mongoose sont typées (ex: `ValidationError`).
- Les index uniques peuvent déclencher des erreurs de duplication.

Exemple d’handler Express simplifié :

```js
export function errorHandler(err, req, res, next) {
  if (err?.name === 'ValidationError') {
    return res.status(400).json({ error: 'ValidationError', details: err.errors });
  }

  // duplication (selon driver)
  if (err?.code === 11000) {
    return res.status(409).json({ error: 'DuplicateKey', details: err.keyValue });
  }

  console.error(err);
  res.status(500).json({ error: 'InternalError' });
}
```

### 5.6 Relations et référence (basique)

MongoDB n’impose pas de relations, mais vous pouvez référencer :

```js
const postSchema = new mongoose.Schema({
  title: String,
  authorId: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
});
```

Puis « peupler » :

```js
Post.find().populate('authorId').exec();
```

---

## 6. MySQL / PostgreSQL avec Sequelize

### 6.1 Installation

Pour PostgreSQL :

```bash
npm i sequelize pg pg-hstore dotenv
```

Pour MySQL :

```bash
npm i sequelize mysql2 dotenv
```

### 6.2 Initialiser Sequelize

Fichier `src/db/sql.js` :

```js
import { Sequelize } from 'sequelize';

export function createSequelize() {
  const dialect = process.env.DB_DIALECT;
  const host = process.env.DB_HOST;
  const port = Number(process.env.DB_PORT ?? 5432);
  const database = process.env.DB_NAME;
  const username = process.env.DB_USER;
  const password = process.env.DB_PASSWORD;

  if (!dialect) throw new Error('DB_DIALECT manquant');

  const sequelize = new Sequelize(database, username, password, {
    host,
    port,
    dialect,
    logging: false, // true pour debug (ou fonction)
    pool: {
      max: 10,
      min: 0,
      acquire: 10000,
      idle: 10000,
    },
    define: {
      underscored: true,
      timestamps: true,
    },
  });

  return sequelize;
}

export async function testSqlConnection(sequelize) {
  await sequelize.authenticate();
  console.log('[sql] connected');
}
```

### 6.3 Définir un modèle Sequelize

Exemple `src/modelsSql/User.js` :

```js
import { DataTypes } from 'sequelize';

export function defineUser(sequelize) {
  return sequelize.define(
    'User',
    {
      id: {
        type: DataTypes.UUID,
        defaultValue: DataTypes.UUIDV4,
        primaryKey: true,
      },
      email: {
        type: DataTypes.STRING(255),
        allowNull: false,
        unique: true,
        validate: { isEmail: true },
      },
      name: {
        type: DataTypes.STRING(120),
        allowNull: false,
      },
    },
    {
      tableName: 'users',
    }
  );
}
```

### 6.4 Charger les modèles et associer

Fichier `src/db/models.js` :

```js
import { defineUser } from '../modelsSql/User.js';

export function initModels(sequelize) {
  const User = defineUser(sequelize);

  // Associations (exemple)
  // User.hasMany(Post)

  return { User };
}
```

### 6.5 Synchronisation vs migrations

> `sequelize.sync()` est pratique en prototype, mais déconseillé en production.

Prototype :

```js
await sequelize.sync({ alter: true });
```

Production : utilisez des **migrations** (section 7).

### 6.6 CRUD avec Sequelize

Repository `src/repositoriesSql/userRepository.js` :

```js
export function createUser(UserModel, data) {
  return UserModel.create(data);
}

export function findUserByEmail(UserModel, email) {
  return UserModel.findOne({ where: { email } });
}

export function listUsers(UserModel, { limit = 20, offset = 0 } = {}) {
  return UserModel.findAll({
    order: [['createdAt', 'DESC']],
    limit,
    offset,
  });
}

export async function updateUserName(UserModel, id, name) {
  const user = await UserModel.findByPk(id);
  if (!user) return null;
  user.name = name;
  await user.save();
  return user;
}

export async function deleteUser(UserModel, id) {
  const count = await UserModel.destroy({ where: { id } });
  return count > 0;
}
```

### 6.7 Transactions

Exemple : deux opérations atomiques.

```js
await sequelize.transaction(async (t) => {
  const user = await User.create({ email, name }, { transaction: t });
  await AuditLog.create({ action: 'USER_CREATED', userId: user.id }, { transaction: t });
});
```

---

## 7. Gestion des migrations et seed

### 7.1 Pourquoi migrer ?

- Versionner le schéma
- Déployer de manière reproductible
- Faciliter le travail en équipe

### 7.2 Sequelize CLI (approche classique)

Installer :

```bash
npm i -D sequelize-cli
```

Initialiser :

```bash
npx sequelize-cli init
```

Créer une migration :

```bash
npx sequelize-cli migration:generate --name create-users
```

Exemple de migration (simplifiée) :

```js
'use strict';

module.exports = {
  async up(queryInterface, Sequelize) {
    await queryInterface.createTable('users', {
      id: {
        type: Sequelize.UUID,
        defaultValue: Sequelize.literal('gen_random_uuid()'),
        primaryKey: true,
        allowNull: false,
      },
      email: {
        type: Sequelize.STRING(255),
        allowNull: false,
        unique: true,
      },
      name: {
        type: Sequelize.STRING(120),
        allowNull: false,
      },
      created_at: { type: Sequelize.DATE, allowNull: false },
      updated_at: { type: Sequelize.DATE, allowNull: false },
    });
  },

  async down(queryInterface) {
    await queryInterface.dropTable('users');
  },
};
```

> Remarque : la génération d’UUID et les fonctions disponibles diffèrent entre PostgreSQL/MySQL. Adaptez selon votre dialect.

### 7.3 Seed (données d’exemple)

Générer :

```bash
npx sequelize-cli seed:generate --name seed-users
```

Puis insérer des données :

```js
'use strict';

module.exports = {
  async up(queryInterface) {
    await queryInterface.bulkInsert('users', [
      {
        id: '11111111-1111-1111-1111-111111111111',
        email: 'admin@example.com',
        name: 'Admin',
        created_at: new Date(),
        updated_at: new Date(),
      },
    ]);
  },

  async down(queryInterface) {
    await queryInterface.bulkDelete('users', null, {});
  },
};
```

---

## 8. Sécurité et bonnes pratiques

### 8.1 Secrets & configuration

- Utilisez `dotenv` en local uniquement
- En prod : variables d’environnement via l’infra (Kubernetes, systemd, CI/CD)
- Ne versionnez jamais `.env`

### 8.2 Principes de moindre privilège

- Créez un utilisateur DB dédié à l’application
- Donnez uniquement les droits nécessaires

### 8.3 Protection contre injection (SQL injection)

- Utilisez les requêtes paramétrées (ORM/driver)
- Évitez de concaténer du SQL avec de l’entrée utilisateur

### 8.4 Index & performance

- Ajoutez des index sur les champs filtrés fréquemment : `email`, `createdAt`, etc.
- Sur Mongo : index via schema (`unique`, `index: true`) ou migration/outils.
- Sur SQL : index B-tree classiques (selon requêtes).

### 8.5 Observabilité

- Loggez les erreurs DB (sans exposer de secrets)
- Ajoutez des métriques (latence des requêtes, nombre de connexions)

### 8.6 Arrêt propre (graceful shutdown)

- Fermer le serveur HTTP
- Fermer la connexion DB / pool
- Donner un délai maximum (timeout) avant `process.exit(1)`

---

## 9. Patterns d’architecture (Repository, Service, DI)

### 9.1 Séparer les responsabilités

- **Controller** : HTTP, validation d’entrée, codes HTTP
- **Service** : logique métier
- **Repository/DAO** : accès DB

Cela facilite les tests, les refactors et le changement d’ORM.

### 9.2 Exemple (structure projet)

```
src/
  db/
    mongo.js
    sql.js
    models.js
  models/
    User.js
  repositories/
    userRepository.js
  services/
    userService.js
  routes/
    users.js
  index.js
```

### 9.3 Exemple de service

```js
import * as userRepo from '../repositories/userRepository.js';

export async function registerUser(payload) {
  const existing = await userRepo.findUserByEmail(payload.email);
  if (existing) {
    const err = new Error('Email déjà utilisé');
    err.statusCode = 409;
    throw err;
  }

  return userRepo.createUser(payload);
}
```

---

## 10. Tests & stratégie de validation

### 10.1 Niveaux de test

- **Unitaires** : services avec repositories mockés
- **Intégration** : API + vraie DB (idéalement via Docker)
- **E2E** : parcours complets

### 10.2 Exemple d’approche intégration (concept)

- Lancer une DB via Docker compose
- Exécuter migrations
- Lancer tests Jest/Vitest
- Nettoyer (truncate/rollback)

Conseils :

- Utilisez une base de test dédiée
- Isolez les tests (transaction + rollback quand possible)

---

## 11. Exercices guidés

### Exercice 1 — Connexion MongoDB, modèle User

**Objectif** : créer une API Express montrant un CRUD utilisateur avec Mongoose.

1. Créer `connectMongo()`
2. Créer un modèle `User` (email unique + validation)
3. Ajouter routes :
   - `POST /users`
   - `GET /users`
   - `GET /users/:id`
   - `PATCH /users/:id`
   - `DELETE /users/:id`

**Bonus** : pagination `limit/offset` et tri.

### Exercice 2 — Connexion PostgreSQL, modèle User

**Objectif** : reproduire le CRUD avec Sequelize.

1. Configurer Sequelize + `authenticate()`
2. Définir `User`
3. Implémenter CRUD
4. Ajouter une transaction lors d’une création + audit log

### Exercice 3 — Migrations

**Objectif** : versionner votre schéma.

1. Créer migration `create-users`
2. Exécuter `db:migrate`
3. Ajouter une migration `add-index-on-email`

---

## 12. Récapitulatif & check-list de prod

### Check-list

- [ ] Variables d’environnement chargées et validées au démarrage
- [ ] Connexion DB testée (`authenticate` / `connect`)
- [ ] Pool configuré (SQL)
- [ ] Index clés ajoutés (`email`, etc.)
- [ ] Migrations en place (SQL)
- [ ] Gestion d’erreurs centralisée
- [ ] Logs et métriques minimales
- [ ] Arrêt propre (SIGTERM/SIGINT)
- [ ] Secrets gérés par l’infra (pas de `.env` en prod)

### Résumé

- **MongoDB** : Mongoose simplifie la modélisation (schemas, validation, middleware).
- **MySQL/PostgreSQL** : Sequelize fournit un ORM, du pooling et des transactions.
- Quel que soit le SGBD : la robustesse vient du **design de connexion**, de la **sécurité**, d’une **gestion d’erreurs** claire et des **migrations/tests**.
