# Formation React — Props

## Informations
- **Thème** : Les props (properties) en React
- **Public** : développeurs débutants à intermédiaires en React
- **Pré-requis** : JavaScript ES6 (fonctions, objets, destructuring), bases des composants React
- **Objectif général** : maîtriser le passage de données d’un composant parent vers un composant enfant via les **props** afin de **personnaliser** et **réutiliser** les composants.

---

## Plan (structure du cours)
1. Introduction : pourquoi les props ?
2. Définition et règles fondamentales
3. Passer des props : syntaxe JSX et variantes
4. Lire des props dans un composant enfant (function components)
5. Props et enfants : `props.children`
6. Types de props : primitives, objets, tableaux, fonctions
7. Props d’événements : remonter une action vers le parent (callbacks)
8. Props et composition : pattern de réutilisabilité
9. Valeurs par défaut et robustesse (`default values`)
10. Typage et validation (PropTypes / TypeScript)
11. Pièges fréquents et bonnes pratiques
12. Exercices guidés + corrigés

---

# 1) Introduction : pourquoi les props ?

React encourage la construction d’UI en **composants** (petits blocs réutilisables). Pour qu’un même composant serve dans plusieurs contextes, il faut pouvoir :
- lui **donner des données** (ex : un titre, un prix, une URL d’image),
- lui fournir des **options** (ex : variante « primary », taille « small »),
- lui permettre d’exécuter une **action** (ex : au clic sur un bouton).

C’est exactement le rôle des **props** :
> Les *props* sont des **propriétés passées d’un composant parent à un composant enfant**. Elles permettent de **transmettre des données** et de **personnaliser** les composants.

---

# 2) Définition et règles fondamentales

## 2.1 Définition
Les **props** (abréviation de *properties*) sont les paramètres d’entrée d’un composant React.

- Le **parent** fournit les props dans le JSX.
- L’**enfant** reçoit ces props (souvent sous forme d’objet) et les utilise pour rendre l’UI.

## 2.2 Flux de données unidirectionnel
La communication via props se fait **du parent vers l’enfant** (top → down).

- L’enfant **ne modifie pas** les props reçues.
- Si l’enfant a besoin de provoquer un changement, il le fait via une **fonction** passée en prop (callback) et c’est le parent qui met à jour son **state**.

## 2.3 Props = « read-only »
Les props sont **immuables** du point de vue du composant enfant :
- Ne pas faire : `props.title = '...'`
- Si vous devez transformer une valeur, utilisez une variable locale :
  ```js
  const upperTitle = props.title.toUpperCase();
  ```

---

# 3) Passer des props : syntaxe JSX et variantes

## 3.1 Props sous forme d’attributs JSX
Exemple simple :

```jsx
function App() {
  return <Welcome name="Ada" />;
}
```

### Passer une expression JS avec `{}`
```jsx
function App() {
  const user = { name: "Ada", age: 28 };
  return <Welcome name={user.name} age={user.age} />;
}
```

## 3.2 Le « spread » de props
Utile pour transmettre un objet de props :

```jsx
function App() {
  const props = { name: "Ada", age: 28 };
  return <Welcome {...props} />;
}
```

**Attention** : le spread peut masquer l’origine des props si utilisé de manière excessive (à réserver à des cas clairs).

## 3.3 Prop booléenne (shorthand)
```jsx
<Button disabled />
```
Équivalent à :
```jsx
<Button disabled={true} />
```

---

# 4) Lire des props dans un composant enfant

## 4.1 Accès via l’objet `props`

```jsx
function Welcome(props) {
  return <h1>Bonjour {props.name} !</h1>;
}
```

## 4.2 Destructuring des props (recommandé)

```jsx
function Welcome({ name, age }) {
  return (
    <p>
      Bonjour {name}, tu as {age} ans.
    </p>
  );
}
```

### Avantages
- code plus lisible,
- moins de répétitions `props.xxx`,
- plus simple à typer (TypeScript) et à documenter.

---

# 5) Props et enfants : `props.children`

`children` est une prop spéciale qui contient le contenu placé entre les balises du composant.

