# Formation — Structure d’un projet Vue

## Objectifs pédagogiques
À la fin de cette formation, vous serez capable de :

- Expliquer l’architecture typique d’un projet Vue (Vue 3) généré avec Vue CLI ou Vite.
- Identifier le rôle des dossiers clés : `src/`, `components/`, `assets/`, `views/`.
- Décrire le point d’entrée d’une application (`main.js` ou `main.ts`) et son lien avec `App.vue`.
- Appliquer de bonnes pratiques de structuration, de nommage et d’organisation.

## Prérequis
- Connaissances de base en JavaScript (ES6+).
- Notions de composants Vue (template, script, style).
- Node.js et npm/pnpm/yarn installés.

## Durée recommandée
- 1h30 à 2h (avec exercices)

## Public cible
- Développeurs et intégrateurs web
- Débutants/intermédiaires en Vue.js souhaitant comprendre la structure projet

---

# Plan du cours

1. **Panorama d’un projet Vue moderne**
2. **Le dossier racine du projet**
   - Fichiers de configuration et dépendances
3. **Le dossier `src/` (cœur applicatif)**
4. **Le point d’entrée : `main.js` / `main.ts`**
5. **Le composant racine : `App.vue`**
6. **Organisation fonctionnelle : `components/`, `views/`, `assets/`**
7. **Convention de routage : `router/` (si présent)**
8. **Gestion d’état : `store/` (si présent)**
9. **Bonnes pratiques (scalabilité, lisibilité, maintenance)**
10. **Exercices guidés**
11. **Résumé & check-list**

---

# 1) Panorama d’un projet Vue moderne

Un projet Vue (Vue 3 le plus souvent) est généralement créé via :

- **Vite** (recommandé aujourd’hui) : rapide, moderne
- **Vue CLI** (encore utilisé) : historique, robuste

Quel que soit l’outil, l’organisation générale reste similaire :

- Un dossier **racine** avec les configs et dépendances
- Un dossier **`src/`** qui contient le code de l’application
- Un point d’entrée (`main.js`) qui **initialise Vue** et **monte** l’application sur la page
- Un composant racine `App.vue` qui sert de **conteneur** global

---

# 2) Dossier racine du projet

## Structure typique

```txt
my-vue-app/
├─ index.html
├─ package.json
├─ vite.config.js
├─ jsconfig.json (ou tsconfig.json)
├─ .eslintrc.cjs / eslint.config.*
├─ .prettierrc
├─ node_modules/
└─ src/
```

### `package.json`
- Déclare les dépendances (Vue, Router, Pinia…)
- Contient les scripts (dev/build/lint/test)

Exemple :

```json
{
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview"
  }
}
```

### `index.html`
Avec Vite, `index.html` est à la racine et contient le point d’ancrage DOM pour l’app.

```html
<div id="app"></div>
<script type="module" src="/src/main.js"></script>
```

### Fichiers de config
- `vite.config.js` : configuration Vite (alias, plugins…)
- `tsconfig.json` / `jsconfig.json` : configuration TypeScript/JS
- ESLint/Prettier : qualité et format du code

---

# 3) Le dossier `src/` : le cœur de l’application

C’est ici que vit votre application.

Structure courante :

```txt
src/
├─ main.js
├─ App.vue
├─ assets/
├─ components/
├─ views/
├─ router/         (optionnel)
├─ store/          (optionnel)
└─ styles/         (optionnel)
```

---

# 4) Le point d’entrée : `main.js` (ou `main.ts`)

## Rôle
`main.js` est le fichier qui :

1. **importe Vue**
2. **importe le composant racine** (`App.vue`)
3. **crée l’application** via `createApp()`
4. **branche les plugins** (Router, Pinia, i18n…)
5. **monte** l’application sur un élément DOM (souvent `#app`)

## Exemple minimal

```js
// src/main.js
import { createApp } from 'vue'
import App from './App.vue'

createApp(App).mount('#app')
```

## Exemple avec Router et Store

```js
import { createApp } from 'vue'
import App from './App.vue'
import router from './router'
import { createPinia } from 'pinia'

const app = createApp(App)

app.use(createPinia())
app.use(router)

app.mount('#app')
```

### Points clés à retenir
- Le montage `.mount('#app')` doit correspondre à l’id présent dans `index.html`.
- Les plugins sont généralement branchés **avant** `mount()`.

---

# 5) Le composant racine : `App.vue`

## Rôle
`App.vue` est le **composant conteneur** principal. Il :

- définit la structure globale (layout)
- contient souvent le header/footer/navigation
- contient un `router-view` si Vue Router est utilisé

## Exemple simple (sans routes)

```vue
<template>
  <main class="container">
    <h1>Mon application Vue</h1>
    <HelloWorld />
  </main>
</template>

<script setup>
import HelloWorld from './components/HelloWorld.vue'
</script>

<style scoped>
.container {
  padding: 24px;
}
</style>
```

## Exemple avec router

```vue
<template>
  <div class="layout">
    <header>Mon App</header>
    <router-view />
    <footer>© 2026</footer>
  </div>
</template>
```

---

# 6) Organisation fonctionnelle : `components/`, `views/`, `assets/`

## `components/`
Contient des composants **réutilisables**.

Exemples :

- `BaseButton.vue` (composant UI générique)
- `UserCard.vue` (composant métier réutilisable)
- `TheHeader.vue` (composant de structure)

Structure recommandée :

```txt
components/
├─ base/
│  ├─ BaseButton.vue
│  └─ BaseInput.vue
├─ layout/
│  ├─ TheHeader.vue
│  └─ TheFooter.vue
└─ user/
   └─ UserCard.vue
```

