# Formation Vue.js — Réactivité

## Objectifs pédagogiques
À la fin de cette formation, vous serez capable de :

- Expliquer le rôle de la réactivité dans Vue.js et son impact sur le DOM.
- Utiliser correctement `ref()` et `reactive()` pour créer des états réactifs.
- Comprendre le **suivi des dépendances** (tracking) et le **déclenchement des mises à jour** (triggering).
- Identifier les cas d’usage adaptés à `ref` vs `reactive`.
- Éviter les pièges fréquents (destructuring, reassign, mutation, etc.).
- Appliquer de bonnes pratiques pour des composants performants et lisibles.

---

## Prérequis
- JavaScript moderne (ES6+)
- Notions de base de Vue 3 (Single File Components, template, props)
- Connaître le Composition API est un plus (sinon, la formation l’introduit au fil de l’eau)

---

## Public cible
- Développeurs front-end
- Formateurs/enseignants Vue.js
- Développeurs Vue 2 souhaitant migrer leurs réflexes vers Vue 3

---

## Plan de la formation

1. **Comprendre la réactivité dans Vue.js**
   - Qu’est-ce que la réactivité ?
   - DOM, Virtual DOM et rendu
   - Un modèle mental : _data → render → DOM_

2. **Le cœur du système : tracking & triggering**
   - Suivi des dépendances
   - Déclenchement
   - Rôle des Proxies et getters/setters

3. **Créer des données réactives avec `ref()`**
   - Principe
   - `.value` en JavaScript
   - Auto-unwrapping dans le template
   - Références sur les types primitifs vs objets

4. **Créer des données réactives avec `reactive()`**
   - Principe
   - Proxies et mutations
   - Deep reactivity
   - Limitations (réassign, destructuring)

5. **Choisir entre `ref` et `reactive`**
   - Règles simples
   - Patterns courants
   - Cas limites

6. **Mises à jour du DOM & cycle de rendu**
   - Mise à jour asynchrone (batching)
   - `nextTick()`
   - Démonstrations

7. **Exercices & corrections**
   - Exercice 1 : compteur
   - Exercice 2 : formulaire et objet `reactive`
   - Exercice 3 : liste filtrée
   - Exercice 4 : pièges (destructuring/reassign)

8. **Bonnes pratiques & anti-patterns**
   - Lisibilité, performance, cohérence
   - Centraliser l’état (composables)
   - Debug et outils

---

# 1) Comprendre la réactivité dans Vue.js

## 1.1 Définition
La **réactivité** est la capacité d’un système à **réagir automatiquement** à des changements de données.

Dans Vue.js, cela signifie :

- Vous déclarez un état (state) réactif.
- Vous l’utilisez dans le template.
- Quand cet état change, Vue **recalcule** ce qui dépend de cet état et **met à jour le DOM**.

### Modèle mental

```text
État (data) → Rendu (render) → DOM
        ↑                 ↓
      modifications    mise à jour
```

Vous n’avez pas besoin d’écrire manuellement :
- “quand `count` change, change le texte HTML”,
- ni de manipuler le DOM avec `document.querySelector(...)`.

Vue fait le lien **automatiquement**.

## 1.2 Réactivité et DOM
Vue rend votre interface à partir de votre état. Quand l’état change :

- Vue calcule une nouvelle représentation (Virtual DOM)
- puis applique uniquement les différences (diff/patch)

Résultat : le DOM est mis à jour efficacement.

---

# 2) Le cœur du système : tracking & triggering

L’idée principale : Vue doit répondre à deux questions.

1. **Quelles parties du rendu dépendent de quelles données ?** → **Tracking**
2. **Quand une donnée change, que faut-il recalculer ?** → **Triggering**

## 2.1 Tracking (suivi des dépendances)
Quand un rendu (ou une computed, ou un watcher) accède à une valeur réactive, Vue enregistre :

- “Cette fonction dépend de cette propriété”.

En termes simples : *lire une donnée réactive, ça crée une dépendance*.

## 2.2 Triggering (déclenchement)
Quand une valeur réactive est modifiée :

- Vue identifie toutes les fonctions qui dépendent de cette valeur.
- Vue les relance (ou planifie leur relance) pour mettre à jour le rendu.

