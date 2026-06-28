# Formation Vue.js — Tests (Unitaires avec Vue Test Utils & Jest)

> Objectif : apprendre à écrire, exécuter et maintenir des **tests unitaires** pour des composants Vue afin de **vérifier leur comportement** et **garantir la stabilité** du code.

---

## Plan de la formation

1. **Introduction aux tests dans un projet Vue**
   - Pourquoi tester ?
   - Types de tests (unitaire, intégration, E2E)
   - Ce que couvre cette formation : **tests unitaires**

2. **Outillage : Vue Test Utils & Jest**
   - Rôles respectifs
   - Installation et configuration
   - Structure des fichiers de test

3. **Premiers tests de composant**
   - `mount` vs `shallowMount`
   - Sélecteurs, assertions et snapshots (optionnel)
   - Bonnes pratiques de lisibilité

4. **Tester le rendu (template) et les props**
   - Vérifier le DOM
   - Tester les props et valeurs par défaut
   - Slots (base)

5. **Tester les événements et interactions utilisateur**
   - Trigger d’événements (click, input)
   - `v-model` et champs de formulaire
   - Émission d’événements (`$emit`)

6. **Tester la logique interne : data, computed, methods, watchers**
   - Manipuler l’état
   - Tester une computed
   - Tester un watcher (cas courants)

7. **Mocking & stubbing : isoler le composant**
   - Stub de composants enfants
   - Mock de modules / utilitaires
   - Mock d’appels API

8. **Asynchrone : promesses, timers et nextTick**
   - `await flushPromises()`
   - `nextTick`
   - Fake timers (si besoin)

9. **Tests et architecture : composants, composables, services**
   - Où placer la logique testable
   - Tester un composable
   - Tester une fonction utilitaire

10. **Qualité et maintenance**
   - Couverture de code
   - Stratégie de test
   - Anti-patterns

11. **Atelier guidé (TP) : créer une suite de tests**
   - Cas d’usage complet (composant + formulaire + émission)

---

## 1) Introduction aux tests dans un projet Vue

### Pourquoi tester ?
Les tests unitaires servent à :
- **Vérifier le comportement** d’un composant : rendu, interactions, émissions d’événements.
- **Prévenir les régressions** : si une fonctionnalité casse, les tests le signalent.
- **Documenter** le comportement attendu.
- Faciliter les **refactorings** : en conservant une « ceinture de sécurité ».

### Les types de tests (rappel)
- **Tests unitaires** : vérifier un composant/fonction isolé(e).
- **Tests d’intégration** : vérifier plusieurs unités ensemble.
- **Tests E2E** : vérifier un parcours utilisateur complet.

> Dans cette formation, on se concentre sur les **tests unitaires** avec **Vue Test Utils** et **Jest**.

---

## 2) Outillage : Vue Test Utils & Jest

### Qui fait quoi ?
- **Vue Test Utils (VTU)** : fournit des utilitaires pour monter un composant Vue et interagir avec lui.
- **Jest** : framework de test JavaScript : runner, assertions, mocks, couverture.

### Installation (exemple)
> Les commandes exactes dépendent de votre stack (Vue 2/3, Vite/Vue CLI). L’idée est : installer Jest + VTU et configurer l’environnement de test.

Exemple (Vue 3, Jest) :
```bash
npm i -D jest @vue/test-utils
```

> Selon le projet, vous aurez souvent besoin de :
> - `vue-jest` (Vue 2) ou `@vue/vue3-jest` / transform adapté
> - `babel-jest` si vous transpilez
> - `jest-environment-jsdom` (DOM)

### Convention de structure
- Tests à côté du composant :
  - `src/components/MyButton.vue`
  - `src/components/__tests__/MyButton.spec.js`

Ou dossier dédié :
- `tests/unit/MyButton.spec.js`

### Premier fichier de test
```js
// MyButton.spec.js

describe('MyButton', () => {
  test('exemple', () => {
    expect(true).toBe(true)
  })
})
```

---

## 3) Premiers tests de composant

### `mount` vs `shallowMount`
- `mount` : monte le composant **avec** ses enfants (plus proche du réel).
- `shallowMount` : remplace les enfants par des stubs (tests plus isolés).

