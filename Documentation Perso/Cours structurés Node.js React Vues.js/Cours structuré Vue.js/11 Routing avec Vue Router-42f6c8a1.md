# Formation Vue.js — Routing avec Vue Router

**Référence cours :** 11 — Routing avec Vue Router  
**Public :** Développeurs web connaissant JavaScript et les bases de Vue 3  
**Durée conseillée :** 1 journée (6–7h) ou 2 demi-journées  
**Pré-requis :** Vue 3 (Composition API), notions de composants, props/emit, reactivité, build tool (Vite)  

---

## Objectifs pédagogiques

À la fin de cette formation, vous serez capable de :

- Expliquer le rôle du **routing** dans une application **SPA** (Single Page Application).
- Mettre en place **Vue Router** dans un projet Vue 3.
- Définir des **routes** (simples, dynamiques, imbriquées) et associer des **URLs** à des **composants**.
- Utiliser les composants et APIs clés : `RouterLink`, `RouterView`, `useRoute()`, `useRouter()`.
- Mettre en œuvre des **gardes de navigation** (auth, permissions) et la gestion des **redirections**.
- Gérer les **paramètres**, **query strings**, et les comportements de **scroll**.
- Améliorer les performances via le **lazy loading** (import dynamique).
- Diagnostiquer et résoudre les problèmes courants (404, routes non trouvées, navigation incohérente).

---

## Plan de la formation

1. **Introduction au routing et à Vue Router**
2. **Installation et configuration (Vue 3 + Vite)**
3. **Concepts fondamentaux : routes, history, RouterLink/RouterView**
4. **Définir des routes : statiques, dynamiques, props de route**
5. **Navigation : déclarative et programmatique**
6. **Routes imbriquées (nested routes) et layouts**
7. **Redirections, alias, route “catch-all” (404)**
8. **Garde de navigation (guards) : auth, rôle, validation**
9. **Meta fields, titres de page, breadcrumbs**
10. **Performance : lazy loading et découpage de code**
11. **Scroll behavior, liens actifs, UX de navigation**
12. **Atelier : mini-application SPA avec routing complet**
13. **Bonnes pratiques & erreurs fréquentes**
14. **Quiz / Check de fin de module**

---

# 1) Introduction au routing et à Vue Router

## 1.1 Qu’est-ce que le routing en SPA ?

Dans une **SPA**, la page HTML est chargée une fois. Ensuite, la navigation se fait **sans rechargement complet**, en remplaçant dynamiquement les composants affichés.

Le **routing** consiste à :

- Mapper une **URL** (ex: `/products/42`) vers un **composant** (ex: `ProductDetailsView.vue`).
- Gérer l’historique navigateur (boutons **Précédent/Suivant**).
- Contrôler l’accès à certaines pages (authentification, permissions).

## 1.2 Rôle de Vue Router

**Vue Router** est la bibliothèque officielle de routing pour Vue. Elle permet :

- La définition d’un tableau de routes.
- L’usage d’un mode d’historique (`history` ou `hash`).
- Des routes **imbriquées**, **dynamiques**, **lazy-loaded**.
- Gardes de navigation globales / par route.

---

# 2) Installation et configuration (Vue 3 + Vite)

## 2.1 Installation

```bash
npm install vue-router
```

> Pour Vue 3, Vue Router v4 est la version adaptée.

## 2.2 Structure de base

Exemple d’arborescence typique :

```txt
src/
  main.js
  router/
    index.js
  views/
    HomeView.vue
    AboutView.vue
  App.vue
```

## 2.3 Création du routeur

**`src/router/index.js`**

```js
import { createRouter, createWebHistory } from 'vue-router'

import HomeView from '../views/HomeView.vue'
import AboutView from '../views/AboutView.vue'

const routes = [
  { path: '/', name: 'home', component: HomeView },
  { path: '/about', name: 'about', component: AboutView },
]

const router = createRouter({
  history: createWebHistory(),
  routes,
})

export default router
```

## 2.4 Injection dans l’application

**`src/main.js`**

```js
import { createApp } from 'vue'
import App from './App.vue'
import router from './router'

createApp(App)
  .use(router)
  .mount('#app')
```

---

# 3) Concepts fondamentaux : routes, history, RouterLink/RouterView

## 3.1 Route = URL + composant

