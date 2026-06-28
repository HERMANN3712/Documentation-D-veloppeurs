# Formation Vue.js — Props et communication entre composants

> **Objectif général** : maîtriser la transmission de données **du parent vers l’enfant** via les **props**, et la remontée d’informations **de l’enfant vers le parent** via les **événements personnalisés**.  
> **Public** : développeurs Vue.js (débutant à intermédiaire).  
> **Pré-requis** : bases de Vue 3 (SFC, template/script, réactivité).  
> **Durée indicative** : 2h30 à 4h (selon ateliers).

---

## Plan de la formation

1. **Contexte : architecture composant et flux de données**
2. **Props : transmettre des données du parent vers l’enfant**
   1. Définition et cas d’usage
   2. Déclaration des props (Vue 3)
   3. Types, valeurs par défaut, validation
   4. Sens unique : pourquoi les props sont "read-only"
   5. Props et réactivité : pièges et bonnes pratiques
   6. Props avancées : `v-bind`, objets, tableaux, `defineProps` + TS
3. **Événements personnalisés : remonter une information de l’enfant vers le parent**
   1. `$emit` et écoute via `@event`
   2. Payload d’événement et conventions de nommage
   3. Déclaration avec `defineEmits` (et typage)
   4. Pattern "événement + état" et gestion des formulaires
4. **Communication bidirectionnelle contrôlée : `v-model` sur composant**
   1. `modelValue` + `update:modelValue`
   2. `v-model` avec argument (multi v-model)
5. **Ateliers pratiques (pas à pas)**
6. **Erreurs fréquentes + checklist de code review**
7. **Résumé + mini quiz**

---

## 1) Contexte : architecture composant et flux de données

Vue invite à structurer l’UI en **composants**. Dans une application, on a souvent :

- un **composant parent** qui orchestre l’état (state)
- des **composants enfants** qui affichent des informations ou déclenchent des actions

### Règle d’or
- **Parent → Enfant** : via **props** (données)
- **Enfant → Parent** : via **événements** (intention/action)

Ce modèle rend le code plus prévisible : un **flux de données descendant** et une **remontée d’événements**.

---

## 2) Props : transmettre des données du parent vers l’enfant

### 2.1 Définition
Une **prop** est un attribut exposé par un composant enfant pour recevoir une valeur du parent.

Cas d’usage typiques :
- afficher un texte, un prix, une date
- paramétrer un composant (taille, thème, état désactivé)
- passer une liste d’éléments à rendre

---

### 2.2 Déclaration des props (Vue 3)

#### Option A — `<script setup>` avec `defineProps`

**ChildBadge.vue**
```vue
<script setup>
const props = defineProps({
  label: {
    type: String,
    required: true
  },
  tone: {
    type: String,
    default: 'info'
  }
})
</script>

<template>
  <span class="badge" :data-tone="props.tone">
    {{ props.label }}
  </span>
</template>
```

**Parent.vue**
```vue
<script setup>
import ChildBadge from './ChildBadge.vue'
</script>

<template>
  <ChildBadge label="Nouveau" tone="success" />
</template>
```

#### Option B — API Options (pour lecture / legacy)
```js
export default {
  props: {
    label: { type: String, required: true },
    tone: { type: String, default: 'info' }
  }
}
```

---

### 2.3 Types, valeurs par défaut, validation

#### Types primitifs et multiples
```js
defineProps({
  count: Number,
  title: [String, Number]
})
```

#### Valeur par défaut (attention aux objets/tableaux)
Pour **Array/Object**, la valeur par défaut doit être une **fonction**.
```js
defineProps({
  items: {
    type: Array,
    default: () => []
  },
  config: {
    type: Object,
    default: () => ({ pageSize: 20 })
  }
})
```

#### Validation personnalisée
```js
defineProps({
  size: {
    type: String,
    default: 'md',
    validator: (v) => ['sm', 'md', 'lg'].includes(v)
  }
})
```

---

### 2.4 Sens unique : pourquoi les props sont "read-only"

Dans Vue, une prop représente une **valeur possédée par le parent**.

- Le **parent** est la source de vérité.
- L’**enfant** ne doit pas modifier directement cette valeur, sinon le flux devient ambigu.

