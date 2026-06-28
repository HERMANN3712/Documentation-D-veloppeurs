# Formation Vue.js — Computed et Watch (Vue 3)

## Objectifs pédagogiques
À la fin de cette formation, vous saurez :

- Expliquer la différence entre **données**, **computed**, **methods** et **watchers**.
- Créer des **propriétés computed** performantes (caching, getters/setters).
- Utiliser **watch** et **watchEffect** pour réagir aux changements (effets secondaires, appels API, synchronisation).
- Éviter les anti‑patterns (watch inutile, computed avec effets, boucles de watch, sur‑réactivité).
- Structurer proprement la logique de dérivation et d’observation dans des composants Vue.

## Pré‑requis
- Connaissances de base de Vue.js (composants, `data`/`setup`, `props`, `v-model`, directives).
- Compréhension des principes de réactivité.

## Public
Développeurs et développeuses web (intermédiaire) souhaitant maîtriser la logique réactive avancée dans Vue.js.

## Durée recommandée
2h30 à 4h selon la pratique guidée.

---

# Plan de la formation

1. **Rappels : réactivité et cycle d’update**
2. **Computed : valeurs dérivées**
   1. Pourquoi computed ?
   2. Computed vs methods
   3. Computed en Options API
   4. Computed en Composition API
   5. Getters/Setters (computed writable)
   6. Bonnes pratiques et pièges
3. **Watch : observer et déclencher des effets**
   1. Quand utiliser watch ?
   2. Watch en Options API
   3. Watch en Composition API (`watch`)
   4. `watchEffect`
   5. Options utiles : `immediate`, `deep`, `flush`, nettoyage
   6. Annulation d’appels asynchrones et prévention des races
   7. Pièges et anti‑patterns
4. **Atelier guidé : mini‑app de recherche + filtres**
5. **Checklist de décision : computed ou watch ?**

---

# 1) Rappels : réactivité et cycle d’update

Vue suit les **dépendances réactives** : lorsqu’une valeur réactive est lue lors d’un rendu ou d’un effet, Vue enregistre cette dépendance. Quand la valeur change, Vue sait **quoi recalculer**.

Dans ce contexte :

- **Computed** : "je veux une **valeur dérivée**" (pure, sans effet secondaire) → Vue peut la **mémoriser (cache)** et ne la recalculer que si ses dépendances changent.
- **Watch** : "je veux **exécuter du code** quand une valeur change" (effet secondaire : API, storage, router, logs, etc.).

---

# 2) Computed : valeurs dérivées

## 2.1 Pourquoi computed ?

Une propriété computed sert à:

- Centraliser une logique de calcul (formatage, agrégation, filtres, tri, mapping…)
- Garantir la **cohérence** : la valeur dérivée suit automatiquement les données sources
- Améliorer les performances : **cache** tant que les dépendances ne changent pas

### Exemple : total de panier
Vous avez un tableau d’articles réactif. Le total doit se recalculer automatiquement.

---

## 2.2 Computed vs methods

### Différence clé : caching
- **Computed** : résultat mis en cache → recalcul uniquement si dépendances changent.
- **Method** : recalcul à chaque render / appel.

Quand préférer :

- **computed** : valeur utilisée dans le template, ou réutilisée à plusieurs endroits, ou coûteuse.
- **method** : action/traitement ponctuel, dépend d’arguments, ou n’est pas une valeur dérivée stable.

---

## 2.3 Computed en Options API

```vue
<template>
  <div>
    <p>Prénom: {{ firstName }}</p>
    <p>Nom: {{ lastName }}</p>
    <p>Nom complet: {{ fullName }}</p>
  </div>
</template>

<script>
export default {
  data() {
    return {
      firstName: 'Ada',
      lastName: 'Lovelace'
    }
  },
  computed: {
    fullName() {
      return `${this.firstName} ${this.lastName}`
    }
  }
}
</script>
```

**Points importants :**
- `fullName` est une propriété : `{{ fullName }}` (pas `fullName()`).
- Vue sait que `fullName` dépend de `firstName` et `lastName`.

---

## 2.4 Computed en Composition API

Dans `setup()` avec `ref`/`reactive` et `computed()`.

```vue
<script setup>
import { ref, computed } from 'vue'

const firstName = ref('Ada')
const lastName = ref('Lovelace')

const fullName = computed(() => `${firstName.value} ${lastName.value}`)
</script>

<template>
  <p>{{ fullName }}</p>
</template>
```