### Bonnes pratiques
- Préfixer les composants “framework internes” par `Base` ou `App`.
- Utiliser le **PascalCase** pour les noms de fichiers de composants.

---

## `views/`
Contient des composants “pages” : ils correspondent souvent à des routes.

Exemples :

- `HomeView.vue`
- `AboutView.vue`
- `UsersView.vue`

Caractéristiques d’une view :
- moins réutilisable
- assemble plusieurs `components/`
- charge des données liées à la page

Structure :

```txt
views/
├─ HomeView.vue
└─ UsersView.vue
```

---

## `assets/`
Contient des ressources statiques gérées par le bundler :

- images
- icônes
- polices

Exemple :

```txt
assets/
├─ logo.svg
└─ images/
   └─ banner.jpg
```

### Différence avec `public/` (si présent)
- `assets/` : passe par le pipeline (hash, optimisation, import direct)
- `public/` : servi tel quel (URLs directes, pas d’import)

---

# 7) Le dossier `router/` (optionnel mais fréquent)

Si vous utilisez Vue Router :

```txt
router/
└─ index.js
```

Exemple :

```js
import { createRouter, createWebHistory } from 'vue-router'
import HomeView from '../views/HomeView.vue'

const routes = [
  { path: '/', name: 'home', component: HomeView },
  {
    path: '/about',
    name: 'about',
    component: () => import('../views/AboutView.vue') // lazy load
  }
]

export default createRouter({
  history: createWebHistory(),
  routes
})
```

---

# 8) Le dossier `store/` (optionnel)

Avec **Pinia** (recommandé) :

```txt
store/
└─ user.store.js
```

Exemple :

```js
import { defineStore } from 'pinia'

export const useUserStore = defineStore('user', {
  state: () => ({
    user: null
  }),
  actions: {
    setUser(user) {
      this.user = user
    }
  }
})
```

---

# 9) Bonnes pratiques de structuration

## A) Séparer “pages” et “composants”
- `views/` = pages
- `components/` = briques réutilisables

## B) Éviter les dossiers fourre-tout
Au lieu de `components/misc/`, préférez des sous-domaines (`user/`, `product/`, `layout/`).

## C) Mettre les règles d’import en place (alias)
Exemple (Vite) : alias `@` → `src/`.

Vous pouvez ensuite importer :

```js
import UserCard from '@/components/user/UserCard.vue'
```

## D) Un composant = un fichier, responsabilités claires
- un composant d’UI ne doit pas contenir toute la logique métier
- un composant de page orchestre les composants

## E) Styling
Options courantes :
- styles globaux dans `src/styles/`
- styles par composant via `<style scoped>`

---

# 10) Exercices guidés

## Exercice 1 — Identifier les rôles
**Consigne :** À partir de l’arborescence ci-dessous, expliquez le rôle de chaque dossier/fichier.

```txt
src/
├─ main.js
├─ App.vue
├─ components/
│  └─ BaseButton.vue
├─ views/
│  └─ HomeView.vue
└─ assets/
   └─ logo.svg
```

**Attendu :**
- `main.js` initialise et monte l’app
- `App.vue` est le composant racine
- `components/` contient des briques réutilisables
- `views/` contient des pages
- `assets/` contient des ressources statiques packagées

---

## Exercice 2 — Créer une page et un composant
**Consigne :**
1. Créez `src/views/ProfileView.vue`
2. Créez `src/components/user/UserBadge.vue`
3. Utilisez `UserBadge` dans `ProfileView`
4. Affichez `ProfileView` dans `App.vue` (sans router)

**Exemple de correction (simplifiée)**

`src/components/user/UserBadge.vue`

```vue
<template>
  <span class="badge">{{ name }}</span>
</template>

<script setup>
defineProps({
  name: { type: String, required: true }
})
</script>

<style scoped>
.badge {
  padding: 6px 10px;
  border-radius: 999px;
  background: #eef;
}
</style>
```

`src/views/ProfileView.vue`

```vue
<template>
  <section>
    <h2>Profil</h2>
    <UserBadge name="Ada Lovelace" />
  </section>
</template>

<script setup>
import UserBadge from '@/components/user/UserBadge.vue'
</script>
```

`src/App.vue`

```vue
<template>
  <ProfileView />
</template>

<script setup>
import ProfileView from './views/ProfileView.vue'
</script>
```

---

# 11) Résumé & check-list

## Résumé
- Un projet Vue est structuré autour de `src/`.
- `main.js` initialise l’application et monte `App.vue`.
- `App.vue` est le composant racine.
- `components/`: composants réutilisables.
- `views/`: pages (souvent liées aux routes).
- `assets/`: ressources statiques packagées.

## Check-list rapide
- [ ] `main.js` contient `createApp(App).mount('#app')`
- [ ] `App.vue` reste simple et structure l’application
- [ ] Les pages sont dans `views/`
- [ ] Les composants réutilisables sont dans `components/`
- [ ] Les assets importés sont dans `assets/`
- [ ] Alias `@` configuré (si possible)

---

## Annexes — Arborescence type complète (exemple)

```txt
my-vue-app/
├─ index.html
├─ package.json
├─ vite.config.js
└─ src/
   ├─ main.js
   ├─ App.vue
   ├─ assets/
   │  ├─ logo.svg
   │  └─ images/
   ├─ components/
   │  ├─ base/
   │  ├─ layout/
   │  └─ user/
   ├─ views/
   ├─ router/
   │  └─ index.js
   ├─ store/
   └─ styles/
      └─ main.css
```
