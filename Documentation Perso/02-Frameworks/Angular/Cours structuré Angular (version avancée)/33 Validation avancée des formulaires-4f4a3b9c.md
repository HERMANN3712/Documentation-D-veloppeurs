# Angular (Reactive Forms) — Validation avancée des formulaires

> **Public** : développeurs Angular (niveau intermédiaire) qui pratiquent déjà les *Reactive Forms* (FormControl/FormGroup/FormArray).  
> **Durée suggérée** : 1 journée (6–7h) ou 2 demi-journées.  
> **Pré-requis** : RxJS de base (Observable, pipe, map, switchMap), TypeScript, notions de forms Angular.

---

## Objectifs pédagogiques

À l’issue de la formation, vous serez capable de :

- Concevoir des **règles de validation inter-champs** robustes (FormGroup/FormArray).
- Implémenter des **validateurs asynchrones** (backend, disponibilité, unicité) avec une stratégie RxJS correcte.
- Afficher des **messages d’erreurs contextualisés** (i18n, priorités, messages par use-case).
- Mettre en place une **normalisation** (trim, case, format) sans casser la validation ni l’UX.
- Définir une **stratégie d’affichage** des erreurs (quand, où, comment) cohérente à l’échelle d’une application.

---

## Plan de la formation

1. **Rappels et socle** (Reactive Forms, statuts, erreurs)
2. **Validation inter-champs (cross-field)**
3. **Validation asynchrone (AsyncValidator)**
4. **Messages d’erreurs contextualisés**
5. **Normalisation des données** (avant/pendant/après saisie)
6. **Stratégies d’affichage des erreurs** (UX, accessibilité, performance)
7. **Atelier de synthèse** : formulaire complet (création compte / checkout)
8. **Checklist & bonnes pratiques**

---

# 1) Rappels — Comment Angular valide un formulaire

## 1.1. Statuts et propriétés clés

- `valid` / `invalid`
- `pending` : souvent lié à une validation asynchrone
- `disabled` : exclu de la validation et de la valeur
- `touched` / `untouched`, `dirty` / `pristine`

**Erreur** = objet `ValidationErrors`:

```ts
export type ValidationErrors = { [key: string]: any };
```

Un `FormControl` a :

- `errors: ValidationErrors | null`
- `hasError('required')`

## 1.2. Chaîne de validation

1. Angular exécute les **validateurs synchrones**.
2. Si tout est ok (ou même si non, selon cas), Angular peut exécuter les **validateurs asynchrones** (statut `PENDING`).
3. `errors` est fusionné à partir de tous les validateurs.

**Point d’attention** : un AsyncValidator doit **compléter** (Observable qui se termine) et **émettre** `null` ou un objet d’erreurs.

---

# 2) Validation inter-champs (Cross-field)

Les règles inter-champs comparent ou combinent plusieurs champs :

- mot de passe + confirmation
- date début < date fin
- email requis si opt-in newsletter
- au moins un téléphone parmi plusieurs
- contraintes sur un `FormArray` (quantités, unicité)

## 2.1. Pattern recommandé : validateur de FormGroup

### Exemple : mot de passe = confirmation

```ts
import { AbstractControl, ValidationErrors, ValidatorFn } from '@angular/forms';

export function matchFieldsValidator(
  field: string,
  confirmField: string,
  errorKey = 'fieldsMismatch'
): ValidatorFn {
  return (group: AbstractControl): ValidationErrors | null => {
    const a = group.get(field);
    const b = group.get(confirmField);

    if (!a || !b) return null; // group mal câblé

    // Ne pas écraser une autre erreur déjà posée sur confirmField
    const bErrors = b.errors ?? {};

    // Si l'un des champs est vide, laissez required gérer (optionnel)
    if (!a.value || !b.value) {
      // Retire l'erreur spécifique si elle existe
      if (bErrors[errorKey]) {
        delete bErrors[errorKey];
        b.setErrors(Object.keys(bErrors).length ? bErrors : null);
      }
      return null;
    }

    const mismatch = a.value !== b.value;

    if (mismatch) {
      b.setErrors({ ...bErrors, [errorKey]: { field, confirmField } });
      return { [errorKey]: true };
    } else {
      // nettoyage
      if (bErrors[errorKey]) {
        delete bErrors[errorKey];
        b.setErrors(Object.keys(bErrors).length ? bErrors : null);
      }
      return null;
    }
  };
}
```

