# Formation Vue.js — Performance et optimisation

> Public cible : développeurs Vue.js (niveau intermédiaire)
>
> Objectif : rendre une application Vue perçue comme plus rapide et plus fluide en appliquant **lazy loading**, **code splitting** et **optimisation du rendu des listes**.

---

## 0) Informations générales

### Prérequis
- Bonne maîtrise de JavaScript ES6+
- Connaissance de Vue 3 (SFC, Composition API ou Options API)
- Avoir déjà utilisé un bundler (Vite recommandé, Webpack possible)

### Résultats attendus (compétences)
À la fin, vous serez capable de :
- Identifier les principales causes de lenteur (chargement initial, routes lourdes, rendu de listes)
- Mettre en place le **lazy loading** de composants et de routes
- Configurer/contrôler le **code splitting** et l’**optimisation des chunks**
- Accélérer le rendu de listes : stabiliser les clés, réduire les re-renders, virtualiser, paginer
- Mesurer l’impact (Lighthouse, Web Vitals, DevTools)

### Durée suggérée
- 1 journée (7h) ou 2 demi-journées

### Matériel
- Node.js LTS
- Projet Vue 3 (Vite) de démonstration
- Chrome/Edge avec DevTools

---

## 1) Comprendre la performance dans une SPA Vue

### 1.1 Les 3 dimensions clés
1. **Performance de chargement**
   - Taille du bundle initial
   - Nombre de requêtes
   - Temps de parsing/exécution JS
2. **Performance de rendu**
   - Coût du Virtual DOM / patching
   - Fréquence des re-renders
   - Coût des composants et du CSS
3. **Performance d’interaction**
   - Lags au scroll
   - Saisie lente (input)
   - Animations saccadées

### 1.2 Vocabulaire rapide
- **TTFB** : time to first byte
- **FCP** : first contentful paint
- **LCP** : largest contentful paint
- **TTI/TBT** : interactivité / blocage du thread principal
- **CLS** : stabilité visuelle

### 1.3 Outils de mesure (indispensables)
- **Lighthouse** (Chrome DevTools) : audit global
- **Performance tab** : flame chart, scripting, layout, paint
- **Network tab** : waterfall, cache, taille JS/CSS, timing
- **Vue Devtools** : inspection du rendu, composants qui re-rendent

> Règle d’or : **mesurer avant/après**. L’optimisation sans mesure est souvent contre-productive.

---

## 2) Lazy loading des composants

L’idée : **ne pas charger** (ou initialiser) ce qui n’est pas nécessaire tout de suite.

### 2.1 Lazy loading : composants asynchrones
Vue permet de définir un composant chargé à la demande.

#### Exemple (Vue 3) : `defineAsyncComponent`
```ts
// src/components/HeavyChart.async.ts
import { defineAsyncComponent } from 'vue'

export const HeavyChart = defineAsyncComponent({
  loader: () => import('./HeavyChart.vue'),
  // Optionnel : expérience utilisateur + robustesse
  delay: 200,           // avant d’afficher le loader
  timeout: 10000,       // délai avant erreur
  suspensible: true,
  onError(error, retry, fail, attempts) {
    if (attempts <= 3) retry()
    else fail()
  },
})
```

Utilisation :
```vue
<script setup lang="ts">
import { HeavyChart } from '@/components/HeavyChart.async'
</script>

<template>
  <section>
    <h2>Dashboard</h2>
    <HeavyChart />
  </section>
</template>
```

> Résultat : le code de `HeavyChart.vue` n’est téléchargé qu’au moment où il est réellement rendu.

### 2.2 Offrir un fallback (loader, skeleton)
Avec `Suspense` (Vue 3), vous pouvez afficher un fallback pendant le chargement.

```vue
<script setup>
import { HeavyChart } from '@/components/HeavyChart.async'
</script>

<template>
  <Suspense>
    <template #default>
      <HeavyChart />
    </template>

    <template #fallback>
      <div class="skeleton">Chargement du graphe…</div>
    </template>
  </Suspense>
</template>
```

### 2.3 Lazy loading conditionnel
Ne chargez un composant lourd que si une condition le demande.

```vue
<script setup>
import { ref } from 'vue'
import { HeavyChart } from '@/components/HeavyChart.async'

const show = ref(false)
</script>

<template>
  <button @click="show = !show">
    {{ show ? 'Masquer' : 'Afficher' }} le graphe
  </button>

  <Suspense>
    <template #default>
      <HeavyChart v-if="show" />
    </template>
    <template #fallback>
      <div>Chargement…</div>
    </template>
  </Suspense>
</template>
```

### 2.4 Bonnes pratiques
- Lazy load **les features rares** (admin, analytics, exports PDF, editor riche…)
- Évitez de lazy loader des composants **toujours visibles au-dessus de la ligne de flottaison** (header critique)
- Ajoutez un fallback UX pour éviter un “flash” vide
- Pensez aux erreurs réseau : timeouts, retries, message utilisateur

