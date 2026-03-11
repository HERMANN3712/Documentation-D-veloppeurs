# Formation Vue.js — 02 Installation et environnement

> Objectif : savoir **démarrer un projet Vue.js** selon plusieurs approches (CDN, Vite, Vue CLI) et mettre en place un **environnement de développement moderne**.

---

## Plan de la formation

1. **Pré-requis et installation des outils**
   - Node.js, npm/pnpm/yarn
   - Vérifications de versions
   - Éditeur (VS Code) + extensions
2. **Trois façons d’utiliser Vue.js**
   - Via **CDN** (sans build)
   - Via **Vite** (recommandé)
   - Via **Vue CLI** (historique)
3. **Méthode CDN : démarrer vite, comprendre les limites**
   - Fichier HTML minimal
   - Création d’une app Vue
   - Avantages/inconvénients
4. **Méthode moderne : créer un projet avec Vite**
   - `npm create vite@latest`
   - Structure du projet
   - Scripts npm
   - Hot Module Replacement (HMR)
5. **Configuration et bonnes pratiques d’environnement**
   - Variables d’environnement (`.env`)
   - Alias, base path, build
   - Qualité de code : ESLint/Prettier
6. **Alternative : Vue CLI (pour maintenance/legacy)**
   - Scaffolding
   - Différences avec Vite
7. **Atelier récapitulatif**
   - Démarrer, modifier un composant, lancer build, prévisualiser

---

## 1) Pré-requis et installation des outils

### 1.1 Installer Node.js

Vue.js (en mode projet) s’appuie sur l’écosystème Node.

- Installer Node.js via : https://nodejs.org/
- Recommandation : utiliser une version **LTS**.

#### Vérifier l’installation

Ouvrez un terminal et exécutez :

```bash
node -v
npm -v
```

Vous devez obtenir des numéros de version (ex. `v20.x.x` et `10.x.x`).

> Conseil : si vous jonglez entre plusieurs versions de Node, envisagez d’utiliser **nvm** (Node Version Manager).

### 1.2 Choisir un gestionnaire de paquets

Vous pouvez utiliser :

- **npm** (par défaut)
- **pnpm** (rapide, efficace en espace disque)
- **yarn** (historique, encore utilisé)

Dans ce cours, les commandes seront données en **npm**. Les équivalents sont souvent faciles à déduire.

### 1.3 Installer un éditeur et extensions

Recommandation : **Visual Studio Code**.

Extensions utiles :

- **Vue - Official** (Volar)
- **ESLint**
- **Prettier - Code formatter**
- **Path Intellisense**

---

## 2) Trois façons d’utiliser Vue.js

Il existe plusieurs approches, à choisir selon le contexte.

### 2.1 Vue via CDN

- Ajout d’une balise `<script>` pointant vers Vue.
- Aucun bundler, aucun build step.
- Idéal pour : prototypes, intégration progressive dans une page existante.

### 2.2 Vue avec Vite (recommandé)

- Outil moderne de build/dev server.
- Démarrage ultra rapide.
- Hot Module Replacement (HMR) très efficace.
- Standard actuel pour la plupart des nouveaux projets Vue.

### 2.3 Vue CLI (historique)

- Ancien générateur basé sur webpack.
- Utilisé dans de nombreux projets existants.
- Encore valable pour maintenance, mais moins recommandé pour un nouveau projet.

---

## 3) Démarrer Vue.js via CDN (sans build)

### 3.1 Exemple minimal

Créez un fichier `index.html` :

```html
<!doctype html>
<html lang="fr">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Vue via CDN</title>
  </head>
  <body>
    <div id="app">
      <h1>{{ message }}</h1>
      <button @click="increment">Compteur: {{ count }}</button>
    </div>

    <!-- Vue 3 via CDN (production). Pour du dev, vous pouvez utiliser la version dev -->
    <script src="https://unpkg.com/vue@3/dist/vue.global.prod.js"></script>

    <script>
      const { createApp } = Vue

      createApp({
        data() {
          return {
            message: 'Bonjour Vue via CDN',
            count: 0,
          }
        },
        methods: {
          increment() {
            this.count++
          },
        },
      }).mount('#app')
    </script>
  </body>
</html>
```

Ouvrez ce fichier dans le navigateur.

### 3.2 Avantages / limites

**Avantages :**

- Très simple
- Aucun outil à installer
- Parfait pour tester Vue rapidement

**Limites :**

- Pas de SFC (`.vue`) facilement
- Gestion des modules ES plus complexe
- Pas (ou peu) d’outillage : TypeScript, lint, tests, build optimisé…

---

## 4) Méthode moderne : créer une application avec Vite

### 4.1 Créer le projet

Dans un terminal :

```bash
npm create vite@latest
```

Vous serez invité à choisir :

- **Project name** : par ex. `my-vue-app`
- **Framework** : `Vue`
- **Variant** : `JavaScript` ou `TypeScript`

> Alternative directe (exemple) :
>
> ```bash
> npm create vite@latest my-vue-app -- --template vue
> ```

### 4.2 Installer les dépendances

```bash
cd my-vue-app
npm install
```

### 4.3 Lancer le serveur de dev

```bash
npm run dev
```

Vite affiche une URL (souvent `http://localhost:5173`). Ouvrez-la.