**Notes :**
- Dans le template, `fullName` est automatiquement “unwrapped”, pas besoin de `.value`.
- Dans le JS, `fullName.value`.

---

## 2.5 Getters/Setters : computed writable

Un computed peut avoir un **getter** (lecture) et un **setter** (écriture) pour créer une API pratique, par exemple avec un champ `v-model`.

### Options API

```vue
<template>
  <input v-model="fullName" />
  <p>Prénom: {{ firstName }}</p>
  <p>Nom: {{ lastName }}</p>
</template>

<script>
export default {
  data() {
    return {
      firstName: 'Ada',
      lastName: 'Lovelace'
    }
  },
  computed: {
    fullName: {
      get() {
        return `${this.firstName} ${this.lastName}`
      },
      set(value) {
        const [first, ...rest] = value.split(' ')
        this.firstName = first || ''
        this.lastName = rest.join(' ') || ''
      }
    }
  }
}
</script>
```

### Composition API

```vue
<script setup>
import { ref, computed } from 'vue'

const firstName = ref('Ada')
const lastName = ref('Lovelace')

const fullName = computed({
  get: () => `${firstName.value} ${lastName.value}`,
  set: (value) => {
    const [first, ...rest] = value.split(' ')
    firstName.value = first || ''
    lastName.value = rest.join(' ') || ''
  }
})
</script>

<template>
  <input v-model="fullName" />
</template>
```

**Quand utiliser ?**
- Lorsque l’interface utilisateur manipule une **valeur combinée**, mais que le state reste **normalisé**.

---

## 2.6 Bonnes pratiques et pièges (Computed)

### Bonnes pratiques
- Garder un computed **pur** : pas d’appel API, pas de `localStorage`, pas de mutation.
- Préférer computed à un état “dupliqué”.
- Utiliser computed pour :
  - filtres/tri sur une liste
  - dérivations (totaux, compteurs)
  - formatage (dates, devise)

### Pièges fréquents
- **Dupliquer le state** : stocker `fullName` en data + `firstName/lastName` crée des incohérences.
- Mettre des **effets secondaires** dans computed (log, fetch) : risque de comportements imprévisibles.
- Confondre `method` et `computed` : si vous affichez le résultat dans le template, computed est souvent le bon choix.

---

# 3) Watch : observer et déclencher des effets

## 3.1 Quand utiliser watch ?

Utilisez un watcher quand vous devez exécuter un **effet secondaire** suite à un changement :

- Appel API quand un filtre change
- Mise à jour du `localStorage`
- Synchronisation avec le router (query params)
- Validation / analytics / logs
- Déclencher une animation ou un side effect

Règle mnémotechnique :

> **Computed = calcul** (valeur)
> 
> **Watch = réaction** (effet)

---

## 3.2 Watch en Options API

### Watch simple

```vue
<script>
export default {
  data() {
    return { query: '' }
  },
  watch: {
    query(newValue, oldValue) {
      console.log('query a changé:', oldValue, '→', newValue)
      // ex: déclencher une recherche
    }
  }
}
</script>
```

### Avec `immediate`

```js
watch: {
  query: {
    immediate: true,
    handler(newValue) {
      // exécuté au montage ET à chaque changement
      this.fetchResults(newValue)
    }
  }
}
```

### Watcher “deep” (objets)

```js
data() {
  return {
    filters: {
      minPrice: 0,
      maxPrice: 100
    }
  }
},
watch: {
  filters: {
    deep: true,
    handler(newFilters) {
      this.fetchResults(newFilters)
    }
  }
}
```

**Attention** : `deep` peut être coûteux sur des objets volumineux.

---

## 3.3 Watch en Composition API (`watch`)

### Watcher sur un `ref`

```vue
<script setup>
import { ref, watch } from 'vue'

const query = ref('')

watch(query, (newValue, oldValue) => {
  console.log(oldValue, '→', newValue)
})
</script>
```

### Watcher sur une fonction getter (reactive)

```vue
<script setup>
import { reactive, watch } from 'vue'

const filters = reactive({ min: 0, max: 100 })

watch(
  () => filters.min,
  (min) => {
    console.log('min a changé:', min)
  }
)
</script>
```

### Watcher multi‑sources

