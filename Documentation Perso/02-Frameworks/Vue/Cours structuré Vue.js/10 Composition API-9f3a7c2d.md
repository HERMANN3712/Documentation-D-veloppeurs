# Formation Vue.js — Composition API (Vue 3)

> **Objectif** : Maîtriser la **Composition API** pour organiser la logique de composants de manière plus flexible via `setup()`, améliorer la **réutilisabilité** du code et la **maintenabilité** d’applications Vue complexes.

---

## 1) Informations générales

- **Public** : Développeurs ayant déjà pratiqué Vue (Options API ou Vue 3 basique)
- **Prérequis** : JavaScript ES6+, bases Vue (template, props, events), notions de composants
- **Durée conseillée** : 1 jour (7h) ou 2 demi-journées
- **Stack** : Vue 3, Vite, TypeScript optionnel

---

## 2) Plan de formation

1. **Introduction et motivations**
2. **`setup()` et le modèle mental de la Composition API**
3. **Réactivité : `ref`, `reactive`, `computed`, `watch`, `watchEffect`**
4. **Props, emits et interfaces de composant en Composition API**
5. **Lifecycle hooks côté Composition API**
6. **Templates et accès aux refs de template**
7. **Composables : réutilisation de logique (le vrai "super-pouvoir")**
8. **Gestion d’état : patterns avec composables et Pinia (aperçu)**
9. **Bonnes pratiques, pièges et conventions**
10. **Atelier guidé : refactor d’un composant Options API → Composition API**
11. **Annexes : checklists, snippets, ressources**

---

## 3) Introduction et motivations

### 3.1 Pourquoi la Composition API ?

La **Composition API** a été introduite pour répondre à des problèmes fréquents dans les applications Vue qui grandissent :

- **Logique dispersée** : avec l’Options API, une même fonctionnalité est souvent répartie dans `data`, `methods`, `computed`, `watch`, etc.
- **Réutilisation difficile** : les mixins créent des collisions de noms et rendent le code moins explicite.
- **Maintenabilité** : plus un composant est grand, plus il est difficile de regrouper et faire évoluer une fonctionnalité.

### 3.2 Ce que vous gagnez

- **Organisation par fonctionnalité** (et non par option)
- **Composables** : extraction de logique réutilisable de manière claire et typée
- Meilleur support **TypeScript**
- Composants plus **testables** (logique isolée)

---

## 4) `setup()` et le modèle mental

### 4.1 Rôle de `setup()`

`setup()` est le point d’entrée de la Composition API dans un composant. On y :

- déclare l’état (réactif)
- écrit la logique (fonctions)
- expose des valeurs au template

> Tout ce qui est retourné par `setup()` est accessible dans le template.

### 4.2 Exemple minimal

```vue
<script>
import { ref } from 'vue'

export default {
  setup() {
    const count = ref(0)

    function increment() {
      count.value++
    }

    return { count, increment }
  }
}
</script>

<template>
  <button @click="increment">Count: {{ count }}</button>
</template>
```

### 4.3 Version recommandée : `<script setup>`

Vue 3 propose une syntaxe SFC dédiée qui rend tout plus concis.

```vue
<script setup>
import { ref } from 'vue'

const count = ref(0)
const increment = () => count.value++
</script>

<template>
  <button @click="increment">Count: {{ count }}</button>
</template>
```

**Règles clés** :
- Pas de `return` explicite : les variables du script sont directement exposées au template.
- Code plus lisible et adapté à TypeScript.

---

## 5) Réactivité en profondeur

### 5.1 `ref()`

- Pour les **primitives** (nombre, string, boolean) et aussi des objets si besoin.
- Accès via `.value` en JavaScript.

```js
import { ref } from 'vue'

const name = ref('Ada')
name.value = 'Grace'
```

Dans le template, Vue "déplie" automatiquement `.value` :

```html
<p>{{ name }}</p>
```

### 5.2 `reactive()`

- Pour créer un **objet réactif** (proxy).
- Accès direct aux propriétés.

```js
import { reactive } from 'vue'

const form = reactive({
  email: '',
  password: ''
})

form.email = 'a@b.com'
```

**Piège courant** : déstructurer un objet `reactive()` casse la réactivité.

```js
const { email } = form // ❌ email n’est plus réactif
```

Solution : `toRefs()`

```js
import { toRefs } from 'vue'

const { email, password } = toRefs(form) // ✅ email/password sont des refs
```

### 5.3 `computed()`

- Valeur dérivée, mise en cache, recalculée si dépendances changent.

```js
import { ref, computed } from 'vue'

const price = ref(100)
const quantity = ref(2)

const total = computed(() => price.value * quantity.value)
```

Computed writable (getter + setter) :

```js
const fullName = computed({
  get: () => `${first.value} ${last.value}`,
  set: (v) => {
    const [f, l] = v.split(' ')
    first.value = f
    last.value = l
  }
})
```

### 5.4 `watch()`

- Observe une source et exécute une fonction quand elle change.