Si dans un enfant vous tentez de faire :
```js
props.label = '...' 
```
Vue vous avertira (et en mode strict cela peut être bloquant selon contexte).

#### Que faire si l’enfant doit “éditer” une valeur reçue ?
Deux approches courantes :

1) **Copier en état local** (pour édition temporaire) :
```vue
<script setup>
import { ref, watch } from 'vue'

const props = defineProps({
  model: { type: String, required: true }
})

const localValue = ref(props.model)

watch(() => props.model, (nv) => {
  localValue.value = nv
})
</script>
```

2) **Remonter l’intention via un événement** (recommandé)

---

### 2.5 Props et réactivité : pièges et bonnes pratiques

#### Piège : déstructurer les props sans précaution
En `<script setup>`, si vous faites :
```js
const { label } = defineProps({ label: String })
```
Vous risquez de **perdre la réactivité** dans certains cas (selon usages). Préférez :
- utiliser `props.label` directement
- ou `toRefs(props)` si vous avez besoin d’extraire

```js
import { toRefs } from 'vue'
const props = defineProps({ label: String })
const { label } = toRefs(props)
```

#### Bonnes pratiques
- Utiliser des props **petites et cohérentes** (éviter de passer un énorme objet "fourre-tout")
- Documenter les props (types, valeurs possibles)
- Préférer des noms explicites : `isDisabled`, `variant`, `items`, `onConfirm` (éviter `data`)

---

### 2.6 Props avancées : `v-bind`, objets, tableaux, TypeScript

#### Passer un objet complet avec `v-bind`
```vue
<ChildBadge v-bind="{ label: 'Promo', tone: 'warning' }" />
```

#### Avec TypeScript (typé)
```vue
<script setup lang="ts">
type Tone = 'info' | 'success' | 'warning' | 'danger'

defineProps<{ 
  label: string
  tone?: Tone
}>()
</script>
```

---

## 3) Événements personnalisés : remonter une info de l’enfant vers le parent

### 3.1 `$emit` et écoute via `@event`

Les **événements personnalisés** permettent à l’enfant d’exprimer une intention.

- Enfant : `emit('confirm')`
- Parent : `@confirm="handler"`

**ChildConfirmButton.vue**
```vue
<script setup>
const emit = defineEmits(['confirm'])
</script>

<template>
  <button type="button" @click="emit('confirm')">
    Confirmer
  </button>
</template>
```

**Parent.vue**
```vue
<script setup>
import ChildConfirmButton from './ChildConfirmButton.vue'

function onConfirm() {
  // action côté parent
  console.log('confirm reçu')
}
</script>

<template>
  <ChildConfirmButton @confirm="onConfirm" />
</template>
```

---

### 3.2 Payload d’événement et conventions de nommage

L’événement peut transporter des données (payload).

**ChildItem.vue**
```vue
<script setup>
const props = defineProps({ id: Number, label: String })
const emit = defineEmits(['remove'])
</script>

<template>
  <div class="row">
    <span>{{ props.label }}</span>
    <button @click="emit('remove', props.id)">Supprimer</button>
  </div>
</template>
```

**Parent.vue**
```vue
<template>
  <ChildItem
    v-for="it in items"
    :key="it.id"
    :id="it.id"
    :label="it.label"
    @remove="removeById"
  />
</template>
```

**Conventions**
- Nom d’événement en **kebab-case** côté template: `@item-selected`
- Côté `emit`, utiliser la même chaîne: `emit('item-selected', payload)`
- Choisir des noms orientés action : `save`, `cancel`, `remove`, `update`, `select`

---

### 3.3 Déclaration avec `defineEmits` (et typage)

#### Version simple
```js
const emit = defineEmits(['save', 'cancel'])
```

#### Version TypeScript (recommandée)
```ts
const emit = defineEmits<{
  (e: 'save', payload: { id: number; label: string }): void
  (e: 'cancel'): void
}>()
```

Bénéfices : autocomplétion + sécurité sur les payloads.

---

### 3.4 Pattern "événement + état" (forme mentale)

- Le **parent** détient l’état : liste, sélection, valeur de formulaire
- L’**enfant** émet l’action : "je veux être supprimé", "ma valeur a changé"

Ce pattern garde un flux clair et évite les dépendances implicites.

