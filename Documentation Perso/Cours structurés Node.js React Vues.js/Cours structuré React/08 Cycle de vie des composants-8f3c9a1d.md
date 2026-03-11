# Formation React — Cycle de vie des composants (fonctionnels) avec `useEffect`

**Public visé :** développeurs débutants à intermédiaires en React

**Prérequis :**
- JavaScript ES6+
- Bases de React (JSX, props, state)

**Objectifs pédagogiques :**
- Comprendre le « cycle de vie » d’un composant fonctionnel
- Maîtriser `useEffect` pour gérer **montage**, **mise à jour** et **démontage**
- Éviter les pièges courants (boucles, dépendances, effets non nettoyés)
- Savoir structurer des effets pour des cas concrets (fetch, subscriptions, timers)

---

## Plan de la formation

1. **Rappels : cycle de vie en React**
   - Qu’est-ce qu’un cycle de vie ?
   - Rendu, re-rendu et démontage
   - Différence classe vs fonctionnel (contexte historique)

2. **Comprendre `useEffect`**
   - Définition d’un effet
   - Quand un effet s’exécute ?
   - Signature et rôle de la fonction de nettoyage (cleanup)

3. **Montage (équivalent `componentDidMount`)**
   - Exécuter un effet une seule fois
   - Cas d’usage : initialisation, fetch initial

4. **Mise à jour (équivalent `componentDidUpdate`)**
   - Exécuter un effet lors d’un changement de dépendances
   - Synchronisation avec des props/state

5. **Démontage (équivalent `componentWillUnmount`)**
   - Nettoyage : abonnements, timers, listeners
   - Pourquoi c’est indispensable (fuites mémoire, comportements imprévisibles)

6. **La liste de dépendances : règles & bonnes pratiques**
   - Comment la déterminer
   - Stale closures (valeurs « figées »)
   - Linter `eslint-plugin-react-hooks`

7. **Cas pratiques guidés**
   - 1 : Fetch de données avec annulation
   - 2 : Timer/interval et cleanup
   - 3 : Abonnement à un événement (window)
   - 4 : Synchroniser un state avec une prop & éviter les effets inutiles

8. **Pièges et anti-patterns**
   - Boucles de rendu / effets
   - Oublier le cleanup
   - Dépendances manquantes
   - Mettre trop de choses dans un seul `useEffect`

9. **Synthèse & exercices**
   - Résumé des patterns
   - Exercices à réaliser

---

## 1) Rappels : cycle de vie en React

### 1.1 Qu’est-ce que le cycle de vie ?

Le **cycle de vie** décrit les différentes étapes d’existence d’un composant :

- **Montage (mount)** : le composant est créé et inséré dans le DOM.
- **Mise à jour (update)** : le composant se re-rend suite à un changement de `state` ou de `props`.
- **Démontage (unmount)** : le composant est retiré du DOM.

Dans React moderne (avec les composants fonctionnels), on gère principalement ces moments via des **hooks** — notamment **`useEffect`**.

### 1.2 Rendu vs effet

- Le **rendu** (render) est une phase où React calcule l’UI à afficher.
- Un **effet** (effect) est du code “impératif” exécuté **après** le rendu, typiquement pour :
  - accéder à des APIs externes
  - lancer des requêtes HTTP
  - manipuler des abonnements
  - déclencher / arrêter des timers

> Idée clé : le rendu décrit **quoi afficher**, l’effet gère **les interactions avec l’extérieur**.

---

## 2) Comprendre `useEffect`

### 2.1 Définition

`useEffect` permet d’exécuter une **fonction d’effet** après que React ait rendu le composant.

```jsx
import { useEffect } from "react";

useEffect(() => {
  // code exécuté après le rendu
}, [/* dépendances */]);
```

### 2.2 Les 2 paramètres importants

1) **La fonction d’effet** :

```jsx
useEffect(() => {
  // side effect
});
```

2) **Le tableau de dépendances** :

```jsx
useEffect(() => {
  // side effect
}, [a, b]);
```

- Si `a` ou `b` change entre deux rendus, l’effet est relancé.

### 2.3 La fonction de cleanup (nettoyage)

Un `useEffect` peut **retourner une fonction**, appelée **cleanup**, qui sera exécutée :

- **avant** de relancer l’effet (lors d’une mise à jour)
- et **au démontage** du composant

```jsx
useEffect(() => {
  console.log("effect start");

  return () => {
    console.log("cleanup");
  };
}, []);
```

---

## 3) Montage : exécuter un effet une seule fois

### 3.1 Pattern « au montage »

Pour exécuter un code **uniquement au montage**, on utilise un tableau de dépendances **vide** :

```jsx
useEffect(() => {
  // S’exécute 1 fois au montage
}, []);
```

### 3.2 Exemple : initialiser un titre de page

