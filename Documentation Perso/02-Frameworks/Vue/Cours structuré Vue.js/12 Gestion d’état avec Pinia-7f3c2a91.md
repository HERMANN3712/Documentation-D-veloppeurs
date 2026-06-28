# Formation Vue.js — Gestion d’état avec Pinia

> **Objectif** : maîtriser Pinia (solution officielle et moderne de gestion d’état pour Vue 3) afin de centraliser, structurer et fiabiliser l’état partagé d’une application (SPA), avec de bonnes pratiques de conception, de test et de performance.

---

## Sommaire

1. [Présentation & objectifs pédagogiques](#1-présentation--objectifs-pédagogiques)
2. [Prérequis & contexte](#2-prérequis--contexte)
3. [Plan détaillé de la formation](#3-plan-détaillé-de-la-formation)
4. [Module 1 — Problématique de l’état dans Vue](#4-module-1--problématique-de-létat-dans-vue)
5. [Module 2 — Installation et mise en route](#5-module-2--installation-et-mise-en-route)
6. [Module 3 — Créer une store Pinia (state/getters/actions)](#6-module-3--créer-une-store-pinia-stategettersactions)
7. [Module 4 — Utiliser une store dans les composants](#7-module-4--utiliser-une-store-dans-les-composants)
8. [Module 5 — Stores "setup" vs stores "options"](#8-module-5--stores-setup-vs-stores-options)
9. [Module 6 — Gestion d’asynchrone (API, loading, erreurs)](#9-module-6--gestion-dasynchrone-api-loading-erreurs)
10. [Module 7 — Composition, partage, et architecture des stores](#10-module-7--composition-partage-et-architecture-des-stores)
11. [Module 8 — Persistance (localStorage) & sécurité](#11-module-8--persistance-localstorage--sécurité)
12. [Module 9 — Pinia et le Router (guards, auth)](#12-module-9--pinia-et-le-router-guards-auth)
13. [Module 10 — Devtools, debug, performance](#13-module-10--devtools-debug-performance)
14. [Module 11 — Tests unitaires des stores](#14-module-11--tests-unitaires-des-stores)
15. [Atelier final — Mini-projet guidé](#15-atelier-final--mini-projet-guidé)
16. [Cheat sheet & bonnes pratiques](#16-cheat-sheet--bonnes-pratiques)

---

## 1) Présentation & objectifs pédagogiques

### Pourquoi Pinia ?
Pinia est la solution moderne (et officielle) de gestion d’état globale pour Vue. Elle permet de **centraliser les données partagées** entre plusieurs composants, de rendre l’état **prévisible**, **testable**, et de simplifier le **debug** via les Devtools.

### Objectifs pédagogiques
À la fin de cette formation, vous saurez :

- expliquer quand et pourquoi utiliser une gestion d’état globale
- installer et configurer Pinia dans un projet Vue 3
- créer des stores avec `state`, `getters`, `actions` (et en style `setup`)
- consommer une store depuis des composants (Options API / Composition API)
- gérer les appels API (chargement, erreurs, cache, invalidation)
- structurer une application avec plusieurs stores
- persister une partie de l’état (ex: panier, session)
- intégrer Pinia avec Vue Router pour une gestion d’authentification
- utiliser les Devtools pour inspecter et rejouer les actions
- tester des stores Pinia avec Vitest

---

## 2) Prérequis & contexte

### Prérequis techniques
- Vue 3 (Composition API)
- Notions de composants, props/emit, slots
- Notions de reactivité (`ref`, `reactive`, `computed`)
- Bases de TypeScript conseillées (optionnel mais recommandé)

### Environnement recommandé
- Node.js LTS
- Vite + Vue 3
- Vue Devtools
- (Optionnel) Vitest

---

## 3) Plan détaillé de la formation

- **M1** — Comprendre le besoin : état local vs prop drilling vs global
- **M2** — Installation et initialisation de Pinia
- **M3** — Concevoir une store : `state`, `getters`, `actions`
- **M4** — Utiliser une store dans les composants
- **M5** — Stores `options` vs `setup` (quand choisir quoi)
- **M6** — Asynchrone : fetch, loading, erreurs, cache
- **M7** — Architecture : découpage en stores, composition, dépendances
- **M8** — Persistance : plugins, localStorage, SSR (notes)
- **M9** — Auth et router : guards, redirections
- **M10** — Devtools, bonnes pratiques de perf
- **M11** — Tests unitaires
- **Atelier** — Mini-projet : Auth + catalogue + panier

---

## 4) Module 1 — Problématique de l’état dans Vue

### 4.1 État local
Dans un composant, l’état local est idéal quand :
- il ne sert qu’à ce composant
- sa durée de vie est celle du composant

Exemple : ouverture/fermeture d’une modale.

### 4.2 Partage d’état via props/emit
Approche classique :
- **parent → enfant** via `props`
- **enfant → parent** via `emit`

Limite : quand l’application grossit, on fait du **prop drilling** (props en cascade) et la logique devient difficile à maintenir.

### 4.3 Besoin d’état global
On passe à un store global quand :
- plusieurs branches de composants consomment les mêmes données
- on veut un point unique de vérité (single source of truth)
- on veut tracer les mutations/actions et débugger facilement

Exemples typiques :
- utilisateur connecté + droits
- panier e-commerce
- préférences UI
- cache de données API

### 4.4 Pourquoi Pinia plutôt que Vuex ?
Pinia :
- API plus simple et plus typée
- pas de mutations séparées (actions mutent directement le state)
- meilleure DX, plus “Vue 3 native”

---

## 5) Module 2 — Installation et mise en route

### 5.1 Installation
```bash
npm i pinia
```

### 5.2 Enregistrement dans l’application
`main.ts` :
```ts
import { createApp } from 'vue'
import { createPinia } from 'pinia'
import App from './App.vue'

const app = createApp(App)
app.use(createPinia())
app.mount('#app')
```

### 5.3 Vérification Devtools
Dans Vue Devtools, vous devriez voir un onglet **Pinia** permettant :
- inspection du state
- historique des actions
- time-travel/debug

---

## 6) Module 3 — Créer une store Pinia (state/getters/actions)

Pinia fonctionne autour de **stores** identifiées par une clé, via `defineStore`.

### 6.1 Store "options" (style objet)
`src/stores/counter.ts` :
```ts
import { defineStore } from 'pinia'

export const useCounterStore = defineStore('counter', {
  state: () => ({
    count: 0
  }),
  getters: {
    double(state) {
      return state.count * 2
    }
  },
  actions: {
    increment() {
      this.count++
    },
    add(n: number) {
      this.count += n
    }
  }
})
```

**Points importants** :
- `state` est une **fonction** qui retourne l’état initial
- `getters` sont des computed dérivés
- `actions` portent la logique métier 

### 6.2 Accès au state : `this` dans les actions
Dans une store “options”, `this` pointe vers l’instance de store.

---

## 7) Module 4 — Utiliser une store dans les composants

### 7.1 Composition API
`CounterView.vue` :
```vue
<script setup lang="ts">
import { computed } from 'vue'
import { useCounterStore } from '@/stores/counter'

const counter = useCounterStore()

const count = computed(() => counter.count)
const double = computed(() => counter.double)

function onAdd10() {
  counter.add(10)
}
</script>

<template>
  <section>
    <p>Count: {{ count }}</p>
    <p>Double: {{ double }}</p>

    <button @click="counter.increment()">+1</button>
    <button @click="onAdd10">+10</button>
  </section>
</template>
```

### 7.2 ⚠️ Déstructuration et réactivité
Si vous faites :
```ts
const { count } = useCounterStore()
```
`count` **perd** la réactivité (car extrait de l’objet réactif). La solution : **storeToRefs**.

### 7.3 `storeToRefs`
```ts
import { storeToRefs } from 'pinia'

const counter = useCounterStore()
const { count } = storeToRefs(counter)
```

- `count` est un `ref`
- idéal pour la déstructuration sans casser la réactivité

### 7.4 Options API
```js
import { useCounterStore } from '@/stores/counter'

export default {
  computed: {
    counter() {
      return useCounterStore()
    }
  },
  methods: {
    inc() {
      this.counter.increment()
    }
  }
}
```

> Recommandation : privilégier Composition API sur Vue 3.

---

## 8) Module 5 — Stores "setup" vs stores "options"

### 8.1 Store "setup" (recommandée quand vous avez l’habitude de la Composition API)
`src/stores/todos.ts` :
```ts
import { defineStore } from 'pinia'
import { computed, ref } from 'vue'

export const useTodosStore = defineStore('todos', () => {
  const todos = ref<{ id: string; text: string; done: boolean }[]>([])
  const filter = ref<'all' | 'done' | 'todo'>('all')

  const filteredTodos = computed(() => {
    if (filter.value === 'done') return todos.value.filter(t => t.done)
    if (filter.value === 'todo') return todos.value.filter(t => !t.done)
    return todos.value
  })

  function addTodo(text: string) {
    todos.value.push({ id: crypto.randomUUID(), text, done: false })
  }

  function toggle(id: string) {
    const t = todos.value.find(t => t.id === id)
    if (t) t.done = !t.done
  }

  return { todos, filter, filteredTodos, addTodo, toggle }
})
```

### 8.2 Comment choisir ?
- **Options store** : simple, proche de Vuex, facile à lire
- **Setup store** : flexible (utilise directement `ref`, `computed`, composables), souvent plus naturelle en Vue 3

---

## 9) Module 6 — Gestion d’asynchrone (API, loading, erreurs)

Objectif : encapsuler la logique d’appel HTTP dans la store et exposer un état clair :
- `data`
- `isLoading`
- `error`

### 9.1 Exemple : store de produits
`src/stores/products.ts` :
```ts
import { defineStore } from 'pinia'
import { computed, ref } from 'vue'

type Product = { id: number; title: string; price: number }

export const useProductsStore = defineStore('products', () => {
  const items = ref<Product[]>([])
  const isLoading = ref(false)
  const error = ref<string | null>(null)
  const lastLoadedAt = ref<number | null>(null)

  const count = computed(() => items.value.length)

  async function fetchAll() {
    isLoading.value = true
    error.value = null

    try {
      const res = await fetch('https://fakestoreapi.com/products')
      if (!res.ok) throw new Error(`HTTP ${res.status}`)
      const data = (await res.json()) as any[]

      items.value = data.map(p => ({
        id: p.id,
        title: p.title,
        price: p.price
      }))
      lastLoadedAt.value = Date.now()
    } catch (e: any) {
      error.value = e?.message ?? 'Erreur inconnue'
    } finally {
      isLoading.value = false
    }
  }

  function invalidate() {
    lastLoadedAt.value = null
  }

  function shouldRefetch(maxAgeMs = 60_000) {
    return !lastLoadedAt.value || Date.now() - lastLoadedAt.value > maxAgeMs
  }

  return { items, count, isLoading, error, lastLoadedAt, fetchAll, invalidate, shouldRefetch }
})
```

### 9.2 Usage côté composant
```vue
<script setup lang="ts">
import { onMounted } from 'vue'
import { storeToRefs } from 'pinia'
import { useProductsStore } from '@/stores/products'

const productsStore = useProductsStore()
const { items, isLoading, error } = storeToRefs(productsStore)

onMounted(async () => {
  if (productsStore.shouldRefetch()) {
    await productsStore.fetchAll()
  }
})
</script>

<template>
  <div>
    <p v-if="isLoading">Chargement...</p>
    <p v-else-if="error">Erreur: {{ error }}</p>

    <ul v-else>
      <li v-for="p in items" :key="p.id">
        {{ p.title }} — {{ p.price }} €
      </li>
    </ul>
  </div>
</template>
```

### 9.3 Bonnes pratiques asynchrones
- centraliser l’API dans les stores (ou dans des services) 
- exposer `isLoading`/`error` pour l’UI
- prévoir invalidation/cache léger
- ne pas mélanger logique UI (toasts) directement dans la store si possible

---

## 10) Module 7 — Composition, partage, et architecture des stores

### 10.1 Découpage en plusieurs stores
Recommandation : découper par **capabilité métier** :
- `useAuthStore`
- `useCartStore`
- `useProductsStore`
- `useUiStore`

Éviter : un store “fourre-tout” global.

### 10.2 Une store peut utiliser une autre store
Exemple : le panier dépend des produits.

`src/stores/cart.ts` :
```ts
import { defineStore } from 'pinia'
import { computed, ref } from 'vue'
import { useProductsStore } from './products'

type CartLine = { productId: number; qty: number }

export const useCartStore = defineStore('cart', () => {
  const lines = ref<CartLine[]>([])

  const productsStore = useProductsStore()

  const detailedLines = computed(() => {
    return lines.value.map(line => {
      const product = productsStore.items.find(p => p.id === line.productId)
      return {
        ...line,
        product,
        lineTotal: (product?.price ?? 0) * line.qty
      }
    })
  })

  const total = computed(() => detailedLines.value.reduce((acc, l) => acc + l.lineTotal, 0))

  function add(productId: number, qty = 1) {
    const existing = lines.value.find(l => l.productId === productId)
    if (existing) existing.qty += qty
    else lines.value.push({ productId, qty })
  }

  function remove(productId: number) {
    lines.value = lines.value.filter(l => l.productId !== productId)
  }

  function clear() {
    lines.value = []
  }

  return { lines, detailedLines, total, add, remove, clear }
})
```

### 10.3 Anti-pattern : dépendances circulaires
Éviter que `auth` importe `cart` qui importe `auth` etc. Préférer :
- extraire une fonction commune
- ou inverser la dépendance (un orchestrateur)

---

## 11) Module 8 — Persistance (localStorage) & sécurité

### 11.1 Pourquoi persister ?
Ex : conserver le panier après refresh, mémoriser un thème, une langue.

### 11.2 Approche simple "maison" (watch)
```ts
import { watch } from 'vue'

watch(lines, (val) => {
  localStorage.setItem('cart', JSON.stringify(val))
}, { deep: true })
```

Pour recharger :
```ts
const saved = localStorage.getItem('cart')
if (saved) lines.value = JSON.parse(saved)
```

### 11.3 Approche plugin (recommandée)
Il existe des plugins comme `pinia-plugin-persistedstate`.

Exemple (principe) :
```bash
npm i pinia-plugin-persistedstate
```

`main.ts` :
```ts
import { createPinia } from 'pinia'
import piniaPersistedstate from 'pinia-plugin-persistedstate'

const pinia = createPinia()
pinia.use(piniaPersistedstate)
app.use(pinia)
```

Dans la store :
```ts
export const useCartStore = defineStore('cart', {
  state: () => ({ lines: [] as { productId:number; qty:number }[] }),
  persist: true
})
```

### 11.4 ⚠️ Sécurité
- Ne stockez pas de secrets dans `localStorage`.
- Pour l’auth, préférez des cookies HTTPOnly côté serveur quand possible.
- Si vous stockez un token, anticipez XSS (CSP, sanitation, libs sûres).

---

## 12) Module 9 — Pinia et le Router (guards, auth)

### 12.1 Store d’authentification (exemple)
`src/stores/auth.ts` :
```ts
import { defineStore } from 'pinia'
import { computed, ref } from 'vue'

type User = { id: string; email: string }

export const useAuthStore = defineStore('auth', () => {
  const user = ref<User | null>(null)
  const token = ref<string | null>(null)

  const isAuthenticated = computed(() => !!token.value)

  async function login(email: string, password: string) {
    // Démo : simule un login
    await new Promise(r => setTimeout(r, 300))
    token.value = 'demo-token'
    user.value = { id: 'u1', email }
  }

  function logout() {
    token.value = null
    user.value = null
  }

  return { user, token, isAuthenticated, login, logout }
})
```

### 12.2 Guard de route
`src/router/index.ts` :
```ts
import { createRouter, createWebHistory } from 'vue-router'
import { useAuthStore } from '@/stores/auth'

const router = createRouter({
  history: createWebHistory(),
  routes: [
    { path: '/login', component: () => import('@/views/LoginView.vue') },
    { path: '/admin', meta: { requiresAuth: true }, component: () => import('@/views/AdminView.vue') }
  ]
})

router.beforeEach((to) => {
  const auth = useAuthStore()
  if (to.meta.requiresAuth && !auth.isAuthenticated) {
    return { path: '/login', query: { redirect: to.fullPath } }
  }
})

export default router
```

**Important** : les stores sont accessibles dans les guards. 
- Assurez-vous que Pinia est bien initialisé avant la navigation (dans `main.ts`).

---

## 13) Module 10 — Devtools, debug, performance

### 13.1 Devtools
Pinia expose :
- état actuel de chaque store
- timeline des actions
- possibilité de modifier le state (en dev)

### 13.2 Performance & bonnes pratiques
- évitez de stocker des objets énormes si inutile
- préférez des références (ids) + getters computés pour dériver les vues
- éviter de recalculer coûteusement dans des getters non nécessaires
- utiliser `storeToRefs` pour n’extraire que ce dont on a besoin

### 13.3 $patch et modifications groupées
Pinia permet de patcher :
```ts
counter.$patch({ count: 42 })
```
Ou fonctionnel :
```ts
counter.$patch((state) => {
  state.count++
})
```

---

## 14) Module 11 — Tests unitaires des stores

### 14.1 Pourquoi tester les stores ?
Les stores contiennent la logique métier :
- calculs
- règles de gestion
- orchestration d’appels API

### 14.2 Mise en place de base (Vitest)
Installation indicative :
```bash
npm i -D vitest jsdom
```

### 14.3 Exemple de test
`src/stores/counter.spec.ts` :
```ts
import { describe, it, expect, beforeEach } from 'vitest'
import { setActivePinia, createPinia } from 'pinia'
import { useCounterStore } from './counter'

describe('counter store', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
  })

  it('increment increases count', () => {
    const store = useCounterStore()
    expect(store.count).toBe(0)
    store.increment()
    expect(store.count).toBe(1)
  })

  it('double getter works', () => {
    const store = useCounterStore()
    store.add(2)
    expect(store.double).toBe(4)
  })
})
```

### 14.4 Tester l’asynchrone
- mocker `fetch` (ou utiliser MSW)
- vérifier `isLoading` + `error` + données finales

---

## 15) Atelier final — Mini-projet guidé

### Sujet
Construire une mini-app Vue 3 avec :
- Auth (login/logout)
- Catalogue produits (fetch)
- Panier (ajout/suppression)
- Persistance du panier

### Étapes guidées
1. **Créer `useAuthStore`** + page `/login`
2. **Protéger `/admin`** via router guard
3. **Créer `useProductsStore`** (fetch + cache simple)
4. **Créer `useCartStore`** (lignes + total)
5. **Persister le panier** via plugin ou watch
6. **Ajouter une UI** : liste produits + boutons “Ajouter au panier”
7. **Afficher le panier** : lignes détaillées + total

### Critères de succès
- pas de prop drilling pour les éléments globaux
- le panier reste après refresh
- l’UI gère loading et erreurs
- logique métier testable (au moins une store testée)

---

## 16) Cheat sheet & bonnes pratiques

### Choisir entre état local et global
- local : UI éphémère (modales, inputs)
- global : session, cache, panier, préférences, données transverses

### Patterns recommandés
- une store par domaine
- actions pour la logique métier
- getters pour dériver et formater
- `storeToRefs` pour déstructurer dans les composants
- `isLoading/error` pour l’asynchrone

### Anti-patterns
- stocker des composants/instances non sérialisables dans le state
- créer des dépendances circulaires entre stores
- sur-utiliser le global pour de l’état purement local

---

## Annexes — Snippets utiles

### A) Reset d’une store
Pinia expose `$reset()` sur les stores options (state défini en fonction). Sinon, faites votre propre fonction :
```ts
function reset() {
  items.value = []
  error.value = null
  isLoading.value = false
}
```

### B) Accès hors composant
Dans certains utilitaires, vous pouvez importer `useXStore()` tant qu’une instance Pinia est active. Sinon, injecter Pinia explicitement (avancé).

---

**Fin de formation**