---

## 4) Communication bidirectionnelle contrôlée : `v-model` sur composant

`v-model` est une **convention** qui combine :
- une prop `modelValue`
- un événement `update:modelValue`

### 4.1 Implémenter un composant compatible `v-model`

**ChildTextInput.vue**
```vue
<script setup>
const props = defineProps({
  modelValue: { type: String, default: '' }
})

const emit = defineEmits(['update:modelValue'])
</script>

<template>
  <input
    :value="props.modelValue"
    @input="emit('update:modelValue', $event.target.value)"
  />
</template>
```

**Parent.vue**
```vue
<script setup>
import { ref } from 'vue'
import ChildTextInput from './ChildTextInput.vue'

const name = ref('Ada')
</script>

<template>
  <ChildTextInput v-model="name" />
  <p>Bonjour {{ name }}</p>
</template>
```

### 4.2 Multi `v-model` (argument)

**ChildRange.vue** (exemple : min/max)
```vue
<script setup>
const props = defineProps({
  min: { type: Number, required: true },
  max: { type: Number, required: true }
})

const emit = defineEmits(['update:min', 'update:max'])
</script>

<template>
  <input type="number" :value="props.min" @input="emit('update:min', +$event.target.value)" />
  <input type="number" :value="props.max" @input="emit('update:max', +$event.target.value)" />
</template>
```

**Parent.vue**
```vue
<ChildRange v-model:min="min" v-model:max="max" />
```

---

## 5) Ateliers pratiques (pas à pas)

### Atelier 1 — Carte produit (props)
**Objectif** : afficher un produit via props.

1) Créer `ProductCard.vue` recevant :
- `title` (String, required)
- `price` (Number, required)
- `isAvailable` (Boolean, default true)

2) Dans un parent, boucler sur une liste et afficher les cartes.

**Indications**
- Utiliser `v-for`, `:key`
- Formatter le prix dans le composant enfant (computed) sans modifier la prop.

---

### Atelier 2 — Liste avec suppression (events)
**Objectif** : l’enfant demande sa suppression via un event.

- Parent détient `items` (ref d’un tableau)
- Enfant `ListRow.vue` reçoit `item` en prop
- Enfant émet `remove` avec `item.id`
- Parent filtre le tableau

---

### Atelier 3 — Champ contrôlé (v-model)
**Objectif** : créer un champ de saisie réutilisable.

- `BaseInput.vue` accepte `modelValue`, `label`, `error`
- Émet `update:modelValue`
- Parent gère validation et affiche erreurs

---

## 6) Erreurs fréquentes + checklist

### Erreurs fréquentes
- Modifier une prop directement au lieu d’émettre un événement
- Oublier la factory function pour les defaults d’objets/tableaux (`default: () => ({})`)
- Passer des props non typées / non documentées (difficile à maintenir)
- Noms d’événements ambigus (`change` partout) plutôt que `remove`, `select`, `save`

### Checklist review
- [ ] Le parent est-il la source de vérité ?
- [ ] L’enfant n’émet-il que des intentions (événements) ?
- [ ] Les props ont-elles un type et des defaults corrects ?
- [ ] Les événements sont-ils nommés clairement et portent-ils le bon payload ?
- [ ] `v-model` est-il utilisé quand c’est pertinent (input-like) ?

---

## 7) Résumé + mini quiz

### Résumé
- **Props** : données du **parent → enfant** (configuration/affichage)
- **Events** : intentions du **enfant → parent** (`emit` + `@event`)
- **`v-model` composant** : prop `modelValue` + event `update:modelValue`

### Mini quiz
1) Pourquoi une prop ne doit pas être modifiée dans l’enfant ?
2) Comment passer une valeur par défaut pour une prop de type objet ?
3) Quelle paire prop/event est utilisée par `v-model` ?
4) Donnez un exemple de nom d’événement orienté action.

---

## Annexes — Snippets utiles

### A) `toRefs` pour extraire des props réactives
```js
import { toRefs } from 'vue'
const props = defineProps({ label: String })
const { label } = toRefs(props)
```

### B) `defineEmits` typé (rappel)
```ts
const emit = defineEmits<{
  (e: 'remove', id: number): void
}>()
```
