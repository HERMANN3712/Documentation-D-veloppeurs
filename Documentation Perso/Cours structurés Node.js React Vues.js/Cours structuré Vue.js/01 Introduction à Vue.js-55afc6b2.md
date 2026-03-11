# Formation 01 — Introduction à Vue.js

> **Public cible** : développeurs (débutants à intermédiaires) en JavaScript souhaitant découvrir Vue.js
> 
> **Format** : cours + démos + exercices
> 
> **Durée indicative** : 1 journée (6–7h) ou 2 demi-journées
> 
> **Pré-requis** : JavaScript ES6, HTML/CSS, notions de DOM et de modules npm

---

## 1) Objectifs pédagogiques

À la fin de cette formation, l’apprenant sera capable de :

- Expliquer ce qu’est **Vue.js** et son positionnement (framework progressif, approche orientée UI)
- Mettre en place un projet Vue avec **Vite**
- Comprendre la structure d’une app Vue : **composants**, **template**, **script**, **style**
- Utiliser les fonctionnalités essentielles :
  - **Interpolation** et **directives** (`v-if`, `v-for`, `v-bind`, `v-on`, `v-model`)
  - **Réactivité** (state) et cycle de vie
  - **Props** et **events** pour communiquer entre composants
  - **Computed** et **watchers**
- Produire une mini-application (ex : Todo / catalogue) en appliquant les bonnes pratiques de base

---

## 2) Plan de la formation

1. **Introduction & contexte**
   - MVC, couche “View”, SPA et composants UI
   - Vue.js : philosophie “progressive”
2. **Installation & démarrage avec Vite**
   - Prérequis, outillage
   - Structure d’un projet
3. **Les bases de Vue**
   - Instance/app Vue, template, rendu
   - Réactivité : `ref`, `reactive`
4. **Templates & directives**
   - Interpolation, attributs dynamiques
   - Événements, formulaires, conditionnels, listes
5. **Composants**
   - SFC (`.vue`) et composition
   - Props, events, slots (notions)
6. **Computed & watchers**
   - Différences et cas d’usage
7. **Cycle de vie & effets**
   - Hooks principaux
8. **Mini projet guidé**
   - Conception, implémentation, améliorations
9. **Bonnes pratiques & suite**
   - Organisation, style, tests (pistes), écosystème (router, pinia)

---

# 3) Contenu détaillé (cours complet)

## Module 1 — Introduction & positionnement de Vue.js

### 1.1 Qu’est-ce que Vue.js ?

**Vue.js** est un framework JavaScript **progressif** pour construire des **interfaces utilisateur** et des **applications web modernes**. Il se concentre principalement sur la **couche “View”** du modèle **MVC** (Model–View–Controller).

- **View** : ce que l’utilisateur voit (UI), et comment cela réagit aux interactions.
- Vue est particulièrement efficace pour :
  - rendre un UI dynamique à partir de données,
  - structurer l’interface en **composants réutilisables**,
  - maintenir l’état de l’application grâce à un système de **réactivité**.

### 1.2 Pourquoi “progressif” ?

Vue est dit **progressif** car vous pouvez l’adopter :

1. **Dans une page existante**, en ajoutant un petit composant pour une zone spécifique.
2. **Dans une application complète**, structurée en SPA/MPA avec routing, state management, build tooling.

> Cela permet une adoption graduelle : pas d’obligation de “tout refaire”.

### 1.3 Vue vs autres approches

- **Vanilla JS + DOM** : fonctionne, mais devient complexe quand l’UI et l’état grandissent.
- **Vue** apporte :
  - un **DOM déclaratif** (template),
  - une **réactivité** pour synchroniser état ↔ UI,
  - une architecture en **composants**.

### 1.4 Vue 3 en bref

Vue 3 introduit (entre autres) :

- La **Composition API** (recommandée pour organiser la logique)
- Meilleures performances et meilleure typage (TypeScript friendly)

Dans ce cours, on utilise surtout **Vue 3 + Composition API**.

---

## Module 2 — Démarrage : installation et projet avec Vite

### 2.1 Pré-requis techniques

- Node.js (LTS conseillé)
- npm/pnpm/yarn
- Un IDE (VS Code recommandé)

### 2.2 Créer un projet Vue avec Vite

```bash
npm create vite@latest
```

Choisir :
- Framework : **Vue**
- Variant : **JavaScript** (ou TypeScript si souhaité)

Puis :

```bash
cd mon-projet-vue
npm install
npm run dev
```

### 2.3 Structure de projet (typique)