Exemple :
```js
import { mount, shallowMount } from '@vue/test-utils'
import MyCard from '../MyCard.vue'

test('mount rend la structure complète', () => {
  const wrapper = mount(MyCard)
  expect(wrapper.exists()).toBe(true)
})

test('shallowMount isole les enfants', () => {
  const wrapper = shallowMount(MyCard)
  expect(wrapper.exists()).toBe(true)
})
```

### Bonnes pratiques rapides
- Nommer les tests comme une **spécification** : « should … » / « affiche … ».
- Tester un comportement par test.
- Préférer les sélecteurs stables (`data-test`) plutôt que des classes CSS.

Exemple dans le template :
```html
<button data-test="submit">Envoyer</button>
```

---

## 4) Tester le rendu (template) et les props

### Composant d’exemple : `UserBadge.vue`
```vue
<template>
  <div class="badge" :class="{ online }">
    <span data-test="name">{{ name }}</span>
    <span v-if="online" data-test="status">En ligne</span>
    <span v-else data-test="status">Hors ligne</span>
  </div>
</template>

<script>
export default {
  name: 'UserBadge',
  props: {
    name: { type: String, required: true },
    online: { type: Boolean, default: false }
  }
}
</script>
```

### Tests : rendu et props
```js
import { mount } from '@vue/test-utils'
import UserBadge from '../UserBadge.vue'

describe('UserBadge', () => {
  test('affiche le nom fourni en prop', () => {
    const wrapper = mount(UserBadge, {
      props: { name: 'Ada Lovelace' }
    })

    expect(wrapper.get('[data-test="name"]').text()).toBe('Ada Lovelace')
  })

  test('affiche "Hors ligne" par défaut', () => {
    const wrapper = mount(UserBadge, { props: { name: 'Ada' } })
    expect(wrapper.get('[data-test="status"]').text()).toBe('Hors ligne')
  })

  test('affiche "En ligne" lorsque online=true', () => {
    const wrapper = mount(UserBadge, {
      props: { name: 'Ada', online: true }
    })

    expect(wrapper.get('[data-test="status"]').text()).toBe('En ligne')
    expect(wrapper.classes()).toContain('online')
  })
})
```

---

## 5) Tester les événements et interactions utilisateur

### Composant : `Counter.vue`
```vue
<template>
  <div>
    <p data-test="count">{{ count }}</p>
    <button data-test="inc" @click="increment">+</button>
  </div>
</template>

<script>
export default {
  name: 'Counter',
  data() {
    return { count: 0 }
  },
  methods: {
    increment() {
      this.count += 1
    }
  }
}
</script>
```

### Test : clic et mise à jour DOM
```js
import { mount } from '@vue/test-utils'
import Counter from '../Counter.vue'

test('incrémente le compteur au clic', async () => {
  const wrapper = mount(Counter)

  expect(wrapper.get('[data-test="count"]').text()).toBe('0')

  await wrapper.get('[data-test="inc"]').trigger('click')

  expect(wrapper.get('[data-test="count"]').text()).toBe('1')
})
```

### Tester `v-model`
Composant : `EmailInput.vue`
```vue
<template>
  <label>
    Email
    <input data-test="email" v-model="email" />
  </label>
</template>

<script>
export default {
  name: 'EmailInput',
  data() {
    return { email: '' }
  }
}
</script>
```

Test :
```js
import { mount } from '@vue/test-utils'
import EmailInput from '../EmailInput.vue'

test('met à jour la donnée email via v-model', async () => {
  const wrapper = mount(EmailInput)
  const input = wrapper.get('[data-test="email"]')

  await input.setValue('test@example.com')

  expect(wrapper.vm.email).toBe('test@example.com')
})
```

### Tester l’émission d’événements
Composant : `ConfirmButton.vue`
```vue
<template>
  <button data-test="confirm" @click="$emit('confirm')">Confirmer</button>
</template>

<script>
export default { name: 'ConfirmButton' }
</script>
```

Test :
```js
import { mount } from '@vue/test-utils'
import ConfirmButton from '../ConfirmButton.vue'

test('émet un événement confirm au clic', async () => {
  const wrapper = mount(ConfirmButton)

  await wrapper.get('[data-test="confirm"]').trigger('click')

  expect(wrapper.emitted('confirm')).toBeTruthy()
  expect(wrapper.emitted('confirm').length).toBe(1)
})
```

---

## 6) Tester la logique interne : data, computed, methods, watchers