```js
import { ref, watch } from 'vue'

const query = ref('')

watch(query, (newValue, oldValue) => {
  console.log('query changed:', oldValue, '->', newValue)
})
```

Sur plusieurs sources :

```js
watch([a, b], ([newA, newB]) => {
  // ...
})
```

Options utiles :

```js
watch(query, () => {}, { immediate: true })
watch(obj, () => {}, { deep: true }) // à utiliser avec prudence
```

### 5.5 `watchEffect()`

- Exécute automatiquement et **traque** les dépendances utilisées.
- Pratique pour synchroniser un effet.

```js
import { ref, watchEffect } from 'vue'

const id = ref(1)

watchEffect(async (onCleanup) => {
  const controller = new AbortController()
  onCleanup(() => controller.abort())

  const res = await fetch(`/api/users/${id.value}`, { signal: controller.signal })
  // ...
})
```

**Choisir entre `watch` et `watchEffect`**
- `watch` : quand on veut contrôler précisément la/les sources.
- `watchEffect` : quand on veut un effet synchronisé sans déclarer explicitement les sources.

---

## 6) Props et emits en Composition API

### 6.1 Avec `<script setup>`

```vue
<script setup>
const props = defineProps({
  label: { type: String, required: true },
  modelValue: { type: Number, default: 0 }
})

const emit = defineEmits(['update:modelValue'])

function inc() {
  emit('update:modelValue', props.modelValue + 1)
}
</script>

<template>
  <button @click="inc">{{ label }}: {{ modelValue }}</button>
</template>
```

### 6.2 `v-model` et conventions

- `v-model` correspond au couple :
  - prop `modelValue`
  - event `update:modelValue`

Pour plusieurs v-model :

```vue
<script setup>
const props = defineProps({
  title: String,
  open: Boolean
})
const emit = defineEmits(['update:title', 'update:open'])
</script>
```

### 6.3 Avec TypeScript (aperçu)

```ts
<script setup lang="ts">
const props = defineProps<{
  label: string
  modelValue?: number
}>()

const emit = defineEmits<{
  (e: 'update:modelValue', v: number): void
}>()
</script>
```

---

## 7) Lifecycle hooks

En Composition API, les hooks sont des fonctions importées :

- `onMounted`
- `onUpdated`
- `onUnmounted`
- `onBeforeMount`, `onBeforeUpdate`, `onBeforeUnmount`

```vue
<script setup>
import { onMounted, onUnmounted } from 'vue'

function onResize() {
  console.log(window.innerWidth)
}

onMounted(() => {
  window.addEventListener('resize', onResize)
})

onUnmounted(() => {
  window.removeEventListener('resize', onResize)
})
</script>
```

---

## 8) Template refs et interactions DOM

### 8.1 Référence à un élément

```vue
<script setup>
import { ref, onMounted } from 'vue'

const inputEl = ref(null)

onMounted(() => {
  inputEl.value?.focus()
})
</script>

<template>
  <input ref="inputEl" />
</template>
```

### 8.2 Référence à un composant enfant

```vue
<script setup>
import { ref } from 'vue'
import Child from './Child.vue'

const childRef = ref(null)
</script>

<template>
  <Child ref="childRef" />
</template>
```

> Pour exposer explicitement des méthodes côté enfant : `defineExpose({ ... })`.

---

## 9) Composables : réutiliser la logique

### 9.1 Définition

Un **composable** est une fonction (souvent nommée `useXxx`) qui encapsule :

- état réactif
- computed
- watchers
- hooks
- fonctions utilitaires

But : **partager une fonctionnalité** entre composants sans mixins.

### 9.2 Exemple : `useCounter()`

`src/composables/useCounter.js`

```js
import { ref, computed } from 'vue'

export function useCounter(initial = 0) {
  const count = ref(initial)
  const doubled = computed(() => count.value * 2)

  function inc() {
    count.value++
  }

  function dec() {
    count.value--
  }

  return { count, doubled, inc, dec }
}
```

Utilisation :

```vue
<script setup>
import { useCounter } from '@/composables/useCounter'

const { count, doubled, inc, dec } = useCounter(10)
</script>

<template>
  <div>count: {{ count }} / doubled: {{ doubled }}</div>
  <button @click="inc">+</button>
  <button @click="dec">-</button>
</template>
```

### 9.3 Composable avec side-effects + cleanup : `useEventListener()`

```js
import { onMounted, onUnmounted } from 'vue'

export function useEventListener(target, event, handler) {
  onMounted(() => target.addEventListener(event, handler))
  onUnmounted(() => target.removeEventListener(event, handler))
}
```

Usage :

```js
useEventListener(window, 'resize', () => console.log('resize'))
```

### 9.4 Bonnes pratiques Composables

- Préfixer par `use` (`useAuth`, `useFetch`, `useForm`)
- Retourner uniquement ce qui est utile
- Documenter les paramètres et valeurs retournées
- Gérer le cleanup (events, timers, subscriptions)
- Éviter la magie : rester explicite