```txt
mon-projet-vue/
  index.html
  src/
    main.js
    App.vue
    assets/
    components/
  package.json
  vite.config.js
```

- `main.js` : point d’entrée, montage de l’app
- `App.vue` : composant racine
- `components/` : composants réutilisables

---

## Module 3 — Premiers pas : application, composant, template

### 3.1 Monter l’application

`src/main.js` :

```js
import { createApp } from 'vue'
import App from './App.vue'

createApp(App).mount('#app')
```

- `createApp(App)` crée l’application avec `App` comme composant racine.
- `.mount('#app')` attache l’app à l’élément `#app` du `index.html`.

### 3.2 SFC : Single File Components

Un composant Vue moderne est souvent un **SFC** (`.vue`) composé de :

```vue
<template>
  <h1>{{ title }}</h1>
</template>

<script setup>
import { ref } from 'vue'

const title = ref('Bonjour Vue !')
</script>

<style scoped>
h1 { color: #42b883; }
</style>
```

- `<template>` : UI déclarative
- `<script setup>` : logique du composant (Composition API)
- `<style scoped>` : styles limités au composant

---

## Module 4 — Réactivité : état et rendu

### 4.1 Le concept de réactivité

La réactivité signifie :

- vous modifiez une **donnée** (state)
- Vue met automatiquement à jour l’interface (DOM)

Sans vous obliger à manipuler le DOM manuellement.

### 4.2 `ref` (valeur primitive / simple)

```vue
<script setup>
import { ref } from 'vue'

const count = ref(0)
const increment = () => {
  count.value++
}
</script>

<template>
  <p>Compteur : {{ count }}</p>
  <button @click="increment">+1</button>
</template>
```

- `ref(0)` renvoie un objet réactif.
- Dans le script : on utilise `count.value`.
- Dans le template : Vue “unwrap” automatiquement le `.value`.

### 4.3 `reactive` (objets)

```js
import { reactive } from 'vue'

const user = reactive({
  name: 'Ada',
  role: 'Admin'
})

user.name = 'Grace'
```

---

## Module 5 — Templates & directives essentielles

Les **directives** Vue sont des attributs spéciaux (préfixe `v-`) qui ajoutent un comportement au DOM.

### 5.1 Interpolation

```vue
<p>Bonjour {{ userName }} !</p>
```

### 5.2 `v-bind` (binding d’attributs)

```vue
<img :src="avatarUrl" :alt="`Avatar de ${userName}`" />
```

`:` est un alias de `v-bind:`.

### 5.3 `v-on` (événements) + modificateurs

```vue
<button @click="doSomething">Clique</button>
<form @submit.prevent="onSubmit">
  ...
</form>
```

- `@` est un alias de `v-on:`
- `.prevent` évite le rechargement de page.

### 5.4 `v-if`, `v-else-if`, `v-else`

```vue
<p v-if="isLoggedIn">Bienvenue !</p>
<p v-else>Veuillez vous connecter.</p>
```

### 5.5 `v-show` (affichage conditionnel via CSS)

- `v-if` ajoute/retire du DOM.
- `v-show` garde dans le DOM et toggle `display: none`.

### 5.6 `v-for` (listes)

```vue
<ul>
  <li v-for="item in items" :key="item.id">
    {{ item.label }}
  </li>
</ul>
```

> `:key` est crucial pour la performance et la stabilité du rendu.

### 5.7 `v-model` (liaison bidirectionnelle)

```vue
<script setup>
import { ref } from 'vue'
const message = ref('')
</script>

<template>
  <input v-model="message" placeholder="Tapez..." />
  <p>Vous avez écrit : {{ message }}</p>
</template>
```

`v-model` gère :
- l’écoute d’événements `input/change`
- la mise à jour de la valeur

---

## Module 6 — Composants : composition, props, events

### 6.1 Pourquoi des composants ?

- Réutilisabilité
- Lisibilité
- Séparation des responsabilités
- Testabilité

### 6.2 Créer et utiliser un composant

`src/components/BaseButton.vue`

```vue
<template>
  <button class="btn" type="button">
    <slot />
  </button>
</template>

<style scoped>
.btn {
  padding: 8px 12px;
  border: 1px solid #ddd;
  border-radius: 8px;
  background: white;
}
</style>
```

Dans `App.vue` :

```vue
<script setup>
import BaseButton from './components/BaseButton.vue'
</script>

<template>
  <BaseButton>Valider</BaseButton>
</template>
```

### 6.3 Props (données parent → enfant)

`UserCard.vue` :

