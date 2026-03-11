# Formation Vue.js — Directives Vue

> **Public** : développeurs ayant déjà des bases HTML/JS et une première exposition à Vue (ou un autre framework).  
> **Niveau** : débutant → intermédiaire  
> **Durée indicative** : 2h30 à 4h (selon exercices)

---

## Objectifs pédagogiques

À la fin de cette formation, vous serez capable de :

- Définir ce qu’est une **directive Vue** et reconnaître la syntaxe `v-`.
- Utiliser correctement :
  - `v-if` / `v-else-if` / `v-else` pour le **rendu conditionnel**.
  - `v-for` pour **itérer sur des listes** et comprendre les clés (`:key`).
  - `v-bind` (et son raccourci `:`) pour **lier des attributs** et des props.
  - `v-on` (et son raccourci `@`) pour **gérer les événements**.
- Combiner les directives sans tomber dans les pièges courants.
- Écrire un code plus lisible grâce aux bonnes pratiques de templating.

---

## Prérequis techniques

- Node.js + un projet Vue fonctionnel (Vue CLI ou Vite).
- Connaissances :
  - HTML/CSS de base
  - JavaScript (variables, objets, tableaux, fonctions)
  - Notions de composants (`<template>`, `<script>`, `<style>`) recommandées

---

## Plan de la formation

1. **Introduction aux directives Vue**
   - Définition
   - Expressions dans le template
   - Raccourcis (`:` et `@`)
2. **Rendu conditionnel : `v-if` et variantes**
   - `v-if`, `v-else-if`, `v-else`
   - `v-if` vs `v-show`
   - Bonnes pratiques et erreurs fréquentes
3. **Rendu de listes : `v-for`**
   - Itération sur tableaux, objets, ranges
   - `:key` et stabilité du DOM
   - `v-for` avec `v-if` (anti-pattern et alternatives)
4. **Liaison d’attributs : `v-bind`**
   - Attributs HTML, classes, styles
   - Liaison d’objets et d’attributs multiples
   - Valeurs booléennes, `null`/`undefined`
5. **Gestion des événements : `v-on`**
   - Écouter des événements DOM
   - Modifiers (`.prevent`, `.stop`, `.once`, `.capture`, `.self`, `.passive`)
   - Key modifiers (`.enter`, `.esc`, etc.)
   - Passer des paramètres, `$event`
6. **Atelier (fil rouge) : mini-composant “Todo”**
   - Affichage conditionnel
   - Liste + clés
   - Binding + événements
7. **Quiz & checklist de bonnes pratiques**

---

# 1) Introduction aux directives Vue

## 1.1 Qu’est-ce qu’une directive ?

Les **directives** sont des **attributs spéciaux** dans les templates Vue, **préfixés par `v-`**, qui indiquent à Vue d’appliquer un comportement réactif au DOM.

Exemples :

- `v-if` : affiche/retire un élément selon une condition
- `v-for` : répète un bloc pour chaque élément d’une liste
- `v-bind` : lie la valeur d’un attribut à une expression JS
- `v-on` : écoute un événement et déclenche du code

> Une directive se place sur un **élément** (ex: `<div>`) ou un **composant** (ex: `<MyButton>`).

## 1.2 Expressions dans le template

Les directives évaluent des **expressions JavaScript** dans le contexte du composant.

```vue
<template>
  <p v-if="isLoggedIn">Bonjour {{ user.name }}</p>
</template>

<script setup>
const isLoggedIn = true
const user = { name: 'Sam' }
</script>
```

Règles importantes :

- Dans le template, vous écrivez des **expressions** (pas des statements).
  - ✅ `user.name`, `count + 1`, `isAdmin && isActive`
  - ❌ `if (...) { ... }`, `for (...) { ... }`

## 1.3 Raccourcis utiles (`:` et `@`)

Deux directives sont si courantes qu’elles ont des raccourcis :

- `v-bind:href="..."` → `:href="..."`
- `v-on:click="..."` → `@click="..."`

```vue
<template>
  <a :href="profileUrl" @click="trackClick">Profil</a>
</template>
```

---

# 2) Rendu conditionnel — `v-if`, `v-else-if`, `v-else`

## 2.1 `v-if` : afficher ou non un élément

`v-if` **ajoute ou retire** l’élément du DOM selon la condition.

```vue
<template>
  <button @click="isOpen = !isOpen">Toggle</button>

  <p v-if="isOpen">Le panneau est ouvert</p>
</template>

<script setup>
import { ref } from 'vue'
const isOpen = ref(false)
</script>
```

### À retenir

- Si `isOpen` est `false`, le `<p>` **n’existe pas** (pas juste caché).
- Les composants/enfants à l’intérieur sont **montés/démontés**.

