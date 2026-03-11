# Formation React — 05 Composants React

## Objectifs pédagogiques
À la fin de cette formation, vous serez capable de :

- Expliquer ce qu’est un composant React et son rôle dans une application.
- Créer des composants **fonctionnels** et comprendre l’historique des **composants de classe**.
- Passer des **props**, gérer le **children**, et structurer l’UI en éléments réutilisables.
- Composer des composants (composition), factoriser l’UI et éviter la duplication.
- Comprendre le **cycle de rendu** (render), les notions de **pureté** et d’**idempotence**.
- Appliquer de bonnes pratiques d’organisation (naming, structure de fichiers, responsabilités).

## Pré-requis
- JavaScript (ES6+) : fonctions, objets, destructuring, modules.
- Bases de React : projet initialisé (Vite/CRA/Next) et outil de build fonctionnel.
- Connaissances HTML/CSS.

## Durée suggérée
- 2h à 3h (cours + démonstrations + exercices)

## Public cible
- Développeurs et apprenants souhaitant maîtriser la création et l’usage des composants React.

---

# 1. Introduction — Qu’est-ce qu’un composant React ?

Les **composants** sont les **blocs de base** d’une application React. Une UI est découpée en petites pièces autonomes et **réutilisables**.

Un composant React :

- Représente une partie de l’interface (ex. `Header`, `Button`, `ProductCard`).
- Reçoit éventuellement des **données en entrée** (les **props**).
- Retourne une description de l’UI (via **JSX**).

> Idée clé : on pense l’UI comme un arbre de composants.

### 1.1. Pourquoi des composants ?
- **Réutilisabilité** : un bouton ou une carte produit peut servir partout.
- **Lisibilité** : découper l’UI clarifie l’organisation.
- **Testabilité** : une partie isolée est plus simple à tester.
- **Maintenance** : modifier un composant met à jour tous ses usages.

---

# 2. Les deux formes historiques : fonctions et classes

React supporte deux façons de définir un composant :

1. **Composants fonctionnels** (recommandé aujourd’hui)
2. **Composants de classe** (historique, encore présent dans des codebases)

## 2.1. Composant fonctionnel (moderne)

Un composant fonctionnel est une fonction JavaScript qui **retourne du JSX**.

```jsx
function Hello() {
  return <h1>Bonjour React</h1>;
}
```

### 2.1.1. Avec une fonction fléchée

```jsx
const Hello = () => {
  return <h1>Bonjour React</h1>;
};
```

### 2.1.2. Règles essentielles
- Le nom du composant commence par une **majuscule** (`Hello`, `UserCard`).
- Le composant doit retourner :
  - du JSX valide, ou
  - `null` pour ne rien afficher.

## 2.2. Composant de classe (ancien)

Avant les Hooks, les classes étaient la méthode principale pour gérer l’état et le cycle de vie.

```jsx
import React from "react";

class Hello extends React.Component {
  render() {
    return <h1>Bonjour React</h1>;
  }
}
```

### 2.2.1. Pourquoi encore en parler ?
- Vous pouvez en rencontrer dans des projets existants.
- Comprendre la différence facilite la maintenance et les migrations.

> Aujourd’hui, on privilégie les **composants fonctionnels** + **Hooks**.

---

# 3. JSX : le langage des composants

JSX est une syntaxe qui ressemble à HTML mais qui est en réalité du JavaScript.

```jsx
function Title() {
  const name = "Ada";
  return <h1>Hello {name}</h1>;
}
```

## 3.1. Expressions et logique

- On injecte une expression JS avec `{ ... }`.
- Pour une condition :

```jsx
function Badge({ isAdmin }) {
  return (
    <div>
      {isAdmin ? <span>Admin</span> : <span>Utilisateur</span>}
    </div>
  );
}
```

## 3.2. Lists et clés (clé = identity)

```jsx
function UserList({ users }) {
  return (
    <ul>
      {users.map((u) => (
        <li key={u.id}>{u.name}</li>
      ))}
    </ul>
  );
}
```

- `key` doit être **stable** et **unique** dans la liste.
- Éviter `index` comme clé si la liste peut être réordonnée/supprimée.

---

# 4. Props : données en entrée

Les **props** sont l’équivalent d’arguments de fonction : elles alimentent le composant.