## 2.3 Comment Vue fait techniquement ?
- `reactive()` s’appuie sur des **Proxies**.
- `ref()` encapsule une valeur dans un objet réactif spécial.

Vue intercepte :
- les **lectures** (get) pour tracker
- les **écritures** (set) pour trigger

---

# 3) Créer des données réactives avec `ref()`

## 3.1 Principe
`ref()` sert à créer une **référence réactive** autour d’une valeur.

```js
import { ref } from 'vue'

const count = ref(0)
```

Ici :
- `count` est un objet ref.
- La valeur est accessible via `count.value`.

## 3.2 `.value` en JavaScript

```js
count.value++
console.log(count.value)
```

👉 Pourquoi `.value` ?
- Parce qu’un ref **encapsule** la valeur.
- Cela permet à Vue d’intercepter les accès et modifications.

## 3.3 Auto-unwrapping dans le template
Dans le template, Vue “déplie” automatiquement le ref :

```vue
<script setup>
import { ref } from 'vue'

const count = ref(0)
</script>

<template>
  <button @click="count++">Clicked {{ count }} times</button>
</template>
```

- En template : `count` est utilisé directement.
- En JS : vous utilisez `count.value`.

## 3.4 `ref` avec objets et tableaux
`ref()` peut aussi contenir un objet.

```js
const user = ref({ name: 'Ada', age: 28 })

user.value.name = 'Grace'
```

Dans ce cas :
- `user` est une ref
- l’objet interne est **réactif** (Vue rend les objets imbriqués réactifs)

### Exemple : tableau

```js
const items = ref(['A', 'B'])
items.value.push('C')
```

## 3.5 Quand utiliser `ref()` ?
- Pour des **primitifs** : number, string, boolean
- Pour des références simples : éléments DOM, timers, etc.
- Quand vous avez besoin d’une variable “single value” claire

---

# 4) Créer des données réactives avec `reactive()`

## 4.1 Principe
`reactive()` prend un objet (ou tableau) et retourne un **Proxy** réactif.

```js
import { reactive } from 'vue'

const state = reactive({
  count: 0,
  user: { name: 'Ada' }
})
```

Ici :
- `state` se manipule directement
- pas de `.value`

## 4.2 Mutation : le cas naturel de `reactive`

```js
state.count++
state.user.name = 'Grace'
```

## 4.3 Deep reactivity (réactivité profonde)
Par défaut, Vue rend **réactives** les propriétés imbriquées :

```js
state.user.name = 'Alan'
```

Cela déclenche bien une mise à jour si le template dépend de `state.user.name`.

## 4.4 Limitations importantes

### 4.4.1 Réassigner un objet `reactive`
Si vous faites :

```js
let state = reactive({ count: 0 })
state = { count: 10 } // ❌ casse le lien réactif
```

Vous remplacez la référence par un objet non réactif.

✅ Solution : muter les propriétés existantes :

```js
state.count = 10
```

ou bien utiliser un `ref` pour pouvoir réassigner :

```js
const state = ref({ count: 0 })
state.value = { count: 10 } // ✅
```

### 4.4.2 Destructuring (déstructuration)
Piège classique :

```js
const state = reactive({ count: 0 })
const { count } = state

count++ // ❌ ne met pas à jour state.count
```

Car `count` devient une **copie** de la valeur.

✅ Solution : utiliser `toRefs()` si vous devez déstructurer :

```js
import { reactive, toRefs } from 'vue'

const state = reactive({ count: 0 })
const { count } = toRefs(state)

count.value++ // ✅ met à jour state.count
```

---

# 5) Choisir entre `ref` et `reactive`

## 5.1 Règles simples
- **Primitif** → `ref()`
- **Objet avec plusieurs champs (state “structuré”)** → `reactive()`
- **Besoin de réassigner un objet complet** → `ref({ ... })`

## 5.2 Exemples par scénario

### Scénario A: compteur

```js
const count = ref(0)
```

### Scénario B: formulaire

```js
const form = reactive({
  email: '',
  password: '',
  remember: true
})
```

### Scénario C: état d’un composable
Composables : usage mixte très courant.

```js
export function useCounter() {
  const count = ref(0)
  const inc = () => count.value++
  return { count, inc }
}
```

---

# 6) Mises à jour du DOM & cycle de rendu