```js
watch(
  [() => filters.min, () => filters.max],
  ([min, max], [oldMin, oldMax]) => {
    console.log('min/max:', oldMin, oldMax, '→', min, max)
  }
)
```

---

## 3.4 `watchEffect` : effet auto‑dépendant

`watchEffect` exécute une fonction immédiatement et **tracke automatiquement** les valeurs réactives lues.

```vue
<script setup>
import { ref, watchEffect } from 'vue'

const query = ref('')

watchEffect(() => {
  // dès que query.value est lu ici, il devient une dépendance
  console.log('Recherche pour:', query.value)
})
</script>
```

### Quand préférer `watchEffect` ?
- Vous voulez un effet immédiatement au montage.
- Vous ne voulez pas lister explicitement les sources.

### Quand éviter ?
- Si vous avez besoin de `oldValue`.
- Si vous voulez contrôler précisément *quoi* déclenche l’effet.

---

## 3.5 Options utiles : `immediate`, `deep`, `flush`, nettoyage

### Nettoyage (cleanup) et annulation
En Composition API, `watch` et `watchEffect` acceptent un `onCleanup` (alias courant `onInvalidate`).

```js
import { ref, watch } from 'vue'

const query = ref('')

watch(query, async (q, _oldQ, onCleanup) => {
  let cancelled = false
  onCleanup(() => { cancelled = true })

  const res = await fetch(`/api/search?q=${encodeURIComponent(q)}`)
  const data = await res.json()

  if (cancelled) return
  // appliquer le résultat seulement si non annulé
  console.log('Résultats:', data)
})
```

### `flush`
- `flush: 'pre'` (par défaut) : avant le rendu.
- `flush: 'post'` : après le rendu (utile si vous devez lire le DOM à jour).
- `flush: 'sync'` : synchrone (à utiliser avec prudence).

```js
watch(query, () => {
  // ...
}, { flush: 'post' })
```

---

## 3.6 Prévenir les boucles et anti‑patterns

### Anti‑pattern : “watch pour calculer une valeur dérivée”

```js
// Mauvaise idée : dupliquer un état dérivé
watch(firstName, (v) => {
  fullName.value = v + ' ' + lastName.value
})
```

**Correct :**

```js
const fullName = computed(() => `${firstName.value} ${lastName.value}`)
```

### Anti‑pattern : muter la même source dans son watcher

```js
watch(query, (q) => {
  query.value = q.trim() // risque de boucle / comportements étranges
})
```

**Alternative :**
- Normaliser à l’entrée (dans l’input avec `@change`/
  `@blur`) ou utiliser un computed writable.

### Watch deep sur gros objets
- Préférer watcher des champs spécifiques.
- Ou utiliser des structures immuables / snapshots ciblés.

---

# 4) Atelier guidé : mini‑app de recherche + filtres

Objectif : mettre en pratique computed + watch en séparant clairement :
- **computed** pour dériver l’affichage
- **watch** pour les effets (appel API, persistance)

## 4.1 Énoncé
Vous avez :
- Une liste d’articles (mock)
- Un champ de recherche
- Un filtre “prix max”
- Une pagination simple

Attendus :
- La liste visible est une **computed** basée sur `items`, `query`, `maxPrice`, `page`.
- Un **watch** sauvegarde `query` et `maxPrice` dans `localStorage`.
- Un **watch** déclenche un fetch (mock) au changement de `query` (debounce simple).

## 4.2 Implémentation (Composition API)