### Exemple : computed
Composant : `FullName.vue`
```vue
<template>
  <p data-test="full">{{ fullName }}</p>
</template>

<script>
export default {
  name: 'FullName',
  props: {
    first: { type: String, required: true },
    last: { type: String, required: true }
  },
  computed: {
    fullName() {
      return `${this.first} ${this.last}`
    }
  }
}
</script>
```

Test :
```js
import { mount } from '@vue/test-utils'
import FullName from '../FullName.vue'

test('calcule le nom complet', () => {
  const wrapper = mount(FullName, { props: { first: 'Ada', last: 'Lovelace' } })
  expect(wrapper.get('[data-test="full"]').text()).toBe('Ada Lovelace')
})
```

### Exemple : watcher (cas fréquent)
Composant : `SearchBox.vue`
```vue
<template>
  <input data-test="q" v-model="q" placeholder="Rechercher..." />
</template>

<script>
export default {
  name: 'SearchBox',
  emits: ['search'],
  data() {
    return { q: '' }
  },
  watch: {
    q(newValue) {
      this.$emit('search', newValue)
    }
  }
}
</script>
```

Test :
```js
import { mount } from '@vue/test-utils'
import SearchBox from '../SearchBox.vue'

test('émet search à chaque modification de q', async () => {
  const wrapper = mount(SearchBox)

  await wrapper.get('[data-test="q"]').setValue('vue')

  expect(wrapper.emitted('search')).toBeTruthy()
  expect(wrapper.emitted('search')[0]).toEqual(['vue'])
})
```

---

## 7) Mocking & stubbing : isoler le composant

### Stub d’un composant enfant
Composant : `UserCard.vue`
```vue
<template>
  <div>
    <h3>{{ user.name }}</h3>
    <Avatar :url="user.avatarUrl" />
  </div>
</template>

<script>
import Avatar from './Avatar.vue'

export default {
  name: 'UserCard',
  components: { Avatar },
  props: {
    user: { type: Object, required: true }
  }
}
</script>
```

Test (isoler `UserCard` sans tester `Avatar`) :
```js
import { shallowMount } from '@vue/test-utils'
import UserCard from '../UserCard.vue'

test('affiche le nom sans dépendre du rendu du composant Avatar', () => {
  const wrapper = shallowMount(UserCard, {
    props: { user: { name: 'Ada', avatarUrl: '/ada.png' } },
    global: {
      stubs: { Avatar: true }
    }
  })

  expect(wrapper.text()).toContain('Ada')
})
```

### Mock d’un module utilitaire
Supposons une fonction `formatDate` utilisée dans un composant.

`utils/formatDate.js` :
```js
export function formatDate(date) {
  return new Intl.DateTimeFormat('fr-FR').format(date)
}
```

Dans un test Jest :
```js
jest.mock('../../utils/formatDate', () => ({
  formatDate: () => '01/01/2000'
}))
```

> Objectif : rendre le test déterministe.

---

## 8) Asynchrone : promesses, timers et `nextTick`

### Cas : une action async met à jour le DOM
Composant : `AsyncMessage.vue`
```vue
<template>
  <div>
    <button data-test="load" @click="load">Charger</button>
    <p v-if="message" data-test="msg">{{ message }}</p>
  </div>
</template>

<script>
export default {
  name: 'AsyncMessage',
  data() {
    return { message: '' }
  },
  methods: {
    async load() {
      // Simule un appel async
      const result = await Promise.resolve('Hello')
      this.message = result
    }
  }
}
</script>
```

Test :
```js
import { mount } from '@vue/test-utils'
import AsyncMessage from '../AsyncMessage.vue'

test('affiche le message après chargement', async () => {
  const wrapper = mount(AsyncMessage)

  await wrapper.get('[data-test="load"]').trigger('click')
  await wrapper.vm.$nextTick()

  expect(wrapper.get('[data-test="msg"]').text()).toBe('Hello')
})
```

### `flushPromises` (utile si chaînes de Promises)
Vous pouvez ajouter un helper :
```js
export const flushPromises = () => new Promise(setImmediate)
```
Puis :
```js
await flushPromises()
```

---

## 9) Tests et architecture : composants, composables, services

### Principe
Plus la logique est :
- isolée dans des **fonctions pures** (utils)
- ou des **composables**

…plus elle est simple à tester.