Une route associe :

- `path` : l’URL
- `component` : le composant affiché
- `name` (optionnel) : un identifiant stable pour naviguer

## 3.2 History modes

### `createWebHistory()` (recommandé)

- URLs propres : `/about`
- Nécessite une configuration serveur : toutes les routes doivent renvoyer `index.html`.

### `createWebHashHistory()`

- URLs avec hash : `/#/about`
- Moins beau mais fonctionne sans config serveur (utile pour hébergements statiques simples).

## 3.3 `RouterView` : zone d’affichage

Dans `App.vue`, on place la “fenêtre” où les pages s’affichent :

```vue
<template>
  <header>
    <nav>
      <RouterLink to="/">Accueil</RouterLink>
      <RouterLink to="/about">À propos</RouterLink>
    </nav>
  </header>

  <main>
    <RouterView />
  </main>
</template>
```

## 3.4 `RouterLink` : navigation déclarative

`RouterLink` empêche le rechargement complet et gère l’historique.

- `to` peut être une string (`"/about"`)
- ou un objet (`{ name: 'about' }`)

---

# 4) Définir des routes : statiques, dynamiques, props de route

## 4.1 Routes statiques

```js
{ path: '/contact', name: 'contact', component: () => import('../views/ContactView.vue') }
```

## 4.2 Routes dynamiques (params)

Pour des détails produit, profil utilisateur, etc. :

```js
{ path: '/products/:id', name: 'product', component: () => import('../views/ProductView.vue') }
```

Ici, `:id` est un paramètre.

### Accès au paramètre

Dans le composant, via `useRoute()` :

```vue
<script setup>
import { computed } from 'vue'
import { useRoute } from 'vue-router'

const route = useRoute()
const productId = computed(() => route.params.id)
</script>

<template>
  <h1>Produit {{ productId }}</h1>
</template>
```

## 4.3 Passer les params en props (recommandé)

Pour découpler le composant du routeur :

```js
{
  path: '/products/:id',
  name: 'product',
  component: () => import('../views/ProductView.vue'),
  props: true,
}
```

Puis :

```vue
<script setup>
defineProps({
  id: { type: String, required: true }
})
</script>

<template>
  <h1>Produit {{ id }}</h1>
</template>
```

## 4.4 Query string

Ex: `/search?q=vue&sort=asc`

```js
const route = useRoute()
route.query.q
route.query.sort
```

> Les valeurs de `query` sont des strings (ou tableaux) — pensez à convertir si nécessaire.

---

# 5) Navigation : déclarative et programmatique

## 5.1 Navigation déclarative avec `RouterLink`

```vue
<RouterLink :to="{ name: 'product', params: { id: '42' } }">
  Voir le produit 42
</RouterLink>
```

## 5.2 Navigation programmatique avec `useRouter()`

```vue
<script setup>
import { useRouter } from 'vue-router'

const router = useRouter()

function goToHome() {
  router.push({ name: 'home' })
}
</script>

<template>
  <button @click="goToHome">Retour</button>
</template>
```

### `push` vs `replace`

- `push()` ajoute une entrée dans l’historique
- `replace()` remplace l’entrée courante (utile après login pour éviter “retour” vers la page de login)

---

# 6) Routes imbriquées (nested routes) et layouts

## 6.1 Pourquoi des routes imbriquées ?

Pour des sections avec layout commun (Dashboard, Admin…), où seule une partie change.

## 6.2 Exemple : `/admin` avec sous-pages

**Router**

```js
{
  path: '/admin',
  component: () => import('../layouts/AdminLayout.vue'),
  children: [
    { path: '', name: 'admin-home', component: () => import('../views/admin/AdminHomeView.vue') },
    { path: 'users', name: 'admin-users', component: () => import('../views/admin/AdminUsersView.vue') },
  ],
}
```

**`AdminLayout.vue`**

```vue
<template>
  <div class="admin">
    <aside>
      <RouterLink :to="{ name: 'admin-home' }">Accueil Admin</RouterLink>
      <RouterLink :to="{ name: 'admin-users' }">Utilisateurs</RouterLink>
    </aside>

    <section class="content">
      <RouterView />
    </section>
  </div>
</template>
```

> Le `RouterView` du layout affichera les enfants.

---

# 7) Redirections, alias, route “catch-all” (404)

