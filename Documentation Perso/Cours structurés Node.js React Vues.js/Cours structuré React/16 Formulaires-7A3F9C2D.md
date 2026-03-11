# 16 — Formulaires (React)

> **Idée clé** : en React, les formulaires utilisent généralement des **composants contrôlés**, où la valeur des champs est **liée à l’état** du composant. Cela permet une **source de vérité unique**, une validation plus simple, et des comportements UI cohérents.

---

## Objectifs pédagogiques

À la fin de cette formation, vous serez capable de :

- Expliquer la différence entre **composants contrôlés** et **non contrôlés**.
- Construire un formulaire React complet (texte, checkbox, radio, select, textarea).
- Gérer la soumission, la validation, les erreurs, et l’accessibilité.
- Mettre en place des patterns réutilisables (hooks, composant `<Field />`).
- Éviter les pièges courants (performance, reset, valeurs initiales, champs dynamiques).

---

## Prérequis

- JavaScript ES6 (const/let, destructuring, spread)
- Bases de React (composants, props, state, hooks)

---

## Plan de formation

1. **Rappels : comment fonctionne un formulaire HTML**
2. **Composants contrôlés : le standard en React**
3. **Gestion d’état d’un formulaire**
4. **Gestion des évènements : `onChange`, `onBlur`, `onSubmit`**
5. **Champs courants : input, textarea, select, checkbox, radio**
6. **Validation** (synchrone, messages d’erreur, “touched”)
7. **Accessibilité (a11y) et ergonomie**
8. **Reset, valeurs initiales, préremplissage**
9. **Champs dynamiques (listes)**
10. **Patterns avancés** : hook `useForm`, composant `Field`, reducers
11. **Exercice fil rouge** : formulaire complet commenté

---

# 1) Rappels : formulaire HTML

En HTML, un formulaire typique :

- Les champs ont une valeur interne gérée par le navigateur.
- Le submit déclenche l’envoi de données (souvent via rechargement de page) sauf si on empêche le comportement.

En React, on préfère :

- Empêcher le rechargement (`event.preventDefault()`)
- Récupérer/contrôler les valeurs via `state`.

---

# 2) Concept central : composants contrôlés

## 2.1 Définition

Un **composant contrôlé** est un champ dont :

- la valeur (`value` / `checked`) vient d’un **state React**
- toute modification passe par un handler (`onChange`) qui met à jour ce state

> La **source de vérité** n’est plus le DOM : c’est **React**.

## 2.2 Exemple minimal (input texte)

```jsx
import { useState } from "react";

export function ExampleControlled() {
  const [name, setName] = useState("");

  return (
    <div>
      <label>
        Nom
        <input
          value={name}
          onChange={(e) => setName(e.target.value)}
          placeholder="Votre nom"
        />
      </label>

      <p>Valeur en state : {name}</p>
    </div>
  );
}
```

**À retenir** :

- `value={name}` impose la valeur affichée
- `onChange` met à jour `name`

---

# 3) Composants non contrôlés (comparaison)

Un champ **non contrôlé** délègue la valeur au DOM, et vous lisez la valeur via un `ref`.

```jsx
import { useRef } from "react";

export function ExampleUncontrolled() {
  const inputRef = useRef(null);

  function handleSubmit(e) {
    e.preventDefault();
    console.log("Nom:", inputRef.current.value);
  }

  return (
    <form onSubmit={handleSubmit}>
      <input ref={inputRef} defaultValue="" />
      <button type="submit">Envoyer</button>
    </form>
  );
}
```

## Quand utiliser l’un ou l’autre ?

- **Contrôlé** : validation, formats, UI conditionnelle, synchronisation, exigences produit → **souvent recommandé**.
- **Non contrôlé** : formulaires très simples, intégration rapide, ou champs très volumineux/rarement lus.

---

# 4) Structurer l’état d’un formulaire

## 4.1 Un champ = un `useState`

Simple, lisible, mais vite verbeux :

```jsx
const [email, setEmail] = useState("");
const [password, setPassword] = useState("");
```

## 4.2 Un objet `form` (pattern fréquent)

```jsx
const [form, setForm] = useState({
  email: "",
  password: "",
  remember: false,
});

function handleChange(e) {
  const { name, value, type, checked } = e.target;
  setForm((prev) => ({
    ...prev,
    [name]: type === "checkbox" ? checked : value,
  }));
}
```