**Pourquoi poser l’erreur sur `confirmField` ?**
- L’utilisateur voit l’erreur sur le champ qu’il corrige.
- Le `FormGroup` est invalide, et le champ est aussi invalide.

### Usage

```ts
this.form = this.fb.group(
  {
    password: ['', [Validators.required, Validators.minLength(12)]],
    passwordConfirm: ['', [Validators.required]],
  },
  { validators: [matchFieldsValidator('password', 'passwordConfirm')] }
);
```

## 2.2. Exemple : date début < date fin

```ts
export function dateRangeValidator(
  startKey: string,
  endKey: string,
  errorKey = 'dateRange'
): ValidatorFn {
  return (group: AbstractControl): ValidationErrors | null => {
    const start = group.get(startKey)?.value;
    const end = group.get(endKey)?.value;
    if (!start || !end) return null;

    const startDate = new Date(start);
    const endDate = new Date(end);

    if (Number.isNaN(startDate.getTime()) || Number.isNaN(endDate.getTime())) {
      return null; // laissez un validator de format gérer
    }

    return startDate <= endDate
      ? null
      : { [errorKey]: { startKey, endKey, start, end } };
  };
}
```

## 2.3. Validation conditionnelle (règles dynamiques)

### Cas : email requis si newsletter = true

Deux approches :

1) **ValidatorFn** sur le group
2) **(re)configuration** des validators du contrôle en réponse à une valeur

Approche 2 (souvent la plus simple à maintenir) :

```ts
this.form = this.fb.group({
  newsletter: [false],
  email: ['', [Validators.email]],
});

this.form.get('newsletter')!.valueChanges.subscribe((optIn: boolean) => {
  const email = this.form.get('email')!;

  if (optIn) {
    email.addValidators([Validators.required]);
  } else {
    email.removeValidators([Validators.required]);
  }

  // Important : recalculer la validation
  email.updateValueAndValidity({ emitEvent: false });
});
```

**Bonnes pratiques** :
- Prévoir un `takeUntilDestroyed()` (Angular 16+) ou équivalent pour éviter les fuites.

## 2.4. Validation d’un FormArray (règles globales)

Exemple : au moins un item et quantités > 0.

```ts
export function minItemsValidator(min: number, errorKey = 'minItems'): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    const value = control.value as unknown[];
    return Array.isArray(value) && value.length >= min ? null : { [errorKey]: { min } };
  };
}
```

Usage :

```ts
this.form = this.fb.group({
  lines: this.fb.array([], [minItemsValidator(1)]),
});
```

---

# 3) Validation asynchrone (AsyncValidator)

Valider via backend :

- disponibilité d’un username
- unicité email
- code promo valide

## 3.1. Rappels : signature

```ts
import { AbstractControl, AsyncValidatorFn, ValidationErrors } from '@angular/forms';
import { Observable, of, timer } from 'rxjs';
import { catchError, map, switchMap } from 'rxjs/operators';

export function myAsyncValidator(): AsyncValidatorFn {
  return (control: AbstractControl): Observable<ValidationErrors | null> => {
    return of(null);
  };
}
```

**Règles** :
- retourner `of(null)` si la validation est non applicable (champ vide, format invalide, etc.)
- gérer `catchError` pour que le formulaire ne reste pas bloqué en `PENDING`
- utiliser un mécanisme de **debounce** si la validation appelle le réseau

## 3.2. Exemple complet : username disponible (avec debounce)

Service :

```ts
@Injectable({ providedIn: 'root' })
export class UsersService {
  constructor(private http: HttpClient) {}

  isUsernameAvailable(username: string) {
    return this.http.get<{ available: boolean }>(`/api/users/username-available`, {
      params: { username },
    });
  }
}
```

