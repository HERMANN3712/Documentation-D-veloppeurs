# Formation Node.js — 02 Installation et environnement

## Objectifs pédagogiques
À la fin de ce module, vous serez capable de :

- Installer **Node.js** depuis le site officiel **nodejs.org**.
- Vérifier correctement l’installation via les commandes **`node -v`** et **`npm -v`**.
- Expliquer le rôle de **npm** en tant que gestionnaire de paquets permettant d’installer des bibliothèques JavaScript.

---

## Public visé & prérequis

- **Public** : développeurs débutants à intermédiaires, étudiants, professionnels souhaitant démarrer avec Node.js.
- **Prérequis** : savoir utiliser un terminal (Windows PowerShell / macOS Terminal / Linux shell) et connaître les bases de JavaScript.

---

## Plan de la formation

1. **Comprendre ce qu’on installe**
   - Node.js (runtime JavaScript)
   - npm (gestionnaire de paquets)
2. **Installation de Node.js depuis nodejs.org**
   - Choisir la version (LTS vs Current)
   - Télécharger et installer
3. **Vérification de l’installation**
   - `node -v`
   - `npm -v`
   - Interpréter la sortie
4. **Comprendre npm (essentiel)**
   - Rôle de npm
   - Installer des bibliothèques JavaScript

---

## 1) Comprendre ce qu’on installe

### 1.1 Node.js

**Node.js** est un environnement d’exécution (runtime) qui permet d’exécuter du **JavaScript en dehors du navigateur**, notamment :

- sur un serveur (API, back-end)
- en ligne de commande (scripts, outils)
- pour des tâches d’automatisation

En résumé : **Node.js = JavaScript côté serveur + outils d’exécution**.

### 1.2 npm

Lorsque vous installez Node.js, vous installez généralement aussi **npm**.

- **npm** signifie historiquement *Node Package Manager*.
- C’est un **gestionnaire de paquets** : il sert à **installer**, **mettre à jour** et **gérer** des bibliothèques (des *packages*) JavaScript.

Exemples typiques de bibliothèques installées via npm :

- frameworks serveur (ex. Express)
- outils de build (ex. Vite, Webpack)
- utilitaires (ex. lodash)

---

## 2) Installation de Node.js depuis nodejs.org

### 2.1 Télécharger Node.js

1. Ouvrez votre navigateur
2. Allez sur : **https://nodejs.org/**

Vous verrez généralement **deux versions** proposées :

- **LTS (Long Term Support)** : version recommandée pour la stabilité (production, formation, entreprises)
- **Current** : version plus récente, avec les dernières fonctionnalités, mais potentiellement moins stable

> Recommandation : choisissez **LTS** pour la plupart des usages.

### 2.2 Installer Node.js

Une fois l’installeur téléchargé :

- Lancez l’installeur
- Acceptez les conditions
- Conservez les options par défaut si vous débutez

> Important : l’installation configure généralement le **PATH** (variable d’environnement) pour rendre `node` et `npm` accessibles depuis le terminal.

#### Remarque sur le terminal

Après installation :

- Fermez puis rouvrez votre terminal
- (Sinon, il peut ne pas « voir » immédiatement `node` et `npm`)

---

## 3) Vérification de l’installation

Une fois Node.js installé, la toute première étape consiste à vérifier que :

- `node` est disponible
- `npm` est disponible

### 3.1 Vérifier la version de Node.js

Dans un terminal, exécutez :

```bash
node -v
```

**Ce que ça fait :**
- affiche la version de Node.js installée

**Exemple de sortie :**

```bash
v20.11.1
```

> La présence du `v` est normale : c’est le format habituel des versions Node.js.

### 3.2 Vérifier la version de npm

Dans le même terminal :

```bash
npm -v
```

**Ce que ça fait :**
- affiche la version de npm installée

**Exemple de sortie :**

```bash
10.2.4
```

### 3.3 En cas de problème

Si l’une des commandes renvoie un message du type :

- « command not found » (macOS/Linux)
- « n’est pas reconnu en tant que commande interne » (Windows)

Cela signifie généralement que :

- Node.js n’est pas installé correctement, **ou**
- le terminal ne voit pas Node.js (PATH non configuré ou terminal non redémarré)

Actions rapides à tenter :

1. Fermer/réouvrir le terminal
2. Réinstaller Node.js depuis **nodejs.org**

---

## 4) Comprendre npm (essentiel)

### 4.1 npm : gestionnaire de paquets

**npm** est l’outil qui permet d’installer des bibliothèques JavaScript via le registre npm.

En pratique, vous l’utiliserez pour :

- ajouter une dépendance à un projet (une bibliothèque)
- mettre à jour des packages
- exécuter certains scripts (via `npm run ...` — sera détaillé dans un module ultérieur)

### 4.2 Installer une bibliothèque JavaScript

Le point clé : **npm permet d’installer des bibliothèques JavaScript**.

Un exemple générique (vous le ferez réellement dans un prochain cours) :

```bash
npm install nom-du-package
```

**Ce que cela implique généralement :**

- `npm` télécharge le package
- le package est ajouté aux dépendances du projet
- les fichiers sont placés dans un dossier `node_modules/`

> Pour l’instant, retenez surtout le rôle de npm : **installer et gérer des packages JavaScript**.

---

## Synthèse (à retenir)

- Installez Node.js depuis **https://nodejs.org/** (préférez **LTS**)
- Vérifiez l’installation :
  - `node -v`
  - `npm -v`
- **npm** est le gestionnaire de paquets qui permet d’installer des bibliothèques JavaScript.

---

## Mini-checklist de fin de module

- [ ] Node.js est installé depuis nodejs.org
- [ ] `node -v` affiche une version
- [ ] `npm -v` affiche une version
- [ ] Je peux expliquer en une phrase le rôle de npm
