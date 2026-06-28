# Formation React — JSX

> **Résumé** : JSX est une extension syntaxique de JavaScript qui permet d’écrire du pseudo-HTML directement dans le code JavaScript. **JSX est ensuite compilé** (par Babel/TypeScript) **en appels JavaScript** (ex. `React.createElement`) utilisés par React.

---

## 1) Objectifs pédagogiques

À la fin de cette formation, vous serez capable de :

- Expliquer **ce qu’est JSX** et pourquoi React l’utilise.
- Lire et écrire du JSX correct (syntaxe, expressions, conditions, listes).
- Comprendre comment JSX est **transformé** en JavaScript.
- Éviter les erreurs courantes (attributs, `className`, `htmlFor`, clés, fragments, etc.).
- Structurer des composants React à l’aide de JSX de manière lisible et maintenable.

---

## 2) Prérequis

- Bases de JavaScript (variables, fonctions, objets, tableaux).
- Notions de DOM/HTML/CSS.
- Connaissance minimale de React (composant fonctionnel, props) recommandée.

---

## 3) Public cible

- Développeurs débutants à intermédiaires en React.
- Formateurs/mentors souhaitant formaliser un cours sur JSX.

---

## 4) Durée & format

- **Durée conseillée** : 2h30 à 4h (selon la profondeur et les exercices).
- Alternance : explications → démonstrations → exercices → correction.

---

## 5) Plan de la formation

1. **Introduction à JSX**
2. **Syntaxe de base** : balises, imbrication, un seul parent
3. **JSX = JavaScript + expressions**
4. **Attributs et différences vs HTML**
5. **Style en JSX** (inline style, classes)
6. **Conditions** (ternaires, `&&`, fonctions)
7. **Listes et rendu itératif** (`map`, `key`)
8. **Fragments, commentaires, composants**
9. **Événements en JSX**
10. **Sous le capot** : compilation, `React.createElement`, runtime JSX
11. **Bonnes pratiques & pièges fréquents**
12. **Exercices de synthèse**

---

# 1. Introduction à JSX

## 1.1 Qu’est-ce que JSX ?

**JSX** (JavaScript XML) est une **extension syntaxique** qui permet d’écrire des balises ressemblant à du HTML **au sein du code JavaScript**.

Exemple :

```jsx
const element = <h1>Bonjour JSX</h1>;
```

Important :

- JSX **n’est pas** du HTML.
- JSX **n’est pas** exécuté tel quel par le navigateur.
- JSX est **transformé** (compilé) en JavaScript standard.

## 1.2 Pourquoi JSX ?

JSX offre :

- **Lisibilité** : proche du rendu final.
- **Composition** : facile d’imbriquer des composants.
- **Puissance** : tout ce qui est JavaScript reste disponible (conditions, variables, fonctions).

---

# 2. Syntaxe de base

## 2.1 Balises et imbrication

```jsx
function App() {
  return (
    <section>
      <h1>Titre</h1>
      <p>Un paragraphe</p>
    </section>
  );
}
```

## 2.2 Un seul élément parent

Un `return` JSX doit renvoyer **un seul nœud racine**.

❌ Incorrect :

```jsx
return (
  <h1>Titre</h1>
  <p>Paragraphe</p>
);
```

✅ Correct (avec un wrapper) :

```jsx
return (
  <div>
    <h1>Titre</h1>
    <p>Paragraphe</p>
  </div>
);
```

✅ Correct (avec un fragment) :

```jsx
return (
  <>
    <h1>Titre</h1>
    <p>Paragraphe</p>
  </>
);
```

## 2.3 Balises auto-fermantes

En JSX, une balise sans contenu doit être **auto-fermante**.

✅

```jsx
<img src="/logo.png" alt="Logo" />
<input type="text" />
```

---

# 3. JSX = JavaScript + expressions

## 3.1 Injecter une expression avec `{}`

Dans JSX, on insère du JavaScript avec des **accolades** : `{ expression }`.

```jsx
const name = "Ada";

function Hello() {
  return <p>Bonjour {name}</p>;
}
```

Ce qu’on peut mettre dans `{}` :

