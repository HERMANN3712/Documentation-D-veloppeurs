# Formation React — Gestion de l’état avec `useState`

> **Objectif** : maîtriser la gestion de l’état local dans des composants fonctionnels React grâce au hook **`useState`**. 

---

## 1) Pré-requis

- Bases de JavaScript (ES6+) : fonctions fléchées, destructuring, modules.
- Connaissance minimale de React : composants, props, JSX.
- Environnement : Node.js + projet React (Vite, CRA, Next.js…).

---

## 2) Contexte : qu’est-ce que “l’état” en React ?

### 2.1 Définition

- **Props** : données *entrantes* passées à un composant ; immuables côté composant.
- **State (état)** : données *internes* au composant ; peuvent changer au cours du temps.

L’état sert à représenter une information qui évolue : compteur, valeur d’un champ, liste, statut de chargement, onglet actif, etc.

### 2.2 Effet principal

Quand une valeur d’état change, **React re-rend** le composant (et ses descendants) afin de **mettre à jour l’interface**.

> ⚠️ Modifier une variable locale (ex: `let count = 0`) **ne met pas à jour l’UI** : React ne “voit” pas ce changement. Pour déclencher un rendu, on utilise l’état.

---

## 3) Le hook `useState` : rôle et signature

### 3.1 Rôle

Le hook **`useState`** permet de :

- Stocker une valeur d’état **locale** à un composant fonctionnel.
- Obtenir une fonction pour **mettre à jour** cette valeur.
- Déclencher automatiquement un **re-render** du composant quand l’état change.

### 3.2 Signature

```js
const [state, setState] = useState(initialValue);
```

- `state` : la valeur actuelle.
- `setState` : la fonction de mise à jour.
- `initialValue` : valeur initiale lors du premier rendu.

### 3.3 Import

```js
import { useState } from "react";
```

---

## 4) Premier exemple : compteur

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

### Ce qu’il faut retenir

- **`setCount`** déclenche un nouveau rendu.
- On n’écrit pas `count = count + 1`.
- L’UI reflète l’état : `{count}`.

---

## 5) Comprendre le re-render : le “cycle” simplifié

Quand on appelle `setCount` :

1. React enregistre la **nouvelle valeur d’état**.
2. React relance le rendu du composant.
3. Le JSX est recalculé.
4. Le DOM est mis à jour uniquement de ce qui a changé.

> **Important** : un re-render n’est pas forcément un “repaint complet” du DOM. React met à jour efficacement.

---

## 6) Mise à jour basée sur la valeur précédente (functional update)

### 6.1 Problème courant

Dans certains cas (événements rapides, batch updates), utiliser directement `count + 1` peut produire des résultats inattendus.

### 6.2 Solution : forme fonctionnelle

```jsx
setCount((prev) => prev + 1);
```

Exemple avec incrément multiple :

```jsx
<button
  onClick={() => {
    setCount((c) => c + 1);
    setCount((c) => c + 1);
    setCount((c) => c + 1);
  }}
>
  +3
</button>
```

Ici, le résultat est bien `+3`.

---

## 7) `useState` avec des objets : bonnes pratiques

### 7.1 Ne pas muter l’objet

❌ À éviter :

```js
user.name = "Alice";
setUser(user);
```

Même si vous appelez `setUser`, vous réutilisez la même référence : cela peut mener à des comportements confus.

### 7.2 Bon réflexe : créer une nouvelle référence

```jsx
import { useState } from "react";

export function ProfileForm() {
  const [user, setUser] = useState({ name: "", email: "" });

  return (
    <form>
      <label>
        Nom
        <input
          value={user.name}
          onChange={(e) => setUser({ ...user, name: e.target.value })}
        />
      </label>

      <label>
        Email
        <input
          value={user.email}
          onChange={(e) => setUser({ ...user, email: e.target.value })}
        />
      </label>

      <pre>{JSON.stringify(user, null, 2)}</pre>
    </form>
  );
}
```

### 7.3 Variante robuste : mise à jour fonctionnelle

```js
setUser((prev) => ({ ...prev, name: newName }));
```

---

## 8) `useState` avec des tableaux

### 8.1 Ajouter un item

