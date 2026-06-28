# Formation React — Hooks (useState, useEffect, useContext, useRef, useMemo)

> **Public** : développeurs React (débutant → intermédiaire)\
> **Pré-requis** : JavaScript ES6+, notions de composants React, props, JSX\
> **Durée conseillée** : 1 journée (6–7h) ou 2 demi-journées\
> **Objectif** : savoir utiliser les hooks React pour gérer l’état, les effets, le contexte, les références et l’optimisation dans des composants fonctionnels.

---

## Sommaire

1. [Introduction : pourquoi les hooks ?](#1-introduction--pourquoi-les-hooks-)
2. [Rappels : composants fonctionnels, rendu et re-render](#2-rappels--composants-fonctionnels-rendu-et-re-render)
3. [Hook 1 — useState](#3-hook-1--usestate)
4. [Hook 2 — useEffect](#4-hook-2--useeffect)
5. [Hook 3 — useContext](#5-hook-3--usecontext)
6. [Hook 4 — useRef](#6-hook-4--useref)
7. [Hook 5 — useMemo](#7-hook-5--usememo)
8. [Atelier fil rouge : mini-app “Dashboard”](#8-atelier-fil-rouge--mini-app-dashboard)
9. [Bonnes pratiques, anti-patterns et checklists](#9-bonnes-pratiques-anti-patterns-et-checklists)
10. [Quiz / Évaluation](#10-quiz--évaluation)
11. [Annexes : snippets et mémo](#11-annexes--snippets-et-mémo)

---

## 1. Introduction : pourquoi les hooks ?

### 1.1 Définition
Les **hooks** sont des fonctions fournies par React qui permettent d’“accrocher” (hook into) des fonctionnalités React (état, cycle de vie, contexte, refs, optimisations…) dans des **composants fonctionnels**.

Avant les hooks, on utilisait principalement :
- des **class components** pour l’état (`this.state`) et le cycle de vie (`componentDidMount`, `componentDidUpdate`, `componentWillUnmount`),
- des **HOCs** ou **render props** pour réutiliser de la logique.

Les hooks ont été introduits pour :
- **simplifier la composition** de logique (réutilisation via custom hooks),
- **réduire la complexité** des classes,
- rendre le code plus **déclaratif** et **modulaire**.

### 1.2 Règles des hooks (à connaître absolument)
React impose deux règles :

1. **Appeler les hooks uniquement au niveau racine** d’un composant fonctionnel (ou d’un custom hook).\
   → Pas dans des boucles, conditions, fonctions imbriquées.
2. **Appeler les hooks uniquement depuis React** : composant fonctionnel ou custom hook.

> Objectif : garantir un ordre d’appel stable des hooks à chaque rendu.

Exemple incorrect :
```jsx
if (isOpen) {
  const [x, setX] = useState(0); // ❌
}
```

Exemple correct :
```jsx
const [x, setX] = useState(0);
if (isOpen) {
  // ...
}
```

---

## 2. Rappels : composants fonctionnels, rendu et re-render

### 2.1 Le rendu React en bref
- Un **rendu** correspond à l’exécution de la fonction composant.
- Un changement de **state** ou de **props** déclenche un nouveau rendu.
- React compare l’arbre virtuel (Virtual DOM) et applique les changements nécessaires.

### 2.2 Variables locales vs state
Une variable locale dans le composant :
- est **réinitialisée** à chaque rendu,
- ne déclenche pas de re-render quand elle change.

Le state via `useState` :
- est **persisté** entre les rendus,
- `setState` déclenche un re-render.

---

## 3. Hook 1 — useState

### 3.1 Objectif
`useState` permet de **déclarer un état local** dans un composant fonctionnel.

Signature :
```jsx
const [state, setState] = useState(initialState);
```

- `initialState` peut être une valeur (nombre, objet…) ou une fonction d’initialisation.
- `setState` déclenche un re-render.

### 3.2 Exemple basique : compteur
```jsx
import { useState } from "react";

export function Counter() {
  const [count, setCount] = useState(0);

  return (
    <div>
      <p>Compteur : {count}</p>
      <button onClick={() => setCount(count + 1)}>+1</button>
      <button onClick={() => setCount(count - 1)}>-1</button>
      <button onClick={() => setCount(0)}>Reset</button>
    </div>
  );
}
```

### 3.3 Mise à jour basée sur l’état précédent (forme fonctionnelle)
Quand le nouvel état dépend de l’ancien, utilisez la forme fonctionnelle :
```jsx
setCount((prev) => prev + 1);
```

Pourquoi ?
- évite des bugs quand plusieurs updates sont “batchées” (groupées) par React.

Exemple :
```jsx
<button onClick={() => {
  setCount((c) => c + 1);
  setCount((c) => c + 1);
}}>
  +2
</button>
```

### 3.4 State objet : attention à l’immutabilité
```jsx
const [user, setUser] = useState({ name: "Ada", age: 28 });

function birthday() {
  setUser((u) => ({ ...u, age: u.age + 1 }));
}
```

> Ne jamais muter : `user.age++` puis `setUser(user)` ❌

### 3.5 Initialisation coûteuse : lazy initialization
Si l’état initial est coûteux à calculer :
```jsx
const [data, setData] = useState(() => expensiveCompute());
```

### 3.6 Exercice
1. Créer un `TodoList` minimal avec :
   - un champ texte contrôlé,
   - un bouton “Ajouter”,
   - une liste de todos,
   - suppression d’un todo.

Points d’attention :
- state immuable,
- ne pas utiliser l’index comme clé si possible (utiliser un `id`).

---

## 4. Hook 2 — useEffect

### 4.1 Objectif
`useEffect` permet de synchroniser le composant avec un **effet** :
- appel API,
- abonnement (events),
- timers,
- interaction avec le DOM / environnement externe,
- logging, analytics…

Signature :
```jsx
useEffect(() => {
  // effet
  return () => {
    // cleanup (optionnel)
  };
}, [dependencies]);
```

### 4.2 Comprendre les dépendances
Le tableau de dépendances contrôle **quand** l’effet s’exécute :

- **Sans tableau** : à chaque rendu
  ```jsx
  useEffect(() => { /* ... */ });
  ```
- **Tableau vide `[]`** : une fois au montage (et cleanup au démontage)
  ```jsx
  useEffect(() => { /* ... */ }, []);
  ```
- **Avec dépendances** : au montage + à chaque changement d’une dépendance
  ```jsx
  useEffect(() => { /* ... */ }, [userId]);
  ```

> Important : les dépendances doivent inclure tout ce qui est utilisé dans l’effet et provient du scope (props, state, fonctions définies dans le composant), sauf exceptions maîtrisées.

### 4.3 Exemple : fetch de données
```jsx
import { useEffect, useState } from "react";

export function UserProfile({ userId }) {
  const [user, setUser] = useState(null);
  const [status, setStatus] = useState("idle"); // idle | loading | success | error

  useEffect(() => {
    let cancelled = false;

    async function load() {
      setStatus("loading");
      try {
        const res = await fetch(`https://jsonplaceholder.typicode.com/users/${userId}`);
        if (!res.ok) throw new Error("HTTP error");
        const json = await res.json();
        if (!cancelled) {
          setUser(json);
          setStatus("success");
        }
      } catch (e) {
        if (!cancelled) setStatus("error");
      }
    }

    load();

    return () => {
      cancelled = true;
    };
  }, [userId]);

  if (status === "loading") return <p>Chargement...</p>;
  if (status === "error") return <p>Erreur</p>;
  if (!user) return null;

  return (
    <div>
      <h2>{user.name}</h2>
      <p>{user.email}</p>
    </div>
  );
}
```

**Pourquoi un flag `cancelled` ?**
- éviter de mettre à jour le state après un démontage ou un changement rapide de `userId`.
- En pratique, vous pouvez aussi utiliser `AbortController`.

### 4.4 Exemple : abonnement + cleanup
```jsx
import { useEffect, useState } from "react";

export function WindowWidth() {
  const [width, setWidth] = useState(window.innerWidth);

  useEffect(() => {
    function onResize() {
      setWidth(window.innerWidth);
    }

    window.addEventListener("resize", onResize);
    return () => window.removeEventListener("resize", onResize);
  }, []);

  return <p>Largeur : {width}px</p>;
}
```

### 4.5 Pièges fréquents
- **Boucle infinie** :
  - effet qui fait `setState` et dépend de ce state, sans condition.
- **Dépendances manquantes** : valeurs “stales” (obsolètes) dans l’effet.
- **Effets pour dériver l’UI** : souvent un `useMemo` ou un calcul direct suffit.

### 4.6 Exercice
Créer un composant `SearchUsers` :
- input contrôlé `query`
- `useEffect` déclenche recherche simulée après 300ms (debounce simple avec `setTimeout` + cleanup)
- afficher `loading`, `results`, `error`.

---

## 5. Hook 3 — useContext

### 5.1 Objectif
`useContext` permet de lire une valeur de **contexte** React (state global-ish), évitant le **prop drilling**.

Cas d’usage :
- thème (dark/light),
- langue,
- utilisateur connecté,
- configuration,
- dépendances partagées (clients API, feature flags).

> Le contexte n’est pas une solution universelle de state management. Pour des états complexes et très dynamiques, évaluer Zustand/Redux/Jotai, etc.

### 5.2 Création d’un contexte
1) Créer le contexte :
```jsx
import { createContext } from "react";

export const ThemeContext = createContext("light");
```

2) Fournir la valeur avec un Provider :
```jsx
import { ThemeContext } from "./ThemeContext";

export function App() {
  return (
    <ThemeContext.Provider value="dark">
      <Page />
    </ThemeContext.Provider>
  );
}
```

3) Consommer via `useContext` :
```jsx
import { useContext } from "react";
import { ThemeContext } from "./ThemeContext";

export function Button() {
  const theme = useContext(ThemeContext);
  return <button className={`btn btn--${theme}`}>OK</button>;
}
```

### 5.3 Contexte avec objet (valeur + actions)
Souvent, on fournit un objet :
```jsx
export const AuthContext = createContext(null);

export function AuthProvider({ children }) {
  const [user, setUser] = useState(null);

  const login = async (email, password) => {
    // ...
    setUser({ id: "u1", email });
  };

  const logout = () => setUser(null);

  const value = { user, login, logout };

  return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>;
}

export function useAuth() {
  const ctx = useContext(AuthContext);
  if (!ctx) throw new Error("useAuth must be used within AuthProvider");
  return ctx;
}
```

> Pattern recommandé : exposer un **custom hook** `useAuth()`.

### 5.4 Performance : éviter les re-renders inutiles
Quand la valeur du provider change, **tous** les consommateurs re-render.

Bonnes pratiques :
- mémoriser la `value` avec `useMemo` (si nécessaire) :
```jsx
const value = useMemo(() => ({ user, login, logout }), [user]);
```
- séparer les contextes (ex : `AuthUserContext`, `AuthActionsContext`).

### 5.5 Exercice
Mettre en place un `ThemeProvider` :
- `theme` dans le state
- `toggleTheme()`
- composants `Toolbar` et `Card` qui consomment le thème sans props.

---

## 6. Hook 4 — useRef

### 6.1 Objectif
`useRef` fournit un conteneur **mutable** qui persiste entre les rendus **sans déclencher de re-render** quand sa valeur change.

Deux usages principaux :
1) **Référence DOM** (accéder à un élément)
2) **Stockage mutable** (timeout id, valeur précédente, instance de librairie, etc.)

Signature :
```jsx
const ref = useRef(initialValue);
// ref.current est la valeur mutable
```

### 6.2 Référence JavaScript vers un élément DOM
```jsx
import { useEffect, useRef } from "react";

export function FocusInput() {
  const inputRef = useRef(null);

  useEffect(() => {
    inputRef.current?.focus();
  }, []);

  return <input ref={inputRef} placeholder="Je suis focus au montage" />;
}
```

### 6.3 Stocker une valeur sans re-render : timer
```jsx
import { useEffect, useRef, useState } from "react";

export function DebouncedCounter() {
  const [count, setCount] = useState(0);
  const timerRef = useRef(null);

  function increment() {
    if (timerRef.current) clearTimeout(timerRef.current);

    timerRef.current = setTimeout(() => {
      setCount((c) => c + 1);
    }, 300);
  }

  useEffect(() => {
    return () => {
      if (timerRef.current) clearTimeout(timerRef.current);
    };
  }, []);

  return (
    <div>
      <p>{count}</p>
      <button onClick={increment}>+1 (debounced)</button>
    </div>
  );
}
```

### 6.4 Garder la valeur précédente (previous value)
```jsx
import { useEffect, useRef, useState } from "react";

export function PreviousValueDemo() {
  const [value, setValue] = useState("");
  const prevRef = useRef("");

  useEffect(() => {
    prevRef.current = value;
  }, [value]);

  return (
    <div>
      <input value={value} onChange={(e) => setValue(e.target.value)} />
      <p>Actuel : {value}</p>
      <p>Précédent : {prevRef.current}</p>
    </div>
  );
}
```

### 6.5 Exercice
Créer un composant `Stopwatch` :
- `Start/Stop/Reset`
- `setInterval` stocké dans une ref
- cleanup au démontage

---

## 7. Hook 5 — useMemo

### 7.1 Objectif
`useMemo` mémorise le résultat d’un calcul pour éviter de le recalculer à chaque rendu.

Signature :
```jsx
const memoizedValue = useMemo(() => compute(a, b), [a, b]);
```

### 7.2 Quand l’utiliser ?
**À utiliser** si :
- le calcul est **coûteux** (ex : tri/filtrage de grande liste, opérations lourdes),
- et qu’il dépend de valeurs qui changent rarement.

**À éviter** si :
- le calcul est trivial,
- ou si on l’utilise partout “par réflexe” (complexifie le code).

> `useMemo` est une optimisation, pas un outil de logique métier.

### 7.3 Exemple : filtrage + tri d’une liste
```jsx
import { useMemo, useState } from "react";

export function ProductList({ products }) {
  const [query, setQuery] = useState("");
  const [sort, setSort] = useState("price"); // price | name

  const visibleProducts = useMemo(() => {
    const q = query.trim().toLowerCase();

    const filtered = q
      ? products.filter((p) => p.name.toLowerCase().includes(q))
      : products;

    const sorted = [...filtered].sort((a, b) => {
      if (sort === "price") return a.price - b.price;
      return a.name.localeCompare(b.name);
    });

    return sorted;
  }, [products, query, sort]);

  return (
    <section>
      <input value={query} onChange={(e) => setQuery(e.target.value)} placeholder="Rechercher" />
      <select value={sort} onChange={(e) => setSort(e.target.value)}>
        <option value="price">Prix</option>
        <option value="name">Nom</option>
      </select>

      <ul>
        {visibleProducts.map((p) => (
          <li key={p.id}>{p.name} — {p.price}€</li>
        ))}
      </ul>
    </section>
  );
}
```

### 7.4 useMemo vs useEffect
- `useMemo` : calcule une valeur dérivée *pendant* le rendu.
- `useEffect` : exécute un effet *après* le rendu.

Si vous voulez “dériver” des données pour l’affichage, préférez :
- un calcul direct,
- ou `useMemo` si nécessaire,
plutôt que `useEffect + setState`.

### 7.5 Exercice
À partir d’une liste de 10 000 items, implémenter :
- un champ de recherche
- un filtre
- un tri
- optimiser avec `useMemo` (et mesurer en ajoutant un `console.time`).

---

## 8. Atelier fil rouge : mini-app “Dashboard”

### 8.1 But
Construire un mini dashboard qui utilise les 5 hooks :
- `useState` : état UI (onglet actif, recherche)
- `useEffect` : chargement des données et abonnements
- `useContext` : thème et utilisateur
- `useRef` : focus + timers
- `useMemo` : filtrage et tri

### 8.2 Spécifications
- Un `AuthProvider` (mock) fourni `user` et `logout()`.
- Un `ThemeProvider` gère `theme` + `toggleTheme()`.
- Une page `Dashboard` :
  - fetch d’une liste d’items (users/products)
  - recherche + tri
  - un champ search autofocused
  - un message “Dernière recherche” stocké en ref

### 8.3 Structure de fichiers (suggestion)
```
src/
  providers/
    AuthProvider.jsx
    ThemeProvider.jsx
  hooks/
    useAuth.js
    useTheme.js
  pages/
    Dashboard.jsx
  components/
    SearchBar.jsx
    ProductList.jsx
```

### 8.4 Conseils d’implémentation
- Isoler la logique réutilisable (ex : un custom hook pour fetch) si l’exercice est long.
- Ajouter de la gestion d’état `loading/error`.
- Vérifier la propreté des `useEffect` (cleanup, dépendances).

---

## 9. Bonnes pratiques, anti-patterns et checklists

### 9.1 Checklist useState
- [ ] État minimal (éviter la duplication)
- [ ] Mises à jour immuables
- [ ] Forme fonctionnelle si dépendance sur le state précédent

### 9.2 Checklist useEffect
- [ ] Les dépendances sont correctes
- [ ] Les abonnements/timers sont nettoyés (cleanup)
- [ ] Éviter `useEffect` pour dériver l’affichage
- [ ] Attention aux conditions de course (race conditions)

### 9.3 Checklist useContext
- [ ] Le contexte est utilisé pour des données globales stables
- [ ] Éviter un gros contexte fourre-tout
- [ ] Fournir un custom hook `useX()`

### 9.4 Checklist useRef
- [ ] Utilisé pour DOM ou valeurs mutables non-rendues
- [ ] Pas pour “cacher” des erreurs de dépendances d’effets

### 9.5 Checklist useMemo
- [ ] Optimisation justifiée (mesurée)
- [ ] Dépendances correctes
- [ ] Pas d’abus : le code reste lisible

---

## 10. Quiz / Évaluation

1. Pourquoi les hooks ne doivent pas être appelés dans une condition ?
2. Quand utiliser la forme fonctionnelle de `setState` ?
3. Différence entre `useMemo` et `useEffect` ?
4. Quel est le rôle du cleanup dans `useEffect` ?
5. Pourquoi `useRef` ne déclenche pas de re-render ?
6. Quel problème `useContext` résout-il ? Et quel est son risque côté performance ?

---

## 11. Annexes : snippets et mémo

### 11.1 Mémo rapide
- `useState` : état local
- `useEffect` : synchronisation avec le monde extérieur
- `useContext` : accès à des valeurs partagées (Provider)
- `useRef` : valeur mutable persistante / ref DOM
- `useMemo` : mémoïsation d’un calcul

### 11.2 Erreur classique : dépendances manquantes
```jsx
useEffect(() => {
  doSomething(userId);
}, []); // ❌ userId manquant
```

Correct :
```jsx
useEffect(() => {
  doSomething(userId);
}, [userId]);
```

### 11.3 Pattern : custom hook pour contexte
```jsx
export function useTheme() {
  const ctx = useContext(ThemeContext);
  if (!ctx) throw new Error("useTheme must be used within ThemeProvider");
  return ctx;
}
```

---

## Fin de formation

### Prochaines étapes
- Ajouter les hooks complémentaires : `useCallback`, `useReducer`, `useLayoutEffect`.
- Introduire les **Custom Hooks** pour factoriser la logique (fetch, forms, etc.).
- Optimisation : memoisation de composants (`React.memo`) + `useCallback`.