- variables
- appels de fonctions
- opérations (ex. `a + b`)
- ternaires (`cond ? A : B`)
- `map(...)`

Ce qu’on ne met pas directement :

- instructions (`if`, `for`, `while`) directement dans `{}` (voir section conditions/lists).

## 3.2 JSX et prévention des injections

Par défaut, React **échappe** les chaînes de caractères rendues dans JSX.

```jsx
const unsafe = "<script>alert('xss')</script>";
return <div>{unsafe}</div>; // affiché en texte, pas exécuté
```

Pour injecter du HTML (à éviter), on utilise `dangerouslySetInnerHTML`.

---

# 4. Attributs : différences JSX vs HTML

## 4.1 Noms d’attributs en camelCase

JSX utilise des noms d’attributs proches du DOM.

- `class` → `className`
- `for` → `htmlFor`
- `tabindex` → `tabIndex`

Exemple :

```jsx
<label htmlFor="email" className="label">Email</label>
<input id="email" tabIndex={0} />
```

## 4.2 Types des attributs

- Chaînes : `"..."`
- Expressions : `{...}`

```jsx
const disabled = true;
<button disabled={disabled}>Valider</button>
```

## 4.3 Booléens

En JSX :

```jsx
<input required />        // équivaut à required={true}
<input required={false} />
```

---

# 5. Style en JSX

## 5.1 `className`

```jsx
<div className="card card--primary">...</div>
```

## 5.2 Style inline via objet

En JSX, `style` prend **un objet JavaScript** (pas une string CSS).

```jsx
const boxStyle = {
  backgroundColor: "black",
  color: "white",
  padding: 12,
};

function Box() {
  return <div style={boxStyle}>Box</div>;
}
```

Remarques :

- propriétés en **camelCase** (`background-color` → `backgroundColor`)
- valeurs numériques = pixels par défaut (selon propriété)

---

# 6. Rendu conditionnel en JSX

## 6.1 Opérateur ternaire

```jsx
function Status({ online }) {
  return <p>{online ? "En ligne" : "Hors ligne"}</p>;
}
```

## 6.2 `&&` (rendu si vrai)

```jsx
function AdminBadge({ isAdmin }) {
  return <div>{isAdmin && <span>Admin</span>}</div>;
}
```

Piège : si `isAdmin` peut être `0`, `""`, etc., React peut afficher cette valeur. Préférez une condition explicite.

## 6.3 `if` avant le `return`

```jsx
function Profile({ user }) {
  if (!user) {
    return <p>Chargement...</p>;
  }
  return <p>Bienvenue {user.name}</p>;
}
```

---

# 7. Listes et rendu itératif

## 7.1 `map`

```jsx
function TodoList({ todos }) {
  return (
    <ul>
      {todos.map((t) => (
        <li key={t.id}>{t.label}</li>
      ))}
    </ul>
  );
}
```

## 7.2 La prop `key`

- `key` aide React à **identifier** les éléments d’une liste.
- Elle doit être **stable** et **unique** dans la liste.

✅ Recommandé : un id métier.

❌ À éviter : l’index du tableau (surtout si la liste peut changer : insertion/suppression/tri).

---

# 8. Fragments, commentaires, composants

## 8.1 Fragments

Réduisent les wrappers inutiles.

```jsx
function TableRow() {
  return (
    <>
      <td>A</td>
      <td>B</td>
    </>
  );
}
```

Fragment avec `key` (utile dans des listes) :

```jsx
import { Fragment } from "react";

items.map((item) => (
  <Fragment key={item.id}>
    <dt>{item.name}</dt>
    <dd>{item.value}</dd>
  </Fragment>
));
```

## 8.2 Commentaires en JSX

```jsx
return (
  <div>
    {/* Ceci est un commentaire JSX */}
    <p>Texte</p>
  </div>
);
```

## 8.3 Composants vs balises HTML

- Balises **minuscule** : éléments HTML (`div`, `span`).
- Balises **Majuscule** : composants React (`MyButton`).

```jsx
function MyButton({ label }) {
  return <button>{label}</button>;
}

function App() {
  return <MyButton label="Clique" />;
}
```