## 2.2 `v-else-if` et `v-else`

```vue
<template>
  <p v-if="status === 'loading'">Chargement…</p>
  <p v-else-if="status === 'error'">Erreur</p>
  <p v-else>OK</p>
</template>

<script setup>
import { ref } from 'vue'
const status = ref('loading')
</script>
```

Contraintes :

- `v-else-if` et `v-else` doivent **immédiatement** suivre un élément `v-if`/`v-else-if`.

## 2.3 `v-if` vs `v-show`

Même si ce cours se concentre sur les directives principales demandées, il est utile de comprendre la différence :

- `v-if` : **vrai/faux** → création/suppression du DOM (coût au toggle)
- `v-show` : l’élément reste dans le DOM, Vue joue sur `display: none` (coût initial)

| Besoin | Préférer |
|---|---|
| Affichage rare (souvent faux) | `v-if` |
| Toggle fréquent (UI qui cache/affiche souvent) | `v-show` |

## 2.4 Bonnes pratiques & pièges

- Évitez les expressions peu lisibles :
  - ❌ `v-if="a && b && c && !d && (x > y ? p : q)"`
  - ✅ calculer dans un `computed` (ou une fonction) puis utiliser `v-if="canDisplay"`

- Attention à la perte d’état : un composant dans un `v-if` est détruit quand la condition passe à faux.

---

# 3) Rendu de listes — `v-for`

## 3.1 Itérer sur un tableau

```vue
<template>
  <ul>
    <li v-for="item in items" :key="item.id">
      {{ item.label }}
    </li>
  </ul>
</template>

<script setup>
const items = [
  { id: 1, label: 'Apprendre v-if' },
  { id: 2, label: 'Apprendre v-for' },
  { id: 3, label: 'Apprendre v-bind' }
]
</script>
```

Forme avec index :

```vue
<li v-for="(item, index) in items" :key="item.id">
  #{{ index }} — {{ item.label }}
</li>
```

## 3.2 `:key` : pourquoi c’est obligatoire (ou presque)

La clé (`key`) aide Vue à :

- identifier chaque nœud de façon stable
- réutiliser correctement des éléments lors des modifications (insertions, suppressions, tri)

### Bonnes clés

- ✅ ID unique et stable (`item.id`)

### Mauvaises clés

- ⚠️ index (`:key="index"`) : acceptable uniquement si la liste ne change jamais d’ordre et qu’on n’insère/supprime pas.

## 3.3 Itérer sur un objet (key/value)

```vue
<template>
  <ul>
    <li v-for="(value, key) in settings" :key="key">
      {{ key }}: {{ value }}
    </li>
  </ul>
</template>

<script setup>
const settings = {
  theme: 'dark',
  lang: 'fr',
  debug: false
}
</script>
```

## 3.4 Itérer sur un range

```vue
<template>
  <p>
    <span v-for="n in 5" :key="n">{{ n }} </span>
  </p>
</template>
```

## 3.5 `v-for` et `v-if` ensemble : anti-pattern courant

### Problème

Écrire `v-if` et `v-for` sur le **même élément** crée souvent une ambiguïté et nuit à la lisibilité.

```vue
<!-- À éviter -->
<li v-for="item in items" v-if="item.active" :key="item.id">
  {{ item.label }}
</li>
```

### Alternatives recommandées

1) Filtrer en amont (computed) :

```vue
<script setup>
import { computed } from 'vue'

const items = [
  { id: 1, label: 'A', active: true },
  { id: 2, label: 'B', active: false },
]

const activeItems = computed(() => items.filter(i => i.active))
</script>

<template>
  <li v-for="item in activeItems" :key="item.id">{{ item.label }}</li>
</template>
```

2) Mettre le `v-if` sur un wrapper :

```vue
<template>
  <template v-for="item in items" :key="item.id">
    <li v-if="item.active">{{ item.label }}</li>
  </template>
</template>
```

---

# 4) Liaison d’attributs — `v-bind`

## 4.1 Principe

`v-bind` permet de lier dynamiquement un attribut HTML (ou une prop de composant) à une expression.

```vue
<template>
  <img v-bind:src="avatarUrl" v-bind:alt="userName" />
</template>

<script setup>
const avatarUrl = 'https://example.com/avatar.png'
const userName = 'Sam'
</script>
```

Avec le raccourci :

```vue
<img :src="avatarUrl" :alt="userName" />
```

## 4.2 Lier des classes

### Classe conditionnelle (objet)

```vue
<template>
  <button :class="{ primary: isPrimary, disabled: isDisabled }">
    Valider
  </button>
</template>

<script setup>
import { ref } from 'vue'
const isPrimary = ref(true)
const isDisabled = ref(false)
</script>
```

