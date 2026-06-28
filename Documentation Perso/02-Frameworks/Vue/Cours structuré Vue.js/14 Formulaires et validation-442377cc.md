# Formation Vue.js (Module 14) — Formulaires et validation

> **Objectif du module** : maîtriser la création de formulaires robustes dans Vue.js, depuis les bases de `v-model` (liaison de données) jusqu’à des validations avancées et maintenables avec **VeeValidate**.

---

## Prérequis

- Connaissances de base en Vue.js (composants, props/emit, directives).
- Notions JavaScript (ES6), événements et promesses.
- (Recommandé) Vue 3 + Composition API.

---

## Public & durée

- **Public** : développeurs Vue.js débutants à intermédiaires, formateurs, fullstack.
- **Durée suggérée** : 3h à 6h selon le niveau et la part de pratique.

---

## Plan (structure pédagogique)

1. **Introduction** : enjeux des formulaires (UX, qualité des données, sécurité)
2. **Rappels HTML & événements** : `input`, `change`, `submit`
3. **`v-model` en profondeur**
   - Text, number, checkbox, radio, select
   - Modificateurs `lazy`, `trim`, `number`
   - `v-model` sur composants (contrats `modelValue` / `update:modelValue`)
4. **Validation “maison”**
   - Validation synchrone
   - Gestion des états (dirty/touched)
   - Affichage des erreurs et accessibilité
5. **Introduction à VeeValidate**
   - Concepts : `Form`, `Field`, `ErrorMessage`, schema
   - Validation par règles et par schéma (Yup/Zod)
6. **Validation avancée**
   - Validation asynchrone (ex: email unique)
   - Validation conditionnelle, champs dynamiques, tableaux
   - Validation cross-field (ex: confirmation mot de passe)
7. **Stratégies UX**
   - Quand valider (à la saisie, au blur, au submit)
   - Messages d’erreur utiles + i18n
   - Focus management, scroll vers erreur
8. **Cas pratique complet** : formulaire d’inscription + mise en forme + API
9. **Check-list & bonnes pratiques**

---

# 1) Introduction : pourquoi les formulaires sont difficiles

Les formulaires sont un point critique d’une application :

- Ils touchent directement à la **qualité des données** (mauvais formats, champs oubliés).
- Ils impactent l’**expérience utilisateur** (friction, messages d’erreur, validation trop agressive).
- Ils posent des enjeux de **sécurité** (données côté client ≠ données fiables côté serveur).

> La validation côté client améliore l’UX mais **ne remplace pas** la validation côté serveur.

---

# 2) Rappels HTML : événements utiles

- `input` : déclenché à chaque frappe (selon type)
- `change` : déclenché quand la valeur est “commise” (ex: select)
- `submit` : déclenché sur l’envoi du formulaire

En Vue, on intercepte `submit` :

```html
<form @submit.prevent="onSubmit">
  <!-- champs -->
  <button>Envoyer</button>
</form>
```

- `@submit.prevent` empêche le rechargement de la page.

---

# 3) `v-model` en profondeur

## 3.1 `v-model` sur un champ texte

```vue
<script setup>
import { ref } from 'vue'

const email = ref('')
</script>

<template>
  <label>
    Email
    <input type="email" v-model="email" placeholder="vous@domaine.com" />
  </label>
  <p>Valeur : {{ email }}</p>
</template>
```

- `v-model` réalise une **liaison bidirectionnelle** entre la valeur du champ et l’état du composant.

## 3.2 Modificateurs : `lazy`, `trim`, `number`

```html
<input v-model.trim="name" />
<input v-model.lazy="comment" />
<input v-model.number="age" type="number" />
```

- `trim` : supprime les espaces en début/fin.
- `lazy` : met à jour au `change` (souvent au blur) plutôt que `input`.
- `number` : convertit en nombre.

## 3.3 Checkbox

### Cas booléen

```vue
<script setup>
import { ref } from 'vue'
const accepted = ref(false)
</script>

<template>
  <label>
    <input type="checkbox" v-model="accepted" />
    J’accepte les CGU
  </label>
  <p>accepted: {{ accepted }}</p>
</template>
```

### Cas liste (multi-checkbox)

```vue
<script setup>
import { ref } from 'vue'
const skills = ref([]) // ex: ['vue', 'node']
</script>

<template>
  <label><input type="checkbox" value="vue" v-model="skills" /> Vue</label>
  <label><input type="checkbox" value="node" v-model="skills" /> Node</label>
  <label><input type="checkbox" value="css" v-model="skills" /> CSS</label>

  <pre>{{ skills }}</pre>
</template>
```

