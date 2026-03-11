# Formation Vue.js — Consommation d’API (REST) avec `fetch` & `axios`

> **Public** : développeurs Vue.js (débutant → intermédiaire)  
> **Prérequis** : bases Vue 3 (SFC), JavaScript (promesses/async), notions HTTP (GET/POST)  
> **Durée conseillée** : 1 jour (7h) ou 2×3h30  
> **Objectif** : savoir consommer une API REST, gérer l’état et l’affichage dynamique, les erreurs, le chargement, la pagination, l’annulation, et structurer proprement le code.

---

## Plan de la formation

1. **Rappels : HTTP & REST pour frontends**
   - Méthodes, codes de statut, headers, JSON
   - Modèles de données, endpoints, query params
2. **Préparer un projet Vue pour consommer une API**
   - Variables d’environnement, base URL
   - Organisation (services, composables)
3. **Consommer une API avec `fetch`**
   - GET simple, POST, gestion JSON
   - `async/await`, erreurs, timeouts, abort
4. **Consommer une API avec `axios`**
   - Instance axios, interceptors, erreurs
   - Annulation, timeouts, auth token
5. **Stocker et afficher les données**
   - `ref`, `reactive`, `computed`
   - États UI : loading / error / empty / success
6. **Bonnes pratiques de composition**
   - Composables : `useApi`, `useUsers`
   - Services : `ApiClient`, `UserService`
7. **Cas fréquents**
   - Recherche / filtres / tri
   - Pagination ou « infinite scroll »
   - Synchronisation du formulaire (POST/PUT/PATCH)
8. **Robustesse & UX**
   - Cache simple, deduplication
   - Retry, backoff (concept)
   - Optimistic UI (concept)
9. **Atelier final**
   - Mini-application : liste + détail + création
   - Refactor : extraction services & composables

---

# 1) Rappels : HTTP & REST côté frontend

## 1.1 Méthodes HTTP

- **GET** : récupérer une ressource (liste ou détail)
- **POST** : créer
- **PUT** : remplacer
- **PATCH** : modifier partiellement
- **DELETE** : supprimer

## 1.2 Codes de statut utiles

- **200 OK** : tout va bien
- **201 Created** : création réussie
- **204 No Content** : succès sans corps
- **400 Bad Request** : paramètres invalides
- **401 Unauthorized** : non authentifié
- **403 Forbidden** : authentifié mais non autorisé
- **404 Not Found** : ressource absente
- **409 Conflict** : conflit (ex : ressource déjà existante)
- **422 Unprocessable Entity** : validation
- **500** : erreur serveur

## 1.3 JSON & headers

Une API REST renvoie souvent du JSON :

- Requête JSON : `Content-Type: application/json`
- Réponse JSON : `Accept: application/json`

---

# 2) Préparer un projet Vue pour consommer une API

## 2.1 Variables d’environnement (Vite)

Dans un projet Vue 3 avec Vite :

**`.env`**
```bash
VITE_API_BASE_URL=https://api.exemple.com
```

Dans le code :
```js
const baseUrl = import.meta.env.VITE_API_BASE_URL
```

## 2.2 Structurer le code

Une organisation simple et scalable :

```
src/
  api/
    apiClient.js
    users.service.js
  composables/
    useUsers.js
  components/
  views/
```

- `apiClient` : configuration commune (baseURL, headers, interceptors)
- `*.service` : fonctions métier (users, auth, etc.)
- `use*` : composables Vue (state + appel API)

---

# 3) Consommer une API avec `fetch`

`fetch` est natif et basé sur les promesses. Il ne rejette **pas** automatiquement en cas de code HTTP 4xx/5xx : il faut le vérifier.

## 3.1 GET simple

```js
async function fetchUsers() {
  const res = await fetch(`${import.meta.env.VITE_API_BASE_URL}/users`)
  if (!res.ok) {
    throw new Error(`HTTP ${res.status}`)
  }
  return await res.json()
}
```

## 3.2 Stocker les données dans l’état d’un composant

Exemple SFC Vue 3 :

```vue
<script setup>
import { ref, onMounted } from 'vue'

const users = ref([])
const loading = ref(false)
const error = ref(null)

async function load() {
  loading.value = true
  error.value = null

  try {
    const res = await fetch(`${import.meta.env.VITE_API_BASE_URL}/users`)
    if (!res.ok) throw new Error(`HTTP ${res.status}`)
    users.value = await res.json()
  } catch (e) {
    error.value = e
  } finally {
    loading.value = false
  }
}

onMounted(load)
</script>

<template>
  <section>
    <h1>Utilisateurs</h1>

    <p v-if="loading">Chargement…</p>
    <p v-else-if="error">Erreur : {{ error.message }}</p>
    <ul v-else>
      <li v-for="u in users" :key="u.id">{{ u.name }}</li>
    </ul>
  </section>
</template>
```

### Points clés

- **`loading`** et **`error`** rendent l’UI robuste.
- On remplit l’état **après** conversion `res.json()`.

## 3.3 POST JSON (création)

