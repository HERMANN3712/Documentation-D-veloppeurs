# 01 — Introduction à React

> **Public** : débutants en React (bases HTML/CSS/JS nécessaires)  
> **Durée conseillée** : 3h à 1 journée (selon ateliers)  
> **Pré-requis** : JavaScript ES6 (let/const, fonctions fléchées, modules, destructuring), notions de DOM et d’événements

---

## Objectifs pédagogiques

À la fin de cette formation, vous saurez :

- Expliquer ce qu’est React et dans quels cas l’utiliser.
- Comprendre le **modèle mental** : UI = fonction de l’état.
- Créer et composer des **composants** réutilisables.
- Comprendre le rôle du **Virtual DOM** et le rendu déclaratif.
- Passer des données via **props** et gérer un état simple via **useState**.
- Mettre en place un petit projet avec un outil moderne (Vite) et structurer un code React.

---

## Plan de la formation

1. **Contexte & pourquoi React**
2. **Principes fondamentaux**
   - Composants
   - JSX
   - Rendu déclaratif
3. **Virtual DOM & performance (concepts)**
4. **Premiers pas : créer une application React**
5. **Props, state et événements**
6. **Cycle de rendu & bonnes pratiques de base**
7. **Atelier : mini application (liste filtrable)**
8. **Ressources & prochaines étapes**

---

## 1) Contexte & pourquoi React

### Qu’est-ce que React ?

**React** est une bibliothèque JavaScript développée par **Meta** (Facebook) qui permet de construire des **interfaces utilisateur dynamiques**.

React se concentre principalement sur la **couche vue** (UI). On l’utilise souvent avec d’autres bibliothèques/outils pour la gestion de routage, de données, etc.

### Pourquoi utiliser React ?

- **Composants réutilisables** : on découpe l’interface en briques isolées.
- **UI déclarative** : on décrit *ce qu’on veut afficher* plutôt que *comment manipuler le DOM*.
- **Écosystème** : outils, patterns, composants UI, communauté.
- **Performance** : React optimise les mises à jour via un **Virtual DOM** (voir section dédiée).

### Quand React est-il pertinent ?

- Applications riches (tableaux, filtres, formulaires complexes)
- UI très interactive (dashboards, SPA)
- Projets où la **maintenabilité** et la **réutilisabilité** comptent

### Quand ce n’est pas nécessaire ?

- Page statique simple
- Site vitrine léger où quelques lignes de JS suffisent

---

## 2) Principes fondamentaux

### 2.1 Le modèle mental : UI = f(state)

En React, l’interface est une **fonction** de l’état.

- Vous stockez des données (l’**état**)
- React calcule à quoi l’UI doit ressembler
- À chaque changement d’état, React **re-rend** ce qui doit changer

> **Idée clé** : vous évitez les manipulations DOM manuelles (querySelector, innerHTML, etc.) autant que possible.

### 2.2 Les composants

Un **composant** est une unité d’interface réutilisable. Il peut :

- Afficher des données
- Recevoir des **props** (propriétés) en entrée
- Gérer un état interne (state)
- Émettre des événements (via callbacks)

#### Exemple : composant simple

```jsx
function Hello() {
  return <h1>Bonjour React</h1>;
}
```

#### Composer des composants

```jsx
function Header() {
  return <header>Mon site</header>;
}

function App() {
  return (
    <div>
      <Header />
      <main>Contenu</main>
    </div>
  );
}
```

**Composition** = assembler des briques simples pour former une interface complexe.

### 2.3 JSX : écrire l’UI avec une syntaxe proche HTML

JSX est une extension de syntaxe qui permet d’écrire :

```jsx
const title = "Introduction à React";

function App() {
  return <h1>{title}</h1>;
}
```

Points importants :

- JSX **n’est pas** du HTML : c’est du JavaScript qui produit des éléments React.
- Interpolation via `{...}`
- Attributs : `className` au lieu de `class`
- Styles inline : objet JS

```jsx
const style = { color: "tomato", fontWeight: 700 };

function Banner() {
  return <p style={style}>Texte important</p>;
}
```

### 2.4 Rendu conditionnel et listes

#### Conditionnel

```jsx
function Welcome({ isLoggedIn }) {
  if (isLoggedIn) return <p>Content de vous revoir !</p>;
  return <p>Veuillez vous connecter.</p>;
}
```

Ou version courte :

```jsx
{isLoggedIn ? <Dashboard /> : <Login />}
```

#### Listes + `key`

```jsx
function List({ items }) {
  return (
    <ul>
      {items.map((item) => (
        <li key={item.id}>{item.label}</li>
      ))}
    </ul>
  );
}
```

> **Règle** : `key` doit être stable et unique (éviter l’index si la liste change).

---

## 3) Virtual DOM & performance (concepts)

### 3.1 DOM vs Virtual DOM

- Le **DOM** (Document Object Model) est la représentation de la page dans le navigateur.
- Mettre à jour le DOM peut être coûteux lorsqu’on multiplie les opérations.

Le **Virtual DOM** est une représentation **en mémoire** de l’UI.

### 3.2 Comment React met à jour l’UI

1. Votre état change (ex : `setCount(count + 1)`)
2. React **recalcule** le rendu (nouvel arbre d’éléments)
3. React compare l’ancien et le nouveau (diff)
4. React applique **uniquement les changements nécessaires** au DOM réel

Ce processus s’appelle souvent la **reconciliation**.

> À retenir : React n’“évite pas” tous les coûts, mais il standardise et optimise les mises à jour.

### 3.3 Ce que vous pouvez faire pour aider React

