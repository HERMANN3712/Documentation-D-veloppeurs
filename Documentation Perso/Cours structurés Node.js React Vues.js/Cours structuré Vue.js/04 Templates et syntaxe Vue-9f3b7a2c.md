# Formation Vue.js — Templates et syntaxe Vue

> Public : développeurs (débutants à intermédiaires) souhaitant maîtriser la syntaxe de template Vue (Vue 3) et ses directives pour produire un DOM dynamique, lisible et maintenable.

---

## Objectifs pédagogiques

À l’issue de cette formation, vous saurez :

- Expliquer le rôle du **template** dans un composant Vue.
- Utiliser les **interpolations** (moustaches `{{ }}`) et comprendre leurs limites.
- Manipuler les attributs HTML avec **`v-bind`** et les événements avec **`v-on`**.
- Contrôler le rendu via les **directives** (`v-if`, `v-show`, `v-for`, `v-model`, etc.).
- Composer une UI dynamique en respectant les bonnes pratiques (lisibilité, performance, sécurité).

---

## Prérequis

- JavaScript (ES6+), HTML/CSS.
- Notions de composants.
- Un projet Vue 3 initialisé (Vite recommandé) ou Vue Playground.

---

## Plan de la formation

1. **Rappels : structure d’un composant Vue**
2. **Templates : principes et contraintes**
3. **Interpolations `{{ }}` : expressions JavaScript dans le HTML**
4. **Directives Vue : contrôler le DOM dynamiquement**
   - `v-bind` (liaison d’attributs)
   - `v-on` (gestion d’événements)
   - `v-if / v-else-if / v-else` (conditionnel)
   - `v-show` (affichage conditionnel)
   - `v-for` (listes)
   - `v-model` (liaison bidirectionnelle)
   - `v-html` (injection HTML) et sécurité
   - `v-pre`, `v-once`, `v-memo` (optimisations)
5. **Syntaxe abrégée (shorthands) et conventions**
6. **Bonnes pratiques et pièges fréquents**
7. **Ateliers guidés (exercices + corrections)**
8. **Synthèse et checklist**

---

## 1) Rappels : structure d’un composant Vue

En Vue 3 (SFC — Single File Component), un composant est typiquement découpé en :

- `<template>` : description déclarative du DOM.
- `<script setup>` (ou `<script>`) : logique, état, fonctions.
- `<style>` : styles.

Exemple minimal :

```vue
<script setup>
import { ref } from 'vue'

const count = ref(0)
const increment = () => count.value++
</script>

<template>
  <button @click="increment">Compteur: {{ count }}</button>
</template>
```

>:warning: Remarque importante

Dans un template, **`ref` est auto-déballé (auto-unwrapped)** : on écrit `{{ count }}` et non `{{ count.value }}`.

---

## 2) Templates : principes et contraintes

### 2.1 Le template : une surcouche HTML

Vue utilise une syntaxe de template **basée sur HTML**. Vous écrivez un HTML « augmentée » :

- vous conservez la structure HTML,
- vous ajoutez des **directives** (attributs spéciaux `v-...`),
- vous insérez des **expressions JavaScript** via `{{ ... }}`.

### 2.2 Un template doit renvoyer un seul nœud racine

En Vue 3, un template de composant peut avoir **plusieurs racines** (fragment) dans un SFC. Mais selon les contextes (certaines intégrations, transitions, etc.), un wrapper peut rester utile.

Exemple avec deux racines :

```vue
<template>
  <header>...</header>
  <main>...</main>
</template>
```

### 2.3 Ce que peut contenir une expression de template

Dans `{{ ... }}` et certains bindings, vous écrivez **des expressions**, pas des instructions.

✅ Autorisé :

- opérations : `a + b`, `price * 1.2`
- ternaires : `isAdmin ? 'Admin' : 'User'`
- appels de fonctions : `format(date)`
- accès objet : `user.name`

❌ Non autorisé (dans une moustache) :

- `if (...) { ... }`
- `for (...) { ... }`
- `const x = ...`

> Conseil : si votre expression devient longue, déplacez-la dans une **computed** ou une fonction.

---

## 3) Interpolations `{{ }}` : expressions JavaScript dans le HTML

### 3.1 Afficher une variable

```vue
<script setup>
import { ref } from 'vue'
const name = ref('Ada')
</script>

<template>
  <p>Bonjour {{ name }}.</p>
</template>
```

### 3.2 Interpolation d’expression