---

## 3) Division du code (Code Splitting)

Le code splitting découpe l’application en **chunks** chargés au bon moment. Avec Vite (Rollup) ou Webpack, l’instruction dynamique `import()` est le levier principal.

### 3.1 Code splitting via le router (lazy routes)
Typiquement, chaque page (route) peut être un chunk.

```ts
// src/router/index.ts
import { createRouter, createWebHistory } from 'vue-router'

const routes = [
  {
    path: '/',
    name: 'home',
    component: () => import('@/views/HomeView.vue'),
  },
  {
    path: '/admin',
    name: 'admin',
    component: () => import('@/views/AdminView.vue'),
  },
]

export const router = createRouter({
  history: createWebHistory(),
  routes,
})
```

> Effet immédiat : le bundle initial diminue. Les pages non visitées n’alourdissent pas le premier chargement.

### 3.2 Nommer les chunks (selon bundler)
- **Vite/Rollup** : le nom final dépend souvent de la stratégie de build, mais vous pouvez orienter la séparation via `manualChunks`.
- **Webpack** : `/* webpackChunkName: "admin" */` dans l’import.

Exemple Webpack (si applicable) :
```ts
component: () => import(/* webpackChunkName: "admin" */ '@/views/AdminView.vue')
```

### 3.3 Précharger (preload/prefetch) intelligemment
- **Prefetch** : on charge “quand le navigateur a le temps”
- **Preload** : on charge très tôt car probablement nécessaire

Approche simple côté application : précharger une route probable sur interaction.

```ts
// Au survol d’un lien, on peut déclencher l’import
const preloadAdmin = () => import('@/views/AdminView.vue')
```

Dans un composant :
```vue
<template>
  <RouterLink to="/admin" @mouseenter="preload">
    Admin
  </RouterLink>
</template>

<script setup>
const preload = () => {
  import('@/views/AdminView.vue')
}
</script>
```

> But : améliorer la perception. L’utilisateur ne “paie” plus le coût au clic.

### 3.4 Éviter les pièges fréquents
- Trop de chunks minuscules = surcharge de requêtes (surtout sans HTTP/2/3)
- Trop gros chunks = bénéfice limité
- Les dépendances partagées peuvent se répéter si le bundler ne les factorise pas correctement

### 3.5 Optimiser les chunks avec Vite (`manualChunks`)
Exemple (indicatif) :
```ts
// vite.config.ts
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'

export default defineConfig({
  plugins: [vue()],
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          // Mettre les libs dans un chunk vendor
          vue: ['vue', 'vue-router', 'pinia'],
          // Exemple de lib lourde isolée
          charts: ['echarts'],
        },
      },
    },
  },
})
```

> Attention : `manualChunks` doit refléter votre usage. Une mauvaise configuration peut dégrader le cache.

---

## 4) Optimisation du rendu des listes

Les listes sont une source majeure de lenteur, surtout quand :
- Il y a beaucoup d’items (1000+)
- Chaque item contient des sous-composants lourds
- Chaque interaction déclenche des re-renders

### 4.1 Les fondamentaux : `key` stable et unique
Toujours fournir une `key` stable (id) — jamais l’index si la liste peut changer d’ordre.

```vue
<li v-for="user in users" :key="user.id">
  {{ user.name }}
</li>
```

Pourquoi ?
- Vue réutilise mieux les éléments DOM
- Moins de patching inutile
- Évite des bugs subtils (inputs réutilisés, focus)

### 4.2 Réduire le coût de chaque item

#### a) Décomposer intelligemment
Si un item est complexe, isolez-le dans un composant, mais assurez-vous qu’il ne re-render pas “pour rien”.

```vue
<!-- UserRow.vue -->
<script setup>
defineProps({
  user: { type: Object, required: true },
})
</script>

<template>
  <div class="row">
    <span>{{ user.name }}</span>
    <span>{{ user.email }}</span>
  </div>
</template>
```

#### b) Éviter les calculs coûteux dans le template
Mauvais :
```vue
<li v-for="u in users" :key="u.id">
  {{ expensiveFormat(u) }}
</li>
```

Mieux : pré-calculer (computed) ou préparer les données.
```ts
import { computed } from 'vue'

const formattedUsers = computed(() =>
  users.value.map(u => ({
    ...u,
    label: `${u.name} — ${u.email}`
  }))
)
```

```vue
<li v-for="u in formattedUsers" :key="u.id">
  {{ u.label }}
</li>
```

> Objectif : éviter de recalculer une fonction lourde à chaque patch.

### 4.3 Limiter les re-renders