```js
async function createUser(payload) {
  const res = await fetch(`${import.meta.env.VITE_API_BASE_URL}/users`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Accept': 'application/json'
    },
    body: JSON.stringify(payload)
  })

  if (!res.ok) {
    // Souvent: API renvoie un JSON d’erreur
    let details = null
    try { details = await res.json() } catch {}
    const msg = details?.message ?? `HTTP ${res.status}`
    throw new Error(msg)
  }

  return await res.json()
}
```

## 3.4 Gestion d’erreurs plus fine

### Différencier réseau vs HTTP

- **Erreur réseau** (DNS, offline, CORS bloqué) : `fetch` rejette
- **Erreur HTTP** (404/500) : `fetch` résout, mais `res.ok === false`

## 3.5 Timeout & annulation (AbortController)

```js
async function fetchWithTimeout(url, { timeoutMs = 8000 } = {}) {
  const controller = new AbortController()
  const timer = setTimeout(() => controller.abort(), timeoutMs)

  try {
    const res = await fetch(url, { signal: controller.signal })
    if (!res.ok) throw new Error(`HTTP ${res.status}`)
    return await res.json()
  } finally {
    clearTimeout(timer)
  }
}
```

### Exemple : annuler une requête quand l’utilisateur retape

- cas typique : champ de **recherche** (autocomplete)

---

# 4) Consommer une API avec `axios`

`axios` est une lib populaire :

- Rejette en 4xx/5xx (comportement pratique)
- Transforme JSON automatiquement
- Offre interceptors, instance, timeouts, annulation…

## 4.1 Installation

```bash
npm i axios
```

## 4.2 Créer une instance `axios`

**`src/api/apiClient.js`**
```js
import axios from 'axios'

export const apiClient = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL,
  timeout: 8000,
  headers: {
    Accept: 'application/json'
  }
})
```

### Pourquoi une instance ?

- centraliser `baseURL`, `timeout`, headers
- ajouter facilement l’authentification

## 4.3 Interceptors (auth + logging)

**`src/api/apiClient.js`**
```js
import axios from 'axios'

export const apiClient = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL,
  timeout: 8000,
})

apiClient.interceptors.request.use((config) => {
  const token = localStorage.getItem('token')
  if (token) {
    config.headers.Authorization = `Bearer ${token}`
  }
  return config
})

apiClient.interceptors.response.use(
  (response) => response,
  (error) => {
    // Normaliser le message
    const message =
      error.response?.data?.message ??
      error.message ??
      'Erreur inconnue'

    error.normalizedMessage = message
    return Promise.reject(error)
  }
)
```

## 4.4 GET / POST avec axios

**Service Users**

**`src/api/users.service.js`**
```js
import { apiClient } from './apiClient'

export async function getUsers(params = {}) {
  const { data } = await apiClient.get('/users', { params })
  return data
}

export async function createUser(payload) {
  const { data } = await apiClient.post('/users', payload)
  return data
}
```

---

# 5) Stocker et afficher dynamiquement les données dans l’interface

## 5.1 Pattern UI : loading / error / empty / success

Ce pattern évite une UI « silencieuse ».

- `loading = true` pendant la requête
- `error` si échec
- `items.length === 0` : état vide

## 5.2 Exemple : liste avec recherche

**`src/views/UsersView.vue`**
```vue
<script setup>
import { computed, ref, watch } from 'vue'
import { getUsers } from '@/api/users.service'

const query = ref('')
const users = ref([])
const loading = ref(false)
const error = ref(null)

const hasUsers = computed(() => users.value.length > 0)

let debounceTimer
watch(query, () => {
  clearTimeout(debounceTimer)
  debounceTimer = setTimeout(load, 300)
})

async function load() {
  loading.value = true
  error.value = null

  try {
    users.value = await getUsers({ q: query.value })
  } catch (e) {
    error.value = new Error(e.normalizedMessage ?? e.message)
  } finally {
    loading.value = false
  }
}

load()
</script>

<template>
  <section>
    <h1>Utilisateurs</h1>

    <label>
      Recherche
      <input v-model="query" placeholder="Nom…" />
    </label>

    <p v-if="loading">Chargement…</p>
    <p v-else-if="error">Erreur : {{ error.message }}</p>
    <p v-else-if="!hasUsers">Aucun résultat.</p>

    <ul v-else>
      <li v-for="u in users" :key="u.id">
        {{ u.name }} — {{ u.email }}
      </li>
    </ul>
  </section>
</template>
```

---

# 6) Bonnes pratiques : Composables & Services

## 6.1 Séparer « appel API » et « état UI »

- Le **service** sait parler à l’API.
- Le **composable** gère l’état côté Vue (loading/error/data) et expose des actions.

## 6.2 Composable `useUsers`