```vue
<template>
  <p>Total: {{ quantity * unitPrice }} €</p>
  <p>{{ isOnline ? 'En ligne' : 'Hors ligne' }}</p>
</template>
```

### 3.3 Interpolation et texte brut

Si vous voulez afficher les moustaches **littéralement** (sans interprétation), utilisez `v-pre` sur un bloc :

```html
<p v-pre>Affiche littéralement: {{ message }}</p>
```

### 3.4 Limite : interpolation dans les attributs HTML

On **n’utilise pas** `{{ }}` dans la plupart des attributs HTML. Pour ça, on utilise `v-bind`.

Mauvais :

```html
<img src="{{ imageUrl }}" />
```

Correct :

```html
<img :src="imageUrl" />
```

---

## 4) Directives Vue : contrôler le DOM dynamiquement

Les **directives** sont des attributs commençant par `v-` et qui pilotent le rendu.

### 4.1 `v-bind` — Lier dynamiquement des attributs / props

#### 4.1.1 Bind d’attribut simple

```vue
<script setup>
import { ref } from 'vue'
const avatar = ref('https://example.com/me.png')
</script>

<template>
  <img :src="avatar" alt="Avatar" />
</template>
```

#### 4.1.2 Bind d’une classe (string, objet, tableau)

**String** :

```html
<div :class="'card is-active'">...</div>
```

**Objet** (recommandé) :

```vue
<script setup>
const isActive = true
const hasError = false
</script>

<template>
  <div :class="{ active: isActive, error: hasError }">...</div>
</template>
```

**Tableau** :

```html
<div :class="['card', isActive ? 'active' : '']">...</div>
```

#### 4.1.3 Bind d’un style

```vue
<script setup>
const color = 'rebeccapurple'
const size = 18
</script>

<template>
  <p :style="{ color, fontSize: size + 'px' }">Texte stylé</p>
</template>
```

#### 4.1.4 Bind de plusieurs attributs avec un objet

```vue
<script setup>
const attrs = {
  id: 'main-title',
  title: 'Survoler',
  'aria-label': 'Titre principal'
}
</script>

<template>
  <h1 v-bind="attrs">Bienvenue</h1>
</template>
```

---

### 4.2 `v-on` — Écouter des événements

#### 4.2.1 Exemple simple

```vue
<script setup>
import { ref } from 'vue'
const count = ref(0)
</script>

<template>
  <button @click="count++">+1</button>
  <p>Count: {{ count }}</p>
</template>
```

#### 4.2.2 Passer des arguments

```vue
<script setup>
const select = (id) => console.log('selected', id)
</script>

<template>
  <button @click="select(42)">Sélectionner</button>
</template>
```

#### 4.2.3 Modifiers (modificateurs)

- `.prevent` : `event.preventDefault()`
- `.stop` : `event.stopPropagation()`
- `.capture`, `.once`, `.passive`
- Modifiers clavier : `.enter`, `.esc`, etc.

Exemples :

```html
<form @submit.prevent="onSubmit">
  <input @keyup.enter="onSubmit" />
</form>
```

---

### 4.3 `v-if / v-else-if / v-else` — Rendu conditionnel

`v-if` **ajoute/retire réellement** les nœuds du DOM.

```vue
<script setup>
import { ref } from 'vue'
const isLogged = ref(false)
</script>

<template>
  <button @click="isLogged = !isLogged">
    Toggle
  </button>

  <p v-if="isLogged">Bienvenue !</p>
  <p v-else>Veuillez vous connecter.</p>
</template>
```

#### 4.3.1 `template` + condition

Quand vous devez conditionner **plusieurs éléments** sans wrapper visible :

```html
<template v-if="isLogged">
  <h2>Dashboard</h2>
  <p>Contenu privé</p>
</template>
```

---

### 4.4 `v-show` — Affichage conditionnel

`v-show` **ne retire pas** l’élément : il change son `display` CSS.

```html
<p v-show="isOpen">Je suis visible si isOpen est vrai</p>
```

**Quand utiliser quoi ?**

- `v-if` : coût initial plus faible si rarement affiché, mais toggles plus chers (création/destruction).
- `v-show` : coût initial plus élevé (rendu initial), toggles très rapides.

---

### 4.5 `v-for` — Rendu de listes

#### 4.5.1 Parcourir un tableau

```vue
<script setup>
const users = [
  { id: 1, name: 'Ada' },
  { id: 2, name: 'Alan' }
]
</script>

<template>
  <ul>
    <li v-for="user in users" :key="user.id">
      {{ user.name }}
    </li>
  </ul>
</template>
```

