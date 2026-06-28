# Formation React — Rendering conditionnel

> **Objectif** : maîtriser l’affichage conditionnel en React en s’appuyant sur les expressions JavaScript dans le JSX (opérateur ternaire, opérateurs logiques, structures de contrôle via variables/fonctions), tout en évitant les pièges courants.

---

## Sommaire

1. [Pré-requis et objectifs pédagogiques](#1-pré-requis-et-objectifs-pédagogiques)
2. [Rappel : JSX = JavaScript + UI](#2-rappel--jsx--javascript--ui)
3. [Les différentes techniques de rendu conditionnel](#3-les-différentes-techniques-de-rendu-conditionnel)
   1. [If/else “classique” hors JSX](#31-ifelse-classique-hors-jsx)
   2. [L’opérateur ternaire (`cond ? A : B`)](#32-lopérateur-ternaire-cond--a--b)
   3. [Le ET logique (`cond && <Component />`)](#33-le-et-logique-cond--component-)
   4. [Le OU logique (`a || b`) : usage avec précaution](#34-le-ou-logique-a--b--usage-avec-précaution)
   5. [Retourner `null` pour ne rien rendre](#35-retourner-null-pour-ne-rien-rendre)
   6. [Switch/cas multiples et mapping d’états](#36-switchcas-multiples-et-mapping-détats)
   7. [Extraire le rendu dans une fonction / composant](#37-extraire-le-rendu-dans-une-fonction--composant)
4. [Patterns recommandés (lisibilité & maintenabilité)](#4-patterns-recommandés-lisibilité--maintenabilité)
5. [Pièges fréquents et anti-patterns](#5-pièges-fréquents-et-anti-patterns)
6. [Atelier guidé : construire un composant “Dashboard”](#6-atelier-guidé--construire-un-composant-dashboard)
7. [Exercices (avec corrigés)](#7-exercices-avec-corrigés)
8. [Checklist de validation](#8-checklist-de-validation)

---

## 1) Pré-requis et objectifs pédagogiques

### Pré-requis
- Connaître les bases de React (composants fonctionnels, props, state).
- Connaître les bases de JavaScript (conditions, opérateurs, fonctions, tableaux).

### Objectifs pédagogiques
À la fin, vous saurez :
- Choisir la bonne technique de rendu conditionnel (ternaire, `&&`, `null`, etc.).
- Gérer plusieurs états d’UI (chargement/erreur/succès/vide).
- Éviter les bugs liés aux valeurs “falsy” (`0`, `""`, `false`, `null`, `undefined`).
- Rendre votre JSX plus lisible en extrayant des composants ou fonctions.

---

## 2) Rappel : JSX = JavaScript + UI

En React, **le JSX est une syntaxe qui se compile en appels JavaScript** et qui permet d’imbriquer des expressions JS dans le rendu :

```jsx
function Hello({ name }) {
  return <h1>Bonjour {name}</h1>;
}
```

> Dans un JSX, vous ne pouvez pas écrire un `if` directement comme une instruction au milieu du markup (car `if` est une *statement*, pas une *expression*). En revanche, vous pouvez utiliser :
- des **expressions** (ternaire, `&&`, appels de fonctions)
- ou préparer une variable avant le `return`

---

## 3) Les différentes techniques de rendu conditionnel

### 3.1) If/else “classique” hors JSX

Quand la logique est trop complexe pour tenir proprement dans le JSX, on prépare le rendu avant :

```jsx
function Profile({ user }) {
  if (!user) {
    return <p>Veuillez vous connecter.</p>;
  }

  return (
    <section>
      <h2>Profil</h2>
      <p>Nom : {user.name}</p>
    </section>
  );
}
```

**Avantages** :
- Très lisible.
- Excellent pour les “early returns” (cas d’erreur, chargement, absence de données).

**Inconvénient** :
- Moins pratique quand il s’agit juste de masquer/afficher un petit fragment.

---

### 3.2) L’opérateur ternaire (`cond ? A : B`)

Le ternaire est une **expression** : il est donc directement utilisable dans le JSX.

```jsx
function Greeting({ isLoggedIn }) {
  return (
    <div>
      {isLoggedIn ? <p>Bienvenue !</p> : <p>Connectez-vous.</p>}
    </div>
  );
}
```

#### Exemple : afficher un bouton différent selon l’état

```jsx
function SubscribeButton({ isSubscribed, onSubscribe, onUnsubscribe }) {
  return isSubscribed ? (
    <button onClick={onUnsubscribe}>Se désabonner</button>
  ) : (
    <button onClick={onSubscribe}>S’abonner</button>
  );
}
```

#### Bonnes pratiques
- Garder le ternaire **court**.
- Éviter l’empilement de ternaires (nested ternaries) : cela devient vite illisible.

**À éviter** (illisible) :

```jsx
{status === 'loading'
  ? <Spinner />
  : status === 'error'
    ? <ErrorPanel />
    : <Content />
}
```

On verra plus loin des alternatives propres.

---

### 3.3) Le ET logique (`cond && <Component />`)

Très utilisé pour rendre un élément **uniquement si une condition est vraie**.

```jsx
function CartSummary({ items }) {
  return (
    <section>
      <h2>Panier</h2>
      {items.length > 0 && <p>Vous avez {items.length} article(s).</p>}
    </section>
  );
}
```

#### ⚠️ Attention aux valeurs falsy rendues
En JS :
- `true && <X />` rend `<X />`
- `false && <X />` rend `false` (React n’affiche rien)
- **mais** `0 && <X />` rend **0** (React peut afficher `0` !)

Exemple problématique :

```jsx
{count && <span>Count: {count}</span>}
```

Si `count = 0`, React affiche `0` (ou un espace inattendu), ce qui est rarement voulu.

✅ Solution : rendre la condition explicitement booléenne :

```jsx
{count > 0 && <span>Count: {count}</span>}
```

ou

```jsx
{Boolean(count) && <span>Count: {count}</span>}
```

---

### 3.4) Le OU logique (`a || b`) : usage avec précaution

`||` renvoie la première valeur “truthy”. On peut l’utiliser pour des **valeurs par défaut** :

```jsx
function UserName({ user }) {
  return <p>Nom : {user.name || 'Anonyme'}</p>;
}
```

#### ⚠️ Piège : `""` et `0`
Si `user.name` peut être `""`, `0`, etc., `||` remplacera ces valeurs par le fallback.

✅ Alternative : l’opérateur de coalescence nulle `??` (si vous le pouvez)

```jsx
<p>Nom : {user.name ?? 'Anonyme'}</p>
```

- `??` ne remplace que `null` ou `undefined`.

---

### 3.5) Retourner `null` pour ne rien rendre

Un composant React peut retourner `null` : cela signifie **rien afficher**.

```jsx
function AdminBadge({ isAdmin }) {
  if (!isAdmin) return null;
  return <span className="badge">Admin</span>;
}
```

C’est souvent plus lisible qu’un `isAdmin && ...` lorsque le composant est dédié.

---

### 3.6) Switch/cas multiples et mapping d’états

Quand vous avez **plusieurs états** (ex : `loading`, `error`, `success`, `empty`), un `switch` ou une table de correspondance rend le code plus clair.

#### Option A — `switch` + early return

```jsx
function DataPanel({ status, data }) {
  switch (status) {
    case 'loading':
      return <Spinner />;
    case 'error':
      return <ErrorPanel />;
    case 'success':
      return <pre>{JSON.stringify(data, null, 2)}</pre>;
    default:
      return <p>État inconnu</p>;
  }
}
```

#### Option B — mapping (table d’affichage)

```jsx
const STATUS_VIEW = {
  loading: <Spinner />,
  error: <ErrorPanel />,
};

function DataPanel({ status, data }) {
  if (status === 'success') {
    return <pre>{JSON.stringify(data, null, 2)}</pre>;
  }

  return STATUS_VIEW[status] ?? <p>État inconnu</p>;
}
```

**Conseil** : si la vue dépend de props (e.g `data`), préférez une fonction :

```jsx
const statusToView = (status, data) => {
  switch (status) {
    case 'success':
      return <pre>{JSON.stringify(data, null, 2)}</pre>;
    case 'loading':
      return <Spinner />;
    case 'error':
      return <ErrorPanel />;
    default:
      return <p>État inconnu</p>;
  }
};
```

---

### 3.7) Extraire le rendu dans une fonction / composant

Quand le JSX grossit, vous pouvez extraire une partie conditionnelle.

#### Extraire une fonction de rendu

```jsx
function ProductCard({ product }) {
  const renderBadge = () => {
    if (product.stock === 0) return <span className="badge">Rupture</span>;
    if (product.stock < 5) return <span className="badge">Bientôt épuisé</span>;
    return null;
  };

  return (
    <article>
      <h3>{product.name}</h3>
      {renderBadge()}
    </article>
  );
}
```

#### Extraire un composant dédié

```jsx
function StockBadge({ stock }) {
  if (stock === 0) return <span className="badge">Rupture</span>;
  if (stock < 5) return <span className="badge">Bientôt épuisé</span>;
  return null;
}

function ProductCard({ product }) {
  return (
    <article>
      <h3>{product.name}</h3>
      <StockBadge stock={product.stock} />
    </article>
  );
}
```

**Avantages** :
- Meilleure lisibilité.
- Meilleure testabilité.
- Réutilisable.

---

## 4) Patterns recommandés (lisibilité & maintenabilité)

### Pattern 1 — “Early returns” pour les états majeurs

Très utile dans les composants qui chargent des données :

```jsx
function UsersList({ status, users, error }) {
  if (status === 'loading') return <Spinner />;
  if (status === 'error') return <p>Erreur : {error.message}</p>;
  if (!users || users.length === 0) return <p>Aucun utilisateur.</p>;

  return (
    <ul>
      {users.map(u => (
        <li key={u.id}>{u.name}</li>
      ))}
    </ul>
  );
}
```

### Pattern 2 — Favoriser `&&` pour un fragment simple

```jsx
{isOpen && <Modal />}
```

### Pattern 3 — Favoriser le ternaire si vous avez un `else` clair

```jsx
{isDark ? <DarkTheme /> : <LightTheme />}
```

### Pattern 4 — Limiter la logique dans le JSX
- Pré-calculer des booléens :

```jsx
const hasItems = items.length > 0;
...
{hasItems && <ItemsTable items={items} />}
```

---

## 5) Pièges fréquents et anti-patterns

### 5.1) Empiler des ternaires (nested)
- Difficile à lire.
- Difficile à modifier.

✅ Alternative : `switch`, mapping, extraire un composant.

### 5.2) Utiliser `&&` avec une valeur non booléenne (ex: `0`)
- Risque d’afficher `0`.

✅ Utiliser une condition explicite : `count > 0 && ...`.

### 5.3) Confondre “ne pas afficher” et “afficher un placeholder”
- `null` : rien
- placeholder : un élément visible

### 5.4) États d’UI mal définis
Évitez les états flous comme `isLoaded` + `hasError` + `isEmpty` si cela peut se contredire.

✅ Préférez un champ `status` unique :
- `idle | loading | error | success`

---

## 6) Atelier guidé : construire un composant “Dashboard”

### Contexte
On veut un composant qui gère :
- un chargement
- une erreur
- une liste vide
- une liste d’éléments

### Modèle de données

```js
// status: 'loading' | 'error' | 'success'
// error: { message: string }
// projects: Array<{ id: string, name: string }>
```

### Implémentation progressive

#### Étape 1 — gérer loading et error (early returns)

```jsx
function Dashboard({ status, error, projects }) {
  if (status === 'loading') return <Spinner />;
  if (status === 'error') return <ErrorPanel message={error.message} />;

  return <div>...</div>;
}
```

#### Étape 2 — gérer le cas “vide”

```jsx
function Dashboard({ status, error, projects }) {
  if (status === 'loading') return <Spinner />;
  if (status === 'error') return <ErrorPanel message={error.message} />;

  if (!projects || projects.length === 0) {
    return (
      <section>
        <h2>Projets</h2>
        <p>Aucun projet pour le moment.</p>
        <button>Créer un projet</button>
      </section>
    );
  }

  return (
    <section>
      <h2>Projets</h2>
      <ul>
        {projects.map(p => (
          <li key={p.id}>{p.name}</li>
        ))}
      </ul>
    </section>
  );
}
```

#### Étape 3 — ajouter un détail conditionnel simple avec `&&`

On affiche un badge si la liste est “grande”.

```jsx
function Dashboard({ status, error, projects }) {
  if (status === 'loading') return <Spinner />;
  if (status === 'error') return <ErrorPanel message={error.message} />;

  const hasProjects = (projects?.length ?? 0) > 0;
  const isBigList = (projects?.length ?? 0) >= 10;

  if (!hasProjects) {
    return (
      <section>
        <h2>Projets</h2>
        <p>Aucun projet pour le moment.</p>
        <button>Créer un projet</button>
      </section>
    );
  }

  return (
    <section>
      <h2>
        Projets {isBigList && <span className="badge">+10</span>}
      </h2>
      <ul>
        {projects.map(p => (
          <li key={p.id}>{p.name}</li>
        ))}
      </ul>
    </section>
  );
}
```

---

## 7) Exercices (avec corrigés)

### Exercice 1 — Badge “Nouveau”

**Énoncé** : Sur une carte produit, afficher le badge “Nouveau” si `isNew` est `true`.

**Starter** :

```jsx
function Product({ name, isNew }) {
  return (
    <div>
      <h3>{name}</h3>
      {/* TODO */}
    </div>
  );
}
```

**Corrigé (avec `&&`)** :

```jsx
function Product({ name, isNew }) {
  return (
    <div>
      <h3>{name}</h3>
      {isNew && <span className="badge">Nouveau</span>}
    </div>
  );
}
```

---

### Exercice 2 — Bouton “Voir plus / Réduire”

**Énoncé** : Afficher un bouton dont le texte dépend de `expanded`.

**Corrigé (ternaire)** :

```jsx
function ToggleButton({ expanded, onToggle }) {
  return (
    <button onClick={onToggle}>
      {expanded ? 'Réduire' : 'Voir plus'}
    </button>
  );
}
```

---

### Exercice 3 — Gérer 3 états (loading / error / data)

**Énoncé** : Implémenter un composant `UserPanel`.

**Corrigé (early returns)** :

```jsx
function UserPanel({ status, user, error }) {
  if (status === 'loading') return <Spinner />;
  if (status === 'error') return <p>Erreur : {error.message}</p>;
  if (!user) return <p>Aucun utilisateur.</p>;

  return (
    <div>
      <h3>{user.name}</h3>
      <p>Email : {user.email}</p>
    </div>
  );
}
```

---

## 8) Checklist de validation

- [ ] Ai-je choisi `&&` pour un affichage “si vrai seulement” ?
- [ ] Ai-je choisi un ternaire quand j’ai un `else` naturel ?
- [ ] Ai-je évité `count && ...` si `count` peut valoir `0` ?
- [ ] Mes états d’UI sont-ils exclusifs (ex: `status`) ?
- [ ] Ma logique est-elle lisible (extraction de composants si nécessaire) ?

---

## Annexes — Mini aide-mémoire

### Raccourcis
- **Afficher si vrai** : `cond && <X />`
- **Afficher A sinon B** : `cond ? <A /> : <B />`
- **Ne rien afficher** : `return null`
- **Cas multiples** : `switch(status)` ou mapping

### Valeurs falsy en JavaScript
- `false`, `0`, `""`, `null`, `undefined`, `NaN`

---

### Fin de la formation

Si vous souhaitez, je peux fournir une version “slides” (format plan + points clés) ou une série de quiz rapides pour vos apprenants.