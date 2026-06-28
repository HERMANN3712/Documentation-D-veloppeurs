# 03 — Structure d’un projet React

## Objectifs pédagogiques
À la fin de cette formation, vous serez capable de :

- Expliquer la structure typique d’un projet React.
- Identifier le rôle de chaque dossier/fichier clé (notamment `src`, `components`, `App` et le point d’entrée `main.jsx`/`index.jsx`).
- Mettre en place une organisation simple et maintenable.
- Appliquer de bonnes pratiques de nommage et de séparation des responsabilités.

## Pré‑requis
- Notions de base en JavaScript moderne (ES6+).
- Connaissances élémentaires de React : composants, props, state.
- Node.js et npm/pnpm/yarn installés.

## Public cible
- Développeurs débutants à intermédiaires en React.
- Formateurs/équipes souhaitant standardiser la structure d’un projet.

## Durée suggérée
- 45 à 60 minutes (selon niveau et échanges).

---

## Plan de la formation

1. **Vue d’ensemble d’un projet React**
2. **Le dossier `src` : le cœur de l’application**
3. **Le dossier `components` : composants réutilisables**
4. **Le point d’entrée : `main.jsx` ou `index.jsx`**
5. **Le composant racine `App`**
6. **Exemple complet de structure**
7. **Bonnes pratiques d’organisation**
8. **Exercice guidé**
9. **Résumé**

---

## 1) Vue d’ensemble d’un projet React

Un projet React est généralement généré via un outil (Vite, Create React App, Next.js, etc.). Quel que soit l’outil, on retrouve presque toujours :

- Un dossier **`src/`** : contient le code source React.
- Un dossier **`components/`** (souvent dans `src/`) : regroupe les composants réutilisables.
- Un fichier principal **`main.jsx`** (Vite) ou **`index.jsx`** (CRA) : initialise l’application et **monte le composant racine** (souvent `App`).

L’objectif de cette structure est de :

- **Séparer** l’assemblage de l’application (point d’entrée) des composants UI.
- Faciliter la **maintenance**, la **réutilisation** et la **lecture**.
- Limiter le couplage entre les fonctionnalités.

---

## 2) Le dossier `src` : le cœur de l’application

### Rôle
`src/` (pour *source*) contient **tout ce qui est compilé et livré** à votre application front :

- Composants React
- Pages/vues (si vous utilisez un routeur)
- Styles (CSS, modules CSS, Sass…)
- Hooks personnalisés
- Fonctions utilitaires
- Assets (selon conventions)

### Contenu typique
Selon les projets, on peut trouver :

- `src/main.jsx` ou `src/index.jsx` : point d’entrée.
- `src/App.jsx` : composant racine.
- `src/components/` : composants réutilisables.
- `src/assets/` : images, icônes (parfois).
- `src/styles/` : styles globaux.

### Pourquoi centraliser dans `src` ?
- **Clarté** : ce qui est dans `src` correspond au code applicatif.
- **Outillage** : bundlers et linters scannent `src` par défaut.
- **Évolutivité** : on sait où ajouter du code sans tout mélanger.

---

## 3) Le dossier `components` : composants réutilisables

### Rôle
Le dossier `components/` (souvent `src/components/`) regroupe des composants **réutilisables**, plutôt orientés UI.

Exemples :
- `Button.jsx`
- `Modal.jsx`
- `Header.jsx`
- `Card.jsx`

### Pourquoi séparer `components` ?
- **Réutilisation**: un composant peut être utilisé à plusieurs endroits.
- **Lisibilité**: on distingue facilement les briques UI.
- **Maintenance**: on localise rapidement l’origine d’un bug UI.

### Conventions courantes
- **Un composant = un fichier** (souvent).
- PascalCase pour les composants : `UserCard.jsx`.
- Export par défaut fréquent, mais export nommé possible selon standards d’équipe.

### Exemple simple
`src/components/Button.jsx`

```jsx
export default function Button({ children, onClick, variant = "primary" }) {
  return (
    <button className={`btn btn--${variant}`} onClick={onClick}>
      {children}
    </button>
  );
}
```

---

## 4) Le point d’entrée : `main.jsx` ou `index.jsx`

### Rôle
Le fichier `main.jsx` (souvent avec Vite) ou `index.jsx` (souvent avec CRA) sert à :

1. Importer React et la couche DOM (`react-dom/client`).
2. Cibler un élément HTML “racine” (ex. `<div id="root"></div>` dans `index.html`).
3. Monter le composant racine `App`.

### Exemple avec Vite (`src/main.jsx`)

```jsx
import React from "react";
import ReactDOM from "react-dom/client";
import App from "./App.jsx";
import "./index.css";

ReactDOM.createRoot(document.getElementById("root")).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
);
```

### Que fait `ReactDOM.createRoot(...).render(...)` ?
- `createRoot` initialise un *root* React (React 18+).
- `render` démarre le rendu et attache l’application au DOM.