### Classe (tableau)

```vue
<template>
  <div :class="['card', sizeClass, { highlighted: isHighlighted }]">
    Contenu
  </div>
</template>

<script setup>
const sizeClass = 'card--lg'
const isHighlighted = true
</script>
```

## 4.3 Lier des styles

```vue
<template>
  <div :style="{ color: textColor, fontSize: fontSize + 'px' }">
    Texte stylé
  </div>
</template>

<script setup>
const textColor = 'rebeccapurple'
const fontSize = 18
</script>
```

## 4.4 Liaison d’attributs multiples (objet) avec `v-bind="..."`

Vue permet de binder un objet entier d’attributs.

```vue
<template>
  <input v-bind="inputAttrs" />
</template>

<script setup>
const inputAttrs = {
  type: 'email',
  placeholder: 'Votre email',
  autocomplete: 'email'
}
</script>
```

> Utile pour propager des attributs, factoriser, ou construire des “config objects”.

## 4.5 Attributs booléens, `null` et `undefined`

- Un attribut booléen (ex: `disabled`) :

```vue
<button :disabled="isDisabled">Envoyer</button>
```

- Si la valeur liée est `null`/`undefined`, Vue **retire** généralement l’attribut.

---

# 5) Gestion des événements — `v-on`

## 5.1 Écouter un événement DOM

```vue
<template>
  <button v-on:click="count++">Clique</button>
  <p>Count: {{ count }}</p>
</template>

<script setup>
import { ref } from 'vue'
const count = ref(0)
</script>
```

Avec le raccourci :

```vue
<button @click="count++">Clique</button>
```

## 5.2 Appeler une méthode

```vue
<template>
  <button @click="increment">+1</button>
</template>

<script setup>
import { ref } from 'vue'
const count = ref(0)

function increment() {
  count.value++
}
</script>
```

## 5.3 Passer des paramètres et utiliser `$event`

```vue
<template>
  <button @click="add(5)">Ajouter 5</button>
  <input @input="onInput($event)" />
</template>

<script setup>
import { ref } from 'vue'

const count = ref(0)

function add(amount) {
  count.value += amount
}

function onInput(event) {
  // event est un InputEvent ; event.target contient la valeur
  console.log(event.target.value)
}
</script>
```

## 5.4 Event modifiers (les plus utiles)

Vue propose des modificateurs pour éviter du code “boilerplate”.

- `.prevent` → `event.preventDefault()`
- `.stop` → `event.stopPropagation()`
- `.once` → écoute une seule fois
- `.self` → déclenche seulement si la cible est l’élément lui-même

Exemples :

```vue
<template>
  <form @submit.prevent="save">
    <button type="submit">Enregistrer</button>
  </form>

  <a href="https://example.com" @click.prevent="track">Ne navigue pas</a>

  <div class="backdrop" @click.self="close">
    <div class="modal">Clique dehors pour fermer</div>
  </div>
</template>

<script setup>
function save() { /* ... */ }
function track() { /* ... */ }
function close() { /* ... */ }
</script>
```

## 5.5 Key modifiers (clavier)

```vue
<template>
  <input
    placeholder="Tapez et Entrée pour valider"
    @keyup.enter="submit"
    @keyup.esc="reset"
  />
</template>

<script setup>
function submit() { /* ... */ }
function reset() { /* ... */ }
</script>
```

---

# 6) Atelier fil rouge — Mini composant “Todo”

Objectif : mettre en pratique `v-if`, `v-for`, `v-bind`, `v-on` dans un composant unique.

## 6.1 Spécifications

- Afficher une liste de tâches.
- Ajouter une tâche.
- Marquer une tâche comme terminée.
- Filtrer l’affichage : toutes / actives / terminées.
- Afficher un message si la liste est vide.

## 6.2 Implémentation (exemple complet)

> À coller dans un composant `TodoList.vue`.