## 3.4 Radio

```vue
<script setup>
import { ref } from 'vue'
const plan = ref('free')
</script>

<template>
  <label><input type="radio" value="free" v-model="plan" /> Gratuit</label>
  <label><input type="radio" value="pro" v-model="plan" /> Pro</label>
  <p>Plan : {{ plan }}</p>
</template>
```

## 3.5 Select

```vue
<script setup>
import { ref } from 'vue'
const country = ref('FR')
</script>

<template>
  <select v-model="country">
    <option value="FR">France</option>
    <option value="BE">Belgique</option>
    <option value="CH">Suisse</option>
  </select>
</template>
```

## 3.6 `v-model` sur un composant (pattern recommandé)

### Objectif
Créer un composant `BaseInput` réutilisable.

### Composant enfant : `BaseInput.vue`

```vue
<script setup>
const props = defineProps({
  modelValue: {
    type: [String, Number],
    default: ''
  },
  label: {
    type: String,
    default: ''
  },
  id: {
    type: String,
    default: undefined
  }
})

const emit = defineEmits(['update:modelValue'])

function onInput(e) {
  emit('update:modelValue', e.target.value)
}
</script>

<template>
  <label :for="id">
    <span v-if="label">{{ label }}</span>
    <input :id="id" :value="modelValue" @input="onInput" />
  </label>
</template>
```

### Utilisation

```vue
<script setup>
import { ref } from 'vue'
import BaseInput from './BaseInput.vue'

const username = ref('')
</script>

<template>
  <BaseInput v-model="username" label="Nom d’utilisateur" id="username" />
  <p>{{ username }}</p>
</template>
```

> Le `v-model` côté parent se traduit en: `:modelValue="username"` + `@update:modelValue="..."`.

---

# 4) Validation “maison” (sans bibliothèque)

Approche utile pour comprendre les concepts : erreurs, états, messages.

## 4.1 Exemple : formulaire simple (email + mot de passe)

```vue
<script setup>
import { reactive, computed } from 'vue'

const form = reactive({
  email: '',
  password: '',
})

const touched = reactive({
  email: false,
  password: false,
})

function validateEmail(value) {
  if (!value) return 'Email requis'
  const re = /^[^\s@]+@[^\s@]+\.[^\s@]+$/
  if (!re.test(value)) return 'Format email invalide'
  return null
}

function validatePassword(value) {
  if (!value) return 'Mot de passe requis'
  if (value.length < 8) return 'Minimum 8 caractères'
  return null
}

const errors = computed(() => ({
  email: validateEmail(form.email),
  password: validatePassword(form.password),
}))

const isValid = computed(() => !errors.value.email && !errors.value.password)

function onSubmit() {
  // Marquer comme touché pour afficher toutes les erreurs
  touched.email = true
  touched.password = true

  if (!isValid.value) return

  // Envoyer (ex: API)
  console.log('Submit', JSON.stringify(form))
}
</script>

<template>
  <form @submit.prevent="onSubmit" novalidate>
    <div>
      <label>Email</label>
      <input
        type="email"
        v-model.trim="form.email"
        @blur="touched.email = true"
        autocomplete="email"
      />
      <p v-if="touched.email && errors.email" class="error">{{ errors.email }}</p>
    </div>

    <div>
      <label>Mot de passe</label>
      <input
        type="password"
        v-model="form.password"
        @blur="touched.password = true"
        autocomplete="current-password"
      />
      <p v-if="touched.password && errors.password" class="error">{{ errors.password }}</p>
    </div>

    <button :disabled="!isValid">Se connecter</button>
  </form>
</template>

<style scoped>
.error { color: #b00020; margin: 6px 0 0; }
</style>
```

### Points clés

- `touched` évite d’afficher les erreurs avant que l’utilisateur n’interagisse.
- `computed errors` centralise les règles.
- `novalidate` désactive la validation native HTML pour éviter des comportements mixtes.

## 4.2 Limites de la validation “maison”

- Complexité qui augmente vite (tableaux, formulaires dynamiques, async, i18n).
- Beaucoup de code répétitif.
- Standardisation difficile (mêmes messages/règles partout).

C’est là que **VeeValidate** devient très pertinent.

---