### Pourquoi `StrictMode` ?
`React.StrictMode` aide à détecter certains problèmes en développement (effets, patterns obsolètes). Ce n’est **pas** un composant de production au sens rendu visible, mais une aide.

---

## 5) Le composant racine `App`

### Rôle
`App` est le **point central** de votre interface. Souvent, il :

- Contient le layout global (header, footer, contenu).
- Configure le routeur (si utilisé : React Router).
- Initialise certains providers (Context, i18n, thème, etc.).

### Exemple de `App.jsx`

```jsx
import Header from "./components/Header.jsx";
import Button from "./components/Button.jsx";

export default function App() {
  return (
    <div>
      <Header title="Mon application React" />

      <main style={{ padding: 16 }}>
        <h1>Bienvenue</h1>
        <Button onClick={() => alert("Hello React")}>Cliquer</Button>
      </main>
    </div>
  );
}
```

### Différence entre `App` et `main.jsx`
- `main.jsx`/`index.jsx` : **bootstrap** technique (montage React dans le DOM).
- `App.jsx` : **logique UI** racine et composition de l’application.

---

## 6) Exemple complet de structure

Voici un exemple minimal et cohérent avec les éléments demandés : `src`, `components` et un fichier principal `main.jsx`/`index.jsx`.

```text
mon-projet-react/
├─ index.html
├─ package.json
├─ vite.config.js            (selon l’outil)
└─ src/
   ├─ main.jsx               (ou index.jsx)
   ├─ App.jsx
   ├─ index.css
   └─ components/
      ├─ Header.jsx
      └─ Button.jsx
```

### Exemple `Header.jsx`

```jsx
export default function Header({ title }) {
  return (
    <header style={{ padding: 16, borderBottom: "1px solid #ddd" }}>
      <strong>{title}</strong>
    </header>
  );
}
```

---

## 7) Bonnes pratiques d’organisation

### 7.1 Garder une structure simple au démarrage
Au début, un projet peut rester minimal :
- `src/`
- `src/components/`
- `src/App.jsx`
- `src/main.jsx`

Puis évoluer au fur et à mesure.

### 7.2 Éviter les “dossiers fourre‑tout”
Deux pièges fréquents :
- Mettre tout dans `components/` même si ce sont des écrans/pages.
- Mélanger logique métier et UI dans les mêmes fichiers sans limites.

### 7.3 Décider d’une convention : par type ou par feature
- **Par type** (simple) : `components/`, `pages/`, `hooks/`, `utils/`.
- **Par feature** (scalable) : `features/auth/`, `features/cart/`, etc.

Dans cette formation, on reste sur l’approche simple centrée sur `src` + `components` + point d’entrée + `App`.

### 7.4 Nommage cohérent
- Composants : `PascalCase`.
- Fichiers de styles : `kebab-case` ou aligné sur les composants (ex. `Button.module.css`).
- Éviter les noms vagues : `Component1.jsx`, `Test.jsx`.

### 7.5 Responsabilités claires
- `main.jsx` : initialisation, providers globaux.
- `App.jsx` : structure globale, routes.
- `components/` : UI réutilisable.

---

## 8) Exercice guidé (15–20 min)

### Énoncé
Créer la structure suivante et vérifier que l’application s’affiche :

- `src/main.jsx` (ou `index.jsx`) qui monte `<App />`.
- `src/App.jsx` qui affiche un titre et utilise un composant.
- `src/components/Hello.jsx` qui reçoit un `name` en props.

### Étapes
1. Créer `src/components/Hello.jsx` :

```jsx
export default function Hello({ name }) {
  return <p>Bonjour {name} !</p>;
}
```

2. Modifier `src/App.jsx` :

```jsx
import Hello from "./components/Hello.jsx";

export default function App() {
  return (
    <div style={{ padding: 16 }}>
      <h1>Structure de projet React</h1>
      <Hello name="React" />
    </div>
  );
}
```

3. Vérifier `src/main.jsx` :

```jsx
import React from "react";
import ReactDOM from "react-dom/client";
import App from "./App.jsx";

ReactDOM.createRoot(document.getElementById("root")).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
);
```

### Critères de réussite
- L’application se lance sans erreur.
- Le rendu affiche le titre et le texte « Bonjour React ! ».

---

## 9) Résumé

- Un projet React contient généralement un dossier **`src/`** : le code source.
- Un dossier **`components/`** (souvent dans `src`) : composants UI réutilisables.
- Un fichier principal **`main.jsx`** ou **`index.jsx`** : initialise React et monte l’application.
- Le composant **`App`** : composant racine qui compose l’interface.

---

### Annexes — Questions fréquentes

**Où placer les composants ?**  
Dans `src/components/` si ce sont des éléments réutilisables.

**Pourquoi `main.jsx` vs `index.jsx` ?**  
C’est souvent une convention d’outil (Vite utilise fréquemment `main.jsx`, CRA `index.jsx`). Le rôle reste le même.

**Peut-on avoir `components` à la racine ?**  
Oui, mais la convention la plus courante est `src/components` pour regrouper le code applicatif.