```jsx
function Card({ title, children }) {
  return (
    <section className="card">
      <h2>{title}</h2>
      <div className="card-body">{children}</div>
    </section>
  );
}

function App() {
  return (
    <Card title="Profil">
      <p>Nom : Ada</p>
      <p>Rôle : Développeuse</p>
    </Card>
  );
}
```

**Concept clé** : `children` permet une **composition** flexible (le parent fournit le contenu, le composant fournit le cadre).

---

# 6) Types de props : primitives, objets, tableaux, fonctions

## 6.1 Props primitives
```jsx
<ProductBadge label="Nouveau" price={19.99} />
```

## 6.2 Props objets
```jsx
function UserCard({ user }) {
  return (
    <div>
      <strong>{user.name}</strong>
      <span> — {user.role}</span>
    </div>
  );
}

<UserCard user={{ name: "Ada", role: "Admin" }} />
```

## 6.3 Props tableaux
```jsx
function TagList({ tags }) {
  return (
    <ul>
      {tags.map((t) => (
        <li key={t}>{t}</li>
      ))}
    </ul>
  );
}

<TagList tags={["react", "props", "ui"]} />
```

## 6.4 Props fonctions
Les fonctions passées en props sont centrales en React (actions, callbacks, handlers).

```jsx
function Button({ onClick, children }) {
  return (
    <button onClick={onClick}>
      {children}
    </button>
  );
}
```

---

# 7) Props d’événements : remonter une action vers le parent

Les props circulent **du parent vers l’enfant**. Pour qu’un enfant déclenche une action côté parent :
- le parent passe une fonction,
- l’enfant appelle cette fonction.

## Exemple : compteur

```jsx
function Counter({ value, onIncrement }) {
  return (
    <div>
      <p>Valeur : {value}</p>
      <button onClick={onIncrement}>+1</button>
    </div>
  );
}

function App() {
  const [count, setCount] = React.useState(0);

  return (
    <Counter
      value={count}
      onIncrement={() => setCount((c) => c + 1)}
    />
  );
}
```

### À retenir
- `Counter` est **présentationnel** : il affiche et déclenche une action.
- `App` est **conteneur** : il porte le state et décide comment évoluer.

---

# 8) Props et composition : patterns de réutilisabilité

## 8.1 Variants (prop `variant`)

```jsx
function Alert({ variant = "info", message }) {
  return <div className={`alert alert-${variant}`}>{message}</div>;
}

<Alert variant="success" message="Sauvegarde OK" />
<Alert variant="error" message="Erreur réseau" />
```

## 8.2 Rendre un composant générique
Un composant est plus réutilisable si :
- il reçoit ses données via props,
- il expose des points d’extension (`children`, `render props`, callbacks).

### Exemple simple : `Layout`
```jsx
function Layout({ header, children, footer }) {
  return (
    <div>
      <header>{header}</header>
      <main>{children}</main>
      <footer>{footer}</footer>
    </div>
  );
}

<Layout
  header={<h1>Dashboard</h1>}
  footer={<small>© 2026</small>}
>
  <p>Contenu principal…</p>
</Layout>
```

---

# 9) Valeurs par défaut et robustesse

## 9.1 Valeur par défaut via destructuring
C’est la méthode la plus courante.

```jsx
function Avatar({ src, alt = "Avatar", size = 40 }) {
  return <img src={src} alt={alt} width={size} height={size} />;
}
```

## 9.2 Props optionnelles
Si une prop est optionnelle, gérez les cas `undefined` :

```jsx
function Greeting({ name }) {
  return <p>Bonjour {name ?? "inconnu"}</p>;
}
```

---

# 10) Typage et validation (PropTypes / TypeScript)

## 10.1 PropTypes (validation runtime)

```bash
npm i prop-types
```

```jsx
import PropTypes from "prop-types";

function Welcome({ name, age }) {
  return <p>{name} — {age}</p>;
}

Welcome.propTypes = {
  name: PropTypes.string.isRequired,
  age: PropTypes.number,
};
```

## 10.2 TypeScript (recommandé en projets modernes)

```tsx
type WelcomeProps = {
  name: string;
  age?: number;
};

export function Welcome({ name, age }: WelcomeProps) {
  return <p>{name} — {age ?? "?"}</p>;
}
```

---

# 11) Pièges fréquents et bonnes pratiques