---

## 10) Gestion d’état (pattern) : composables vs store

### 10.1 État partagé via composable (singleton)

```js
// useSession.js
import { ref } from 'vue'

const user = ref(null)

export function useSession() {
  function login(u) { user.value = u }
  function logout() { user.value = null }
  return { user, login, logout }
}
```

- Avantage : simple, léger.
- Limite : peut devenir difficile à structurer dans de grosses apps.

### 10.2 Aperçu Pinia

Quand l’app devient plus grande : **Pinia**.
- actions, getters, state
- devtools
- meilleure structuration

---

## 11) Bonnes pratiques & pièges

### 11.1 Conseils d’organisation

- Regrouper la logique **par fonctionnalité** (ex: `useSearch`, `usePagination`)
- Ne pas sur-découper : une composition trop fine peut nuire à la lisibilité
- Nommer clairement : `isLoading`, `error`, `data`

### 11.2 Pièges courants

1. **Oublier `.value`** en JS
2. **Déstructurer un `reactive`** sans `toRefs`
3. **`watch` deep** sur de gros objets → perf
4. **Effets non nettoyés** (listeners, intervals)
5. **Complexifier trop tôt** : utiliser Composition API ne rend pas automatiquement l’architecture meilleure

### 11.3 Convention recommandée

- Préférer `<script setup>`
- Utiliser `ref` pour primitives, `reactive` pour objets structurés
- Utiliser `computed` pour dérivés
- Utiliser `watch` pour sync & side-effects contrôlés

---

## 12) Atelier guidé : refactor Options API → Composition API

### 12.1 Composant Options API (avant)

```vue
<script>
export default {
  props: {
    initial: { type: Number, default: 0 }
  },
  data() {
    return {
      count: this.initial,
      step: 1
    }
  },
  computed: {
    total() {
      return this.count * this.step
    }
  },
  watch: {
    step() {
      console.log('step changed')
    }
  },
  methods: {
    inc() {
      this.count += 1
    }
  },
  mounted() {
    console.log('mounted')
  }
}
</script>

<template>
  <div>
    <p>count: {{ count }}</p>
    <p>total: {{ total }}</p>
    <input type="number" v-model.number="step" />
    <button @click="inc">+</button>
  </div>
</template>
```

### 12.2 Composition API (après) avec `<script setup>`

```vue
<script setup>
import { ref, computed, watch, onMounted } from 'vue'

const props = defineProps({
  initial: { type: Number, default: 0 }
})

const count = ref(props.initial)
const step = ref(1)

const total = computed(() => count.value * step.value)

watch(step, () => {
  console.log('step changed')
})

function inc() {
  count.value += 1
}

onMounted(() => {
  console.log('mounted')
})
</script>

<template>
  <div>
    <p>count: {{ count }}</p>
    <p>total: {{ total }}</p>
    <input type="number" v-model.number="step" />
    <button @click="inc">+</button>
  </div>
</template>
```

### 12.3 Analyse

- Tout est regroupé dans un seul espace (`setup`) : lecture plus "fonctionnelle".
- Facile d’extraire de la logique dans un **composable** (ex: `useCounter`).

---

## 13) Exercices (avec corrigés suggérés)

### Exercice 1 — Convertir un composant

- Prendre un composant existant Options API
- Le convertir en `<script setup>`
- Identifier : état, computed, watch, events

**Critères** : le comportement doit rester identique.

### Exercice 2 — Écrire un composable `useFetch`

**Spécifications** :
- `useFetch(url)` retourne `{ data, error, isLoading, refresh }`
- annule la requête précédente si `url` change

Pseudo-corrigé :

```js
import { ref, watch, unref } from 'vue'

export function useFetch(urlRef) {
  const data = ref(null)
  const error = ref(null)
  const isLoading = ref(false)

  async function run() {
    isLoading.value = true
    error.value = null

    try {
      const url = unref(urlRef)
      const res = await fetch(url)
      data.value = await res.json()
    } catch (e) {
      error.value = e
    } finally {
      isLoading.value = false
    }
  }

  watch(() => unref(urlRef), run, { immediate: true })

  return { data, error, isLoading, refresh: run }
}
```

---

## 14) Checklist de fin de module

- [ ] Je sais expliquer le rôle de `setup()`
- [ ] Je maîtrise `ref` vs `reactive`
- [ ] Je sais écrire `computed`, `watch`, `watchEffect`
- [ ] Je sais gérer props/emits avec `<script setup>`
- [ ] Je sais créer et utiliser un composable
- [ ] Je connais les principaux pièges (déstructuration, `.value`, cleanup)

---

## 15) Ressources

- Documentation Vue 3 — Composition API : https://vuejs.org/guide/extras/composition-api-faq.html
- `<script setup>` : https://vuejs.org/api/sfc-script-setup.html
- Réactivité en profondeur : https://vuejs.org/guide/essentials/reactivity-fundamentals.html

---

**Fin de la formation.**
