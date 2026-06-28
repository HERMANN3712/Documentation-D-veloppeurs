# Formation React — Listes et clés

## Objectifs pédagogiques
À la fin de ce module, vous serez capable de :

- Expliquer pourquoi React a besoin de **clés** lors du rendu de listes.
- Rendre des listes en React avec **`Array.prototype.map()`**.
- Choisir une **clé unique et stable** adaptée au contexte.
- Éviter les pièges courants (index, clés non stables, duplication).
- Comprendre l’impact des clés sur le **DOM virtuel (Virtual DOM)** et la **réconciliation**.

---

## Prérequis
- Bases de React (composants, JSX, props/state)
- JavaScript moderne (tableaux, fonctions fléchées)

---

## Plan de la formation
1. **Introduction** : rendre des collections en UI
2. **Rendre une liste avec `map()`**
3. **La notion de clé (`key`) en React**
4. **Comment React utilise les clés (Virtual DOM & réconciliation)**
5. **Bonnes pratiques pour choisir une clé**
6. **Erreurs fréquentes et anti-patterns**
7. **Exercices guidés + corrigés**
8. **Checklist de fin de module**

---

## 1) Introduction : rendre des collections en UI
Dans une application React, vous affichez très souvent des collections :

- une liste de tâches (todo)
- des produits
- des commentaires
- des lignes d’un tableau

En React, le rendu d’une collection se fait généralement en transformant un tableau de données en un tableau d’éléments JSX.

> Le mécanisme central côté JavaScript : **`map()`**.

---

## 2) Rendre une liste avec `map()`

### 2.1 Exemple minimal
Supposons une liste de prénoms :

```jsx
const names = ["Ada", "Grace", "Linus"];

export function NamesList() {
  return (
    <ul>
      {names.map((name) => (
        <li>{name}</li>
      ))}
    </ul>
  );
}
```

Si vous exécutez ce code, React affichera la liste mais **émettra un warning** en console du type :

> Warning: Each child in a list should have a unique "key" prop.

Ce warning est essentiel : il indique que React a besoin d’une **clé** pour chaque élément de la liste.

---

### 2.2 Ajouter une `key`
On corrige en ajoutant une clé :

```jsx
const names = ["Ada", "Grace", "Linus"];

export function NamesList() {
  return (
    <ul>
      {names.map((name) => (
        <li key={name}>{name}</li>
      ))}
    </ul>
  );
}
```

Ici, `name` sert de clé : c’est acceptable **si** chaque prénom est **unique** et **stable**.

---

### 2.3 Exemple plus réaliste : objets avec identifiant
Dans un vrai projet, vous mappez souvent des objets :

```jsx
const users = [
  { id: "u1", name: "Ada" },
  { id: "u2", name: "Grace" },
  { id: "u3", name: "Linus" },
];

export function UsersList() {
  return (
    <ul>
      {users.map((user) => (
        <li key={user.id}>{user.name}</li>
      ))}
    </ul>
  );
}
```

✅ **Bonne pratique** : utiliser un identifiant unique (souvent un `id` issu de la BDD).

---

## 3) La notion de clé (`key`) en React

### 3.1 Définition
Une **clé** est un attribut spécial que React utilise pour **identifier de manière unique** chaque élément d’une liste au sein de ses frères (siblings).

- La clé n’est **pas** passée comme prop classique au composant (elle est consommée par React).
- Elle doit être **unique** parmi les éléments de la liste rendue.
- Elle doit être **stable** dans le temps : ne pas changer d’un rendu à l’autre si l’élément représente la même entité.

---

### 3.2 Où mettre la `key` ?
La `key` se met sur l’élément **retourné par `map()`** (le nœud le plus haut de ce que vous retournez).

Exemple :

```jsx
{users.map((user) => (
  <UserRow key={user.id} user={user} />
))}
```

Ici, la clé est sur `<UserRow />` (le composant racine créé par le `map`).

---

### 3.3 La `key` n’est pas une prop normale
Si vous écrivez :

```jsx
function UserRow(props) {
  console.log(props.key);
  return <div>{props.user.name}</div>;
}
```

`props.key` sera `undefined`.

Si vous avez besoin de l’identifiant dans le composant, passez-le explicitement :

```jsx
{users.map((user) => (
  <UserRow key={user.id} user={user} userId={user.id} />
))}
```

---

## 4) Comment React utilise les clés (Virtual DOM & réconciliation)

### 4.1 Rappel : Virtual DOM et « diff »
React construit une représentation de l’UI (Virtual DOM). À chaque rendu, React compare l’ancien arbre virtuel au nouveau pour déterminer **quoi mettre à jour** dans le DOM réel.

Ce processus s’appelle la **réconciliation**.

---

### 4.2 Pourquoi les clés aident React ?
Quand vous rendez une liste, React doit comprendre :

- quels éléments existaient avant
- lesquels ont été ajoutés
- lesquels ont été supprimés
- lesquels ont changé d’ordre

Sans clé stable, React peut confondre des éléments et appliquer des mises à jour sur le **mauvais** nœud.

> Les clés donnent une identité aux items : « cet élément-ci est le même qu’avant ».

---

### 4.3 Exemple conceptuel : insertion en tête
Imaginez une liste d’inputs. Vous insérez un item au début.

- Avec des clés stables : chaque input garde sa valeur et son focus associé.
- Avec des clés instables (ex: index) : React peut réutiliser un DOM node pour un autre item → valeurs/focus mélangés.

---

## 5) Bonnes pratiques pour choisir une clé