AsyncValidator :

```ts
export function usernameAvailableValidator(
  users: UsersService,
  opts?: { debounceMs?: number }
): AsyncValidatorFn {
  const debounceMs = opts?.debounceMs ?? 400;

  return (control: AbstractControl) => {
    const value = String(control.value ?? '').trim();

    // Ne pas faire d'appel si vide ou trop court
    if (!value || value.length < 3) return of(null);

    // Si un validator sync (pattern, etc.) échoue, évitez l'appel
    if (control.errors && !control.errors['usernameTaken']) {
      return of(null);
    }

    return timer(debounceMs).pipe(
      switchMap(() => users.isUsernameAvailable(value)),
      map(res => (res.available ? null : { usernameTaken: { username: value } })),
      catchError(() => of(null))
    );
  };
}
```

Usage :

```ts
this.form = this.fb.group({
  username: [
    '',
    [Validators.required, Validators.minLength(3), Validators.pattern(/^[a-z0-9_]+$/i)],
    [usernameAvailableValidator(this.usersService)],
  ],
});
```

## 3.3. Points d’UX : pending, loaders, et conflits

- En statut `PENDING`, afficher une aide : « Validation en cours… »
- Éviter de déclencher l’async validator trop tôt :
  - `updateOn: 'blur'` pour username/email
  - ou debounce comme ci-dessus

Exemple `updateOn: 'blur'` :

```ts
this.form = this.fb.group({
  email: this.fb.control('', {
    validators: [Validators.required, Validators.email],
    asyncValidators: [emailUniqueValidator(this.usersService)],
    updateOn: 'blur',
  }),
});
```

---

# 4) Messages d’erreurs contextualisés

Objectifs :

- messages compréhensibles, actionnables
- priorités (required avant minlength)
- contextualisation (ex: « mot de passe trop court : 12 min »)
- cohérence UI (mêmes rules → mêmes messages)

## 4.1. Mapper d’erreurs → message

Créer une fonction dédiée :

```ts
import { AbstractControl } from '@angular/forms';

export function getErrorMessage(control: AbstractControl | null): string | null {
  if (!control || !control.errors) return null;

  const e = control.errors;

  // Priorité d'affichage
  if (e['required']) return 'Ce champ est obligatoire.';
  if (e['email']) return 'Adresse email invalide.';
  if (e['minlength']) {
    return `Minimum ${e['minlength'].requiredLength} caractères (actuel : ${e['minlength'].actualLength}).`;
  }
  if (e['maxlength']) {
    return `Maximum ${e['maxlength'].requiredLength} caractères.`;
  }
  if (e['pattern']) return 'Format invalide.';

  // Custom
  if (e['usernameTaken']) return 'Ce nom d’utilisateur est déjà pris.';
  if (e['fieldsMismatch']) return 'Les champs ne correspondent pas.';
  if (e['dateRange']) return 'La date de fin doit être postérieure à la date de début.';

  return 'Valeur invalide.';
}
```

## 4.2. Directives/Composants réutilisables (pattern recommandé)

Créer un composant `FieldErrorComponent` :

```ts
@Component({
  selector: 'app-field-error',
  standalone: true,
  template: `
    <p class="field-error" *ngIf="showError">
      {{ message }}
    </p>
  `,
})
export class FieldErrorComponent {
  @Input() control: AbstractControl | null = null;
  @Input() submitted = false;

  get showError(): boolean {
    const c = this.control;
    return !!c && c.invalid && (c.touched || c.dirty || this.submitted);
  }

  get message(): string | null {
    return getErrorMessage(this.control);
  }
}
```

Usage dans un template :

```html
<input type="text" [formControl]="form.controls.username" />
<app-field-error [control]="form.controls.username" [submitted]="submitted" />
```

## 4.3. Internationalisation (i18n)

Deux options courantes :

- `@angular/localize` + messages statiques
- librairie type Transloco / ngx-translate

Le mapping peut retourner des **clés** plutôt que du texte :

```ts
type ErrorMsg = { key: string; params?: Record<string, any> };
```

