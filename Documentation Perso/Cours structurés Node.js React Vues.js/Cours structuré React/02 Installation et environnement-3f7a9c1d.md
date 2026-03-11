# Formation React – 02 Installation et environnement

> **Objectif** : mettre en place un environnement de développement React moderne et fiable, comprendre les options (Create React App, Vite, Next.js) et savoir initialiser rapidement un projet React avec **Vite**.

---

## Public visé
- Développeurs débutants à intermédiaires en React
- Développeurs JavaScript/TypeScript souhaitant démarrer un projet React moderne

## Pré-requis
- Connaissances de base en **HTML/CSS/JavaScript**
- Notions de **Node.js** et **npm** (ou équivalent)

## Durée conseillée
- **1h30 à 2h30** (avec exercices)

## Matériel
- Un ordinateur (Windows/macOS/Linux)
- Un accès Internet (pour l’installation initiale)

---

# Plan de la formation

1. **Comprendre les façons d’utiliser React aujourd’hui**
   - React “vanilla” et outils d’industrialisation
   - Create React App (CRA)
   - Vite (approche moderne)
   - Next.js (framework fullstack)

2. **Installer les prérequis**
   - Node.js + npm (ou pnpm/yarn)
   - Vérifier la version et la configuration
   - Éditeur et extensions conseillées

3. **Créer un projet React avec Vite (méthode moderne)**
   - Commande recommandée : `npm create vite@latest`
   - Choix du template React / React + TypeScript
   - Lancement, structure de base, scripts

4. **Bonnes pratiques d’environnement**
   - Gestion des dépendances et lockfiles
   - Variables d’environnement
   - Configuration du formatage (Prettier) et lint (ESLint)

5. **Exercices**
   - Initialiser un projet Vite React
   - Ajouter une page “Hello React”
   - Créer un build de production

---

# 1) Comprendre les façons d’utiliser React aujourd’hui

React est une bibliothèque UI. En pratique, on l’utilise rarement “à la main” sans outil autour : il faut un serveur de dev, un bundler, une gestion des modules, et souvent du lint/format.

## 1.1 Create React App (CRA)
**Create React App** a longtemps été la solution “standard” pour démarrer.

### Points clés
- Très simple à démarrer
- Configuration masquée (opinionated)

### Limites aujourd’hui
- Moins adapté aux besoins modernes (performances, modularité)
- L’écosystème a largement migré vers **Vite** ou des frameworks (Next.js, Remix…)

> À connaître pour comprendre l’historique et des projets existants, mais **pas** la recommandation moderne pour initier de nouveaux projets.

## 1.2 Vite (recommandé en 202x)
**Vite** est un outillage moderne qui propose :
- Serveur de dev ultra rapide
- Compilation/Build performant (via Rollup)
- Configuration claire et extensible

### Pourquoi Vite est la méthode moderne
- Démarrage très rapide
- Excellente DX (Developer Experience)
- Support natif TS et écosystème plugins

**C’est la méthode recommandée ici**.

## 1.3 Next.js
**Next.js** est un framework React orienté applications web complètes, avec :
- Routing intégré
- SSR/SSG, RSC selon versions
- API routes / fullstack

### Quand choisir Next.js ?
- Sites SEO importants
- Besoin de rendu côté serveur, d’optimisations avancées
- Projet fullstack sur une base React

> Dans cette séance, l’objectif est l’installation et l’environnement **React “classique”** : Vite couvre la majorité des besoins front.

---

# 2) Installer les prérequis

## 2.1 Installer Node.js
React via Vite nécessite **Node.js**.

### Recommandation de version
- Utiliser une version **LTS** récente.

### Vérifier l’installation
Dans un terminal :

```bash
node -v
npm -v
```

Si ces commandes répondent avec des versions, Node et npm sont disponibles.

> Astuce : si vous jonglez entre plusieurs projets, utilisez un gestionnaire de versions Node (ex : nvm). Cela évite les conflits.

## 2.2 Choisir un gestionnaire de paquets
- **npm** (par défaut) : OK pour tous les projets
- **pnpm** : rapide, efficace sur les monorepos
- **yarn** : largement utilisé aussi

Dans cette formation, on utilise **npm** pour rester standard.

## 2.3 Installer un éditeur (recommandé)
### Visual Studio Code (VS Code)
Extensions conseillées :
- **ESLint**
- **Prettier**
- **JavaScript and TypeScript Nightly** (optionnel)

---

# 3) Créer un projet React avec Vite (méthode moderne)