```vue
<template>
  <section class="todo">
    <header class="todo__header">
      <h1>Todos</h1>

      <form class="todo__form" @submit.prevent="addTodo">
        <input
          v-bind="newTodoInputAttrs"
          v-model="newLabel"
        />
        <button type="submit" :disabled="!newLabel.trim()">Ajouter</button>
      </form>

      <nav class="todo__filters">
        <button
          v-for="f in filters"
          :key="f.value"
          :class="{ active: filter === f.value }"
          @click="filter = f.value"
        >
          {{ f.label }}
        </button>
      </nav>
    </header>

    <p v-if="filteredTodos.length === 0" class="todo__empty">
      Aucune tâche à afficher.
    </p>

    <ul v-else class="todo__list">
      <li
        v-for="todo in filteredTodos"
        :key="todo.id"
        :class="{ done: todo.done }"
      >
        <label>
          <input type="checkbox" :checked="todo.done" @change="toggle(todo.id)" />
          <span>{{ todo.label }}</span>
        </label>

        <button class="danger" @click="remove(todo.id)">Supprimer</button>
      </li>
    </ul>

    <footer class="todo__footer">
      <small>
        Total: {{ todos.length }} —
        Actives: {{ activeCount }} —
        Terminées: {{ doneCount }}
      </small>
    </footer>
  </section>
</template>

<script setup>
import { computed, ref } from 'vue'

const newLabel = ref('')
const filter = ref('all')

const todos = ref([
  { id: 1, label: 'Lire la doc Vue', done: true },
  { id: 2, label: 'Pratiquer v-for + key', done: false },
  { id: 3, label: 'Comprendre v-bind / v-on', done: false }
])

const filters = [
  { value: 'all', label: 'Toutes' },
  { value: 'active', label: 'Actives' },
  { value: 'done', label: 'Terminées' }
]

const newTodoInputAttrs = {
  type: 'text',
  placeholder: 'Nouvelle tâche…',
  name: 'todo',
  autocomplete: 'off'
}

const filteredTodos = computed(() => {
  if (filter.value === 'active') return todos.value.filter(t => !t.done)
  if (filter.value === 'done') return todos.value.filter(t => t.done)
  return todos.value
})

const activeCount = computed(() => todos.value.filter(t => !t.done).length)
const doneCount = computed(() => todos.value.filter(t => t.done).length)

function addTodo() {
  const label = newLabel.value.trim()
  if (!label) return

  // id simple pour l'exemple (en vrai, préférez un uuid)
  const id = Date.now()
  todos.value.unshift({ id, label, done: false })
  newLabel.value = ''
}

function toggle(id) {
  const t = todos.value.find(t => t.id === id)
  if (t) t.done = !t.done
}

function remove(id) {
  todos.value = todos.value.filter(t => t.id !== id)
}
</script>

<style scoped>
.todo { max-width: 720px; margin: 2rem auto; font-family: system-ui, sans-serif; }
.todo__header { display: grid; gap: 1rem; }
.todo__form { display: flex; gap: .5rem; }
.todo__form input { flex: 1; padding: .5rem; }
.todo__filters { display: flex; gap: .5rem; flex-wrap: wrap; }
.todo__filters button.active { font-weight: 700; text-decoration: underline; }
.todo__empty { opacity: .8; }
.todo__list { list-style: none; padding: 0; display: grid; gap: .5rem; }
.todo__list li { display: flex; align-items: center; justify-content: space-between; padding: .5rem; border: 1px solid #ddd; border-radius: .5rem; }
.todo__list li.done span { text-decoration: line-through; opacity: .7; }
button.danger { color: #b00020; }
</style>
```

### Où sont les directives ?

- `v-if` / `v-else` : message “liste vide” vs liste
- `v-for` : boutons de filtres + liste de todos
- `v-bind` : `:disabled`, `:class`, `:checked`, `v-bind="newTodoInputAttrs"`
- `v-on` : `@submit.prevent`, `@click`, `@change`

---

# 7) Quiz rapide & checklist

## 7.1 Quiz

1. Quelle est la différence principale entre `v-if` et `v-show` ?
2. Pourquoi `:key` est important avec `v-for` ?
3. Quelle est la différence entre `v-bind:href` et `href` ?
4. À quoi sert `@submit.prevent` ?

## 7.2 Checklist de bonnes pratiques

- [ ] Toujours fournir une `:key` stable avec `v-for`.
- [ ] Éviter `v-if` et `v-for` sur le même élément ; filtrer en amont avec `computed`.
- [ ] Utiliser `:` et `@` pour améliorer la lisibilité.
- [ ] Préférer des conditions simples dans le template ; déplacer la logique complexe dans `computed`.
- [ ] Utiliser les modifiers (`.prevent`, `.stop`, `.once`, `.enter`) pour réduire le boilerplate.

---

## Annexes — Mémo express

```text
v-if        rendu conditionnel (ajoute/retire du DOM)
v-for       itération (tableaux/objets/ranges)
v-bind      liaison d'attributs / props (raccourci :) 
v-on        gestion d'événements (raccourci @)
```

---

### Références

- Documentation officielle Vue — Template Syntax / Directives :
  - https://vuejs.org/guide/essentials/template-syntax.html
  - https://vuejs.org/guide/essentials/conditional.html
  - https://vuejs.org/guide/essentials/list.html
  - https://vuejs.org/guide/essentials/event-handling.html
