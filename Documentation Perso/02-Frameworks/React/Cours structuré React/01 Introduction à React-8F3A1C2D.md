# Formation — Introduction à React

**Public visé :** développeurs web (débutants à intermédiaires) ayant des bases en HTML/CSS/JavaScript ES6.

**Pré-requis :**
- JavaScript moderne (let/const, fonctions fléchées, destructuring)
- Notions de DOM et événements
- Node.js installé (>= 18 recommandé)

**Durée suggérée :** 1 journée (6–7h) ou 2 demi-journées (2×3h30)

**Objectifs pédagogiques :**
- Comprendre ce qu’est React et dans quels cas l’utiliser
- Manipuler le modèle **composant** (fonctionnel) et les **props**
- Gérer l’état via **useState** et réagir aux événements
- Expliquer la notion de **Virtual DOM** et le rendu déclaratif
- Mettre en place un projet React moderne (Vite)
- Comprendre les bases de la liste, des clés, du rendu conditionnel et des formulaires

---

## Plan de la formation

1. [Présentation de React](#1-présentation-de-react)
2. [Installer et démarrer un projet](#2-installer-et-démarrer-un-projet)
3. [Les composants : la brique de base](#3-les-composants--la-brique-de-base)
4. [JSX : écrire l’UI dans JavaScript](#4-jsx--écrire-lui-dans-javascript)
5. [Props : passer des données vers un composant](#5-props--passer-des-données-vers-un-composant)
6. [État et interactions : useState](#6-état-et-interactions--usestate)
7. [Rendu conditionnel](#7-rendu-conditionnel)
8. [Listes et clés (keys)](#8-listes-et-clés-keys)
9. [Formulaires : champs contrôlés](#9-formulaires--champs-contrôlés)
10. [Comprendre le Virtual DOM et le rendu](#10-comprendre-le-virtual-dom-et-le-rendu)
11. [Structurer une petite application](#11-structurer-une-petite-application)
12. [Bonnes pratiques et pièges fréquents](#12-bonnes-pratiques-et-pièges-fréquents)
13. [Exercices (avec corrigés)](#13-exercices-avec-corrigés)
14. [Ressources pour aller plus loin](#14-ressources-pour-aller-plus-loin)

---

## 1. Présentation de React

### Qu’est-ce que React ?
React est une **bibliothèque JavaScript** développée par **Meta** (Facebook) qui permet de construire des **interfaces utilisateur dynamiques**.

React est particulièrement adapté pour :
- les applications web avec beaucoup d’interactions (tableaux, dashboards, backoffices)
- les interfaces où l’état change fréquemment (filtres, recherches, formulaires)
- la construction d’UI à partir de composants réutilisables

### Pourquoi React ?
React se distingue par :
- Un modèle basé sur des **composants réutilisables**
- Un paradigme **déclaratif** : on décrit *ce qu’on veut afficher*, React s’occupe du *comment*
- Un **Virtual DOM** qui optimise les mises à jour de l’interface
- Un large écosystème (routing, state management, UI libraries, outils)

### React n’est pas…
- un framework complet (pas de routage ou d’HTTP « officiel » intégré)
- une solution imposant une architecture unique (plus flexible, mais demande des conventions)

---

## 2. Installer et démarrer un projet

### Outil recommandé : Vite
Aujourd’hui, un moyen simple et rapide de créer un projet React est d’utiliser **Vite**.

#### Création du projet
```bash
npm create vite@latest
```
Choisir :
- Framework : **React**
- Variant : **JavaScript** (ou TypeScript si souhaité)

Puis :
```bash
cd mon-projet
npm install
npm run dev
```

### Structure minimale d’un projet Vite + React
Exemple typique :
```
mon-projet/
  src/
    App.jsx
    main.jsx
    index.css
  index.html
  package.json
```

- `main.jsx` : point d’entrée qui monte l’application React
- `App.jsx` : composant racine

#### Exemple `main.jsx`
```jsx
import React from 'react'
import ReactDOM from 'react-dom/client'
import App from './App.jsx'
import './index.css'

ReactDOM.createRoot(document.getElementById('root')).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
)
```

---

## 3. Les composants : la brique de base

### Définition
Un **composant** est une fonction (dans React moderne) qui **retourne du JSX**. Un composant encapsule :
- une structure UI
- éventuellement un état
- des interactions

### Composant fonctionnel simple
```jsx
function Hello() {
  return <h1>Bonjour React</h1>
}

export default Hello
```

### Utilisation d’un composant
```jsx
import Hello from './Hello.jsx'

function App() {
  return (
    <div>
      <Hello />
    </div>
  )
}
```

### Règles de base
- Les noms de composants commencent par une **majuscule** (`Hello`, `UserCard`)
- Un composant doit **retourner un seul élément parent** (ou un fragment)

Fragment :
```jsx
return (
  <>
    <h1>Titre</h1>
    <p>Texte</p>
  </>
)
```

---

## 4. JSX : écrire l’UI dans JavaScript

### Qu’est-ce que le JSX ?
Le **JSX** est une syntaxe proche du HTML, mais qui est en réalité du JavaScript. Il est transpilé (par Babel / Vite) en appels `React.createElement`.

### Interpoler des expressions
Dans un JSX, on utilise `{}` pour insérer du JavaScript :
```jsx
const name = 'Ada'
return <p>Bonjour {name}</p>
```

### Attributs et différences avec le HTML
- `class` devient `className`
- les attributs sont souvent en camelCase (`onClick`, `tabIndex`)

```jsx
return <button className="primary" onClick={handleClick}>OK</button>
```

### Styles inline
Le style inline attend un objet JS :
```jsx
return <div style={{ backgroundColor: 'black', color: 'white' }}>Box</div>
```

---

## 5. Props : passer des données vers un composant

### Définition
Les **props** (propriétés) sont des données passées d’un composant parent vers un enfant.

#### Exemple : composant paramétrable
```jsx
function Greeting({ name }) {
  return <p>Bonjour {name} !</p>
}

function App() {
  return (
    <div>
      <Greeting name="Marie" />
      <Greeting name="Samir" />
    </div>
  )
}
```

### Props immuables
Un composant ne doit pas modifier ses props : elles sont **en lecture seule**.

### Props `children`
`children` permet d’imbriquer du contenu :
```jsx
function Card({ title, children }) {
  return (
    <section style={{ border: '1px solid #ddd', padding: 12 }}>
      <h2>{title}</h2>
      <div>{children}</div>
    </section>
  )
}

function App() {
  return (
    <Card title="Profil">
      <p>Nom : Ada</p>
      <p>Rôle : Dev</p>
    </Card>
  )
}
```

---

## 6. État et interactions : useState

### Pourquoi un état ?
L’**état** (state) représente des données qui évoluent dans le temps et qui doivent déclencher un rerender.

### Hook `useState`
```jsx
import { useState } from 'react'

function Counter() {
  const [count, setCount] = useState(0)

  return (
    <div>
      <p>Compteur : {count}</p>
      <button onClick={() => setCount(count + 1)}>+1</button>
    </div>
  )
}
```

- `count` : valeur actuelle
- `setCount` : fonction de mise à jour (déclenche un rerender)

### Mise à jour basée sur la valeur précédente
Recommandé quand la nouvelle valeur dépend de l’ancienne :
```jsx
setCount(prev => prev + 1)
```

### Événements
React utilise un système d’événements proche du DOM :
```jsx
function App() {
  function handleClick() {
    console.log('click')
  }

  return <button onClick={handleClick}>Clique</button>
}
```

---

## 7. Rendu conditionnel

### Avec un `if`
```jsx
function Welcome({ isLoggedIn }) {
  if (isLoggedIn) return <p>Bienvenue !</p>
  return <p>Connecte-toi</p>
}
```

### Avec l’opérateur ternaire
```jsx
return <p>{isLoggedIn ? 'Bienvenue' : 'Connecte-toi'}</p>
```

### Avec `&&`
Affiche uniquement si vrai :
```jsx
return <>{hasError && <p style={{ color: 'red' }}>Erreur</p>}</>
```

---

## 8. Listes et clés (keys)

### Rendering d’une liste
```jsx
const users = [
  { id: 1, name: 'Ada' },
  { id: 2, name: 'Linus' },
]

function UserList() {
  return (
    <ul>
      {users.map(user => (
        <li key={user.id}>{user.name}</li>
      ))}
    </ul>
  )
}
```

### Pourquoi une `key` ?
La `key` aide React à identifier chaque élément entre 2 rendus (ajout/suppression/réordonnancement) et à mettre à jour efficacement.

**Bonnes clés :** un identifiant stable (`id`)

**À éviter :** l’index du tableau comme `key` (sauf liste statique sans réordonnancement)

---

## 9. Formulaires : champs contrôlés

### Champ contrôlé
En React, on relie la valeur d’un champ à l’état.

```jsx
import { useState } from 'react'

function NewsletterForm() {
  const [email, setEmail] = useState('')

  function handleSubmit(e) {
    e.preventDefault()
    alert(`Email envoyé : ${email}`)
  }

  return (
    <form onSubmit={handleSubmit}>
      <label>
        Email
        <input
          value={email}
          onChange={e => setEmail(e.target.value)}
          type="email"
          placeholder="vous@exemple.com"
        />
      </label>
      <button type="submit">S’inscrire</button>
    </form>
  )
}
```

### Principes
- `value` vient du state
- `onChange` met à jour le state
- `onSubmit` gère la soumission

---

## 10. Comprendre le Virtual DOM et le rendu

### DOM réel vs Virtual DOM
- Le **DOM réel** : structure interne du navigateur représentant la page.
- Le **Virtual DOM** : représentation JavaScript en mémoire de l’interface.

React procède généralement ainsi :
1. Vous changez l’état (`setState` via `useState`)
2. React re-rend le composant (exécute la fonction et produit un nouveau Virtual DOM)
3. React compare l’ancien et le nouveau Virtual DOM (**diffing**)
4. React applique un minimum de changements au DOM réel (**reconciliation**)

### Résultat
Vous écrivez du code déclaratif :
- Vous décrivez l’UI en fonction de l’état
- React optimise la mise à jour

### Important
Le Virtual DOM n’est **pas** « plus rapide par magie » dans tous les cas, mais il aide à :
- simplifier le modèle mental
- éviter de manipuler le DOM à la main
- minimiser les opérations DOM inutiles

---

## 11. Structurer une petite application

### Exemple fil rouge : Todo minimaliste
Objectif : lister des tâches, ajouter, supprimer, marquer comme faite.

#### Modèle de donnée
```js
{ id: 123, label: 'Apprendre React', done: false }
```

#### Composant `App`
```jsx
import { useState } from 'react'

function App() {
  const [todos, setTodos] = useState([
    { id: 1, label: 'Installer Vite', done: true },
    { id: 2, label: 'Comprendre les composants', done: false },
  ])
  const [label, setLabel] = useState('')

  function addTodo(e) {
    e.preventDefault()
    const trimmed = label.trim()
    if (!trimmed) return

    const newTodo = {
      id: Date.now(),
      label: trimmed,
      done: false,
    }

    setTodos(prev => [newTodo, ...prev])
    setLabel('')
  }

  function toggleTodo(id) {
    setTodos(prev =>
      prev.map(t => (t.id === id ? { ...t, done: !t.done } : t))
    )
  }

  function removeTodo(id) {
    setTodos(prev => prev.filter(t => t.id !== id))
  }

  return (
    <div style={{ maxWidth: 520, margin: '40px auto', fontFamily: 'system-ui' }}>
      <h1>Todo React</h1>

      <form onSubmit={addTodo} style={{ display: 'flex', gap: 8 }}>
        <input
          value={label}
          onChange={e => setLabel(e.target.value)}
          placeholder="Nouvelle tâche"
          style={{ flex: 1 }}
        />
        <button type="submit">Ajouter</button>
      </form>

      <ul style={{ paddingLeft: 18, marginTop: 16 }}>
        {todos.map(todo => (
          <li key={todo.id} style={{ margin: '8px 0' }}>
            <label style={{ display: 'flex', alignItems: 'center', gap: 8 }}>
              <input
                type="checkbox"
                checked={todo.done}
                onChange={() => toggleTodo(todo.id)}
              />
              <span style={{ textDecoration: todo.done ? 'line-through' : 'none' }}>
                {todo.label}
              </span>
            </label>
            <button
              onClick={() => removeTodo(todo.id)}
              style={{ marginLeft: 12 }}
              type="button"
            >
              Supprimer
            </button>
          </li>
        ))}
      </ul>

      <footer style={{ marginTop: 24, color: '#666' }}>
        <small>
          Astuce : observez comment l’interface se met à jour uniquement via l’état.
        </small>
      </footer>
    </div>
  )
}

export default App
```

### Points clés pédagogiques
- L’UI est une fonction de `todos` et `label`
- On crée un nouvel état au lieu de muter l’existant
- Les opérations (ajout/suppression/toggle) sont des transformations de tableau

---

## 12. Bonnes pratiques et pièges fréquents

### 12.1 Ne pas muter le state
**À éviter :**
```js
todos.push(newTodo) // mutation
setTodos(todos)
```

**Correct :**
```js
setTodos(prev => [newTodo, ...prev])
```

### 12.2 Comprendre le rerender
Un rerender :
- ré-exécute la fonction composant
- recalcul le JSX
- met à jour le DOM via reconciliation

### 12.3 Découper en composants
Quand un fichier devient « trop gros » :
- extraire `TodoItem`, `TodoList`, `TodoForm`
- conserver l’état au bon niveau (lifting state up)

### 12.4 Éviter les `key` instables
Une `key` instable provoque des effets visuels et des bugs (focus perdu, champs qui « changent »).

---

## 13. Exercices (avec corrigés)

### Exercice 1 — Composant de profil
**Énoncé :** créer un composant `ProfileCard` qui affiche `name`, `role`, `city` via des props.

**Attendu :**
```jsx
function ProfileCard({ name, role, city }) {
  return (
    <div style={{ border: '1px solid #ddd', padding: 12, borderRadius: 8 }}>
      <h3>{name}</h3>
      <p>Rôle : {role}</p>
      <p>Ville : {city}</p>
    </div>
  )
}
```

### Exercice 2 — Compteur avec pas
**Énoncé :** ajouter un champ `step` (number) et un bouton « +step ».

**Corrigé :**
```jsx
import { useState } from 'react'

function StepCounter() {
  const [count, setCount] = useState(0)
  const [step, setStep] = useState(1)

  return (
    <div>
      <p>Valeur : {count}</p>

      <label>
        Pas :
        <input
          type="number"
          value={step}
          onChange={e => setStep(Number(e.target.value))}
        />
      </label>

      <button onClick={() => setCount(prev => prev + step)}>+step</button>
      <button onClick={() => setCount(0)}>Reset</button>
    </div>
  )
}
```

### Exercice 3 — Liste filtrable
**Énoncé :** afficher une liste d’utilisateurs et filtrer par texte.

**Corrigé :**
```jsx
import { useMemo, useState } from 'react'

const USERS = ['Ada Lovelace', 'Linus Torvalds', 'Grace Hopper', 'Alan Turing']

function FilterableList() {
  const [query, setQuery] = useState('')

  const filtered = useMemo(() => {
    const q = query.trim().toLowerCase()
    if (!q) return USERS
    return USERS.filter(u => u.toLowerCase().includes(q))
  }, [query])

  return (
    <div>
      <input
        value={query}
        onChange={e => setQuery(e.target.value)}
        placeholder="filtrer..."
      />

      <ul>
        {filtered.map(name => (
          <li key={name}>{name}</li>
        ))}
      </ul>
    </div>
  )
}
```

> Remarque : `useMemo` n’est pas obligatoire ici, mais permet d’introduire l’idée d’optimisation.

---

## 14. Ressources pour aller plus loin

- Documentation officielle : https://react.dev/
- React + TypeScript : https://react.dev/learn/typescript
- Vite : https://vitejs.dev/
- Testing Library (tests) : https://testing-library.com/

---

## Annexe — Récapitulatif

- **Composants** : fonctions qui retournent du JSX
- **Props** : données du parent vers l’enfant (lecture seule)
- **State** (`useState`) : données internes qui déclenchent un rerender
- **Rendu conditionnel** : `if`, ternaire, `&&`
- **Listes** : `map` + `key` stable
- **Formulaires** : champs contrôlés (`value` + `onChange`)
- **Virtual DOM** : comparaison + mise à jour optimisée du DOM réel