Usage :

```jsx
<input name="email" value={form.email} onChange={handleChange} />
<input name="password" value={form.password} onChange={handleChange} />
<input name="remember" type="checkbox" checked={form.remember} onChange={handleChange} />
```

**Avantages** :

- handler unique
- facile à passer à une API

**Limite** :

- attention aux transformations/validations champ par champ

---

# 5) Évènements importants

## 5.1 `onChange`

- Déclenché à chaque modification
- Sert à maintenir l’état React à jour

## 5.2 `onBlur`

- Déclenché quand le champ perd le focus
- Très utile pour marquer un champ comme “touché” (afficher l’erreur seulement après interaction)

## 5.3 `onSubmit`

- Déclenché à la soumission du `<form>`
- On empêche le reload : `e.preventDefault()`

---

# 6) Champs courants (avec exemples)

## 6.1 Input texte

```jsx
<input
  name="firstName"
  value={form.firstName}
  onChange={handleChange}
/>
```

## 6.2 Textarea

```jsx
<textarea
  name="bio"
  value={form.bio}
  onChange={handleChange}
/>
```

## 6.3 Select

```jsx
<select name="role" value={form.role} onChange={handleChange}>
  <option value="">-- Choisir --</option>
  <option value="student">Étudiant</option>
  <option value="dev">Développeur</option>
  <option value="trainer">Formateur</option>
</select>
```

## 6.4 Checkbox

Une checkbox est contrôlée avec `checked` (pas `value`).

```jsx
<input
  type="checkbox"
  name="acceptTerms"
  checked={form.acceptTerms}
  onChange={handleChange}
/>
```

## 6.5 Radio

Les radios partagent le même `name`, et `checked` se déduit via comparaison.

```jsx
<input
  type="radio"
  name="level"
  value="beginner"
  checked={form.level === "beginner"}
  onChange={handleChange}
/>
<input
  type="radio"
  name="level"
  value="advanced"
  checked={form.level === "advanced"}
  onChange={handleChange}
/>
```

---

# 7) Validation (pratique et maintenable)

Objectifs :

- empêcher une soumission invalide
- afficher des messages clairs
- ne pas harceler l’utilisateur avant interaction

## 7.1 États recommandés

- `form` : les valeurs
- `errors` : messages par champ
- `touched` : champs déjà visités (blur)
- `isSubmitting` : pour désactiver le bouton

## 7.2 Fonction de validation

```jsx
function validate(form) {
  const errors = {};

  if (!form.email) {
    errors.email = "L'email est requis.";
  } else if (!/^[^@]+@[^@]+\.[^@]+$/.test(form.email)) {
    errors.email = "Format d'email invalide.";
  }

  if (!form.password) {
    errors.password = "Mot de passe requis.";
  } else if (form.password.length < 8) {
    errors.password = "8 caractères minimum.";
  }

  if (!form.acceptTerms) {
    errors.acceptTerms = "Vous devez accepter les conditions.";
  }

  return errors;
}
```

## 7.3 Affichage conditionnel

- Afficher une erreur si `touched[field]` et `errors[field]`.

---

# 8) Accessibilité et UX (indispensable)

## 8.1 Associer label et champ

Utiliser `htmlFor` + `id` :

```jsx
<label htmlFor="email">Email</label>
<input id="email" name="email" value={form.email} onChange={handleChange} />
```

## 8.2 Messages d’erreur accessibles

- `aria-invalid="true"` si erreur
- `aria-describedby` vers l’élément d’erreur

```jsx
<input
  id="email"
  name="email"
  value={form.email}
  onChange={handleChange}
  aria-invalid={Boolean(touched.email && errors.email)}
  aria-describedby={errors.email ? "email-error" : undefined}
/>

{touched.email && errors.email ? (
  <p id="email-error" role="alert">{errors.email}</p>
) : null}
```

## 8.3 Désactiver le submit pendant envoi

```jsx
<button type="submit" disabled={isSubmitting}>
  {isSubmitting ? "Envoi..." : "Envoyer"}
</button>
```

---

# 9) Reset et valeurs initiales

## 9.1 Valeurs initiales