```vue
<script setup>
const props = defineProps({
  name: { type: String, required: true },
  role: { type: String, default: 'User' }
})
</script>

<template>
  <div class="card">
    <h3>{{ props.name }}</h3>
    <p>Rôle : {{ props.role }}</p>
  </div>
</template>
```

Utilisation :

```vue
<UserCard name="Ada" role="Admin" />
```

### 6.4 Events (enfant → parent)

`CounterPanel.vue` :

```vue
<script setup>
const emit = defineEmits(['increment'])
</script>

<template>
  <button @click="emit('increment')">Increment</button>
</template>
```

Parent :

```vue
<script setup>
import { ref } from 'vue'
import CounterPanel from './components/CounterPanel.vue'

const count = ref(0)
</script>

<template>
  <p>{{ count }}</p>
  <CounterPanel @increment="count++" />
</template>
```

### 6.5 Slots (injection de contenu)

- Déjà aperçu avec `<slot />`
- Sert à rendre un composant “contenant” flexible (cards, modals, layouts)

---

## Module 7 — Computed vs Watch

### 7.1 `computed` : une valeur dérivée

```vue
<script setup>
import { ref, computed } from 'vue'

const firstName = ref('Ada')
const lastName = ref('Lovelace')

const fullName = computed(() => `${firstName.value} ${lastName.value}`)
</script>

<template>
  <p>Nom complet : {{ fullName }}</p>
</template>
```

- Cache automatiquement le résultat tant que les dépendances ne changent pas.
- Idéal pour calculs dérivés et filtrages.

### 7.2 `watch` : réagir à un changement

```js
import { ref, watch } from 'vue'

const query = ref('')

watch(query, (newValue, oldValue) => {
  // ex: déclencher une recherche
  console.log('query changed', oldValue, '=>', newValue)
})
```

Cas d’usage :
- synchroniser avec une API
- enregistrer en localStorage
- déclencher un effet asynchrone

---

## Module 8 — Cycle de vie & effets

### 8.1 Hooks principaux

Avec `script setup`, on utilise :

- `onMounted` : après montage initial
- `onUpdated` : après une mise à jour du DOM
- `onUnmounted` : avant destruction

```js
import { onMounted, onUnmounted } from 'vue'

onMounted(() => {
  // fetch initial, abonnement events, etc.
})

onUnmounted(() => {
  // cleanup : removeEventListener, clearInterval...
})
```

---

# 4) Atelier / mini-projet guidé : Todo List (essentiels Vue)

## 4.1 Objectif

Créer une mini application “Todo” :

- Ajouter une tâche
- Marquer une tâche comme terminée
- Supprimer une tâche
- Filtrer (Toutes / Actives / Terminées)
- Persister dans `localStorage`

## 4.2 Modèle de données

```js
{
  id: 'uuid-ou-timestamp',
  label: 'Apprendre Vue',
  done: false
}
```

## 4.3 Implémentation (App.vue)

