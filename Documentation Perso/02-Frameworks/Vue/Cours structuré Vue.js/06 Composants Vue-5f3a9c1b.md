# Formation Vue.js — 06 Composants Vue

> **Objectif général** : comprendre, créer et organiser des composants Vue pour structurer une application en blocs **réutilisables**, **maintenables** et **imbriqués**.

---

## Sommaire

1. [Pourquoi les composants ?](#1-pourquoi-les-composants-)
2. [Anatomie d’un composant Vue (template, logique, style)](#2-anatomie-dun-composant-vue-template-logique-style)
3. [Créer des composants (SFC) et bonnes pratiques de base](#3-créer-des-composants-sfc-et-bonnes-pratiques-de-base)
4. [Enregistrer et utiliser un composant](#4-enregistrer-et-utiliser-un-composant)
5. [Imbrication de composants (parent/enfant)](#5-imbrication-de-composants-parentenfant)
6. [Communication entre composants (props & events)](#6-communication-entre-composants-props--events)
7. [Slots : composer des interfaces réutilisables](#7-slots--composer-des-interfaces-réutilisables)
8. [Composants dynamiques & rendu conditionnel](#8-composants-dynamiques--rendu-conditionnel)
9. [Organisation du code à l’échelle (structure de dossiers & conventions)](#9-organisation-du-code-à-léchelle-structure-de-dossiers--conventions)
10. [Atelier fil rouge : construire une mini-UI en composants](#10-atelier-fil-rouge--construire-une-mini-ui-en-composants)
11. [Récapitulatif & checklist](#11-récapitulatif--checklist)

---

## Prérequis

- Connaissances de base en HTML/CSS/JS
- Notions de base Vue (réactivité, directives `v-if`, `v-for`, `v-model`)
- Projet Vue 3 (Vite recommandé)

> Les exemples ci-dessous ciblent **Vue 3** avec la syntaxe **SFC** (Single File Component) et, selon les cas, `script setup`.

---

## 1. Pourquoi les composants ?

### 1.1 Définition
Un **composant** est un **bloc fonctionnel** et **réutilisable** de l’interface. Il encapsule :

- un **template** (la structure HTML)
- une **logique** (données, méthodes, réactivité)
- un **style** (CSS, potentiellement scoped)

### 1.2 Bénéfices
- **Réutilisation** : écrire une fois, utiliser partout.
- **Lisibilité** : découper une page complexe en unités simples.
- **Testabilité** : tester un composant isolément.
- **Maintenabilité** : limiter l’impact des changements.
- **Collaboration** : répartir le travail (un composant = une responsabilité).

### 1.3 Approche “component-driven”
On conçoit l’UI comme un arbre de composants :

- `App.vue`
  - `Layout`
    - `Navbar`
    - `Sidebar`
    - `Content`
      - `Card`
      - `Button`

C’est exactement ce que vous évoquez : **les composants peuvent être imbriqués**.

---

## 2. Anatomie d’un composant Vue (template, logique, style)

### 2.1 Single File Component (SFC)
Un composant s’écrit souvent dans un fichier `.vue` :

```vue
<template>
  <article class="card">
    <h2>{{ title }}</h2>
    <p v-if="description">{{ description }}</p>
  </article>
</template>

<script setup>
defineProps({
  title: { type: String, required: true },
  description: { type: String, default: '' }
})
</script>

<style scoped>
.card {
  border: 1px solid #ddd;
  padding: 16px;
  border-radius: 8px;
}
</style>
```

### 2.2 Rôles des trois sections
- **`<template>`** : ce qui est rendu (HTML + directives Vue).
- **`<script>` / `<script setup>`** : état, props, events, computed, watchers…
- **`<style>`** : styles (global, scoped, modules CSS...).

### 2.3 Encapsulation des styles
- `scoped` limite les styles au composant.
- Sans `scoped`, les styles sont globaux.

Bon usage : `scoped` pour des composants “UI”, global pour le design system global (variables, reset, typographie).

---

## 3. Créer des composants (SFC) et bonnes pratiques de base

### 3.1 Un composant = une responsabilité
Exemples :
- `BaseButton.vue` : bouton générique
- `UserCard.vue` : affiche un utilisateur
- `TodoList.vue` : liste de tâches

Éviter les composants “fourre-tout” (`Everything.vue`).

### 3.2 Naming & conventions
- Noms en **PascalCase** : `UserCard.vue`, `BaseButton.vue`.
- Composants “de base” préfixés : `Base*` (bouton, input, modal...).
- Composants “layout” : `AppLayout`, `MainHeader`.

### 3.3 Props immutables
- Une prop est une **entrée** : on ne la modifie pas directement dans l’enfant.
- Si l’enfant a besoin d’un état modifiable, il crée un état local basé sur la prop (ou utilise `v-model` événementialisé).

---

## 4. Enregistrer et utiliser un composant

### 4.1 Import local (recommandé)

```vue
<script setup>
import UserCard from '@/components/UserCard.vue'

const user = { id: 1, name: 'Ada Lovelace', role: 'Dev' }
</script>

<template>
  <UserCard :user="user" />
</template>
```

Avantage : explicite, facilite le tree-shaking.

### 4.2 Enregistrement global (à utiliser avec parcimonie)
Possible dans `main.js/ts` via `app.component('BaseButton', BaseButton)`.

Convient souvent aux composants **vraiment** transverses (`BaseButton`, `BaseIcon`).

---

## 5. Imbrication de composants (parent/enfant)

### 5.1 Exemple d’arbre
- `TodoPage.vue` (parent)
  - `TodoForm.vue` (enfant)
  - `TodoList.vue` (enfant)
    - `TodoItem.vue` (petit-enfant)

La page orchestre l’état global de la fonctionnalité, et délègue l’affichage/interaction à des sous-composants.

### 5.2 Exemple minimal

**Parent** :
```vue
<script setup>
import ProductCard from './ProductCard.vue'

const products = [
  { id: 1, name: 'Clavier', price: 79 },
  { id: 2, name: 'Souris', price: 39 }
]
</script>

<template>
  <section>
    <h1>Catalogue</h1>
    <div class="grid">
      <ProductCard
        v-for="p in products"
        :key="p.id"
        :product="p"
      />
    </div>
  </section>
</template>
```

**Enfant** :
```vue
<script setup>
defineProps({
  product: { type: Object, required: true }
})
</script>

<template>
  <article class="card">
    <h2>{{ product.name }}</h2>
    <p>{{ product.price }} €</p>
  </article>
</template>
```

---

## 6. Communication entre composants (props & events)

> Règle d’or : **les données descendent (props)**, **les actions remontent (events)**.

### 6.1 Props (parent → enfant)

```vue
<!-- Parent -->
<UserCard :user="user" />

<!-- Enfant -->
<script setup>
defineProps({
  user: { type: Object, required: true }
})
</script>
```

**Bonnes pratiques** :
- typer/valider les props (type, required, default)
- éviter les objets trop gros (privilégier explicitement les props pertinentes)

### 6.2 Events (enfant → parent)

**Enfant** :
```vue
<script setup>
const emit = defineEmits(['delete'])

function onDeleteClick() {
  emit('delete')
}
</script>

<template>
  <button type="button" @click="onDeleteClick">Supprimer</button>
</template>
```

**Parent** :
```vue
<TodoItem @delete="remove(todo.id)" />
```

### 6.3 Événements avec payload

**Enfant** :
```vue
<script setup>
const emit = defineEmits(['select'])

defineProps({ id: Number })

function select() {
  emit('select', { id: 42 })
}
</script>
```

**Parent** :
```vue
<UserRow @select="onSelect" />

<script setup>
function onSelect(payload) {
  // payload = { id: 42 }
}
</script>
```

### 6.4 `v-model` sur composant (pattern props + event)
`v-model` est un sucre syntaxique basés sur :
- prop `modelValue`
- event `update:modelValue`

**Enfant** :
```vue
<script setup>
const props = defineProps({
  modelValue: { type: String, default: '' }
})

const emit = defineEmits(['update:modelValue'])

function onInput(e) {
  emit('update:modelValue', e.target.value)
}
</script>

<template>
  <input :value="props.modelValue" @input="onInput" />
</template>
```

**Parent** :
```vue
<script setup>
import BaseInput from './BaseInput.vue'
import { ref } from 'vue'

const search = ref('')
</script>

<template>
  <BaseInput v-model="search" />
  <p>Recherche : {{ search }}</p>
</template>
```

---

## 7. Slots : composer des interfaces réutilisables

Les **slots** permettent de réutiliser une structure, tout en laissant le parent injecter du contenu.

### 7.1 Slot par défaut

**BaseCard.vue**
```vue
<template>
  <div class="card">
    <slot />
  </div>
</template>

<style scoped>
.card { border: 1px solid #ddd; padding: 16px; border-radius: 8px; }
</style>
```

**Utilisation**
```vue
<BaseCard>
  <h2>Titre</h2>
  <p>Contenu libre injecté par le parent.</p>
</BaseCard>
```

### 7.2 Slots nommés

**Modal.vue**
```vue
<template>
  <div class="modal">
    <header class="modal__header">
      <slot name="title" />
    </header>

    <section class="modal__body">
      <slot />
    </section>

    <footer class="modal__footer">
      <slot name="actions" />
    </footer>
  </div>
</template>
```

**Utilisation**
```vue
<Modal>
  <template #title>
    <h2>Confirmation</h2>
  </template>

  Voulez-vous vraiment supprimer cet élément ?

  <template #actions>
    <button>Annuler</button>
    <button class="danger">Supprimer</button>
  </template>
</Modal>
```

### 7.3 Slot avec props (scoped slot)
Le composant enfant expose des valeurs au parent.

```vue
<!-- DataList.vue -->
<script setup>
defineProps({
  items: { type: Array, default: () => [] }
})
</script>

<template>
  <ul>
    <li v-for="(item, index) in items" :key="index">
      <slot :item="item" :index="index">
        {{ item }}
      </slot>
    </li>
  </ul>
</template>
```

Utilisation :
```vue
<DataList :items="users">
  <template #default="{ item }">
    <strong>{{ item.name }}</strong>
  </template>
</DataList>
```

---

## 8. Composants dynamiques & rendu conditionnel

### 8.1 `component :is`
Permet de changer de composant à afficher.

```vue
<script setup>
import TabA from './TabA.vue'
import TabB from './TabB.vue'
import { computed, ref } from 'vue'

const active = ref('a')

const currentTab = computed(() => (active.value === 'a' ? TabA : TabB))
</script>

<template>
  <button @click="active = 'a'">Onglet A</button>
  <button @click="active = 'b'">Onglet B</button>

  <component :is="currentTab" />
</template>
```

### 8.2 `keep-alive`
Garder l’état des composants dynamiques (ex : formulaire).

```vue
<keep-alive>
  <component :is="currentTab" />
</keep-alive>
```

---

## 9. Organisation du code à l’échelle (structure de dossiers & conventions)

### 9.1 Structure recommandée (exemple)

```
src/
  components/
    base/
      BaseButton.vue
      BaseInput.vue
    layout/
      AppHeader.vue
      AppFooter.vue
  features/
    todos/
      components/
        TodoForm.vue
        TodoList.vue
        TodoItem.vue
      TodoPage.vue
  App.vue
  main.js
```

### 9.2 Séparer “UI générique” et “métier”
- **Base** : neutre, sans dépendance métier.
- **Feature** : lié à un domaine (todos, users, billing...).

### 9.3 Conventions de props/events
- Props : `kebab-case` en template, `camelCase` côté JS.
- Events : `kebab-case` (`@update:modelValue`, `@submit`, `@delete`).
- Documenter les contrats : quelles props attendues ? quels events émis ?

---

## 10. Atelier fil rouge : construire une mini-UI en composants

### Objectif
Créer un mini-module “Gestion de tâches” en composants imbriqués :

- `TodoPage` (orchestrateur)
- `TodoForm` (saisie)
- `TodoList` (liste)
- `TodoItem` (élément)

### 10.1 Modèle de données
Une todo :
```js
{ id: number, label: string, done: boolean }
```

### 10.2 `TodoItem.vue`
```vue
<script setup>
const props = defineProps({
  todo: { type: Object, required: true }
})

const emit = defineEmits(['toggle', 'remove'])
</script>

<template>
  <li class="todo-item">
    <label>
      <input
        type="checkbox"
        :checked="props.todo.done"
        @change="emit('toggle', props.todo.id)"
      />
      <span :class="{ done: props.todo.done }">{{ props.todo.label }}</span>
    </label>

    <button type="button" @click="emit('remove', props.todo.id)">
      Supprimer
    </button>
  </li>
</template>

<style scoped>
.todo-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 8px 0;
}
.done { text-decoration: line-through; opacity: 0.7; }
</style>
```

### 10.3 `TodoList.vue`
```vue
<script setup>
import TodoItem from './TodoItem.vue'

const props = defineProps({
  todos: { type: Array, default: () => [] }
})

const emit = defineEmits(['toggle', 'remove'])
</script>

<template>
  <ul>
    <TodoItem
      v-for="t in props.todos"
      :key="t.id"
      :todo="t"
      @toggle="emit('toggle', $event)"
      @remove="emit('remove', $event)"
    />
  </ul>
</template>
```

### 10.4 `TodoForm.vue` (avec `v-model` sur composant natif)

```vue
<script setup>
import { ref } from 'vue'

const emit = defineEmits(['add'])

const label = ref('')

function submit() {
  const value = label.value.trim()
  if (!value) return
  emit('add', value)
  label.value = ''
}
</script>

<template>
  <form @submit.prevent="submit" class="todo-form">
    <input v-model="label" placeholder="Nouvelle tâche..." />
    <button type="submit">Ajouter</button>
  </form>
</template>

<style scoped>
.todo-form { display: flex; gap: 8px; margin-bottom: 12px; }
.todo-form input { flex: 1; }
</style>
```

### 10.5 `TodoPage.vue` (parent orchestrateur)

```vue
<script setup>
import { ref } from 'vue'
import TodoForm from './components/TodoForm.vue'
import TodoList from './components/TodoList.vue'

const nextId = ref(3)
const todos = ref([
  { id: 1, label: 'Comprendre les composants', done: true },
  { id: 2, label: 'Pratiquer props & events', done: false }
])

function addTodo(label) {
  todos.value.unshift({ id: nextId.value++, label, done: false })
}

function toggleTodo(id) {
  const t = todos.value.find(x => x.id === id)
  if (t) t.done = !t.done
}

function removeTodo(id) {
  todos.value = todos.value.filter(x => x.id !== id)
}
</script>

<template>
  <section>
    <h1>Mes tâches</h1>

    <TodoForm @add="addTodo" />

    <TodoList
      :todos="todos"
      @toggle="toggleTodo"
      @remove="removeTodo"
    />
  </section>
</template>
```

### 10.6 Points pédagogiques à retenir
- `TodoPage` **possède l’état** et la logique.
- `TodoForm` **émet** un événement `add`.
- `TodoList` **relaye** les événements issus de `TodoItem`.
- `TodoItem` est un composant “feuille” : simple, focalisé.

---

## 11. Récapitulatif & checklist

### 11.1 À retenir
- Les composants structurent une app en **blocs réutilisables**.
- Chaque composant regroupe généralement **template + logique + style**.
- Les composants forment un **arbre**, et peuvent être **imbriqués**.
- Communication standard : **props down, events up**.
- `v-model` sur composant = `modelValue` + `update:modelValue`.
- Les **slots** permettent la composition d’UI.

### 11.2 Checklist qualité
- [ ] Nom explicite et responsabilité unique
- [ ] Props validées (type/default/required)
- [ ] Aucun “mutating prop”
- [ ] Événements documentés et cohérents
- [ ] Composants base vs feature bien séparés
- [ ] Styles encapsulés (`scoped`) quand pertinent

---

## Exercices proposés (optionnels)

1. **Refactor** : transformer une page monolithique en 3 composants.
2. **Custom Input** : créer un `BaseInput` compatible `v-model` + validation.
3. **Slot** : faire une `BaseCard` avec slot `title` + slot `actions`.
4. **Composant dynamique** : implémenter un système d’onglets.

---

### Fin du module
Si vous souhaitez, je peux produire la version **avec slides** (plan + messages clés) ou une version **TP guidé** avec étapes, tests et critères de validation.