## 11.1 Ne pas muter un objet reçu en prop
Mauvais :
```js
function Bad({ user }) {
  user.name = "X"; // ❌ mutation
  return null;
}
```
Bon :
```js
function Good({ user }) {
  const safeUser = { ...user, name: "X" }; // ✅ copie
  return <div>{safeUser.name}</div>;
}
```

## 11.2 Attention à la stabilité des fonctions inline
```jsx
<Child onClick={() => doSomething()} />
```
Créer une nouvelle fonction à chaque rendu peut provoquer des rerenders inutiles si `Child` est mémoïsé.

Solution (selon le contexte) : `useCallback`.

```jsx
const onClick = React.useCallback(() => doSomething(), [doSomething]);
<Child onClick={onClick} />
```

## 11.3 Nommer clairement les props
- `onXxx` pour les événements : `onClose`, `onSubmit`
- `is/has/can` pour les booléens : `isOpen`, `hasError`
- éviter `data` trop générique.

## 11.4 Props vs State
- **Props** : données fournies *au composant*.
- **State** : données internes *au composant* (qui changent dans le temps).

Règle pratique : si une donnée vient du parent → **props**. Si elle est gérée localement → **state**.

---

# 12) Exercices guidés (avec corrigés)

## Exercice 1 — Composant `UserBadge`
**Objectif** : créer un composant réutilisable.

### Énoncé
Créer un composant `UserBadge` qui accepte :
- `name` (string),
- `role` (string),
- `isOnline` (boolean).

Affichage attendu (exemple) :
- `Ada (Admin) — En ligne`
- `Linus (User) — Hors ligne`

### Corrigé
```jsx
function UserBadge({ name, role, isOnline }) {
  return (
    <p>
      {name} ({role}) — {isOnline ? "En ligne" : "Hors ligne"}
    </p>
  );
}

function App() {
  return (
    <div>
      <UserBadge name="Ada" role="Admin" isOnline={true} />
      <UserBadge name="Linus" role="User" isOnline={false} />
    </div>
  );
}
```

---

## Exercice 2 — Passer un callback `onDelete`
**Objectif** : remonter une action enfant → parent via props.

### Énoncé
Créer un composant `TodoItem` recevant :
- `label`
- `onDelete`

Le composant affiche le label et un bouton "Supprimer".

### Corrigé
```jsx
function TodoItem({ label, onDelete }) {
  return (
    <div>
      <span>{label}</span>
      <button onClick={onDelete}>Supprimer</button>
    </div>
  );
}

function App() {
  const [todos, setTodos] = React.useState([
    { id: 1, label: "Apprendre les props" },
    { id: 2, label: "Pratiquer les callbacks" },
  ]);

  return (
    <div>
      {todos.map((t) => (
        <TodoItem
          key={t.id}
          label={t.label}
          onDelete={() => setTodos((prev) => prev.filter((x) => x.id !== t.id))}
        />
      ))}
    </div>
  );
}
```

---

## Exercice 3 — `Card` avec `children`
**Objectif** : utiliser `props.children` pour composer.

### Énoncé
Créer un composant `Card` avec :
- `title`
- `children`

Il affiche un titre puis le contenu.

### Corrigé
```jsx
function Card({ title, children }) {
  return (
    <section style={{ border: "1px solid #ddd", padding: 12 }}>
      <h3>{title}</h3>
      <div>{children}</div>
    </section>
  );
}

function App() {
  return (
    <Card title="Informations">
      <p>Les props rendent les composants configurables.</p>
    </Card>
  );
}
```

---

# Synthèse
- Les **props** sont des propriétés passées **du parent vers l’enfant**.
- Elles servent à **transmettre des données**, **personnaliser** des composants, et favoriser la **réutilisation**.
- L’enfant ne modifie pas les props : elles sont **read-only**.
- Pour remonter une action, on passe un **callback** en prop.
- `children` est la clé de la **composition**.

---

# Annexes — Checklist rapide
- [ ] Ai-je un composant réutilisable ? → définir des props explicites.
- [ ] Mes booléens sont nommés `is/has/can`.
- [ ] Mes événements sont nommés `onXxx`.
- [ ] Je ne mute aucune prop (objets/tableaux inclus).
- [ ] J’utilise `children` si je veux laisser le parent injecter du contenu.