### 4.4 Comprendre la structure du projet

Structure typique :

```
my-vue-app/
  index.html
  package.json
  vite.config.js (ou .ts)
  src/
    main.js
    App.vue
    assets/
    components/
  public/
```

#### Rôle des fichiers clés

- `index.html` : point d’entrée HTML (Vite y injecte le bundle dev/build)
- `src/main.js` : bootstrap de l’application Vue
- `src/App.vue` : composant racine
- `src/components/` : composants réutilisables
- `public/` : fichiers servis tels quels (sans transformation)

### 4.5 Vue en Single File Components (SFC)

Un fichier `.vue` contient :

- `<template>` : HTML déclaratif
- `<script>` ou `<script setup>` : logique JS/TS
- `<style>` : styles (scopables)

Exemple `src/components/HelloCounter.vue` :

```vue
<template>
  <section>
    <h2>{{ title }}</h2>
    <button @click="count++">Compteur: {{ count }}</button>
  </section>
</template>

<script setup>
import { ref } from 'vue'

defineProps({
  title: { type: String, default: 'Compteur' },
})

const count = ref(0)
</script>

<style scoped>
button {
  padding: 0.5rem 0.75rem;
}
</style>
```

Puis utilisez-le dans `App.vue` :

```vue
<template>
  <main>
    <HelloCounter title="Mon premier composant" />
  </main>
</template>

<script setup>
import HelloCounter from './components/HelloCounter.vue'
</script>
```

### 4.6 Scripts npm importants

Dans `package.json`, vous trouverez typiquement :

- `npm run dev` : serveur de dev + HMR
- `npm run build` : build de production
- `npm run preview` : aperçu du build localement

Exécution :

```bash
npm run build
npm run preview
```

### 4.7 Pourquoi Vite est la méthode moderne

- Démarrage quasi instantané
- HMR performant
- Config plus légère
- Écosystème actuel Vue officiel

---

## 5) Configuration d’environnement et bonnes pratiques

### 5.1 Variables d’environnement

Vite supporte des fichiers `.env` :

- `.env` (valeurs communes)
- `.env.development`
- `.env.production`

Règle importante : **toute variable exposée au client doit commencer par `VITE_`**.

Exemple `.env.development` :

```env
VITE_API_URL=http://localhost:3000
```

Utilisation :

```js
console.log(import.meta.env.VITE_API_URL)
```

> Note : ne mettez jamais de secrets (clés privées) dans des variables exposées côté front.

### 5.2 Configuration Vite (vite.config)

Exemple de configuration d’alias :

```js
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import { fileURLToPath, URL } from 'node:url'

export default defineConfig({
  plugins: [vue()],
  resolve: {
    alias: {
      '@': fileURLToPath(new URL('./src', import.meta.url)),
    },
  },
})
```

Vous pourrez importer :

```js
import MyComp from '@/components/MyComp.vue'
```

### 5.3 Qualité de code : ESLint & Prettier (recommandé)

Approche simple :

- Installer ESLint + plugin Vue
- Installer Prettier
- Configurer des scripts `lint` et `format`

Selon le niveau de la formation, vous pouvez :

- soit ajouter ces outils dès le départ,
- soit les introduire dans un module dédié.

---

## 6) Alternative : Vue CLI (pour maintenance / legacy)

### 6.1 Créer un projet (si nécessaire)

Vue CLI s’installe souvent globalement (ou via npx). Exemple :

```bash
npm install -g @vue/cli
vue create my-cli-project
```

Vous choisirez un preset (Babel, TypeScript, Router, Vuex/Pinia, ESLint...).

### 6.2 Différences clés avec Vite

- Vue CLI repose sur **webpack** (plus lourd, configuration plus complexe)
- Vite repose sur une approche plus moderne (ESM en dev)
- Les temps de démarrage et HMR sont généralement meilleurs avec Vite

> Recommandation pédagogique : connaître Vue CLI pour lire/maintenir l’existant, mais démarrer les nouveaux projets avec Vite.

---

## 7) Atelier récapitulatif (guidé)

### Exercice A — Créer et lancer un projet Vue avec Vite

1. Créez un projet :
   ```bash
   npm create vite@latest
   ```
2. Choisissez `Vue` puis `JavaScript`.
3. Installez les dépendances :
   ```bash
   npm install
   ```
4. Lancez :
   ```bash
   npm run dev
   ```

### Exercice B — Créer un composant

1. Créez `src/components/Greeting.vue`
2. Ajoutez un template simple avec une prop `name`
3. Appelez ce composant depuis `App.vue`

### Exercice C — Build et preview

1. Build :
   ```bash
   npm run build
   ```
2. Preview :
   ```bash
   npm run preview
   ```

---

## Conclusion

À l’issue de ce module, vous savez :

- Utiliser Vue rapidement via **CDN** (prototype / intégration)
- Créer une app Vue moderne avec **Vite** (`npm create vite@latest`)
- Comprendre la structure d’un projet Vue et les scripts principaux
- Identifier la place de **Vue CLI** dans un contexte legacy

---

### Annexe — Commandes utiles (récap)

```bash
# Créer un projet Vite
npm create vite@latest

# Installer
npm install

# Dev server
npm run dev

# Build
npm run build

# Preview du build
npm run preview
```