## 6.1 Mise à jour asynchrone (batching)
Quand vous changez plusieurs fois l’état dans le même “tick”, Vue regroupe les mises à jour.

```js
count.value++
count.value++
count.value++
```

Le DOM ne sera généralement mis à jour qu’une seule fois après ce bloc.

## 6.2 `nextTick()`
Si vous avez besoin d’attendre que le DOM soit mis à jour :

```js
import { nextTick, ref } from 'vue'

const count = ref(0)

async function incAndMeasure() {
  count.value++
  await nextTick()
  // ici, le DOM reflète la nouvelle valeur
}
```

---

# 7) Exercices & corrections

## Exercice 1 — Compteur (ref)
**Objectif :** utiliser `ref` et vérifier la mise à jour du DOM.

**Énoncé :**
- Créer un compteur avec un bouton `+1` et un bouton `Reset`.
- Afficher `count` dans le template.

**Correction :**

```vue
<script setup>
import { ref } from 'vue'

const count = ref(0)

function inc() {
  count.value++
}

function reset() {
  count.value = 0
}
</script>

<template>
  <p>Count: {{ count }}</p>
  <button @click="inc">+1</button>
  <button @click="reset">Reset</button>
</template>
```

---

## Exercice 2 — Formulaire (reactive)
**Objectif :** utiliser `reactive` sur un objet de formulaire.

**Énoncé :**
- Créer un formulaire `email`, `password`.
- Afficher un aperçu en dessous.

**Correction :**

```vue
<script setup>
import { reactive } from 'vue'

const form = reactive({
  email: '',
  password: ''
})
</script>

<template>
  <label>
    Email
    <input v-model="form.email" />
  </label>

  <label>
    Password
    <input type="password" v-model="form.password" />
  </label>

  <pre>{{ form }}</pre>
</template>
```

---

## Exercice 3 — Liste filtrée
**Objectif :** montrer la propagation des changements.

**Énoncé :**
- `query` (ref)
- `items` (ref ou reactive)
- filtrer via une computed (optionnel si vous élargissez)

**Correction (sans computed, simple) :**

```vue
<script setup>
import { ref } from 'vue'

const query = ref('')
const items = ref(['Vue', 'React', 'Svelte', 'Angular'])
</script>

<template>
  <input v-model="query" placeholder="Search..." />

  <ul>
    <li v-for="it in items.filter(i => i.toLowerCase().includes(query.toLowerCase()))" :key="it">
      {{ it }}
    </li>
  </ul>
</template>
```

---

## Exercice 4 — Pièges (destructuring)
**Objectif :** comprendre pourquoi la déstructuration casse la réactivité et comment la restaurer.

**Énoncé :**
- Créer un `state` reactive avec `count`.
- Déstructurer `count` (version bug).
- Corriger avec `toRefs`.

**Correction :**

```js
import { reactive, toRefs } from 'vue'

const state = reactive({ count: 0 })

// ✅ safe destructuring
const { count } = toRefs(state)

count.value++
```

---

# 8) Bonnes pratiques & anti-patterns

## 8.1 Bonnes pratiques
- Préférer `ref` pour les valeurs simples, surtout en composables.
- Utiliser `reactive` pour des objets de formulaire, états structurés.
- Éviter la déstructuration directe d’un `reactive` sans `toRefs`.
- Garder les mutations explicites et localisées.

## 8.2 Anti-patterns fréquents
- Réassigner un objet `reactive` (perte de réactivité).
- Copier des valeurs réactives dans des variables non réactives.
- Faire des mises à jour en cascade dans plusieurs endroits (préférer un composable / store cohérent).

## 8.3 Debug
- Utiliser Vue Devtools pour inspecter l’état et les composants.
- Loguer les `.value` des refs côté JS.

---

## Conclusion
Le système de réactivité de Vue est la base de son ergonomie : vous travaillez avec de la donnée, et Vue s’occupe de synchroniser l’interface. `ref()` et `reactive()` sont les deux principaux outils pour créer des données réactives.

- `ref()` : idéal pour les primitives et les valeurs isolées.
- `reactive()` : idéal pour les objets et états structurés (mutations directes).

En maîtrisant les règles d’usage et les pièges (réassign, destructuring), vous gagnez en fiabilité, lisibilité et efficacité.
