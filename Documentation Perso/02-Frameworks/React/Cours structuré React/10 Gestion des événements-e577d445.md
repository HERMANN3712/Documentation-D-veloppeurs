# Formation React — 10. Gestion des événements

> **Objectif** : maîtriser la gestion des événements en React (clics, saisies, soumissions…) en utilisant des fonctions JavaScript et des attributs JSX (`onClick`, `onChange`, etc.).

---

## Plan de la formation

1. **Rappels : événements côté navigateur vs React**
2. **Syntaxe JSX : attacher un gestionnaire (handler)**
3. **Passer des paramètres à un handler**
4. **Le SyntheticEvent (événement React) et `event`**
5. **Gestion des événements de formulaire**
   - `onChange` (inputs contrôlés)
   - `onSubmit`
6. **Prévenir les comportements par défaut : `preventDefault`**
7. **Propagation : bubbling, capture, `stopPropagation`**
8. **Patterns pratiques (handlers propres et lisibles)**
   - fonctions nommées
   - handlers inline : quand et pourquoi
   - composition de handlers
9. **Cas d’usage complets (mini-projets)**
10. **Erreurs fréquentes et bonnes pratiques**
11. **Exercices + corrigés**

---

## 1) Rappels : événements côté navigateur vs React

En JavaScript “vanilla”, on attache souvent des événements via :

```js
button.addEventListener('click', () => {
  console.log('click');
});
```

En **React**, on attache les événements **directement dans le JSX** via des **props** dont le nom commence par `on` (camelCase) :

- `onClick`
- `onChange`
- `onSubmit`
- `onKeyDown`
- etc.

React utilise un système d’événements unifié (historique : *SyntheticEvent*), ce qui offre une API cohérente entre navigateurs.

---

## 2) Syntaxe JSX : attacher un gestionnaire (handler)

### 2.1 Le principe

Un événement React attend **une fonction**.

✅ Correct : on passe une référence de fonction

```jsx
function App() {
  function handleClick() {
    console.log('Bouton cliqué');
  }

  return <button onClick={handleClick}>Cliquer</button>;
}
```

❌ Incorrect : on exécute la fonction au rendu

```jsx
<button onClick={handleClick()}>Cliquer</button>
```

Ici `handleClick()` est appelé **immédiatement** lors du rendu, au lieu d’être appelé lors du clic.

### 2.2 Handler inline (fonction anonyme)

```jsx
<button onClick={() => console.log('clic')}>Cliquer</button>
```

C’est très pratique pour des actions simples. Pour des composants plus complexes, on privilégie souvent une fonction nommée (lisibilité, testabilité).

---

## 3) Passer des paramètres à un handler

Souvent, vous voulez passer une information (id, index, valeur…) au handler.

### 3.1 Via une closure

```jsx
function List({ items }) {
  function handleSelect(id) {
    console.log('Sélection :', id);
  }

  return (
    <ul>
      {items.map((item) => (
        <li key={item.id}>
          <button onClick={() => handleSelect(item.id)}>
            Choisir {item.name}
          </button>
        </li>
      ))}
    </ul>
  );
}
```

### 3.2 Paramètres + event

Si vous souhaitez aussi utiliser l’événement, vous pouvez le récupérer dans la fonction inline :

```jsx
<button onClick={(event) => handleSelect(item.id, event)}>
  Choisir {item.name}
</button>
```

---

## 4) Le SyntheticEvent (événement React) et `event`

Un handler reçoit généralement un paramètre `event` :

```jsx
function App() {
  function handleClick(event) {
    console.log(event.type); // "click"
  }

  return <button onClick={handleClick}>Cliquer</button>;
}
```

### 4.1 Propriétés utiles

- `event.type` : type d’événement
- `event.target` : élément qui a déclenché l’événement
- `event.currentTarget` : élément sur lequel le handler est attaché
- `event.preventDefault()` : empêche le comportement par défaut (ex : soumission formulaire)
- `event.stopPropagation()` : stoppe la propagation (bubbling)

### 4.2 `target` vs `currentTarget`

```jsx
function App() {
  function handleClick(event) {
    console.log('target:', event.target);
    console.log('currentTarget:', event.currentTarget);
  }

  return (
    <button onClick={handleClick}>
      <span>Cliquer</span>
    </button>
  );
}
```

- Si on clique sur le `span`, `event.target` sera le `span`.
- `event.currentTarget` restera le `button` (là où le handler est défini).

---

## 5) Gestion des événements de formulaire

Les formulaires sont un cas central : la saisie utilisateur met à jour l’état (state) via `onChange`.

### 5.1 Inputs contrôlés (Controlled Components)

Un input est dit **contrôlé** quand sa valeur vient du state React.

```jsx
import { useState } from 'react';

function Signup() {
  const [email, setEmail] = useState('');

  function handleEmailChange(event) {
    setEmail(event.target.value);
  }

  return (
    <div>
      <label>
        Email
        <input
          type="email"
          value={email}
          onChange={handleEmailChange}
          placeholder="ex: dev@exemple.com"
        />
      </label>

      <p>Valeur courante : {email}</p>
    </div>
  );
}
```