**`src/composables/useUsers.js`**
```js
import { ref, computed } from 'vue'
import { getUsers, createUser } from '@/api/users.service'

export function useUsers() {
  const users = ref([])
  const loading = ref(false)
  const error = ref(null)

  const isEmpty = computed(() => !loading.value && !error.value && users.value.length === 0)

  async function load(params) {
    loading.value = true
    error.value = null
    try {
      users.value = await getUsers(params)
    } catch (e) {
      error.value = new Error(e.normalizedMessage ?? e.message)
    } finally {
      loading.value = false
    }
  }

  async function add(payload) {
    loading.value = true
    error.value = null
    try {
      const created = await createUser(payload)
      // Mise à jour locale immédiate
      users.value = [created, ...users.value]
      return created
    } catch (e) {
      error.value = new Error(e.normalizedMessage ?? e.message)
      throw e
    } finally {
      loading.value = false
    }
  }

  return { users, loading, error, isEmpty, load, add }
}
```

### Utilisation

```vue
<script setup>
import { onMounted } from 'vue'
import { useUsers } from '@/composables/useUsers'

const { users, loading, error, isEmpty, load, add } = useUsers()

onMounted(() => load())
</script>
```

---

# 7) Cas fréquents

## 7.1 Pagination (page/limit)

### Côté API (exemple)

- `GET /users?page=1&limit=20`
- Réponse :

```json
{
  "items": [ {"id": 1, "name": "…"} ],
  "page": 1,
  "limit": 20,
  "total": 103
}
```

### Côté Vue

- stocker `page`, `limit`, `total`, `items`
- bouton « suivant/précédent »

```vue
<script setup>
import { computed, ref } from 'vue'
import { apiClient } from '@/api/apiClient'

const items = ref([])
const page = ref(1)
const limit = ref(10)
const total = ref(0)
const loading = ref(false)

const pageCount = computed(() => Math.ceil(total.value / limit.value))
const canPrev = computed(() => page.value > 1)
const canNext = computed(() => page.value < pageCount.value)

async function load() {
  loading.value = true
  try {
    const { data } = await apiClient.get('/users', {
      params: { page: page.value, limit: limit.value }
    })
    items.value = data.items
    total.value = data.total
  } finally {
    loading.value = false
  }
}

async function next() { if (!canNext.value) return; page.value++; await load() }
async function prev() { if (!canPrev.value) return; page.value--; await load() }

load()
</script>

<template>
  <div>
    <p v-if="loading">Chargement…</p>

    <ul>
      <li v-for="u in items" :key="u.id">{{ u.name }}</li>
    </ul>

    <button @click="prev" :disabled="!canPrev">Précédent</button>
    <span>Page {{ page }} / {{ pageCount }}</span>
    <button @click="next" :disabled="!canNext">Suivant</button>
  </div>
</template>
```

## 7.2 Détail (GET /users/:id)

- route `/users/:id`
- charger via `watch` sur le paramètre de route

## 7.3 Formulaire : validation, erreurs API, UX

Bonnes pratiques :

- désactiver le bouton pendant `loading`
- afficher erreurs de validation (422) propres
- reset formulaire au succès

---

# 8) Robustesse & UX

## 8.1 Éviter les « double fetch »

Causes fréquentes :

- `watch` + `onMounted` déclenchant tous deux un appel
- composant monté plusieurs fois

Solutions :

- centraliser l’appel dans une fonction
- vérifier si déjà en cours (`loading`)

## 8.2 Cache simple (mémoire)

Idée : éviter de recharger une ressource déjà demandée.

```js
const cache = new Map()

export async function getCached(url, loader) {
  if (cache.has(url)) return cache.get(url)
  const promise = loader()
  cache.set(url, promise)
  return promise
}
```

> À utiliser avec prudence : invalidation et fraîcheur des données.

## 8.3 Retry (concept)

Utile sur erreurs réseau temporaires ou 502/503.

- retry 1–3 fois
- backoff exponentiel

---

# 9) Atelier final (guidé)

## 9.1 Consigne

Construire une mini app :

- écran **Liste utilisateurs**
  - chargement initial
  - recherche côté API
  - états `loading/error/empty`
- écran **Détail utilisateur**
  - `GET /users/:id`
- écran **Création**
  - formulaire + POST

## 9.2 Étapes proposées

1. Créer `apiClient` axios
2. Créer `users.service.js`
3. Créer `useUsers` + `useUserDetails`
4. Implémenter les vues
5. Gestion des erreurs et messages UX
6. (Bonus) Pagination

## 9.3 Critères de réussite

- code lisible, appels API centralisés
- UI claire en cas d’erreur ou de chargement
- pas de duplication des URLs/API keys

---

# Annexes

## A) Check-list « consommation d’API »

- [ ] Base URL en variable d’environnement
- [ ] `loading/error` gérés pour chaque écran
- [ ] Services dédiés (`*.service.js`)
- [ ] Composables pour l’état et les actions
- [ ] Messages d’erreurs normalisés
- [ ] Annulation/timeout pour recherche ou navigation rapide
- [ ] Pagination/tri/filtres testés

## B) À retenir

- `fetch` est suffisant, mais nécessite plus de garde-fous (gestion HTTP, timeout, etc.)
- `axios` offre une meilleure ergonomie pour les applications « business »
- L’objectif n’est pas seulement « récupérer des données », mais **construire une UI fiable** autour des appels réseau.