---

# 5) Normalisation des données

La normalisation améliore la qualité des données **avant** validation côté serveur :

- trim
- suppression doubles espaces
- passer en minuscules (email)
- homogénéiser les formats (téléphone, codes postaux)

## 5.1. Stratégies possibles

1. **À la saisie** (input event) → UX parfois intrusive
2. **Au blur** (recommandé) → moins perturbant
3. **Avant submit** → simple, mais validation peut se déclencher sur valeur non normalisée

## 5.2. Exemple : normaliser email (trim + lowercase) au blur

```ts
normalizeEmail(control: AbstractControl) {
  const v = String(control.value ?? '');
  const normalized = v.trim().toLowerCase();
  if (normalized !== v) {
    control.setValue(normalized, { emitEvent: false });
  }
}
```

Template :

```html
<input
  type="email"
  [formControl]="form.controls.email"
  (blur)="normalizeEmail(form.controls.email)"
/>
```

## 5.3. Attention : normalisation vs validation

- Si vous normalisez en continu, vous pouvez :
  - provoquer des boucles `valueChanges`
  - casser le curseur dans l’input
- Préférer `emitEvent: false` quand la normalisation ne doit pas redéclencher de logique.
- Pour un téléphone, privilégier une lib dédiée (ex: libphonenumber-js) et valider sur un format E.164 côté backend.

---

# 6) Stratégie d’affichage des erreurs

## 6.1. Quand afficher ?

Règle simple (souvent efficace) :

- Afficher l’erreur quand `invalid && (touched || dirty)`
- Et au submit : forcer l’affichage sur tous les champs

Implémentation côté composant :

```ts
submitted = false;

submit() {
  this.submitted = true;
  this.form.markAllAsTouched();

  if (this.form.invalid) return;

  // ... envoyer
}
```

## 6.2. Où afficher ?

- Sous le champ (pattern standard)
- En résumé en haut de page (utile pour grands formulaires)

Exemple : résumé des erreurs

```ts
function collectErrors(group: AbstractControl, path: string[] = []): Array<{ path: string; errors: any }> {
  const result: Array<{ path: string; errors: any }> = [];

  if (group.errors) {
    result.push({ path: path.join('.'), errors: group.errors });
  }

  const anyGroup = group as any;
  const controls = anyGroup.controls as Record<string, AbstractControl> | undefined;

  if (controls) {
    for (const key of Object.keys(controls)) {
      result.push(...collectErrors(controls[key], [...path, key]));
    }
  }

  return result;
}
```

## 6.3. Accessibilité (a11y)

- Associer `aria-invalid="true"` si invalide
- Lier l’erreur via `aria-describedby`
- Utiliser `role="alert"` pour annoncer les erreurs au lecteur d’écran (à doser)

Exemple simple :

```html
<input
  [formControl]="form.controls.username"
  [attr.aria-invalid]="form.controls.username.invalid"
  aria-describedby="username-error"
/>

<p id="username-error" *ngIf="form.controls.username.invalid && (form.controls.username.touched || submitted)">
  {{ getErrorMessage(form.controls.username) }}
</p>
```

## 6.4. Performance et DX

- Centraliser les validateurs custom dans un dossier `validators/`
- Centraliser les messages dans `errors/`
- Favoriser `ChangeDetectionStrategy.OnPush` et des composants d’erreurs petits
- Éviter les appels async à chaque frappe si inutile (`updateOn: 'blur'`)

---

# 7) Atelier de synthèse — Formulaire complet (exemple)

## 7.1. Spécifications

Créer un formulaire de création de compte avec :

- `username` (required, min 3, pattern, async disponibilité)
- `email` (required, email, normalisation au blur, async unicité au blur)
- `password` (required, min 12, règles de complexité)
- `passwordConfirm` (required, doit matcher)
- `period` : `startDate` + `endDate` (dateRange)
- Affichage d’erreurs : sous champs + au submit

## 7.2. Code de formulaire (extrait)