### Exemple : tester un composable
`useCounter.js` :
```js
import { ref } from 'vue'

export function useCounter() {
  const count = ref(0)
  const inc = () => { count.value += 1 }
  return { count, inc }
}
```

Test :
```js
import { useCounter } from '../useCounter'

test('useCounter incrémente', () => {
  const { count, inc } = useCounter()
  expect(count.value).toBe(0)
  inc()
  expect(count.value).toBe(1)
})
```

---

## 10) Qualité et maintenance

### Couverture de code
La couverture aide à repérer les zones non testées :
- statements
- branches
- functions
- lines

Mais :
- **une bonne couverture ne garantit pas de bons tests**.
- vise surtout les **comportements critiques**.

### Stratégie
Tester en priorité :
- Conditions métiers
- Calculs / transformations
- Comportements de formulaire
- Émissions d’événements utilisés par les parents

Éviter de sur-tester :
- La structure exacte du DOM si elle change souvent
- Des détails de styles

### Anti-patterns
- Tests trop couplés à l’implémentation (ex : tester chaque variable interne)
- Snapshots massifs non relus
- Tests fragiles (sélecteurs instables)

---

## 11) TP guidé — Suite de tests pour un composant de formulaire

### Composant : `LoginForm.vue`
```vue
<template>
  <form @submit.prevent="submit">
    <label>
      Email
      <input data-test="email" v-model.trim="email" type="email" />
    </label>

    <label>
      Mot de passe
      <input data-test="password" v-model="password" type="password" />
    </label>

    <p v-if="error" data-test="error">{{ error }}</p>

    <button data-test="submit" type="submit">Connexion</button>
  </form>
</template>

<script>
export default {
  name: 'LoginForm',
  emits: ['login'],
  data() {
    return {
      email: '',
      password: '',
      error: ''
    }
  },
  methods: {
    submit() {
      this.error = ''
      if (!this.email || !this.password) {
        this.error = 'Champs requis'
        return
      }
      this.$emit('login', { email: this.email, password: this.password })
    }
  }
}
</script>
```

### Tests attendus
1. Affiche une erreur si email ou mot de passe manquent.
2. N’émet pas `login` en cas d’erreur.
3. Émet `login` avec un payload correct si le formulaire est valide.
4. Le `.trim` sur l’email supprime les espaces.

### Implémentation des tests
```js
import { mount } from '@vue/test-utils'
import LoginForm from '../LoginForm.vue'

describe('LoginForm', () => {
  test('affiche une erreur si champs manquants', async () => {
    const wrapper = mount(LoginForm)

    await wrapper.get('[data-test="submit"]').trigger('submit')

    expect(wrapper.get('[data-test="error"]').text()).toBe('Champs requis')
  })

  test('n\'émet pas login si invalide', async () => {
    const wrapper = mount(LoginForm)

    await wrapper.get('form').trigger('submit')

    expect(wrapper.emitted('login')).toBeFalsy()
  })

  test('émet login avec email et password', async () => {
    const wrapper = mount(LoginForm)

    await wrapper.get('[data-test="email"]').setValue('ada@example.com')
    await wrapper.get('[data-test="password"]').setValue('secret')

    await wrapper.get('form').trigger('submit')

    expect(wrapper.emitted('login')).toBeTruthy()
    expect(wrapper.emitted('login')[0]).toEqual([
      { email: 'ada@example.com', password: 'secret' }
    ])
  })

  test('trim sur email', async () => {
    const wrapper = mount(LoginForm)

    await wrapper.get('[data-test="email"]').setValue('  ada@example.com  ')
    await wrapper.get('[data-test="password"]').setValue('secret')

    await wrapper.get('form').trigger('submit')

    expect(wrapper.emitted('login')[0][0].email).toBe('ada@example.com')
  })
})
```

---

## Annexes

### Checklist qualité (avant de committer)
- [ ] Les tests décrivent un comportement observable
- [ ] Les sélecteurs utilisent `data-test` quand c’est pertinent
- [ ] Le test échoue si on casse réellement la fonctionnalité
- [ ] Le test n’est pas fragile (peu dépendant de détails visuels)

### Résumé
- **Vue Test Utils** : monter le composant et interagir.
- **Jest** : exécuter, faire des assertions, mocker.
- Les tests unitaires sont un investissement pour la **stabilité** et la **maintenabilité**.