```jsx
const [items, setItems] = useState([]);

function addItem(label) {
  setItems((prev) => [...prev, { id: crypto.randomUUID(), label }]);
}
```

### 8.2 Supprimer un item

```js
setItems((prev) => prev.filter((it) => it.id !== idToRemove));
```

### 8.3 Mettre à jour un item

```js
setItems((prev) =>
  prev.map((it) => (it.id === id ? { ...it, label: newLabel } : it))
);
```

---

## 9) Initialisation “paresseuse” (lazy initialization)

Si l’état initial est coûteux à calculer, passez une fonction à `useState`. Elle ne sera exécutée **qu’au premier rendu**.

```jsx
function expensiveInit() {
  // calcul lourd
  return { value: 42 };
}

const [data, setData] = useState(() => expensiveInit());
```

---

## 10) Formulaires : inputs contrôlés (controlled components)

### 10.1 Un champ simple

```jsx
import { useState } from "react";

export function Newsletter() {
  const [email, setEmail] = useState("");

  function handleSubmit(e) {
    e.preventDefault();
    // envoyer email...
    console.log("submit", email);
  }

  return (
    <form onSubmit={handleSubmit}>
      <input
        type="email"
        value={email}
        onChange={(e) => setEmail(e.target.value)}
        placeholder="Votre email"
      />
      <button type="submit">S’inscrire</button>
    </form>
  );
}
```

**Pourquoi c’est utile ?**

- L’UI dépend de l’état.
- Validation et transformations faciles.
- État unique de vérité : *source of truth*.

### 10.2 Plusieurs champs : un seul state objet

```jsx
const [form, setForm] = useState({ email: "", password: "" });

<input
  value={form.email}
  onChange={(e) => setForm((p) => ({ ...p, email: e.target.value }))}
/>

<input
  type="password"
  value={form.password}
  onChange={(e) => setForm((p) => ({ ...p, password: e.target.value }))}
/>
```

---

## 11) Pièges classiques et règles d’or

### 11.1 Ne pas dériver de state inutilement

Si une valeur peut être calculée à partir de props/state existant, calculez-la à l’affichage.

```jsx
const fullName = `${user.firstName} ${user.lastName}`;
```

Évitez :

```js
const [fullName, setFullName] = useState(""); // souvent inutile
```

### 11.2 `setState` est asynchrone “logiquement”

`setState` planifie une mise à jour. Ne comptez pas sur l’état mis à jour immédiatement après l’appel.

```js
setCount(count + 1);
console.log(count); // log l’ancienne valeur dans le même tick
```

### 11.3 Éviter les mutations

- objets/arrays : utiliser spread, `map`, `filter`, `concat`.
- ne modifiez pas directement l’état.

### 11.4 Hooks : règles

- Appeler les hooks **au niveau racine** du composant, jamais dans une boucle/condition.
- Toujours dans un composant React ou un hook custom.

---

## 12) Mini atelier (exercices)

### Exercice 1 — Compteur avancé

Créer un compteur avec :

- `+1`, `-1`, `Reset`
- un bouton `+5` (utiliser les **functional updates**)
- affichage conditionnel : si `count < 0`, afficher “Valeur négative”.

### Exercice 2 — Todo list

- Champ texte + bouton Ajouter.
- Afficher la liste.
- Supprimer un item.
- Bonus : marquer comme fait (toggle boolean) en respectant l’immutabilité.

### Exercice 3 — Formulaire d’inscription

- Champs : email, password, confirmPassword.
- Désactiver le bouton si password ≠ confirm.

---

## 13) Résumé

- `useState` gère l’état local des composants fonctionnels.
- Modifier l’état via le setter déclenche un **re-render** et React met à jour l’interface.
- Utiliser la **forme fonctionnelle** quand la nouvelle valeur dépend de l’ancienne.
- Respecter l’**immutabilité** pour objets et tableaux.

---

## 14) Annexes — Checklist rapide

- [ ] Ai-je besoin d’un state ou est-ce calculable ?
- [ ] Est-ce que je mute un objet/array ? (si oui, corriger)
- [ ] Ma mise à jour dépend-elle du state précédent ? (si oui, utiliser `prev => ...`)
- [ ] Mes hooks sont-ils au niveau racine du composant ?

---

*Fin de la formation.*