# 5) VeeValidate : validation avancée et maintenable

## 5.1 Installation

```bash
npm i vee-validate
# Optionnel selon approche (schémas)
npm i yup
# ou
npm i zod
```

## 5.2 Concepts principaux

- `<Form>` : gère le submit, l’état global et l’agrégation des champs.
- `<Field>` : connecte un champ au système de validation.
- `<ErrorMessage>` : affiche les erreurs d’un champ.
- Schéma : centralise les contraintes (recommandé pour gros formulaires).

## 5.3 Exemple minimal avec règles

```vue
<script setup>
import { Form, Field, ErrorMessage } from 'vee-validate'

function required(value) {
  if (!value) return 'Champ requis'
  return true
}

function min8(value) {
  if (!value) return 'Champ requis'
  if (value.length < 8) return 'Minimum 8 caractères'
  return true
}

function onSubmit(values) {
  console.log('Values', values)
}
</script>

<template>
  <Form @submit="onSubmit">
    <div>
      <label>Email</label>
      <Field name="email" type="email" :rules="required" />
      <ErrorMessage name="email" class="error" />
    </div>

    <div>
      <label>Mot de passe</label>
      <Field name="password" type="password" :rules="min8" />
      <ErrorMessage name="password" class="error" />
    </div>

    <button>Envoyer</button>
  </Form>
</template>

<style scoped>
.error { color: #b00020; }
</style>
```

> `rules` peut accepter une fonction, une chaîne, ou une combinaison selon votre style.

---

# 6) Validation par schéma (recommandé) avec Yup

## 6.1 Définir un schéma

```vue
<script setup>
import { Form, Field, ErrorMessage } from 'vee-validate'
import * as yup from 'yup'

const schema = yup.object({
  email: yup.string().required('Email requis').email('Format email invalide'),
  password: yup.string().required('Mot de passe requis').min(8, 'Minimum 8 caractères'),
  confirmPassword: yup
    .string()
    .required('Confirmation requise')
    .oneOf([yup.ref('password')], 'Les mots de passe ne correspondent pas'),
})

function onSubmit(values) {
  console.log(values)
}
</script>

<template>
  <Form :validation-schema="schema" @submit="onSubmit">
    <div>
      <label>Email</label>
      <Field name="email" type="email" />
      <ErrorMessage name="email" class="error" />
    </div>

    <div>
      <label>Mot de passe</label>
      <Field name="password" type="password" />
      <ErrorMessage name="password" class="error" />
    </div>

    <div>
      <label>Confirmer</label>
      <Field name="confirmPassword" type="password" />
      <ErrorMessage name="confirmPassword" class="error" />
    </div>

    <button>S’inscrire</button>
  </Form>
</template>
```

### Avantages

- Règles centralisées et lisibles.
- Facile à partager entre front et (parfois) back.
- Meilleure maintenabilité.

---

# 7) Validation asynchrone (ex: email déjà existant)

## 7.1 Exemple de règle async (approche simple)

```vue
<script setup>
import { Form, Field, ErrorMessage } from 'vee-validate'
import * as yup from 'yup'

async function isEmailAvailable(email) {
  // Simule un appel API
  await new Promise(r => setTimeout(r, 400))
  return email !== 'test@exemple.com'
}

const schema = yup.object({
  email: yup
    .string()
    .required('Email requis')
    .email('Format invalide')
    .test('available', 'Email déjà utilisé', async (value) => {
      if (!value) return false
      return await isEmailAvailable(value)
    }),
})

function onSubmit(values) {
  console.log(values)
}
</script>

<template>
  <Form :validation-schema="schema" @submit="onSubmit">
    <label>Email</label>
    <Field name="email" type="email" />
    <ErrorMessage name="email" class="error" />

    <button>Valider</button>
  </Form>
</template>
```

### Bonnes pratiques async

- Déclencher la validation async au **blur** ou au **submit** (éviter API à chaque frappe).
- Mettre en place du **debounce** côté UI si nécessaire.
- Gérer les erreurs réseau (message générique et retry).

---

# 8) Champs dynamiques (tableaux) : cas “compétences”

Objectif : gérer une liste de champs (ex: expériences, tags, skills…).

> VeeValidate propose des utilitaires pour les tableaux (selon version). À défaut, on peut modéliser un tableau et valider via schéma.

## 8.1 Exemple avec un schéma Yup

