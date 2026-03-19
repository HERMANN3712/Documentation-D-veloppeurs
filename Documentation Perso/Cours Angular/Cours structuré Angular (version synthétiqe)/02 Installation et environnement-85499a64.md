# Formation Angular — 02. Installation et environnement

> **Objectif** : savoir installer Angular proprement, comprendre les prérequis et configurer un environnement de développement fiable (Node.js, npm, Angular CLI), puis créer et exécuter un premier projet.

---

## Plan de la formation

1. **Vue d’ensemble**
   - Qu’est-ce qu’Angular ?
   - Le rôle d’Angular CLI
2. **Prérequis techniques**
   - Node.js et npm
   - Éditeur/IDE (VS Code recommandé)
   - Navigateur et outils de debug
3. **Installation d’Angular CLI**
   - Installation globale via npm
   - Vérifications et commandes utiles
   - Mise à jour / désinstallation
4. **Création d’un projet Angular**
   - `ng new` : options principales
   - Structure générée
5. **Exécution et développement**
   - Serveur de dev (`ng serve`)
   - Hot reload et workflow
6. **Build et déploiement (vue pratique)**
   - `ng build` (dev vs prod)
   - Dossier de sortie
7. **Bonnes pratiques & résolution de problèmes**
   - Versions Node/Angular
   - Proxies, permissions, cache npm
   - Diagnostic rapide
8. **Atelier guidé (pas à pas)**
   - Installation → création → lancement → build

---

## 1) Vue d’ensemble

### 1.1 Angular, en bref
Angular est un framework TypeScript orienté **applications web**. Il propose une architecture standardisée (modules, composants, services) et un outillage solide.

### 1.2 Pourquoi Angular CLI ?
Angular s’installe et se pilote principalement via **Angular CLI**.

Angular CLI :
- **crée** des projets Angular (génération de squelette),
- **génère** du code (composants, services, modules…),
- **lance** un serveur de développement,
- **build** l’application pour la production,
- **facilite** le déploiement en produisant des fichiers statiques optimisés.

---

## 2) Prérequis techniques

### 2.1 Node.js et npm
Angular CLI s’installe avec `npm` (Node Package Manager), donc vous avez besoin de **Node.js**.

#### Vérifier l’installation
Ouvrez un terminal (PowerShell, CMD, Terminal macOS/Linux) et exécutez :

```bash
node -v
npm -v
```

Vous devez obtenir des numéros de versions (par exemple `v20.x.x`).

#### Recommandation
- Utilisez une version **LTS** de Node.js (plus stable pour l’écosystème).
- En environnement d’équipe, standardisez la version de Node (ex. via `nvm`, `fnm` ou `.nvmrc`).

### 2.2 Éditeur/IDE
Recommandation : **Visual Studio Code**.

Extensions utiles :
- Angular Language Service
- ESLint
- Prettier

### 2.3 Navigateur et debug
- Chrome / Edge : DevTools + inspection réseau / performance.
- Pour une expérience de debug TypeScript fluide, gardez les sourcemaps activées (par défaut en dev).

---

## 3) Installation d’Angular CLI

### 3.1 Installation globale
La commande principale (au cœur de cette formation) :

```bash
npm install -g @angular/cli
```

- `npm install` installe un paquet.
- `-g` signifie **global** (CLI disponible partout).
- `@angular/cli` est le paquet officiel.

### 3.2 Vérification
Une fois installé :

```bash
ng version
```

Vous verrez les versions de :
- Angular CLI
- Node
- npm
- packages Angular liés

### 3.3 Mise à jour / réinstallation
Mettre à jour Angular CLI globalement :

```bash
npm install -g @angular/cli@latest
```

Désinstaller :

```bash
npm uninstall -g @angular/cli
```

> **Note** : en pratique, on utilise souvent une CLI **locale au projet** (installée dans `node_modules`) et on exécute la bonne version via `npx` ou via les scripts npm. Mais pour démarrer et apprendre, l’installation globale reste la plus simple.

---

## 4) Création d’un projet Angular

### 4.1 Créer un nouveau projet
Commande fondamentale :

```bash
ng new mon-projet
```