- Utiliser des `key` correctes
- Éviter de recréer inutilement des structures/objets dans le rendu (au besoin)
- Découper en composants cohérents

---

## 4) Premiers pas : créer une application React

### 4.1 Outil recommandé : Vite

Vite est un outil moderne pour créer et développer une app React rapidement.

```bash
npm create vite@latest
# Choisir: React + JavaScript (ou TypeScript)
cd mon-projet
npm install
npm run dev
```

### 4.2 Structure typique

- `src/main.jsx` : point d’entrée
- `src/App.jsx` : composant racine
- `src/components/` : composants réutilisables

#### Exemple de `main.jsx`

```jsx
import { createRoot } from "react-dom/client";
import App from "./App.jsx";

createRoot(document.getElementById("root")).render(<App />);
```

---

## 5) Props, state et événements

### 5.1 Props : données entrantes (immutables)

Les **props** sont des paramètres qu’un parent donne à un enfant.

```jsx
function UserCard({ name }) {
  return <p>Utilisateur : {name}</p>;
}

function App() {
  return <UserCard name="Ada" />;
}
```

**Règle** : un composant ne doit pas modifier ses props.

### 5.2 State : données internes qui évoluent

Pour un état local simple, on utilise `useState`.

```jsx
import { useState } from "react";

function Counter() {
  const [count, setCount] = useState(0);

  return (
    <div>
      <p>Compteur : {count}</p>
      <button onClick={() => setCount(count + 1)}>+1</button>
    </div>
  );
}
```

**À comprendre** :

- `setCount` déclenche un re-rendu.
- Ne modifiez pas l’état “en place” si c’est un objet/tableau (on crée une copie).

#### Exemple avec tableau (immutabilité)

```jsx
import { useState } from "react";

function TodoList() {
  const [todos, setTodos] = useState(["Apprendre React"]);

  function addTodo() {
    setTodos((prev) => [...prev, "Nouvelle tâche"]);
  }

  return (
    <div>
      <button onClick={addTodo}>Ajouter</button>
      <ul>
        {todos.map((t, i) => (
          <li key={t + i}>{t}</li>
        ))}
      </ul>
    </div>
  );
}
```

### 5.3 Gestion des événements

En React, les événements ressemblent à ceux du DOM, mais avec une API cohérente.

```jsx
function Form() {
  function handleSubmit(e) {
    e.preventDefault();
    // traitement
  }

  return (
    <form onSubmit={handleSubmit}>
      <button type="submit">Envoyer</button>
    </form>
  );
}
```

---

## 6) Cycle de rendu & bonnes pratiques de base

### 6.1 Quand un composant se re-rend ?

Un composant se re-rend notamment quand :

- son **state** change
- ses **props** changent
- un parent se re-rend (et lui passe de nouvelles props)

### 6.2 Bonnes pratiques

- Un composant = une responsabilité claire
- Nommer les composants en **PascalCase**
- Garder `App` simple, déléguer aux sous-composants
- Éviter la duplication : factoriser les composants
- Préférer des fonctions pures dans le rendu

---

## 7) Atelier : mini application (liste filtrable)

### Objectif

Créer une liste d’éléments affichée à l’écran et filtrée via un champ de saisie.

### Étapes

1. Initialiser un projet Vite React
2. Créer un composant `FilterableList`
3. Gérer :
   - un état `query`
   - une liste `items`
4. Filtrer en fonction de `query`

### Code complet proposé

```jsx
import { useMemo, useState } from "react";

const initialItems = [
  { id: 1, label: "React" },
  { id: 2, label: "Vue" },
  { id: 3, label: "Svelte" },
  { id: 4, label: "Angular" },
];

export default function FilterableList() {
  const [query, setQuery] = useState("");
  const [items] = useState(initialItems);

  const filtered = useMemo(() => {
    const q = query.trim().toLowerCase();
    if (!q) return items;
    return items.filter((it) => it.label.toLowerCase().includes(q));
  }, [query, items]);

  return (
    <section style={{ maxWidth: 420 }}>
      <h1>Introduction à React — Atelier</h1>

      <label style={{ display: "block", marginBottom: 8 }}>
        Filtrer :
        <input
          value={query}
          onChange={(e) => setQuery(e.target.value)}
          placeholder="Tapez un framework…"
          style={{ display: "block", width: "100%", padding: 8, marginTop: 6 }}
        />
      </label>

      <ul>
        {filtered.map((it) => (
          <li key={it.id}>{it.label}</li>
        ))}
      </ul>

      <p style={{ color: "#666" }}>
        {filtered.length} résultat(s)
      </p>
    </section>
  );
}
```

### Questions de validation

- Qu’est-ce qui déclenche le re-rendu ici ?
- Pourquoi `key={it.id}` est important ?
- Que se passe-t-il si on modifie `items` en place ?

---

## 8) Ressources & prochaines étapes

### Documentation

- React (FR/EN) : https://react.dev/

### Prochaines notions à apprendre

- **useEffect** (effets, appels API)
- **Gestion de formulaires** (contrôlés)
- **Props drilling** vs **Context**
- **Routing** (React Router)
- **State management** (Redux, Zustand…) selon la taille du projet
- **TypeScript** avec React

---

## Résumé

- React permet de construire des interfaces via des **composants**.
- Le rendu est **déclaratif** : l’UI se déduit de l’état.
- Le **Virtual DOM** aide à optimiser la mise à jour de l’interface en appliquant les changements nécessaires.
- `props` = données entrantes, `state` = données internes, `events` = interactions utilisateur.

---

*Fin du cours — « Introduction à React »*