#### 4.5.2 Index

```html
<li v-for="(user, index) in users" :key="user.id">
  {{ index + 1 }} — {{ user.name }}
</li>
```

#### 4.5.3 Parcourir un objet

```vue
<script setup>
const settings = { theme: 'dark', lang: 'fr' }
</script>

<template>
  <ul>
    <li v-for="(value, key) in settings" :key="key">
      {{ key }}: {{ value }}
    </li>
  </ul>
</template>
```

#### 4.5.4 Bonnes pratiques `:key`

- Toujours fournir un `:key` **stable et unique**.
- Éviter `index` comme `key` si la liste peut être réordonnée/filtrée.

---

### 4.6 `v-model` — Liaison bidirectionnelle (forms)

`v-model` synchronise la valeur d’un champ avec une variable.

```vue
<script setup>
import { ref } from 'vue'
const email = ref('')
</script>

<template>
  <label>
    Email
    <input v-model="email" type="email" />
  </label>

  <p>Vous avez saisi: {{ email }}</p>
</template>
```

#### 4.6.1 Modifiers de `v-model`

- `.trim` : supprime espaces début/fin
- `.number` : cast en nombre (quand possible)
- `.lazy` : sync sur `change` au lieu de `input`

```html
<input v-model.trim="name" />
<input v-model.number="age" type="number" />
<input v-model.lazy="query" />
```

---

### 4.7 `v-html` — Injection HTML (avec prudence)

`v-html` insère du contenu HTML brut.

```vue
<script setup>
const raw = '<strong>Attention</strong> : contenu HTML.'
</script>

<template>
  <div v-html="raw"></div>
</template>
```

:warning: **Sécurité (XSS)**

- N’utilisez jamais `v-html` avec du contenu non maîtrisé.
- Préférez construire le rendu en template (composants, slots) plutôt que d’injecter du HTML.

---

### 4.8 Directives “utilitaires” : `v-pre`, `v-once`, `v-memo`

#### 4.8.1 `v-pre`

Ignore la compilation du template dans le bloc (affiche `{{ }}` tel quel).

```html
<code v-pre>{{ notEvaluated }}</code>
```

#### 4.8.2 `v-once`

Rend une seule fois (ne se met plus à jour).

```html
<p v-once>Rendu initial: {{ count }}</p>
```

#### 4.8.3 `v-memo` (Vue 3.2+)

Mémorise un sous-arbre tant que les dépendances ne changent pas.

```html
<div v-memo="[user.id]">
  <!-- sera réutilisé tant que user.id ne change pas -->
  <UserCard :user="user" />
</div>
```

---

## 5) Syntaxe abrégée (shorthands) et conventions

### 5.1 Abréviations

- `v-bind:src="x"` → `:src="x"`
- `v-on:click="fn"` → `@click="fn"`

Exemple :

```html
<a :href="url" @click.prevent="track">Lien</a>
```

### 5.2 Conventions de nommage

- Éviter les templates trop “logiques” : gardez-les lisibles.
- Préférer des fonctions `onClick`, `onSubmit`, `toggleX`.
- Extraire des sous-composants lorsque le template grossit.

---

## 6) Bonnes pratiques et pièges fréquents

### 6.1 Computed vs méthodes dans le template

- **Computed** : mise en cache, dépendances réactives, idéal pour les dérivations.
- **Méthodes** : réévaluées à chaque rendu.

Exemple :

```vue
<script setup>
import { ref, computed } from 'vue'

const first = ref('Ada')
const last = ref('Lovelace')

const fullName = computed(() => `${first.value} ${last.value}`)
</script>

<template>
  <p>{{ fullName }}</p>
</template>
```

### 6.2 `v-if` + `v-for` sur le même élément

À éviter (ambiguïtés et lisibilité). Préférez :

- filtrer la liste en amont (computed), ou
- envelopper avec `<template>`.

```html
<template v-for="u in visibleUsers" :key="u.id">
  <UserRow :user="u" />
</template>
```

### 6.3 Optimiser le rendu de listes

- `:key` stable
- composants “purs” quand possible
- utiliser `v-memo` si nécessaire

### 6.4 Éviter la logique complexe inline

Mauvais :

```html
<p>{{ items.filter(i => i.ok).map(i => i.name).join(', ') }}</p>
```

Mieux :

```vue
<script setup>
import { computed } from 'vue'
const items = [{ name: 'A', ok: true }, { name: 'B', ok: false }]
const okNames = computed(() => items.filter(i => i.ok).map(i => i.name).join(', '))
</script>

<template>
  <p>{{ okNames }}</p>
</template>
```