```jsx
import { useEffect } from "react";

export function Page() {
  useEffect(() => {
    document.title = "Dashboard";
  }, []);

  return <h1>Dashboard</h1>;
}
```

### 3.3 Exemple : fetch initial (sans gestion d’annulation pour l’instant)

```jsx
import { useEffect, useState } from "react";

export function Users() {
  const [users, setUsers] = useState([]);

  useEffect(() => {
    fetch("https://example.com/api/users")
      .then((r) => r.json())
      .then(setUsers);
  }, []);

  return (
    <ul>
      {users.map((u) => (
        <li key={u.id}>{u.name}</li>
      ))}
    </ul>
  );
}
```

---

## 4) Mise à jour : exécuter un effet quand une dépendance change

### 4.1 Pattern « lors d’un changement »

Quand on fournit des dépendances, l’effet s’exécute :

- après le **premier rendu**
- puis **à chaque fois** qu’une dépendance change

```jsx
useEffect(() => {
  // S’exécute au montage puis à chaque changement de query
}, [query]);
```

### 4.2 Exemple : rechercher quand `query` change

```jsx
import { useEffect, useState } from "react";

export function SearchUsers() {
  const [query, setQuery] = useState("");
  const [users, setUsers] = useState([]);

  useEffect(() => {
    if (!query) {
      setUsers([]);
      return;
    }

    fetch(`https://example.com/api/users?q=${encodeURIComponent(query)}`)
      .then((r) => r.json())
      .then(setUsers);
  }, [query]);

  return (
    <div>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      <ul>
        {users.map((u) => (
          <li key={u.id}>{u.name}</li>
        ))}
      </ul>
    </div>
  );
}
```

### 4.3 Ajouter une dépendance : attention aux boucles

Si l’effet **modifie** une valeur présente dans ses dépendances, vous risquez une **boucle**.

Exemple typique d’erreur :

```jsx
useEffect(() => {
  setCount(count + 1);
}, [count]);
```

- `count` change → effet relancé → `count` change → etc.

---

## 5) Démontage : gérer le nettoyage (cleanup)

### 5.1 Pourquoi nettoyer ?

Sans cleanup, vous pouvez avoir :
- des **fuites mémoire**
- des **handlers** déclenchés alors que le composant n’est plus visible
- des erreurs du type « state update on unmounted component » (selon les versions)

### 5.2 Exemple : timer avec cleanup

```jsx
import { useEffect, useState } from "react";

export function Stopwatch() {
  const [seconds, setSeconds] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      setSeconds((s) => s + 1);
    }, 1000);

    return () => clearInterval(id);
  }, []);

  return <p>{seconds}s</p>;
}
```

### 5.3 Exemple : event listener avec cleanup

```jsx
import { useEffect, useState } from "react";

export function WindowWidth() {
  const [width, setWidth] = useState(window.innerWidth);

  useEffect(() => {
    const onResize = () => setWidth(window.innerWidth);
    window.addEventListener("resize", onResize);

    return () => {
      window.removeEventListener("resize", onResize);
    };
  }, []);

  return <p>Width: {width}</p>;
}
```

---

## 6) Dépendances : règles & bonnes pratiques

### 6.1 Comment choisir les dépendances ?

Règle pratique : le tableau de dépendances doit contenir **toutes les variables utilisées dans l’effet** qui proviennent du composant (props, state, variables de scope).

Exemple :

```jsx
useEffect(() => {
  console.log(userId, token);
}, [userId, token]);
```

### 6.2 Stale closures : comprendre le problème

Une **closure** peut capturer une valeur obsolète.

Exemple :

```jsx
useEffect(() => {
  const id = setInterval(() => {
    console.log(seconds); // peut être "figé" si dépendances incorrectes
  }, 1000);
  return () => clearInterval(id);
}, []);
```

Deux solutions fréquentes :

1) Mettre `seconds` dans les dépendances (mais recrée l’interval)
2) Utiliser la forme fonctionnelle du state (souvent la meilleure)

```jsx
setSeconds((s) => s + 1);
```

### 6.3 Utiliser le linter

Activez `eslint-plugin-react-hooks` :
- `react-hooks/rules-of-hooks`
- `react-hooks/exhaustive-deps`

Ce linter vous aide à détecter :
- hooks appelés au mauvais endroit
- dépendances oubliées

---

## 7) Cas pratiques guidés

### Cas pratique 1 — Fetch avec annulation (AbortController)

Objectif : éviter qu’une requête terminée après un changement (ou un démontage) mette à jour l’état de manière incohérente.

```jsx
import { useEffect, useState } from "react";