```jsx
function Welcome({ name }) {
  return <p>Bienvenue {name}</p>;
}

// Usage
<Welcome name="Grace" />
```

## 4.1. Props en lecture seule

Un composant ne doit **pas modifier** ses props.

✅ Bon :

```jsx
function Price({ value }) {
  const formatted = value.toFixed(2);
  return <span>{formatted} €</span>;
}
```

❌ Mauvais :

```jsx
function Price({ value }) {
  value = value + 1; // ne pas faire
  return <span>{value}</span>;
}
```

## 4.2. Valeurs par défaut

### 4.2.1. Destructuring avec défaut

```jsx
function Button({ variant = "primary", label }) {
  return <button className={`btn btn--${variant}`}>{label}</button>;
}
```

### 4.2.2. Paramètre `props` classique

```jsx
function Button(props) {
  const { variant = "primary", label } = props;
  return <button className={`btn btn--${variant}`}>{label}</button>;
}
```

## 4.3. Passer des props dynamiques

```jsx
const user = { name: "Linus", age: 54 };

<UserCard name={user.name} age={user.age} />
```

### 4.3.1. Spread props

```jsx
<UserCard {...user} />
```

> À utiliser avec prudence : la lecture du code peut devenir moins explicite.

---

# 5. `children` : composition naturelle

`children` représente le contenu placé entre les balises du composant.

```jsx
function Card({ title, children }) {
  return (
    <section className="card">
      <h2>{title}</h2>
      <div className="card__body">{children}</div>
    </section>
  );
}

// Usage
<Card title="Profil">
  <p>Contenu libre ici.</p>
</Card>
```

## 5.1. Pourquoi c’est si important ?
- Favorise une approche **composition > héritage**.
- Permet de créer des composants « conteneurs » (layout, sections, modales).

---

# 6. Composition de composants (penser en arbre)

React encourage la construction d’une UI en combinant des composants simples.

Exemple :

```jsx
function Header() {
  return (
    <header>
      <Logo />
      <Nav />
      <UserMenu />
    </header>
  );
}
```

## 6.1. Identifier les composants
Méthode pragmatique :
- Repérer les blocs visuels répétitifs.
- Repérer les zones avec responsabilité claire (navigation, carte, formulaire).
- Refactoriser quand un fichier devient trop long ou trop complexe.

---

# 7. Responsabilités : présentation vs logique

On distingue souvent :

- **Composants de présentation** : affichage (UI), reçoivent des props.
- **Composants conteneurs** : orchestrent la logique (données, appels API), puis passent aux composants UI.

Exemple simplifié :

```jsx
// Présentation
function UserView({ user }) {
  return (
    <div>
      <h3>{user.name}</h3>
      <p>{user.email}</p>
    </div>
  );
}

// Conteneur (exemple illustrative)
function UserContainer() {
  const user = { name: "Ada", email: "ada@exemple.com" };
  return <UserView user={user} />;
}
```

> Dans un vrai projet, le conteneur peut utiliser des Hooks (`useEffect`, `useState`) et/ou un gestionnaire d’état.

---

# 8. Rendu, pureté et pièges courants

## 8.1. Un composant doit rester (idéalement) pur
- Pour des mêmes props (et état), il doit produire le même rendu.
- Évitez les effets de bord dans le rendu.

✅ Bon :

```jsx
function Greeting({ name }) {
  return <p>Bonjour {name}</p>;
}
```

❌ Mauvais (effet de bord dans le rendu) :

```jsx
function Greeting({ name }) {
  console.log("render"); // tolérable en dev mais pas une logique métier
  localStorage.setItem("lastName", name); // à éviter dans le rendu
  return <p>Bonjour {name}</p>;
}
```

Les effets de bord doivent être placés dans des Hooks (ex. `useEffect`) ou des handlers d’événements.

## 8.2. Ne pas déclencher de setState dans le rendu
Même si on ne couvre pas l’état en détail ici :
- Toute mise à jour d’état dans le rendu crée des boucles de rendu.

---

# 9. Organisation du code : conventions utiles

## 9.1. Structure de fichiers (exemple)

```
src/
  components/
    Button/
      Button.jsx
      Button.css
      index.js
    Card/
      Card.jsx
  pages/
    Home/
      Home.jsx
  App.jsx
```