```ts
this.form = this.fb.group(
  {
    username: this.fb.control('', {
      validators: [
        Validators.required,
        Validators.minLength(3),
        Validators.pattern(/^[a-z0-9_]+$/i),
      ],
      asyncValidators: [usernameAvailableValidator(this.usersService)],
      updateOn: 'change',
    }),
    email: this.fb.control('', {
      validators: [Validators.required, Validators.email],
      asyncValidators: [emailUniqueValidator(this.usersService)],
      updateOn: 'blur',
    }),
    password: ['', [Validators.required, Validators.minLength(12)]],
    passwordConfirm: ['', [Validators.required]],
    period: this.fb.group(
      {
        startDate: ['', [Validators.required]],
        endDate: ['', [Validators.required]],
      },
      { validators: [dateRangeValidator('startDate', 'endDate')] }
    ),
  },
  { validators: [matchFieldsValidator('password', 'passwordConfirm')] }
);
```

> **Exercice** : ajouter un validator de complexité mot de passe (au moins 1 majuscule, 1 chiffre, 1 caractère spécial) et un message contextuel basé sur ce qui manque.

---

# 8) Checklist — Bonnes pratiques

## Validation inter-champs

- [ ] Placer les règles globales au niveau `FormGroup`/`FormArray`.
- [ ] Éviter d’écraser des erreurs existantes sur un champ.
- [ ] Nettoyer vos erreurs custom quand la règle redevient valide.

## Validation asynchrone

- [ ] Debounce ou `updateOn: 'blur'` pour limiter les appels.
- [ ] Toujours `catchError(() => of(null))`.
- [ ] Ne pas appeler le backend si les validators sync échouent.

## Messages d’erreurs

- [ ] Prioriser les messages (required en premier).
- [ ] Centraliser le mapping erreurs → message.
- [ ] Préparer l’i18n (clés + params).

## Normalisation

- [ ] Normaliser au blur ou au submit selon le contexte.
- [ ] Utiliser `emitEvent: false` si nécessaire.
- [ ] Tester l’impact sur les validators async.

## UX & a11y

- [ ] Afficher après interaction (touched/dirty) + au submit.
- [ ] Utiliser `aria-invalid` et `aria-describedby`.
- [ ] Prévoir un résumé d’erreurs sur gros formulaires.

---

## Annexes

### A) Validator de complexité mot de passe (exemple)

```ts
export function passwordStrengthValidator(errorKey = 'passwordStrength'): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    const v = String(control.value ?? '');
    if (!v) return null;

    const missing: string[] = [];
    if (!/[A-Z]/.test(v)) missing.push('uppercase');
    if (!/[0-9]/.test(v)) missing.push('number');
    if (!/[^A-Za-z0-9]/.test(v)) missing.push('special');

    return missing.length ? { [errorKey]: { missing } } : null;
  };
}
```

Mapping message :

```ts
if (e['passwordStrength']) {
  const missing = e['passwordStrength'].missing as string[];
  const map: Record<string, string> = {
    uppercase: 'une majuscule',
    number: 'un chiffre',
    special: 'un caractère spécial',
  };
  return `Mot de passe incomplet : ajoutez ${missing.map(m => map[m] ?? m).join(', ')}.`;
}
```

### B) emailUniqueValidator (exemple)

```ts
export function emailUniqueValidator(users: UsersService): AsyncValidatorFn {
  return (control: AbstractControl) => {
    const email = String(control.value ?? '').trim().toLowerCase();
    if (!email) return of(null);

    // Si format email invalide, ne pas appeler le backend
    if (control.errors && control.errors['email']) return of(null);

    return users.isEmailAvailable(email).pipe(
      map(res => (res.available ? null : { emailTaken: true })),
      catchError(() => of(null))
    );
  };
}
```

---

## Fin de formation — Résultat attendu

Vous disposez désormais d’une boîte à outils complète pour :

- écrire des validateurs custom synchrones et asynchrones,
- gérer les dépendances entre champs,
- construire un système de messages d’erreurs cohérent,
- normaliser les données sans nuire à l’expérience utilisateur,
- standardiser l’affichage et l’accessibilité des erreurs.