Points clés :

- `value={email}` impose que l’input reflète le state.
- `onChange` met à jour le state à chaque saisie.

### 5.2 Plusieurs champs : gérer via `name`

```jsx
import { useState } from 'react';

function ProfileForm() {
  const [form, setForm] = useState({ firstName: '', lastName: '' });

  function handleChange(event) {
    const { name, value } = event.target;
    setForm((prev) => ({ ...prev, [name]: value }));
  }

  return (
    <form>
      <input
        name="firstName"
        value={form.firstName}
        onChange={handleChange}
        placeholder="Prénom"
      />
      <input
        name="lastName"
        value={form.lastName}
        onChange={handleChange}
        placeholder="Nom"
      />

      <pre>{JSON.stringify(form, null, 2)}</pre>
    </form>
  );
}
```

### 5.3 Checkbox et radio

- Checkbox : on lit souvent `event.target.checked`

```jsx
import { useState } from 'react';

function Terms() {
  const [accepted, setAccepted] = useState(false);

  return (
    <label>
      <input
        type="checkbox"
        checked={accepted}
        onChange={(e) => setAccepted(e.target.checked)}
      />
      J’accepte les conditions
    </label>
  );
}
```

### 5.4 Soumission : `onSubmit`

```jsx
import { useState } from 'react';

function Login() {
  const [email, setEmail] = useState('');

  function handleSubmit(event) {
    event.preventDefault();
    console.log('submit avec', email);
  }

  return (
    <form onSubmit={handleSubmit}>
      <input value={email} onChange={(e) => setEmail(e.target.value)} />
      <button type="submit">Se connecter</button>
    </form>
  );
}
```

---

## 6) Prévenir les comportements par défaut : `preventDefault`

Certains éléments HTML ont des comportements natifs :

- `<form>` : page reload / navigation
- `<a href="...">` : navigation

React ne change pas ces comportements : vous devez les empêcher si nécessaire.

### Exemple avec un lien

```jsx
function App() {
  function handleLinkClick(e) {
    e.preventDefault();
    console.log('Navigation empêchée');
  }

  return (
    <a href="https://example.com" onClick={handleLinkClick}>
      Ouvrir
    </a>
  );
}
```

---

## 7) Propagation : bubbling, capture, `stopPropagation`

Les événements peuvent se **propager** de l’élément le plus interne vers les parents (**bubbling**).

### 7.1 Exemple : bubbling

```jsx
function App() {
  function handleParentClick() {
    console.log('parent');
  }

  function handleChildClick() {
    console.log('child');
  }

  return (
    <div onClick={handleParentClick} style={{ padding: 16, border: '1px solid' }}>
      <button onClick={handleChildClick}>Cliquer</button>
    </div>
  );
}
```

En cliquant sur le bouton, vous verrez :

1. `child`
2. `parent`

### 7.2 Stopper la propagation

```jsx
function App() {
  function handleParentClick() {
    console.log('parent');
  }

  function handleChildClick(e) {
    e.stopPropagation();
    console.log('child');
  }

  return (
    <div onClick={handleParentClick} style={{ padding: 16, border: '1px solid' }}>
      <button onClick={handleChildClick}>Cliquer</button>
    </div>
  );
}
```

Utilisation typique :

- cartes cliquables (parent), avec un bouton “supprimer” (enfant) qui ne doit pas déclencher le clic sur la carte.

### 7.3 Capture phase (optionnel)

React permet l’écoute en phase de capture via `onClickCapture`, `onChangeCapture`, etc.

```jsx
<div onClickCapture={() => console.log('capture parent')} onClick={() => console.log('bubble parent')}>
  <button onClick={() => console.log('button')}>Test</button>
</div>
```

---

## 8) Patterns pratiques (handlers propres et lisibles)

### 8.1 Préférer des fonctions nommées pour les actions non-triviales

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  function increment() {
    setCount((c) => c + 1);
  }

  function decrement() {
    setCount((c) => c - 1);
  }

  return (
    <div>
      <button onClick={decrement}>-</button>
      <span>{count}</span>
      <button onClick={increment}>+</button>
    </div>
  );
}
```

### 8.2 Composition de handlers

Parfois vous voulez exécuter plusieurs actions :

```jsx
function App() {
  function track(eventName) {
    console.log('track', eventName);
  }

  function doAction() {
    console.log('action');
  }

  function handleClick() {
    track('button_clicked');
    doAction();
  }

  return <button onClick={handleClick}>OK</button>;
}
```

### 8.3 Déléguer à un composant enfant via props

```jsx
function Toolbar({ onSave }) {
  return <button onClick={onSave}>Enregistrer</button>;
}

function App() {
  function handleSave() {
    console.log('Sauvegarde…');
  }

  return <Toolbar onSave={handleSave} />;
}
```

---

## 9) Cas d’usage complets (mini-projets)

### 9.1 TodoList avec suppression (stopPropagation)

**Objectifs** :
- gérer `onSubmit` pour ajouter
- gérer `onChange` pour le champ
- gérer `onClick` pour sélectionner un item
- gérer `stopPropagation` pour supprimer sans sélectionner

```jsx
import { useState } from 'react';