## 7.1 Redirection

```js
{ path: '/home', redirect: { name: 'home' } }
```

## 7.2 Alias

```js
{ path: '/', alias: ['/accueil'], name: 'home', component: HomeView }
```

## 7.3 Route 404 (catch-all)

```js
{
  path: '/:pathMatch(.*)*',
  name: 'not-found',
  component: () => import('../views/NotFoundView.vue'),
}
```

---

# 8) Garde de navigation (guards) : auth, rôle, validation

## 8.1 Types de gardes

- **Globales** : `router.beforeEach`, `router.afterEach`
- **Par route** : `beforeEnter`
- **Dans composant** (moins utilisé en Composition API, mais possible via hooks)

## 8.2 Exemple : auth obligatoire via `meta.requiresAuth`

### Définir le meta

```js
{
  path: '/account',
  name: 'account',
  component: () => import('../views/AccountView.vue'),
  meta: { requiresAuth: true },
}
```

### Garde globale

```js
router.beforeEach((to) => {
  const isLoggedIn = Boolean(localStorage.getItem('token'))

  if (to.meta.requiresAuth && !isLoggedIn) {
    return { name: 'login', query: { redirect: to.fullPath } }
  }
})
```

> Retourner un objet de navigation dans un guard annule la navigation courante et redirige.

## 8.3 Exemple : contrôle de rôle

```js
{
  path: '/admin',
  component: () => import('../layouts/AdminLayout.vue'),
  meta: { requiresAuth: true, role: 'admin' },
  children: [/* ... */]
}
```

```js
router.beforeEach((to) => {
  const token = localStorage.getItem('token')
  const userRole = localStorage.getItem('role') // exemple simplifié

  if (to.meta.requiresAuth && !token) return { name: 'login' }
  if (to.meta.role && to.meta.role !== userRole) return { name: 'forbidden' }
})
```

---

# 9) Meta fields, titres de page, breadcrumbs

## 9.1 Titre de page dynamique

```js
{
  path: '/about',
  name: 'about',
  component: () => import('../views/AboutView.vue'),
  meta: { title: 'À propos' },
}
```

Puis :

```js
router.afterEach((to) => {
  const base = 'Mon App Vue'
  document.title = to.meta.title ? `${to.meta.title} • ${base}` : base
})
```

## 9.2 Breadcrumbs (approche simple)

Vous pouvez mettre une structure dans `meta` :

```js
meta: {
  breadcrumbs: [
    { label: 'Accueil', to: { name: 'home' } },
    { label: 'Admin', to: { name: 'admin-home' } },
  ]
}
```

Puis afficher selon `route.meta.breadcrumbs`.

---

# 10) Performance : lazy loading et découpage de code

## 10.1 Import dynamique pour les vues

Au lieu d’importer toutes les pages dès le départ :

```js
{ path: '/about', component: () => import('../views/AboutView.vue') }
```

- Les blocs JS sont chargés **à la demande** lors de la navigation.
- Améliore le temps de chargement initial.

## 10.2 Grouper des chunks (optionnel)

Avec Vite, vous pouvez aussi configurer un découpage personnalisé côté build (niveau avancé), mais l’import dynamique suffit dans la plupart des cas.

---

# 11) Scroll behavior, liens actifs, UX de navigation

## 11.1 Scroll au changement de route

Configurer le routeur :

```js
const router = createRouter({
  history: createWebHistory(),
  routes,
  scrollBehavior(to, from, savedPosition) {
    if (savedPosition) return savedPosition
    if (to.hash) return { el: to.hash, behavior: 'smooth' }
    return { top: 0 }
  },
})
```

## 11.2 Classes actives de `RouterLink`

Par défaut :

- `router-link-active`
- `router-link-exact-active`

Personnaliser :

```js
const router = createRouter({
  history: createWebHistory(),
  routes,
  linkActiveClass: 'is-active',
  linkExactActiveClass: 'is-exact-active',
})
```

---

# 12) Atelier guidé : mini-application SPA

## 12.1 Cahier des charges

Créer une mini app avec :

- Pages : Accueil, Liste produits, Détail produit, Login, Compte, 404
- Navigation via menu
- Route dynamique `/products/:id`
- Guard d’auth : `/account` protégé
- Redirection vers `/login?redirect=...` puis retour après login