```vue
<script setup>
import { Form, Field, ErrorMessage } from 'vee-validate'
import * as yup from 'yup'
import { ref } from 'vue'

const initialValues = { skills: [''] }

const schema = yup.object({
  skills: yup
    .array()
    .of(yup.string().required('Compétence requise').min(2, 'Trop court'))
    .min(1, 'Au moins une compétence'),
})

const skillsCount = ref(1)

function addSkill(values) {
  values.skills.push('')
  skillsCount.value++
}

function removeSkill(values, idx) {
  values.skills.splice(idx, 1)
  skillsCount.value = values.skills.length
}

function onSubmit(values) {
  console.log(values)
}
</script>

<template>
  <Form :initial-values="initialValues" :validation-schema="schema" @submit="onSubmit" v-slot="{ values }">
    <h3>Compétences</h3>

    <div v-for="(_, idx) in values.skills" :key="idx" style="margin-bottom: 12px;">
      <Field :name="`skills[${idx}]`" placeholder="ex: Vue.js" />
      <ErrorMessage :name="`skills[${idx}]`" class="error" />

      <button type="button" @click="removeSkill(values, idx)" v-if="values.skills.length > 1">
        Supprimer
      </button>
    </div>

    <button type="button" @click="addSkill(values)">Ajouter une compétence</button>
    <button type="submit">Envoyer</button>
  </Form>
</template>

<style scoped>
.error { color: #b00020; display: block; }
</style>
```

### Ce que ça enseigne

- Nommer correctement les champs: `skills[0]`, `skills[1]`, …
- Valider des listes avec un schéma.
- Ajouter/supprimer des items en conservant les erreurs cohérentes.

---

# 9) UX de validation : stratégies et accessibilité

## 9.1 Quand afficher les erreurs ?

Stratégies possibles :

1. **Au submit uniquement** : moins intrusif, mais feedback tardif.
2. **Au blur** : compromis courant.
3. **À la saisie** : utile pour formats stricts (ex: masque), mais attention à la frustration.

Recommandation : **blur + submit**.

## 9.2 Messages d’erreur : règles

- Dire **quoi faire** (“Entrez un email valide”), pas seulement “Erreur”.
- Un message par contrainte principale.
- Cohérence (même style dans toute l’app).

## 9.3 Accessibilité (a11y)

- Associer label et input via `for` / `id`.
- Relier l’erreur au champ via `aria-describedby`.
- Indiquer l’état invalide via `aria-invalid="true"`.

Exemple (pattern) :

```html
<input
  :aria-invalid="hasError"
  :aria-describedby="hasError ? 'email-error' : undefined"
/>
<p id="email-error" v-if="hasError">Message</p>
```

## 9.4 Focus management

À l’envoi, si invalid : focus sur le premier champ en erreur.

- Avec VeeValidate, vous pouvez repérer les erreurs et cibler l’élément correspondant.
- Alternative : ajouter des `ref` sur les inputs.

---

# 10) Cas pratique complet : Formulaire d’inscription

## 10.1 Spécification

Créer un formulaire avec :

- Email (requis, format)
- Mot de passe (requis, min 8)
- Confirmation (identique)
- Pays (requis)
- Acceptation CGU (obligatoire)
- Validation async “email déjà pris” (au blur ou submit)
- Affichage d’un résumé des valeurs à l’envoi

## 10.2 Implémentation (Vue + VeeValidate + Yup)