```vue
<script setup>
import { ref, computed, watch } from 'vue'

const STORAGE_KEY = 'todos-v1'

const newLabel = ref('')
const filter = ref('all') // 'all' | 'active' | 'done'

const todos = ref(loadTodos())

function loadTodos() {
  try {
    const raw = localStorage.getItem(STORAGE_KEY)
    return raw ? JSON.parse(raw) : []
  } catch {
    return []
  }
}

function saveTodos(list) {
  localStorage.setItem(STORAGE_KEY, JSON.stringify(list))
}

function addTodo() {
  const label = newLabel.value.trim()
  if (!label) return

  todos.value.unshift({
    id: String(Date.now()),
    label,
    done: false
  })

  newLabel.value = ''
}

function removeTodo(id) {
  todos.value = todos.value.filter(t => t.id !== id)
}

function toggleTodo(id) {
  const t = todos.value.find(t => t.id === id)
  if (t) t.done = !t.done
}

const filteredTodos = computed(() => {
  if (filter.value === 'active') return todos.value.filter(t => !t.done)
  if (filter.value === 'done') return todos.value.filter(t => t.done)
  return todos.value
})

const remainingCount = computed(() => todos.value.filter(t => !t.done).length)

watch(todos, (list) => saveTodos(list), { deep: true })
</script>

<template>
  <main class="container">
    <header>
      <h1>Todo — Vue.js</h1>
      <p>{{ remainingCount }} tâche(s) restante(s)</p>
    </header>

    <section class="add">
      <input
        v-model="newLabel"
        @keyup.enter="addTodo"
        placeholder="Nouvelle tâche..."
      />
      <button @click="addTodo">Ajouter</button>
    </section>

    <section class="filters">
      <button :class="{ active: filter === 'all' }" @click="filter = 'all'">Toutes</button>
      <button :class="{ active: filter === 'active' }" @click="filter = 'active'">Actives</button>
      <button :class="{ active: filter === 'done' }" @click="filter = 'done'">Terminées</button>
    </section>

    <ul class="list">
      <li v-for="t in filteredTodos" :key="t.id" class="item">
        <label>
          <input type="checkbox" :checked="t.done" @change="toggleTodo(t.id)" />
          <span :class="{ done: t.done }">{{ t.label }}</span>
        </label>

        <button class="danger" @click="removeTodo(t.id)">Supprimer</button>
      </li>
    </ul>

    <footer v-if="todos.length === 0" class="empty">
      Aucune tâche. Ajoutez-en une !
    </footer>
  </main>
</template>

<style scoped>
.container {
  max-width: 720px;
  margin: 24px auto;
  padding: 16px;
  font-family: system-ui, -apple-system, Segoe UI, Roboto, Arial, sans-serif;
}

.add {
  display: flex;
  gap: 8px;
  margin: 16px 0;
}

.add input {
  flex: 1;
  padding: 8px 10px;
  border: 1px solid #ddd;
  border-radius: 8px;
}

.filters {
  display: flex;
  gap: 8px;
  margin-bottom: 12px;
}

.filters button {
  padding: 6px 10px;
  border-radius: 999px;
  border: 1px solid #ddd;
  background: #fff;
}

.filters button.active {
  border-color: #42b883;
  color: #2c7a5b;
}

.list {
  list-style: none;
  padding: 0;
  margin: 0;
}

.item {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
  padding: 10px 0;
  border-bottom: 1px solid #f0f0f0;
}

.done {
  text-decoration: line-through;
  color: #999;
}

button {
  padding: 8px 12px;
  border:  1px solid #ddd;
  border-radius: 8px;
  background: white;
  cursor: pointer;
}

button.danger {
  border-color: #ffcccc;
  color: #a40000;
}

.empty {
  margin-top: 16px;
  color: #666;
}
</style>
```

## 4.4 Points pédagogiques couverts

- `ref` pour l’état (`todos`, `newLabel`, `filter`)
- `v-model` pour la saisie
- `v-for` + `:key` pour le rendu des items
- `@click`, `@keyup.enter` pour les événements
- `computed` pour le filtrage et les métriques (`remainingCount`)
- `watch(..., { deep: true })` pour persister dans `localStorage`

### Exercices (progressifs)

1. Ajouter un bouton “Tout terminer”
2. Ajouter un bouton “Vider les tâches terminées”
3. Ajouter une validation (longueur min, message d’erreur)
4. Extraire des composants : `TodoItem`, `TodoFilters`, `TodoAdd`

---

# 5) Bonnes pratiques (niveau introduction)

## 5.1 Organisation

- Un composant = une responsabilité
- Nommer clairement : `BaseButton`, `UserCard`, `TodoItem`
- Centraliser les constantes (clés storage, enum filters)

## 5.2 Réactivité : recommandations

- Préférer `computed` aux fonctions recalculées dans le template
- Éviter les mutations complexes non contrôlées
- Toujours poser une `key` stable sur les listes

## 5.3 Style & qualité

- Utiliser ESLint/Prettier (optionnel mais recommandé)
- Écrire des composants petits et testables

---

# 6) Pour aller plus loin (suite logique)

- **Vue Router** : navigation (SPA)
- **Pinia** : state management partagé
- **Composables** : factoriser logique réutilisable
- **TypeScript** avec Vue
- Appels API : `fetch/axios`, gestion d’erreurs, loading states
- Tests : Vitest + Vue Test Utils

---

# 7) Annexes

## 7.1 Aide-mémoire : directives courantes

- `{{ ... }}` : interpolation
- `:attr="..."` : `v-bind`
- `@event="..."` : `v-on`
- `v-if / v-else-if / v-else`
- `v-show`
- `v-for="item in items" :key="item.id"`
- `v-model="state"`

## 7.2 Glossaire

- **SFC** : Single File Component (`.vue`)
- **SPA** : Single Page Application
- **Réactivité** : mise à jour automatique UI selon l’état
- **Props** : entrées d’un composant
- **Emit** : sortie (événement) d’un composant

---

## Fin de la formation

Ce support constitue une **introduction** : il donne les bases nécessaires pour commencer à développer une application Vue.js et comprendre l’architecture par composants.