## 12.2 Routes proposées

```js
const routes = [
  { path: '/', name: 'home', component: () => import('../views/HomeView.vue') },

  { path: '/products', name: 'products', component: () => import('../views/ProductsView.vue') },
  {
    path: '/products/:id',
    name: 'product',
    component: () => import('../views/ProductView.vue'),
    props: true,
  },

  { path: '/login', name: 'login', component: () => import('../views/LoginView.vue') },
  {
    path: '/account',
    name: 'account',
    component: () => import('../views/AccountView.vue'),
    meta: { requiresAuth: true },
  },

  { path: '/:pathMatch(.*)*', name: 'not-found', component: () => import('../views/NotFoundView.vue') },
]
```

## 12.3 Garde globale

```js
router.beforeEach((to) => {
  const isLoggedIn = Boolean(localStorage.getItem('token'))
  if (to.meta.requiresAuth && !isLoggedIn) {
    return { name: 'login', query: { redirect: to.fullPath } }
  }
})
```

## 12.4 LoginView : simuler un login + redirection

```vue
<script setup>
import { useRoute, useRouter } from 'vue-router'

const route = useRoute()
const router = useRouter()

function login() {
  localStorage.setItem('token', 'demo')
  const redirectTo = route.query.redirect || { name: 'account' }
  router.replace(redirectTo)
}
</script>

<template>
  <h1>Login</h1>
  <button @click="login">Se connecter</button>
</template>
```

## 12.5 ProductsView : navigation vers un détail

```vue
<script setup>
const products = [
  { id: '1', name: 'Clavier' },
  { id: '2', name: 'Souris' },
  { id: '3', name: 'Écran' },
]
</script>

<template>
  <h1>Produits</h1>
  <ul>
    <li v-for="p in products" :key="p.id">
      <RouterLink :to="{ name: 'product', params: { id: p.id } }">
        {{ p.name }}
      </RouterLink>
    </li>
  </ul>
</template>
```

---

# 13) Bonnes pratiques & erreurs fréquentes

## 13.1 Bonnes pratiques

- Privilégier la navigation par **`name` + `params`** plutôt que par URL en dur.
- Utiliser `props: true` pour rendre les vues testables et découplées.
- Centraliser les guards (auth/roles) au niveau du routeur.
- Toujours prévoir une **route 404**.
- Mettre en place le **lazy loading** pour les vues.

## 13.2 Erreurs fréquentes

- Oublier la config serveur en `createWebHistory()` (résultat : 404 au refresh).
- Confondre `params` et `query`.
- Définir des routes enfants sans `RouterView` dans le layout parent.
- Utiliser des `params` non-string sans conversion (router sérialise en string).

---

# 14) Quiz / Check de fin de module

1. Quelle est la différence entre `createWebHistory()` et `createWebHashHistory()` ?
2. Comment accéder à un paramètre `:id` depuis un composant ?
3. Pourquoi `props: true` est-il recommandé pour une route dynamique ?
4. Quelle est la différence entre `router.push()` et `router.replace()` ?
5. Comment créer une route 404 avec Vue Router ?
6. Comment protéger une route via un guard global ?

---

## Annexes — Snippets utiles

### A) `RouterLink` avec classe active

```vue
<RouterLink
  :to="{ name: 'home' }"
  class="nav-link"
  active-class="is-active"
  exact-active-class="is-exact"
>
  Accueil
</RouterLink>
```

### B) Récupérer l’URL complète

```js
const route = useRoute()
console.log(route.fullPath)
```

### C) Réagir au changement de paramètre

Dans une vue détail, si vous restez sur le même composant mais changez `id`, vous pouvez vouloir recharger :

```vue
<script setup>
import { watch } from 'vue'
import { useRoute } from 'vue-router'

const route = useRoute()

watch(
  () => route.params.id,
  (id) => {
    // re-fetch data
    // fetchProduct(id)
  },
  { immediate: true }
)
</script>
```

---

## Conclusion

Vue Router est le pilier de la navigation en SPA Vue. En associant **URLs → composants**, en gérant l’historique, et en ajoutant des **guards**, vous structurez votre application et améliorez l’UX. La maîtrise des routes dynamiques, des routes imbriquées et du lazy loading permet de créer des applications robustes, performantes et maintenables.