Centraliser les valeurs initiales :

```jsx
const initialForm = {
  email: "",
  password: "",
  acceptTerms: false,
  role: "",
};

const [form, setForm] = useState(initialForm);
```

## 9.2 Reset

```jsx
function handleReset() {
  setForm(initialForm);
  setTouched({});
  setErrors({});
}
```

---

# 10) Champs dynamiques (liste d’items)

Exemple : liste de compétences.

## 10.1 Structure d’état

```jsx
const [form, setForm] = useState({
  skills: [""],
});
```

## 10.2 Ajout / suppression / modification

```jsx
function updateSkill(index, value) {
  setForm((prev) => {
    const skills = [...prev.skills];
    skills[index] = value;
    return { ...prev, skills };
  });
}

function addSkill() {
  setForm((prev) => ({ ...prev, skills: [...prev.skills, ""] }));
}

function removeSkill(index) {
  setForm((prev) => ({
    ...prev,
    skills: prev.skills.filter((_, i) => i !== index),
  }));
}
```

---

# 11) Patterns avancés

## 11.1 `useReducer` pour formulaires complexes

Quand le formulaire devient riche (beaucoup de champs, règles), `useReducer` rend la logique plus prévisible.

Idée : un reducer gère `CHANGE_FIELD`, `BLUR_FIELD`, `SET_ERRORS`, `RESET`.

## 11.2 Hook `useForm` (réutilisable)

Objectif : factoriser `form`, `errors`, `touched` + handlers.

Pseudo-API :

```js
const {
  form,
  errors,
  touched,
  handleChange,
  handleBlur,
  handleSubmit,
  reset,
} = useForm({ initialValues, validate, onSubmit });
```

## 11.3 Composant `<Field />`

Encapsuler : label, input, erreur, a11y.

---

# Exercice fil rouge : Formulaire complet (contrôlé + validation + a11y)

## Spécification

Créer un formulaire d’inscription avec :

- Email (requis, format valide)
- Mot de passe (requis, min 8)
- Rôle (select requis)
- Acceptation des conditions (requis)
- Envoi simulé async (1s)
- Affichage des erreurs uniquement après blur

## Code complet