## 3.1 Générer le projet avec `npm create vite@latest`
Dans le dossier où vous souhaitez créer votre projet :

```bash
npm create vite@latest
```

Le CLI vous pose des questions.

### Choix typique
- **Project name** : `ma-super-app`
- **Select a framework** : `React`
- **Select a variant** :
  - `JavaScript` (si vous débutez)
  - `TypeScript` (recommandé en entreprise)

> Vous pouvez aussi lancer directement une création avec nom de projet :
>
> ```bash
> npm create vite@latest ma-super-app
> ```

## 3.2 Installer les dépendances
Entrer dans le dossier puis installer :

```bash
cd ma-super-app
npm install
```

## 3.3 Démarrer le serveur de développement

```bash
npm run dev
```

Vous obtenez une URL locale (souvent `http://localhost:5173`).

### Ce que fait le serveur de dev
- Sert votre app React localement
- Recharge automatiquement (HMR) quand vous modifiez des fichiers

## 3.4 Scripts principaux
Dans `package.json`, vous trouverez généralement :
- `dev` : lance le serveur de dev
- `build` : build de production
- `preview` : prévisualisation du build

Construire en production :

```bash
npm run build
```

Prévisualiser :

```bash
npm run preview
```

---

# 4) Structure d’un projet Vite React

Après création, vous verrez typiquement :

```
ma-super-app/
  index.html
  package.json
  vite.config.*
  src/
    main.jsx (ou main.tsx)
    App.jsx (ou App.tsx)
    assets/
  public/
```

## 4.1 `index.html`
Dans Vite, `index.html` est au **root** et sert de point d’entrée.

## 4.2 `src/main.jsx` (ou `main.tsx`)
C’est le point d’initialisation React (montage dans le DOM), par exemple :

```jsx
import { createRoot } from 'react-dom/client'
import App from './App.jsx'
import './index.css'

createRoot(document.getElementById('root')).render(
  <App />
)
```

## 4.3 `src/App.jsx`
C’est le composant racine de votre application.

---

# 5) Bonnes pratiques d’environnement

## 5.1 Lockfile et cohérence
Après `npm install`, un fichier `package-lock.json` est généré.

**Règle** : committer le lockfile pour garantir des installations identiques entre machines et CI.

## 5.2 Variables d’environnement
Vite supporte les variables via des fichiers `.env`.

Exemples courants :
- `.env` (par défaut)
- `.env.development`
- `.env.production`

Dans Vite, les variables exposées au client doivent souvent commencer par `VITE_`.

Exemple `.env.development` :

```bash
VITE_API_URL=http://localhost:3000
```

Utilisation :

```js
console.log(import.meta.env.VITE_API_URL)
```

## 5.3 ESLint et Prettier
Même si le template inclut parfois du lint, en équipe on standardise souvent :
- Formatage automatique (Prettier)
- Règles de qualité (ESLint)

Objectif : réduire les divergences de style et attraper des erreurs tôt.

---

# 6) Exercices (avec corrigés)

## Exercice 1 — Créer un projet React avec Vite
1. Créez un projet `react-install-demo`
2. Choisissez `React` + `JavaScript`
3. Lancez le serveur et ouvrez l’URL

### Correction

```bash
npm create vite@latest react-install-demo
cd react-install-demo
npm install
npm run dev
```

---

## Exercice 2 — Modifier le composant App
Objectif : afficher un message personnalisé.

Dans `src/App.jsx`, remplacez le contenu par :

```jsx
export default function App() {
  return (
    <main style={{ padding: 24, fontFamily: 'system-ui' }}>
      <h1>Installation OK</h1>
      <p>Mon environnement React + Vite est prêt.</p>
    </main>
  )
}
```

Vérifiez que la page se met à jour automatiquement.

---

## Exercice 3 — Build et preview
1. Générez un build
2. Lancez un serveur local de preview

### Correction

```bash
npm run build
npm run preview
```

Expliquez la différence :
- `dev` : optimisé pour développer (HMR)
- `build` : bundle optimisé production
- `preview` : sert le build pour tester comme en prod

---

# Synthèse

- React peut être utilisé via **Create React App**, **Vite** ou **Next.js**.
- La méthode moderne et recommandée pour démarrer une application React front est **Vite**.
- Commande clé :

```bash
npm create vite@latest
```

- Ensuite : `npm install` puis `npm run dev`.

---

# Ressources
- Vite : https://vitejs.dev/
- React : https://react.dev/
- Next.js : https://nextjs.org/