#### a) Stabiliser les références
Si vous recréez des objets/arrays à chaque rendu, vous forcez des changements.
- Évitez de faire `:style="{...}"` ou `:class="[{...}]"` ultra dynamiques dans de gros `v-for` si ce n’est pas nécessaire.
- Préférez des computed/mémos.

#### b) `v-memo` (Vue 3.2+)
Permet d’indiquer à Vue de **réutiliser** un sous-arbre tant que certaines dépendances ne changent pas.

```vue
<li
  v-for="u in users"
  :key="u.id"
  v-memo="[u.id, u.status]"
>
  <UserRow :user="u" />
</li>
```

- Si `u.status` ne change pas, Vue peut ignorer le re-render de cette ligne.
- À utiliser en connaissance de cause (mesurer l’impact).

### 4.4 Pagination et “windowing”

#### a) Pagination
Solution simple et robuste : ne rendre que 20–50 items à la fois.

- Avantages : facile, UX claire
- Inconvénients : moins fluide qu’un scroll infini

#### b) Virtualisation (windowing)
Pour les listes très longues, on ne rend que les éléments visibles à l’écran.

Approche : librairie de virtual scroll (ex. `vue-virtual-scroller`) ou composant maison.

Pseudo-objectif :
- 10 000 items en mémoire
- ~20–60 items rendus réellement

Points d’attention :
- Hauteurs d’items (fixes ou mesurées)
- SEO (si SSR), accessibilité
- Gestion du focus et du scroll

### 4.5 Optimiser les interactions dans les listes
- **Debounce** sur recherche/filtrage client
- **Throttle** sur scroll
- Éviter de mettre des watchers profonds (`deep: true`) sur des tableaux énormes

Exemple debounce (sans lib externe) :
```ts
let t: number | undefined

function debounce(fn: () => void, delay = 250) {
  return () => {
    window.clearTimeout(t)
    t = window.setTimeout(fn, delay)
  }
}
```

---

## 5) Atelier guidé (pas à pas)

### Contexte
On part d’une app qui :
- Charge une page “Dashboard” contenant un composant `HeavyChart` (lib de chart)
- A une page “Admin” peu utilisée
- Affiche une liste de 5 000 utilisateurs

### Étape 1 — Mesurer l’existant
1. Lighthouse (mode mobile)
2. Network : taille JS initiale
3. Performance : profiler le rendu de la liste

Livrable : capture des métriques (FCP/LCP/TBT) + taille du bundle.

### Étape 2 — Lazy load du composant lourd
- Convertir `HeavyChart` en async component
- Ajouter `Suspense` + skeleton

Mesure après : taille du chunk initial + amélioration TBT.

### Étape 3 — Lazy routes + preload sur intention
- `AdminView` via `component: () => import()`
- `mouseenter` sur lien Admin pour précharger

Mesure après : waterfall du chargement et temps d’ouverture réel.

### Étape 4 — Optimiser la liste
- `:key` stable
- Pré-calculer les labels (computed)
- Ajouter pagination ou virtualisation
- Tester `v-memo` sur `UserRow`

Mesure après : FPS au scroll + durée des frames dans Performance tools.

---

## 6) Checklist “prête à l’emploi”

### Chargement / bundle
- [ ] Routes en lazy loading (pages)
- [ ] Composants lourds en `defineAsyncComponent`
- [ ] Chunks cohérents (ni trop, ni trop peu)
- [ ] Vendor chunk + libs lourdes isolées si besoin
- [ ] Préload/prefetch sur intention utilisateur

### Rendu
- [ ] `key` stable dans les listes
- [ ] Éviter fonctions coûteuses dans le template
- [ ] Limiter watchers/profondeur sur gros tableaux
- [ ] `v-memo` si bénéfice mesuré
- [ ] Pagination/virtualisation si liste volumineuse

### Mesure
- [ ] Lighthouse avant/après
- [ ] Profiling Performance tab
- [ ] Network tab : cache, compression, taille

---

## 7) QCM / Évaluation rapide

1. Quel est l’intérêt principal du lazy loading ?
   - a) Réduire le nombre de composants
   - b) Réduire le **coût initial** en différant le chargement ✅
   - c) Supprimer le Virtual DOM

2. Pourquoi éviter `:key="index"` sur une liste filtrable/triable ?
   - a) C’est interdit
   - b) Ça force Vue à recréer l’array
   - c) Ça perturbe la réutilisation DOM et peut causer des bugs ✅

3. Pour une liste de 10 000 items, que privilégier ?
   - a) Un unique composant géant
   - b) Virtualisation (windowing) ✅
   - c) Ajouter plus de watchers

---

## 8) Annexes — Références
- Vue — Async Components : https://vuejs.org/guide/components/async.html
- Vue — Performance : https://vuejs.org/guide/best-practices/performance.html
- Web.dev — Core Web Vitals : https://web.dev/vitals/

---

# Fin du cours