```jsx
import { useMemo, useState } from "react";

const initialForm = {
  email: "",
  password: "",
  role: "",
  acceptTerms: false,
};

function validate(form) {
  const errors = {};

  if (!form.email) {
    errors.email = "L'email est requis.";
  } else if (!/^[^@]+@[^@]+\.[^@]+$/.test(form.email)) {
    errors.email = "Format d'email invalide.";
  }

  if (!form.password) {
    errors.password = "Le mot de passe est requis.";
  } else if (form.password.length < 8) {
    errors.password = "8 caractères minimum.";
  }

  if (!form.role) {
    errors.role = "Le rôle est requis.";
  }

  if (!form.acceptTerms) {
    errors.acceptTerms = "Vous devez accepter les conditions.";
  }

  return errors;
}

export function SignupForm() {
  const [form, setForm] = useState(initialForm);
  const [touched, setTouched] = useState({});
  const [isSubmitting, setIsSubmitting] = useState(false);

  const errors = useMemo(() => validate(form), [form]);
  const isValid = Object.keys(errors).length === 0;

  function handleChange(e) {
    const { name, value, type, checked } = e.target;
    setForm((prev) => ({
      ...prev,
      [name]: type === "checkbox" ? checked : value,
    }));
  }

  function handleBlur(e) {
    const { name } = e.target;
    setTouched((prev) => ({ ...prev, [name]: true }));
  }

  async function handleSubmit(e) {
    e.preventDefault();

    // Marquer tout comme touché pour afficher les erreurs si l'utilisateur clique direct sur Envoyer
    setTouched({
      email: true,
      password: true,
      role: true,
      acceptTerms: true,
    });

    if (!isValid) return;

    setIsSubmitting(true);
    try {
      await new Promise((r) => setTimeout(r, 1000));
      alert("Inscription réussie !\n" + JSON.stringify(form, null, 2));
      setForm(initialForm);
      setTouched({});
    } finally {
      setIsSubmitting(false);
    }
  }

  function fieldError(name) {
    return touched[name] ? errors[name] : undefined;
  }

  return (
    <form onSubmit={handleSubmit} noValidate>
      <h2>Inscription</h2>

      <div style={{ marginBottom: 12 }}>
        <label htmlFor="email">Email</label>
        <input
          id="email"
          name="email"
          type="email"
          value={form.email}
          onChange={handleChange}
          onBlur={handleBlur}
          aria-invalid={Boolean(fieldError("email"))}
          aria-describedby={fieldError("email") ? "email-error" : undefined}
        />
        {fieldError("email") ? (
          <p id="email-error" role="alert" style={{ color: "crimson" }}>
            {fieldError("email")}
          </p>
        ) : null}
      </div>

      <div style={{ marginBottom: 12 }}>
        <label htmlFor="password">Mot de passe</label>
        <input
          id="password"
          name="password"
          type="password"
          value={form.password}
          onChange={handleChange}
          onBlur={handleBlur}
          aria-invalid={Boolean(fieldError("password"))}
          aria-describedby={fieldError("password") ? "password-error" : undefined}
        />
        {fieldError("password") ? (
          <p id="password-error" role="alert" style={{ color: "crimson" }}>
            {fieldError("password")}
          </p>
        ) : null}
      </div>

      <div style={{ marginBottom: 12 }}>
        <label htmlFor="role">Rôle</label>
        <select
          id="role"
          name="role"
          value={form.role}
          onChange={handleChange}
          onBlur={handleBlur}
          aria-invalid={Boolean(fieldError("role"))}
          aria-describedby={fieldError("role") ? "role-error" : undefined}
        >
          <option value="">-- Choisir --</option>
          <option value="student">Étudiant</option>
          <option value="dev">Développeur</option>
          <option value="trainer">Formateur</option>
        </select>
        {fieldError("role") ? (
          <p id="role-error" role="alert" style={{ color: "crimson" }}>
            {fieldError("role")}
          </p>
        ) : null}
      </div>

      <div style={{ marginBottom: 12 }}>
        <label>
          <input
            name="acceptTerms"
            type="checkbox"
            checked={form.acceptTerms}
            onChange={handleChange}
            onBlur={handleBlur}
            aria-invalid={Boolean(fieldError("acceptTerms"))}
            aria-describedby={fieldError("acceptTerms") ? "terms-error" : undefined}
          />
          J’accepte les conditions
        </label>
        {fieldError("acceptTerms") ? (
          <p id="terms-error" role="alert" style={{ color: "crimson" }}>
            {fieldError("acceptTerms")}
          </p>
        ) : null}
      </div>

      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? "Envoi..." : "Créer mon compte"}
      </button>

      <pre style={{ marginTop: 16, background: "#f6f8fa", padding: 12 }}>
        {JSON.stringify({ form, touched, errors }, null, 2)}
      </pre>
    </form>
  );
}
```

### Points à discuter en formation

- Pourquoi `useMemo` ici ? (éviter de recalculer des validations coûteuses)
- Pourquoi `noValidate` sur le form ? (pour maîtriser la validation côté React)
- Pourquoi `touched` ? (UX : pas d’erreurs dès le premier rendu)
- Différence `value` vs `checked`

---

## Checklist de bonnes pratiques

- ✅ Utiliser des **composants contrôlés** (valeurs dans le state)
- ✅ Centraliser `initialValues`
- ✅ Gérer `touched` + `errors`
- ✅ Fournir des labels accessibles (`htmlFor`, `id`)
- ✅ Utiliser `aria-invalid`, `aria-describedby`, `role="alert"`
- ✅ Désactiver le submit pendant `isSubmitting`
- ✅ Séparer la validation (`validate(form)`) de l’UI

---

## Exercices (à donner aux apprenants)

1. Ajouter un champ `confirmPassword` et une règle : doit correspondre à `password`.
2. Ajouter un champ `skills` (liste dynamique) avec min 1 compétence non vide.
3. Ajouter une validation “debounced” sur l’email (ex: vérifier disponibilité simulée).
4. Refactorer le formulaire avec un hook `useForm` réutilisable.

---

## Résumé

Les formulaires React reposent généralement sur des **composants contrôlés** : la valeur des champs est liée à l’**état**. Cette approche rend les formulaires **prévisibles**, **testables**, adaptés aux **validations**, et plus simples à intégrer dans des interfaces modernes.