Cette commande :
- crée un dossier `mon-projet/`,
- installe les dépendances (`node_modules`),
- initialise une configuration standard,
- génère une application fonctionnelle.

### 4.2 Options fréquentes (selon version CLI)
Lors de `ng new`, la CLI peut vous poser des questions :
- Type de styles (CSS, SCSS…)
- Routage (Angular Router)

Vous pouvez aussi passer des options en ligne de commande (exemples indicatifs) :

```bash
ng new mon-projet --routing
ng new mon-projet --style=scss
```

### 4.3 Structure typique du projet
Après création, vous trouverez généralement :

- `src/` : code de l’application
  - `main.ts` : point d’entrée
  - `index.html` : page hôte
  - `styles.*` : styles globaux
  - `app/` : composants/services principaux
- `angular.json` : configuration CLI (build, assets, environnements)
- `package.json` : dépendances et scripts
- `tsconfig.json` : configuration TypeScript

---

## 5) Exécution et développement

### 5.1 Lancer le serveur de développement
Dans le dossier du projet :

```bash
cd mon-projet
ng serve
```

Par défaut :
- un serveur local est lancé,
- l’application est accessible via une URL du type :
  - `http://localhost:4200/`

### 5.2 Rechargement automatique (workflow)
- Modifiez un fichier (ex. un composant).
- Le navigateur se met à jour automatiquement.

Ce cycle rapide est la base du développement Angular.

---

## 6) Build et déploiement (vue pratique)

Angular CLI facilite le **build** et prépare l’application pour la mise en production.

### 6.1 Build
Dans le projet :

```bash
ng build
```

Cela génère des fichiers optimisés pour être servis par un serveur web.

> Selon la configuration du projet, vous pouvez cibler le mode production via la configuration. Avec les versions récentes, la CLI applique souvent des optimisations par défaut selon le profil de build.

### 6.2 Dossier de sortie
Le build produit des fichiers dans un dossier du type :
- `dist/mon-projet/`

C’est ce dossier qu’on déploie sur un serveur (Nginx, Apache, CDN, hébergeur statique, etc.).

---

## 7) Bonnes pratiques & résolution de problèmes

### 7.1 Problèmes fréquents

#### 1) `ng` non reconnu
- Vérifiez l’installation globale : `npm list -g --depth=0`
- Sur Windows, redémarrez le terminal.
- Vérifiez le `PATH`.

#### 2) Problèmes de droits (permissions)
- Évitez d’utiliser `sudo` systématiquement.
- Préférez une gestion propre de Node via `nvm/fnm`.

#### 3) Cache et dépendances
En cas d’état incohérent :

```bash
npm cache verify
```

Puis, au besoin (dans un projet) :
- supprimer `node_modules/`
- supprimer `package-lock.json`
- relancer `npm install`

#### 4) Proxies/réseaux d’entreprise
- Vérifiez la configuration proxy npm si nécessaire.
- Utilisez un registre interne si imposé.

### 7.2 Standardiser l’environnement d’équipe
- Documenter la version Node.
- Verrouiller les versions via `package-lock.json`.
- Utiliser des scripts npm pour partager les commandes.

---

## 8) Atelier guidé (pas à pas)

### Étape 1 — Vérifier Node et npm

```bash
node -v
npm -v
```

### Étape 2 — Installer Angular CLI

```bash
npm install -g @angular/cli
```

### Étape 3 — Vérifier l’installation

```bash
ng version
```

### Étape 4 — Créer un projet

```bash
ng new demo-install
cd demo-install
```

### Étape 5 — Lancer l’application

```bash
ng serve
```

Ouvrir : `http://localhost:4200/`

### Étape 6 — Builder le projet

```bash
ng build
```

Vérifier la présence du dossier `dist/`.

---

## Synthèse

- Angular s’installe et se pilote via **Angular CLI**.
- Installation : `npm install -g @angular/cli`.
- Création de projet : `ng new`.
- Angular CLI facilite : **développement**, **build**, **déploiement**.

---

## Annexes — Commandes essentielles (récap)

```bash
# Installer Angular CLI
npm install -g @angular/cli

# Vérifier la version
ng version

# Créer un projet
ng new mon-projet

# Lancer en dev
ng serve

# Builder
ng build
```