### 5.1 Qualités d’une bonne clé
Une bonne clé est :

- **Unique** (au sein de la liste)
- **Stable** (ne change pas entre les rendus si l’item est le même)
- **Prévisible** (reliée à l’identité métier de l’élément)

Dans la majorité des cas : `key={item.id}`.

---

### 5.2 Sources de clés recommandées
- Identifiant BDD (UUID, auto-increment, etc.)
- Identifiant métier stable (slug unique, code produit, etc.)

Exemples :

```jsx
<li key={product.sku}>{product.name}</li>
<li key={article.slug}>{article.title}</li>
```

---

### 5.3 Quand `index` peut être acceptable
Utiliser l’index dans `map((item, index) => ...)` n’est pas toujours catastrophique, mais **c’est risqué**.

✅ Acceptable si :
- la liste est **statique** (ne change jamais d’ordre)
- aucun élément n’est ajouté/supprimé
- les items ne contiennent pas d’état local sensible (inputs, composants stateful)

Exemple (liste purement statique) :

```jsx
const steps = ["Installer", "Configurer", "Déployer"];

export function Steps() {
  return (
    <ol>
      {steps.map((label, index) => (
        <li key={index}>{label}</li>
      ))}
    </ol>
  );
}
```

⚠️ Si la liste peut évoluer (tri, filtre, insertion, suppression), évitez `index`.

---

## 6) Erreurs fréquentes et anti-patterns

### 6.1 Utiliser une valeur non unique
Exemple incorrect :

```jsx
{users.map((u) => (
  <li key={u.name}>{u.name}</li>
))}
```

Si deux utilisateurs s’appellent pareil, vous aurez des collisions de clés.

---

### 6.2 Générer une clé aléatoire à chaque rendu
Exemple incorrect :

```jsx
{users.map((u) => (
  <li key={crypto.randomUUID()}>{u.name}</li>
))}
```

Ici, la clé change à **chaque rendu** → React pense que tous les éléments sont **nouveaux**.

Conséquences :
- performance dégradée (remontage inutile)
- perte d’état (un input perd sa valeur)

---

### 6.3 Utiliser `Date.now()` comme clé
Même problème : pas stable.

---

### 6.4 Mauvais placement de la clé
Si vous retournez un fragment, la clé doit être sur le fragment :

```jsx
{users.map((u) => (
  <React.Fragment key={u.id}>
    <dt>{u.name}</dt>
    <dd>{u.id}</dd>
  </React.Fragment>
))}
```

Ou avec la syntaxe courte (nécessite la forme longue pour la `key`) :

```jsx
{users.map((u) => (
  <React.Fragment key={u.id}>
    <span>{u.name}</span>
    <span>{u.id}</span>
  </React.Fragment>
))}
```

---

## 7) Exercices guidés + corrigés

### Exercice 1 — Ajouter des clés à une liste simple
**Énoncé** : rendre une liste de tags.

Données :

```js
const tags = ["react", "js", "ui"];
```

**Attendu** : afficher les tags dans une liste avec une clé correcte.

✅ Corrigé :

```jsx
export function Tags() {
  const tags = ["react", "js", "ui"];

  return (
    <ul>
      {tags.map((tag) => (
        <li key={tag}>{tag}</li>
      ))}
    </ul>
  );
}
```

---

### Exercice 2 — Corriger une liste avec index problématique
**Énoncé** : vous avez une liste que l’utilisateur peut trier.

Code initial (risqué) :

```jsx
{items.map((item, index) => (
  <Row key={index} item={item} />
))}
```

Questions :
1. Pourquoi `index` pose problème ici ?
2. Proposez une correction.

✅ Corrigé (principe) : utilisez un identifiant stable :

```jsx
{items.map((item) => (
  <Row key={item.id} item={item} />
))}
```

Explication : lors d’un tri, les index changent, donc React associe potentiellement les anciens DOM nodes aux mauvais items.

---

### Exercice 3 — Fragments et clé
**Énoncé** : rendre un `<dl>` avec deux lignes par item.

✅ Corrigé :

```jsx
export function UserDefinitionList({ users }) {
  return (
    <dl>
      {users.map((u) => (
        <React.Fragment key={u.id}>
          <dt>{u.name}</dt>
          <dd>{u.email}</dd>
        </React.Fragment>
      ))}
    </dl>
  );
}
```

---

## 8) Checklist de fin de module
Avant de valider votre implémentation, vérifiez :

- [ ] Chaque élément rendu via `map()` a une prop `key`.
- [ ] La `key` est **unique** parmi les éléments frères.
- [ ] La `key` est **stable** (pas de `randomUUID()`, `Math.random()`, `Date.now()`).
- [ ] Vous n’utilisez pas l’index si la liste peut être triée/filtrée/modifiée.
- [ ] Les composants stateful (inputs, items interactifs) conservent leur état lors des mises à jour.

---

## Résumé
- Les listes en React se rendent principalement avec **`map()`**.
- Chaque élément doit posséder une **clé unique** afin que React puisse **identifier efficacement les changements** dans le **DOM virtuel**.
- Une bonne clé est **unique + stable** (souvent `id`).

---

## Annexes — Snippets de référence

### Pattern standard
```jsx
{items.map((item) => (
  <Item key={item.id} item={item} />
))}
```

### Rendu conditionnel en liste (attention au `key`)
```jsx
{items
  .filter((item) => item.enabled)
  .map((item) => (
    <Item key={item.id} item={item} />
  ))}
```
