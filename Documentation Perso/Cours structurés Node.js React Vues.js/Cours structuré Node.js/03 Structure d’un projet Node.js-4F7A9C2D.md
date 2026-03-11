# 03 — Structure d’un projet Node.js

## Objectifs pédagogiques
À la fin de ce cours, vous saurez :

- Identifier les éléments essentiels d’un projet Node.js (fichiers, dossiers, conventions).
- Comprendre le rôle du `package.json` (dépendances, scripts, métadonnées).
- Expliquer à quoi sert le dossier `node_modules` et pourquoi il n’est pas versionné.
- Définir et organiser un point d’entrée (ex. `index.js`, `app.js`, `server.js`).
- Mettre en place une structure de base propre et maintenable.

---

## Pré-requis
- Connaissances de base en JavaScript.
- Node.js et npm installés (ou yarn/pnpm selon votre environnement).
- Un éditeur (VS Code conseillé).

---

## 1) Anatomie générale d’un projet Node.js

Un projet Node.js « minimal » contient généralement :

- **`package.json`** : la carte d’identité du projet (dépendances, scripts, version, etc.).
- **`node_modules/`** : les dépendances installées localement.
- **Un point d’entrée** : le fichier exécuté au démarrage (`index.js`, `app.js`, etc.).

Exemple d’arborescence très simple :

```txt
mon-projet/
├─ package.json
├─ package-lock.json
├─ node_modules/
└─ index.js
```

### Variantes courantes
Selon les besoins, on ajoute souvent :

- `README.md` : documentation minimale.
- `.gitignore` : exclusions Git (notamment `node_modules`).
- `src/` : code source.
- `test/` ou `__tests__/` : tests.
- `.env` : variables d’environnement (souvent non versionnées).

---

## 2) Le fichier `package.json`

### 2.1 Rôle et importance
Le fichier **`package.json`** est central :

- Il **décrit le projet** (nom, version, description, auteur, licence, etc.).
- Il déclare les **dépendances** nécessaires à l’exécution et au développement.
- Il définit des **scripts** (commandes) pour standardiser les actions : démarrage, test, build…

Il permet à n’importe quel développeur (ou serveur CI) de reconstruire l’environnement avec :

```bash
npm install
```

### 2.2 Structure typique
Exemple de `package.json` :

```json
{
  "name": "mon-projet",
  "version": "1.0.0",
  "description": "Exemple de projet Node.js",
  "main": "index.js",
  "type": "commonjs",
  "scripts": {
    "start": "node index.js",
    "dev": "node index.js",
    "test": "echo \"No tests\" && exit 0"
  },
  "dependencies": {
    "express": "^4.19.2"
  },
  "devDependencies": {
    "nodemon": "^3.1.0"
  }
}
```

#### Champs à connaître
- **`name`** : nom du package/projet.
- **`version`** : version du projet (souvent en SemVer).
- **`main`** : point d’entrée principal pour un package publié (optionnel pour une app).
- **`type`** : module system (`"commonjs"` ou `"module"` pour ESM).
- **`scripts`** : raccourcis de commandes.
- **`dependencies`** : dépendances nécessaires en production.
- **`devDependencies`** : dépendances nécessaires uniquement au développement/test.

### 2.3 Scripts : conventions utiles
Les scripts standard usuels :

- `npm run start` (ou `npm start`) : lancer l’app.
- `npm run dev` : mode développement (reload, logs, etc.).
- `npm test` : exécuter les tests.

Exemple avec `nodemon` en dev :

```json
{
  "scripts": {
    "start": "node index.js",
    "dev": "nodemon index.js"
  }
}
```

Lancement :

```bash
npm run dev
```

---

## 3) Le dossier `node_modules/`

### 3.1 À quoi sert-il ?
Quand vous installez des dépendances :

```bash
npm install
```

npm télécharge les paquets déclarés dans `package.json` et les place dans **`node_modules/`**.

### 3.2 Pourquoi on ne versionne pas `node_modules/`
Le dossier `node_modules/` :

- peut être très volumineux,
- dépend de l’OS et parfois de l’architecture,
- est reproductible via `npm install`.