- Un composant = un dossier quand il grossit.
- Exposer via `index.js` pour simplifier les imports.

## 9.2. Nommage
- `PascalCase` pour composants et fichiers (souvent).
- `camelCase` pour fonctions utilitaires.

## 9.3. Choisir la bonne granularité
- Trop gros : composant illisible.
- Trop petit : fragmentation et difficultés de navigation.

Règle simple : si un bloc est réutilisé **ou** a une responsabilité claire, il mérite un composant.

---

# 10. Exemples complets

## 10.1. Exemple : `ProductCard`

```jsx
function ProductCard({ title, price, onAdd }) {
  return (
    <article className="product">
      <h3>{title}</h3>
      <p>{price.toFixed(2)} €</p>
      <button onClick={onAdd}>Ajouter</button>
    </article>
  );
}

// Usage
function ProductsPage() {
  return (
    <div>
      <ProductCard
        title="Clavier mécanique"
        price={129.99}
        onAdd={() => console.log("added")}
      />
    </div>
  );
}
```

Points clés :
- La carte est réutilisable : le parent passe la donnée et l’action.
- Le composant enfant n’a pas besoin de connaître le panier.

## 10.2. Exemple : Layout avec `children`

```jsx
function PageLayout({ children }) {
  return (
    <div className="layout">
      <Header />
      <main className="layout__main">{children}</main>
      <Footer />
    </div>
  );
}

function Home() {
  return (
    <PageLayout>
      <h1>Accueil</h1>
      <p>Bienvenue sur le site.</p>
    </PageLayout>
  );
}
```

---

# 11. Exercices (avec corrigés)

## Exercice 1 — Créer un composant `Alert`

**Énoncé :**
Créer un composant `Alert` réutilisable.
- Props : `type` ("success" | "error" | "info"), `title`, `children`
- Affichage : un bloc avec un titre et un contenu

**Exemple d’usage :**

```jsx
<Alert type="error" title="Erreur">
  Impossible de sauvegarder.
</Alert>
```

### Corrigé

```jsx
function Alert({ type = "info", title, children }) {
  return (
    <div className={`alert alert--${type}`} role="alert">
      <strong className="alert__title">{title}</strong>
      <div className="alert__content">{children}</div>
    </div>
  );
}
```

## Exercice 2 — Liste avec `key`

**Énoncé :**
Créer un composant `TodoList` qui affiche des todos :
- props : `items` = `{ id, label, done }[]`
- afficher une liste avec un style différent si `done`.

### Corrigé

```jsx
function TodoList({ items }) {
  return (
    <ul>
      {items.map((item) => (
        <li key={item.id} style={{ textDecoration: item.done ? "line-through" : "none" }}>
          {item.label}
        </li>
      ))}
    </ul>
  );
}
```

## Exercice 3 — Refactoriser un composant trop gros

**Énoncé :**
À partir d’une page incluant en dur un header, une navigation et un footer, extraire :
- `Header`
- `Nav`
- `Footer`

**Attendu :**
Le composant `App` ne doit contenir que de la **composition**.

### Corrigé (illustratif)

```jsx
function Header() {
  return <header><h1>Mon site</h1></header>;
}

function Nav() {
  return (
    <nav>
      <a href="#home">Accueil</a> | <a href="#about">À propos</a>
    </nav>
  );
}

function Footer() {
  return <footer>© 2026</footer>;
}

function App() {
  return (
    <div>
      <Header />
      <Nav />
      <main>
        <p>Contenu…</p>
      </main>
      <Footer />
    </div>
  );
}
```

---

# 12. Synthèse

- Un composant React est un bloc UI réutilisable, défini en **fonction** (moderne) ou en **classe** (historique).
- Les **props** permettent de paramétrer un composant.
- `children` est central pour construire des composants conteneurs et favoriser la **composition**.
- La UI se construit comme un arbre de composants.
- Une bonne organisation (granularité, structure, naming) rend le code maintenable.

---

## Annexes — Checklist « bon composant »

- [ ] Nom en `PascalCase`
- [ ] Props claires, typées si possible (TypeScript / PropTypes)
- [ ] `key` stable pour les listes
- [ ] Pas d’effets de bord dans le rendu
- [ ] Responsabilité unique, code lisible
- [ ] Réutilisable ou justifié par une responsabilité nette