export function UserDetails({ userId }) {
  const [user, setUser] = useState(null);
  const [status, setStatus] = useState("idle"); // idle | loading | success | error

  useEffect(() => {
    if (!userId) return;

    const controller = new AbortController();
    setStatus("loading");

    fetch(`https://example.com/api/users/${userId}`, {
      signal: controller.signal,
    })
      .then((r) => {
        if (!r.ok) throw new Error("HTTP error");
        return r.json();
      })
      .then((data) => {
        setUser(data);
        setStatus("success");
      })
      .catch((err) => {
        if (err.name === "AbortError") return; // annulation normale
        setStatus("error");
      });

    return () => {
      controller.abort();
    };
  }, [userId]);

  if (!userId) return <p>Choisissez un utilisateur</p>;
  if (status === "loading") return <p>Chargement…</p>;
  if (status === "error") return <p>Erreur</p>;
  if (!user) return null;

  return (
    <article>
      <h2>{user.name}</h2>
      <p>Email: {user.email}</p>
    </article>
  );
}
```

Points clés :
- L’effet dépend de `userId`
- On annule la requête au cleanup
- On évite les mises à jour d’état après démontage

---

### Cas pratique 2 — Interval dépendant d’une prop

Objectif : relancer un interval si une prop `delay` change.

```jsx
import { useEffect, useState } from "react";

export function AutoCounter({ delay = 1000 }) {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      setCount((c) => c + 1);
    }, delay);

    return () => clearInterval(id);
  }, [delay]);

  return <p>Count: {count}</p>;
}
```

---

### Cas pratique 3 — Abonnement/désabonnement (pub/sub)

Objectif : illustrer un pattern d’abonnement.

```jsx
import { useEffect, useState } from "react";

// Exemple fictif
const chat = {
  subscribe(roomId, cb) {
    // ...
    return () => {
      // unsubscribe
    };
  },
};

export function RoomMessages({ roomId }) {
  const [messages, setMessages] = useState([]);

  useEffect(() => {
    if (!roomId) return;

    const unsubscribe = chat.subscribe(roomId, (msg) => {
      setMessages((prev) => [...prev, msg]);
    });

    return () => unsubscribe();
  }, [roomId]);

  return (
    <ul>
      {messages.map((m, i) => (
        <li key={i}>{m}</li>
      ))}
    </ul>
  );
}
```

---

### Cas pratique 4 — Synchroniser une valeur externe et éviter les effets inutiles

Objectif : mettre à jour le titre selon une valeur sans multiplier les effets.

```jsx
import { useEffect } from "react";

export function DocTitle({ title }) {
  useEffect(() => {
    document.title = title;
  }, [title]);

  return <h1>{title}</h1>;
}
```

---

## 8) Pièges et anti-patterns

### 8.1 Tout mettre dans un seul `useEffect`

Anti-pattern : un effet qui fait tout (fetch + event listeners + timers).

Bonne pratique : **séparer par responsabilité**, par exemple :
- un effet pour le fetch
- un effet pour les listeners
- un effet pour les timers

Cela rend le code :
- plus lisible
- plus simple à déboguer
- plus facile à tester

### 8.2 Dépendances manquantes

Si une variable utilisée dans l’effet n’est pas listée, vous pouvez avoir :
- des valeurs obsolètes
- des bugs difficiles à reproduire

Activez le linter et suivez-le dans la majorité des cas.

### 8.3 Boucles de rendu

Si un effet met à jour un state et que ce state est dépendance de l’effet, vous pouvez créer une boucle.

Solutions :
- vérifier les conditions avant de `setState`
- restructurer la logique
- utiliser `useMemo`, `useCallback` si cela stabilise certaines dépendances (avec parcimonie)

---

## 9) Synthèse & exercices

### 9.1 Mémo des patterns `useEffect`

| Objectif | Dépendances | Cleanup ? |
|---|---:|---:|
| Au montage | `[]` | parfois |
| À chaque rendu | *(sans tableau)* | rarement |
| Quand X change | `[x]` | souvent |
| Au démontage | via `return () => ...` | oui |

> Rappel : le cleanup s’exécute aussi **avant** un effet relancé.

### 9.2 Exercices

1) **Formulaire + autosave**
   - `useEffect` déclenché quand `formState` change
   - ajouter un *debounce* (ex: `setTimeout`) et cleanup (`clearTimeout`)

2) **Composant `OnlineStatus`**
   - écouter `window.addEventListener('online')` / `offline`
   - mettre à jour un state et nettoyer les listeners

3) **Recherche avec annulation**
   - champ `query`
   - lancer un fetch à chaque changement
   - annuler la requête précédente

### 9.3 Conclusion

Dans les composants fonctionnels, le cycle de vie n’est pas exposé via des méthodes comme en composants de classe. Il est **modélisé** par l’usage d’`useEffect` (et d’autres hooks), qui permet de :

- exécuter du code au **montage**
- réagir aux **mises à jour** (dépendances)
- nettoyer au **démontage**

La maîtrise du tableau de dépendances et du cleanup est la clé pour des composants robustes.