On l’ignore généralement dans Git via un `.gitignore` :

```gitignore
node_modules
.env
```

### 3.3 Les fichiers de lock : reproductibilité
- **`package-lock.json`** (npm)
- **`yarn.lock`** (yarn)
- **`pnpm-lock.yaml`** (pnpm)

Ces fichiers figent les versions exactes installées pour garantir des installations cohérentes sur toutes les machines.

---

## 4) Le point d’entrée (`index.js`, `app.js`, etc.)

### 4.1 Définition
Le **point d’entrée** est le fichier exécuté lorsque vous lancez :

```bash
node index.js
```

ou via un script npm :

```bash
npm start
```

Le nom varie selon les conventions :

- `index.js` : très courant pour un projet minimal.
- `app.js` : souvent pour une application web.
- `server.js` : courant pour un serveur HTTP/API.

### 4.2 Exemple minimal
`index.js` :

```js
console.log("Hello Node.js");
```

Script :

```json
{
  "scripts": {
    "start": "node index.js"
  }
}
```

---

## 5) Proposition de structure recommandée (pour une app maintenable)

Même si un projet peut tenir en 2–3 fichiers, une structure claire aide énormément dès que le projet grandit.

### 5.1 Structure type

```txt
mon-projet/
├─ package.json
├─ package-lock.json
├─ .gitignore
├─ README.md
├─ src/
│  ├─ index.js
│  ├─ app.js
│  ├─ routes/
│  ├─ controllers/
│  ├─ services/
│  └─ config/
└─ test/
```

### 5.2 Rôles des dossiers
- `src/` : code source.
- `routes/` : définition des routes (HTTP).
- `controllers/` : couche de contrôle (req/res).
- `services/` : logique métier.
- `config/` : configuration (connexion DB, variables, etc.).
- `test/` : tests unitaires / intégration.

---

## 6) Pas à pas : créer un projet Node.js minimal

### 6.1 Initialisation
1) Créer le dossier :

```bash
mkdir mon-projet
cd mon-projet
```

2) Initialiser `package.json` :

```bash
npm init -y
```

### 6.2 Créer un point d’entrée
Créer `index.js` :

```js
console.log("Projet initialisé !");
```

### 6.3 Ajouter un script de démarrage
Dans `package.json` :

```json
{
  "scripts": {
    "start": "node index.js"
  }
}
```

Lancer :

```bash
npm start
```

### 6.4 Installer une dépendance (exemple)

```bash
npm install express
```

Cela ajoute `express` dans `dependencies` et télécharge le paquet dans `node_modules/`.

---

## 7) Bonnes pratiques (essentielles)

- **Ne pas committer `node_modules/`** : utilisez `.gitignore`.
- **Toujours committer le lockfile** (`package-lock.json`, etc.) pour la reproductibilité.
- **Utiliser `scripts`** pour documenter les tâches : start/dev/test/lint.
- **Séparer le code dans `src/`** si le projet dépasse quelques fichiers.
- **Documenter** : un `README.md` simple suffit (installation, lancement, variables d’environnement).

---

## 8) Exercices

### Exercice 1 — Projet minimal
1. Créez un dossier projet.
2. Générez `package.json`.
3. Ajoutez `index.js` qui affiche un message.
4. Ajoutez le script `start`.

Résultat attendu :

```bash
npm start
```

affiche votre message dans le terminal.

### Exercice 2 — Dépendance et script dev
1. Installez `nodemon` en dépendance de dev :

```bash
npm install -D nodemon
```

2. Ajoutez un script `dev` qui lance `index.js` via nodemon.
3. Modifiez le message dans `index.js` et observez le redémarrage automatique.

---

## Résumé
- Un projet Node.js repose généralement sur **`package.json`**, **`node_modules/`**, et un **point d’entrée** (`index.js` / `app.js`).
- **`package.json`** décrit les dépendances et scripts du projet.
- **`node_modules/`** contient les packages installés et ne doit pas être versionné.
- Un **point d’entrée** propre et des **scripts npm** rendent le projet simple à lancer et à maintenir.