```vue
<script setup>
import { ref, computed, watch } from 'vue'

// --- State ---
const items = ref([
  { id: 1, name: 'Clavier', price: 80 },
  { id: 2, name: 'Souris', price: 35 },
  { id: 3, name: 'Écran', price: 220 },
  { id: 4, name: 'Casque', price: 120 },
  { id: 5, name: 'Webcam', price: 60 }
])

const query = ref(localStorage.getItem('query') ?? '')
const maxPrice = ref(Number(localStorage.getItem('maxPrice') ?? 999))

const page = ref(1)
const pageSize = 2

// --- Computed : filtrage/tri/pagination ---
const filtered = computed(() => {
  const q = query.value.trim().toLowerCase()
  return items.value
    .filter(i => (q ? i.name.toLowerCase().includes(q) : true))
    .filter(i => i.price <= maxPrice.value)
})

const totalPages = computed(() => Math.max(1, Math.ceil(filtered.value.length / pageSize)))

const paginated = computed(() => {
  const start = (page.value - 1) * pageSize
  return filtered.value.slice(start, start + pageSize)
})

// --- Watch : garder page dans les bornes si filtres changent ---
watch([filtered, page], () => {
  if (page.value > totalPages.value) page.value = totalPages.value
  if (page.value < 1) page.value = 1
})

// --- Watch : persistance localStorage (effet secondaire) ---
watch([query, maxPrice], ([q, mp]) => {
  localStorage.setItem('query', q)
  localStorage.setItem('maxPrice', String(mp))
})

// --- Watch : simulation fetch avec debounce ---
let timeoutId
watch(query, (q, _old, onCleanup) => {
  clearTimeout(timeoutId)

  let cancelled = false
  onCleanup(() => {
    cancelled = true
    clearTimeout(timeoutId)
  })

  timeoutId = setTimeout(async () => {
    // Simule un appel réseau
    const result = await new Promise(resolve => {
      setTimeout(() => resolve({ ok: true, q }), 200)
    })

    if (cancelled) return
    console.log('Fetch terminé pour q=', result.q)
  }, 300)
})
</script>

<template>
  <section>
    <h2>Recherche</h2>

    <label>
      Query
      <input v-model="query" placeholder="ex: écran" />
    </label>

    <label>
      Prix max
      <input type="number" v-model.number="maxPrice" />
    </label>

    <h3>Résultats ({{ filtered.length }})</h3>

    <ul>
      <li v-for="item in paginated" :key="item.id">
        {{ item.name }} — {{ item.price }}€
      </li>
    </ul>

    <nav>
      <button @click="page--" :disabled="page <= 1">Précédent</button>
      <span>Page {{ page }} / {{ totalPages }}</span>
      <button @click="page++" :disabled="page >= totalPages">Suivant</button>
    </nav>
  </section>
</template>
```

### Débrief : où est quoi ?
- `filtered`, `totalPages`, `paginated` → **computed** (valeurs dérivées)
- persistance `localStorage` + simulation fetch → **watch** (effets secondaires)
- correction de `page` → **watch** (réaction à un changement)

---

# 5) Checklist : computed ou watch ?

## Utiliser **computed** si…
- Vous calculez une **valeur** à partir d’autres valeurs réactives.
- Le résultat est utilisé dans le template ou dans d’autres computed.
- Vous voulez bénéficier du **cache**.

Exemples :
- `isFormValid`, `cartTotal`, `filteredProducts`, `fullName`.

## Utiliser **watch** si…
- Vous devez produire un **effet secondaire**.
- Vous synchronisez quelque chose **hors du state** (API, router, storage, DOM, logs).
- Vous avez besoin de l’**ancienne valeur**, d’un **debounce**, d’**annuler** des actions.

Exemples :
- `watch(route.query, ...)`
- `watch(searchTerm, fetch...)`
- `watch(form, saveDraft...)`

---

# Résumé

- **Computed** : valeurs dérivées, pures, cachées, idéales pour l’affichage et la logique déterministe.
- **Watch** : exécution de code en réaction à un changement, adaptée aux effets secondaires et à l’asynchrone.

---

# Exercices (à faire en autonomie)

1. **Computed** : créer `priceWithTax` dépendant de `price` et `vatRate`.
2. **Computed writable** : champ `v-model` qui lit/écrit un objet `{ firstName, lastName }`.
3. **Watch** : persister un formulaire dans `localStorage` avec `watch(form, { deep: true })`.
4. **Watch + async** : recherche API avec annulation via `onCleanup`.
5. **Refactor** : remplacer un `watch` qui calcule une valeur dérivée par une `computed`.

---

# Annexes

## A) Exemple : `watchEffect` + DOM (flush post)

```vue
<script setup>
import { ref, watchEffect, nextTick } from 'vue'

const open = ref(false)

watchEffect(async () => {
  if (open.value) {
    await nextTick()
    // DOM à jour ici
    const el = document.querySelector('#modal')
    el?.focus?.()
  }
})
</script>

<template>
  <button @click="open = true">Ouvrir</button>
  <div v-if="open" id="modal" tabindex="-1">Modal</div>
</template>
```

## B) Erreurs courantes
- Mettre un `fetch` dans un computed.
- Sur-utiliser `deep: true`.
- Ne pas gérer l’annulation d’un watcher async.
- Confondre un besoin de **valeur** (computed) avec un besoin d’**effet** (watch).