---

## 7) Ateliers guidés (exercices + corrections)

### Atelier 1 — Carte produit (interpolation + `v-bind`)

**Énoncé** : afficher un produit (nom, prix, image). Ajouter une classe `sale` si `onSale` est vrai.

**Correction** :

```vue
<script setup>
const product = {
  name: 'Clavier mécanique',
  price: 129.9,
  image: 'https://picsum.photos/seed/keyboard/240/160',
  onSale: true
}
</script>

<template>
  <article class="product" :class="{ sale: product.onSale }">
    <img :src="product.image" :alt="product.name" />
    <h3>{{ product.name }}</h3>
    <p>Prix : {{ product.price }} €</p>
    <p v-if="product.onSale">Promo en cours</p>
  </article>
</template>

<style scoped>
.product { border: 1px solid #ddd; padding: 12px; border-radius: 8px; max-width: 320px; }
.sale { border-color: #b00020; }
</style>
```

### Atelier 2 — Toggle (différences `v-if` vs `v-show`)

**Énoncé** : un bouton qui affiche/masque un panneau. Comparez `v-if` et `v-show`.

**Correction** :

```vue
<script setup>
import { ref } from 'vue'
const open = ref(false)
</script>

<template>
  <button @click="open = !open">Toggle</button>

  <section>
    <h4>Avec v-if</h4>
    <p v-if="open">Je suis créé/détruit.</p>

    <h4>Avec v-show</h4>
    <p v-show="open">Je reste dans le DOM (display:none).</p>
  </section>
</template>
```

### Atelier 3 — Liste filtrée (`v-for` + computed)

**Énoncé** : afficher seulement les utilisateurs actifs.

**Correction** :

```vue
<script setup>
import { computed } from 'vue'

const users = [
  { id: 1, name: 'Ada', active: true },
  { id: 2, name: 'Alan', active: false },
  { id: 3, name: 'Grace', active: true }
]

const activeUsers = computed(() => users.filter(u => u.active))
</script>

<template>
  <ul>
    <li v-for="u in activeUsers" :key="u.id">{{ u.name }}</li>
  </ul>
</template>
```

### Atelier 4 — Formulaire (`v-model` + submit)

**Énoncé** : créer un mini formulaire (email + opt-in). Afficher le résultat en dessous.

**Correction** :

```vue
<script setup>
import { reactive } from 'vue'

const form = reactive({
  email: '',
  optin: false
})

const submit = () => {
  // ici on ferait un appel API
  console.log('submit', { ...form })
}
</script>

<template>
  <form @submit.prevent="submit">
    <label>
      Email
      <input v-model.trim="form.email" type="email" required />
    </label>

    <label>
      <input v-model="form.optin" type="checkbox" />
      Recevoir la newsletter
    </label>

    <button type="submit">Envoyer</button>
  </form>

  <pre>{{ form }}</pre>
</template>
```

---

## 8) Synthèse et checklist

### Checklist “Template Vue”

- [ ] J’utilise `{{ }}` uniquement pour du texte/interpolation.
- [ ] J’utilise `:attr` (`v-bind`) pour les attributs/props.
- [ ] J’utilise `@event` (`v-on`) pour les événements.
- [ ] J’utilise `v-if` pour le rendu conditionnel (DOM créé/détruit).
- [ ] J’utilise `v-show` pour des toggles fréquents.
- [ ] J’utilise `v-for` avec une `:key` unique et stable.
- [ ] J’utilise `v-model` pour les formulaires.
- [ ] J’évite `v-html` avec du contenu non sûr.
- [ ] Je garde le template lisible (computed / sous-composants).

---

## Annexes — Mémo rapide des directives

| Besoin | Directive / syntaxe | Exemple |
|---|---|---|
| Afficher texte | `{{ expr }}` | `{{ user.name }}` |
| Attribut dynamique | `:attr="expr"` | `:href="url"` |
| Événement | `@event="handler"` | `@click="save"` |
| Conditionnel | `v-if / v-else` | `v-if="ok"` |
| Afficher/masquer | `v-show` | `v-show="open"` |
| Liste | `v-for` + `:key` | `v-for="i in items"` |
| Formulaire | `v-model` | `v-model="email"` |
| HTML brut | `v-html` | `v-html="raw"` |
| Ignorer compilation | `v-pre` | `<code v-pre>` |
| Une seule fois | `v-once` | `<p v-once>` |
| Mémo | `v-memo` | `v-memo="[id]"` |