export default function TodoApp() {
  const [text, setText] = useState('');
  const [todos, setTodos] = useState([
    { id: 1, label: 'Apprendre React', done: false },
    { id: 2, label: 'Pratiquer les events', done: false },
  ]);
  const [selectedId, setSelectedId] = useState(null);

  function handleSubmit(e) {
    e.preventDefault();
    const label = text.trim();
    if (!label) return;

    setTodos((prev) => prev.concat({ id: Date.now(), label, done: false }));
    setText('');
  }

  function toggleDone(id) {
    setTodos((prev) => prev.map((t) => (t.id === id ? { ...t, done: !t.done } : t)));
  }

  function removeTodo(id) {
    setTodos((prev) => prev.filter((t) => t.id !== id));
    setSelectedId((prev) => (prev === id ? null : prev));
  }

  return (
    <div style={{ maxWidth: 500 }}>
      <h2>Todos</h2>

      <form onSubmit={handleSubmit}>
        <input
          value={text}
          onChange={(e) => setText(e.target.value)}
          placeholder="Ajouter une tâche"
        />
        <button type="submit">Ajouter</button>
      </form>

      <ul style={{ padding: 0, listStyle: 'none' }}>
        {todos.map((t) => (
          <li
            key={t.id}
            onClick={() => setSelectedId(t.id)}
            style={{
              display: 'flex',
              gap: 8,
              padding: 8,
              marginTop: 8,
              border: '1px solid #ddd',
              cursor: 'pointer',
              background: selectedId === t.id ? '#f5f5ff' : 'white',
            }}
          >
            <input
              type="checkbox"
              checked={t.done}
              onChange={() => toggleDone(t.id)}
              onClick={(e) => e.stopPropagation()}
            />

            <span style={{ textDecoration: t.done ? 'line-through' : 'none', flex: 1 }}>
              {t.label}
            </span>

            <button
              onClick={(e) => {
                e.stopPropagation();
                removeTodo(t.id);
              }}
            >
              Supprimer
            </button>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

---

## 10) Erreurs fréquentes et bonnes pratiques

### Erreurs fréquentes

1. **Appeler le handler au lieu de le passer**
   - ❌ `onClick={handleClick()}`
   - ✅ `onClick={handleClick}`

2. **Oublier `preventDefault()` sur un formulaire**
   - Symptôme : la page se recharge.

3. **Mauvais accès à la valeur d’un champ**
   - `event.target.value` pour la majorité des inputs
   - `event.target.checked` pour checkbox

4. **Confondre `target` et `currentTarget`**

### Bonnes pratiques

- Garder les handlers **petits** et **précis** (une responsabilité).
- Pour les updates de state basées sur l’ancien state, utiliser la forme fonctionnelle :

```js
setCount((c) => c + 1);
```

- Extraire les handlers si le JSX devient illisible.
- Utiliser `stopPropagation()` uniquement quand c’est nécessaire (sinon on complique le comportement).

---

## 11) Exercices + corrigés

### Exercice 1 — Bouton compteur

**Énoncé** :
- afficher un compteur
- bouton “+1”
- bouton “Reset”

**Corrigé** :

```jsx
import { useState } from 'react';

export default function CounterEx() {
  const [count, setCount] = useState(0);

  return (
    <div>
      <p>Compteur : {count}</p>
      <button onClick={() => setCount((c) => c + 1)}>+1</button>
      <button onClick={() => setCount(0)}>Reset</button>
    </div>
  );
}
```

### Exercice 2 — Champ contrôlé

**Énoncé** :
- input texte
- afficher en direct la valeur saisie
- interdire les espaces en début/fin au moment du submit

**Corrigé** :

```jsx
import { useState } from 'react';

export default function ControlledInputEx() {
  const [value, setValue] = useState('');
  const [submitted, setSubmitted] = useState('');

  function handleSubmit(e) {
    e.preventDefault();
    setSubmitted(value.trim());
  }

  return (
    <div>
      <form onSubmit={handleSubmit}>
        <input value={value} onChange={(e) => setValue(e.target.value)} />
        <button type="submit">Envoyer</button>
      </form>

      <p>Live : {value}</p>
      <p>Submitted : {submitted}</p>
    </div>
  );
}
```

---

## Résumé

- En React, les événements sont **déclarés dans le JSX** via des attributs comme `onClick`, `onChange`, `onSubmit`.
- On passe **une fonction** en handler (référence), pas le résultat d’un appel.
- `event` donne accès aux infos : `target.value`, `preventDefault()`, `stopPropagation()`.
- Les formulaires se gèrent efficacement avec des **inputs contrôlés**.

---

### Annexes (mémo)

| Besoin | Événement | Accès valeur |
|---|---|---|
| Clic bouton | `onClick` | n/a |
| Champ texte | `onChange` | `e.target.value` |
| Checkbox | `onChange` | `e.target.checked` |
| Formulaire | `onSubmit` | `e.preventDefault()` |