---

# 9. Événements en JSX

## 9.1 Convention de nommage

- `onclick` (HTML) devient `onClick` en JSX.
- On passe une **fonction**, pas une string.

```jsx
function Counter() {
  const handleClick = () => {
    console.log("clicked");
  };

  return <button onClick={handleClick}>OK</button>;
}
```

## 9.2 Passer des paramètres

```jsx
<button onClick={() => doSomething(42)}>Action</button>
```

---

# 10. Sous le capot : compilation de JSX

## 10.1 Transformation classique : `React.createElement`

JSX :

```jsx
const el = <h1 className="title">Hello</h1>;
```

Devient (conceptuellement) :

```js
const el = React.createElement(
  "h1",
  { className: "title" },
  "Hello"
);
```

## 10.2 Runtime JSX moderne (React 17+)

Selon la configuration, JSX peut être transformé vers le runtime automatique (`jsx`, `jsxs`) sans importer React explicitement.

Idée clé : **JSX est toujours compilé**, jamais interprété tel quel.

---

# 11. Bonnes pratiques & pièges fréquents

## 11.1 Lisibilité

- Extraire des sous-composants plutôt que des JSX immenses.
- Nommer les variables de rendu (`const content = ...`).

## 11.2 Ne pas appeler une fonction pendant le rendu si elle a des effets

Éviter :

```jsx
return <div>{sideEffect()}</div>;
```

Le rendu doit idéalement rester **pur** (sans side effects).

## 11.3 Attention aux objets/arrays recréés

Créer des objets inline peut provoquer des rerenders plus fréquents (selon mémoïsation).

```jsx
// parfois OK, mais à connaître
<div style={{ padding: 12 }} />
```

## 11.4 Médaille d’or des erreurs : `key`

- `key` manquante → warning
- `key` instable → bugs d’UI (saisie qui saute, mauvais item mis à jour, etc.)

---

# 12. Exercices (avec corrigés)

## Exercice 1 — Corriger du JSX invalide

### Énoncé
Corriger ce composant :

```jsx
function Card() {
  return (
    <h2>Titre</h2>
    <p class="desc">Description</p>
    <img src="/a.png">
  );
}
```

### Corrigé

```jsx
function Card() {
  return (
    <div>
      <h2>Titre</h2>
      <p className="desc">Description</p>
      <img src="/a.png" alt="" />
    </div>
  );
}
```

---

## Exercice 2 — Affichage conditionnel

### Énoncé
Afficher “Connecté” si `user` existe, sinon “Visiteur”.

### Corrigé

```jsx
function Header({ user }) {
  return <p>{user ? "Connecté" : "Visiteur"}</p>;
}
```

---

## Exercice 3 — Rendre une liste avec `key`

### Énoncé
Rendre une liste de produits :

```js
const products = [
  { id: "p1", name: "Clavier" },
  { id: "p2", name: "Souris" }
];
```

### Corrigé

```jsx
function ProductList({ products }) {
  return (
    <ul>
      {products.map((p) => (
        <li key={p.id}>{p.name}</li>
      ))}
    </ul>
  );
}
```

---

## Exercice 4 — Props et composition

### Énoncé
Créer un composant `Alert` qui prend `type` ("success" | "error") et `children`, et applique une classe CSS.

### Corrigé

```jsx
function Alert({ type, children }) {
  return <div className={`alert alert--${type}`}>{children}</div>;
}

function App() {
  return (
    <>
      <Alert type="success">Opération réussie</Alert>
      <Alert type="error">Une erreur est survenue</Alert>
    </>
  );
}
```

---

# Annexes

## A) Mémo rapide (cheat sheet JSX)

- Un seul parent au `return`.
- Balises sans contenu : `/>`.
- JS dans JSX : `{expression}`.
- Attributs : `className`, `htmlFor`, camelCase.
- Listes : `.map()` + `key` stable.
- Conditions : `? :`, `&&`, ou `if` avant `return`.
- Commentaires : `{/* ... */}`

## B) Message clé à retenir

**JSX est une syntaxe pratique**, mais ce n’est qu’une étape : **tout devient du JavaScript** après compilation.