```vue
<script setup>
import { Form, Field, ErrorMessage } from 'vee-validate'
import * as yup from 'yup'

async function isEmailAvailable(email) {
  await new Promise(r => setTimeout(r, 300))
  // Exemple : on refuse un email de test
  return email !== 'test@exemple.com'
}

const schema = yup.object({
  email: yup
    .string()
    .required('Email requis')
    .email('Format email invalide')
    .test('available', 'Cet email est déjà utilisé', async (value) => {
      if (!value) return false
      return await isEmailAvailable(value)
    }),
  password: yup.string().required('Mot de passe requis').min(8, 'Minimum 8 caractères'),
  confirmPassword: yup
    .string()
    .required('Confirmation requise')
    .oneOf([yup.ref('password')], 'Les mots de passe ne correspondent pas'),
  country: yup.string().required('Pays requis'),
  cgu: yup.boolean().oneOf([true], 'Vous devez accepter les CGU'),
})

function onSubmit(values) {
  // Ici : appel API
  alert('Inscription OK:\n' + JSON.stringify(values, null, 2))
}
</script>

<template>
  <Form :validation-schema="schema" @submit="onSubmit" v-slot="{ errors, meta }">
    <h2>Inscription</h2>

    <div class="field">
      <label for="email">Email</label>
      <Field id="email" name="email" type="email" autocomplete="email" />
      <ErrorMessage name="email" class="error" />
    </div>

    <div class="field">
      <label for="password">Mot de passe</label>
      <Field id="password" name="password" type="password" autocomplete="new-password" />
      <ErrorMessage name="password" class="error" />
    </div>

    <div class="field">
      <label for="confirmPassword">Confirmer</label>
      <Field id="confirmPassword" name="confirmPassword" type="password" autocomplete="new-password" />
      <ErrorMessage name="confirmPassword" class="error" />
    </div>

    <div class="field">
      <label for="country">Pays</label>
      <Field id="country" name="country" as="select">
        <option value="" disabled>Choisir</option>
        <option value="FR">France</option>
        <option value="BE">Belgique</option>
        <option value="CH">Suisse</option>
      </Field>
      <ErrorMessage name="country" class="error" />
    </div>

    <div class="field">
      <label class="checkbox">
        <Field name="cgu" type="checkbox" />
        J’accepte les CGU
      </label>
      <ErrorMessage name="cgu" class="error" />
    </div>

    <div class="actions">
      <button type="submit" :disabled="meta.valid === false && Object.keys(errors).length > 0">
        Créer le compte
      </button>
    </div>

    <details style="margin-top: 16px;">
      <summary>Debug</summary>
      <pre>errors: {{ errors }}</pre>
      <pre>meta: {{ meta }}</pre>
    </details>
  </Form>
</template>

<style scoped>
.field { margin-bottom: 14px; display: grid; gap: 6px; max-width: 420px; }
.error { color: #b00020; font-size: 0.95rem; }
.checkbox { display: flex; gap: 8px; align-items: center; }
.actions { margin-top: 10px; }
button[disabled] { opacity: 0.6; cursor: not-allowed; }
</style>
```

### Points de discussion (formateur)

- Pourquoi un schéma plutôt que des règles dispersées.
- Comment éviter les appels async trop fréquents.
- Comment organiser les composants : `BaseInput` + wrapper VeeValidate.

---

# 11) Atelier (exercices guidés)

## Exercice 1 — Formulaire de contact

Champs : nom, email, message.

- Nom requis (min 2)
- Email requis (format)
- Message requis (min 20)

**Bonus** : compteur de caractères sur message.

## Exercice 2 — Profil utilisateur

Champs : âge, téléphone, site.

- Âge nombre entre 13 et 120
- Téléphone (regex simple)
- Site URL valide

## Exercice 3 — Formulaire dynamique “participants”

- Liste de participants (nom + email)
- Minimum 1 participant
- Email unique dans la liste (validation cross-field)

---

# 12) Check-list bonnes pratiques

- **Toujours** valider côté serveur.
- Centraliser les règles (schéma) pour réduire la duplication.
- Ne pas afficher d’erreurs avant interaction (pattern touched/dirty).
- Messages d’erreur actionnables.
- Utiliser `autocomplete` HTML pour améliorer l’UX.
- Gérer l’accessibilité (`label`, `aria-*`, focus).
- Tester :
  - valeurs vides
  - formats invalides
  - limites (min/max)
  - cas asynchrones (timeout, offline)
  - champs conditionnels

---

## Annexes

### A) Table de correspondance : types de champs & `v-model`

| Type | Recommandation | Notes |
|------|----------------|------|
| texte/email | `v-model.trim` | évite les espaces accidentels |
| number | `v-model.number` | attention aux champs vides (`NaN`) |
| checkbox bool | `v-model` | true/false |
| checkbox liste | `v-model` sur array | pousse/retire `value` |
| radio | `v-model` | valeur unique |
| select | `v-model` | option vide conseillée |

### B) Erreurs fréquentes

- Mélanger validation native HTML + custom (comportements confus)
- Valider de façon agressive à chaque frappe avec messages rouges constants
- Oublier les attributs `name` et l’association label/input
- Ne pas gérer les champs conditionnels (ex: si pays=FR alors code postal requis)

---

## Fin du module

**Livrable attendu** : un formulaire d’inscription complet validé (front) avec VeeValidate, schéma Yup, erreurs accessibles, et une stratégie UX claire.
