# Formation Angular — Formulaires réactifs avancés

- **Référence** : 31
- **Thème** : Reactive Forms (niveau avancé)
- **Public** : Développeurs Angular ayant déjà pratiqué les formulaires réactifs (FormGroup/FormControl)
- **Pré-requis** : TypeScript, RxJS (bases), Angular CLI, notions de validation de formulaire
- **Durée suggérée** : 1 à 2 jours (adaptable)
- **Version Angular** : 16+ (compatible 14+ avec ajustements)

---

## Objectifs pédagogiques

À l’issue de la formation, les participants seront capables de :

1. Concevoir des formulaires réactifs complexes et maintenables.
2. Écrire et composer des validateurs **synchrones** et **asynchrones**.
3. Construire des formulaires **dynamiques** (FormArray, structures imbriquées, génération par configuration).
4. Créer des **Custom Form Controls** (ControlValueAccessor) et exploiter les validateurs via NG_VALIDATORS / NG_ASYNC_VALIDATORS.
5. Mettre en place des stratégies de validation avancées (règles conditionnelles, cross-field, validation progressive, UX).
6. Intégrer RxJS pour gérer performance, latence réseau, et expérience utilisateur.

---

## Plan de la formation

1. **Rappels et cadrage avancé** (architecture, typage, états)
2. **Validateurs synchrones avancés**
3. **Validateurs asynchrones** (API, RxJS, annulation, erreurs)
4. **Composition avancée des règles** (cross-field, conditionnels, orchestration)
5. **Formulaires dynamiques** (FormArray, factories, configuration driven)
6. **Custom Form Controls** (ControlValueAccessor, validation, accessibilité)
7. **Patterns, performance et testing** (unitaires, intégration, diagnostics)
8. **Atelier fil rouge** : formulaire complet (inscription/commande) avec contraintes réelles

---

## 1) Rappels et cadrage avancé

### 1.1 Reactive Forms : modèle mental (avancé)

Les formulaires réactifs reposent sur un **arbre de contrôles** :

- `FormControl` : valeur et état d’un champ
- `FormGroup` : regroupement clé → contrôle (objet)
- `FormArray` : liste ordonnée de contrôles (tableau)

Chaque nœud expose :

- **value** : valeur actuelle
- **status** : `VALID | INVALID | PENDING | DISABLED`
- **errors** : dictionnaire d’erreurs
- **touched/untouched** : interaction (blur)
- **dirty/pristine** : modification
- **valueChanges/statusChanges** : flux RxJS

> En avancé, on raisonne en **composition** : chaque sous-formulaire est un composant (ou au moins une fonction de factory), et chaque règle de validation est un module réutilisable.

### 1.2 Typage des Reactive Forms (Angular 14+)

Les **Typed Forms** améliorent DX et sécurité.

```ts
import { FormControl, FormGroup, FormArray, Validators } from '@angular/forms';

type AddressForm = {
  street: FormControl<string>;
  zip: FormControl<string>;
  city: FormControl<string>;
};

type ProfileForm = {
  email: FormControl<string>;
  password: FormControl<string>;
  address: FormGroup<AddressForm>;
  tags: FormArray<FormControl<string>>;
};

const form = new FormGroup<ProfileForm>({
  email: new FormControl('', { nonNullable: true, validators: [Validators.required, Validators.email] }),
  password: new FormControl('', { nonNullable: true, validators: [Validators.required] }),
  address: new FormGroup<AddressForm>({
    street: new FormControl('', { nonNullable: true }),
    zip: new FormControl('', { nonNullable: true }),
    city: new FormControl('', { nonNullable: true }),
  }),
  tags: new FormArray<FormControl<string>>([
    new FormControl('angular', { nonNullable: true })
  ])
});
```

#### Points clés

- `nonNullable: true` évite le `string | null`
- `form.get('...')` est moins sûr que la navigation typée (`form.controls.email`)

### 1.3 Stratégies de mise à jour (`updateOn`)

On peut décaler la validation/mise à jour :

```ts
new FormControl('', { updateOn: 'blur' });
// ou pour tout un groupe
new FormGroup({...}, { updateOn: 'submit' });
```

**Cas d’usage** :
- éviter des appels API sur chaque frappe
- validation lourde uniquement au blur/submit

---

## 2) Validateurs synchrones avancés

### 2.1 API d’un validateur synchrone

Un validateur :

- prend un `AbstractControl`
- renvoie `ValidationErrors | null`

```ts
import { AbstractControl, ValidationErrors, ValidatorFn } from '@angular/forms';

export const noWhitespaceValidator: ValidatorFn = (control: AbstractControl): ValidationErrors | null => {
  const value = (control.value ?? '') as string;
  if (typeof value !== 'string') return null;
  return value.trim().length === 0 ? { whitespace: true } : null;
};
```

### 2.2 Validateurs paramétrables (factory)

```ts
export function minTrimmedLength(min: number): ValidatorFn {
  return (control) => {
    const v = (control.value ?? '') as string;
    if (typeof v !== 'string') return null;
    return v.trim().length < min ? { minTrimmedLength: { min, actual: v.trim().length } } : null;
  };
}
```

Usage :

```ts
name: new FormControl('', {
  nonNullable: true,
  validators: [Validators.required, minTrimmedLength(3)]
})
```

### 2.3 Validation cross-field (niveau groupe)

Exemple : correspondance password / confirmation.

```ts
import { FormGroup, ValidationErrors, ValidatorFn } from '@angular/forms';

type PasswordGroup = {
  password: FormControl<string>;
  confirm: FormControl<string>;
};

export const passwordMatchValidator: ValidatorFn = (control): ValidationErrors | null => {
  const group = control as FormGroup<PasswordGroup>;
  const p = group.controls.password.value;
  const c = group.controls.confirm.value;
  return p && c && p !== c ? { passwordMismatch: true } : null;
};
```

⚠️ **Bonnes pratiques** :

- Le validateur vit sur le **groupe** (`FormGroup`) plutôt que sur `confirm`.
- L’affichage de l’erreur peut être fait au niveau du champ `confirm`, mais l’origine est group.

### 2.4 Normalisation et validation : éviter les boucles

Si vous corrigez automatiquement la valeur (ex. trimming), faites-le en **écoutant** et en utilisant `emitEvent: false` pour éviter les cycles.

```ts
const c = form.controls.email;

c.valueChanges.subscribe(v => {
  const trimmed = v.trim();
  if (trimmed !== v) {
    c.setValue(trimmed, { emitEvent: false });
  }
});
```

---

## 3) Validateurs asynchrones (Async Validators)

### 3.1 Quand utiliser l’asynchrone ?

- Vérification côté serveur (unicité email/username)
- Règles dépendant d’un référentiel distant
- Checks coûteux (mais souvent mieux en back)

### 3.2 API d’un validateur async

Un validateur async renvoie :

- `Observable<ValidationErrors | null>` ou `Promise<ValidationErrors | null>`

```ts
import { AbstractControl, AsyncValidatorFn, ValidationErrors } from '@angular/forms';
import { Observable, of } from 'rxjs';
import { catchError, map } from 'rxjs/operators';

export function emailTakenValidator(check$: (email: string) => Observable<boolean>): AsyncValidatorFn {
  return (control: AbstractControl): Observable<ValidationErrors | null> => {
    const email = (control.value ?? '') as string;
    if (!email) return of(null);

    return check$(email).pipe(
      map(isTaken => (isTaken ? { emailTaken: true } : null)),
      catchError(() => of(null)) // en cas d’erreur réseau, on n’empêche pas forcément la saisie
    );
  };
}
```

### 3.3 Gestion de la fréquence (debounce) et annulation

Important : un async validator est relancé à chaque changement. On veut souvent :

- `debounceTime` pour réduire les requêtes
- `switchMap` pour annuler l’appel précédent

Pattern : injecter un service qui gère le debouncing.

```ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable, of } from 'rxjs';
import { debounceTime, distinctUntilChanged, switchMap, map, catchError } from 'rxjs/operators';

@Injectable({ providedIn: 'root' })
export class UserApi {
  constructor(private http: HttpClient) {}

  isEmailTaken(email: string): Observable<boolean> {
    return this.http.get<{ taken: boolean }>(`/api/users/email-taken`, { params: { email } })
      .pipe(map(r => r.taken));
  }
}
```

Dans le validateur, on utilise une source basée sur `control.valueChanges` ?

⚠️ En pratique, un `AsyncValidatorFn` ne donne pas directement accès au flux. On applique donc un **debounce local** à partir de la valeur courante en combinant un `timer`.

```ts
import { timer, Observable, of } from 'rxjs';
import { switchMap, map, catchError } from 'rxjs/operators';

export function emailTakenValidatorDebounced(api: UserApi, debounceMs = 400): AsyncValidatorFn {
  return (control): Observable<ValidationErrors | null> => {
    const email = (control.value ?? '') as string;
    if (!email) return of(null);

    return timer(debounceMs).pipe(
      switchMap(() => api.isEmailTaken(email)),
      map(taken => (taken ? { emailTaken: true } : null)),
      catchError(() => of(null))
    );
  };
}
```

### 3.4 Priorité et UX : états `PENDING`

Quand un validateur async tourne :

- `status === 'PENDING'`
- utile pour afficher un spinner ou désactiver le submit

```html
<button type="submit" [disabled]="form.pending || form.invalid">Créer</button>
```

---

## 4) Composition avancée des règles de validation

### 4.1 Combiner des validateurs : `Validators.compose`

```ts
import { Validators } from '@angular/forms';

const v = Validators.compose([
  Validators.required,
  Validators.minLength(8),
  noWhitespaceValidator
]);
```

### 4.2 Validation conditionnelle (règles dépendantes)

Exemple : le champ `companyName` est requis uniquement si `accountType === 'company'`.

```ts
type AccountForm = {
  accountType: FormControl<'person' | 'company'>;
  companyName: FormControl<string>;
};

const f = new FormGroup<AccountForm>({
  accountType: new FormControl('person', { nonNullable: true }),
  companyName: new FormControl('', { nonNullable: true })
});

f.controls.accountType.valueChanges.subscribe(type => {
  const c = f.controls.companyName;
  if (type === 'company') {
    c.addValidators([Validators.required, minTrimmedLength(2)]);
  } else {
    c.clearValidators();
  }
  c.updateValueAndValidity({ emitEvent: false });
});
```

Bonnes pratiques :

- centraliser cette orchestration dans une fonction (`setupConditionalValidation(form)`)
- appeler `updateValueAndValidity` après modification des validateurs

### 4.3 Erreurs au bon endroit (propagation)

Pour une règle cross-field, vous pouvez :

- mettre l’erreur sur le groupe (simple)
- ou **reporter** l’erreur sur un contrôle enfant (pour affichage local)

Pattern : dans le validateur de groupe, utiliser `setErrors` sur l’enfant **avec prudence**.

📌 Recommandation : gardez l’erreur sur le groupe et adaptez l’UI pour lire `formGroup.errors`.

### 4.4 Validation progressive (quand afficher ?)

- Afficher erreurs si `control.touched` ou `form.submitted`
- Ou sur `dirty` selon UX

Exemple d’helper :

```ts
export function shouldShowError(control: AbstractControl, submitted: boolean): boolean {
  return control.invalid && (control.touched || submitted);
}
```

---

## 5) Formulaires dynamiques

### 5.1 FormArray : liste de contrôles

Cas : liste d’emails secondaires.

```ts
type EmailsForm = {
  emails: FormArray<FormControl<string>>;
};

const emailsForm = new FormGroup<EmailsForm>({
  emails: new FormArray<FormControl<string>>([
    new FormControl('', { nonNullable: true, validators: [Validators.required, Validators.email] })
  ])
});

function addEmail() {
  emailsForm.controls.emails.push(
    new FormControl('', { nonNullable: true, validators: [Validators.required, Validators.email] })
  );
}

function removeEmail(i: number) {
  emailsForm.controls.emails.removeAt(i);
}
```

Template :

```html
<div formArrayName="emails">
  <div *ngFor="let c of emailsForm.controls.emails.controls; let i = index">
    <input [formControlName]="i" placeholder="Email" />
    <button type="button" (click)="removeEmail(i)">Supprimer</button>
    <div class="error" *ngIf="c.touched && c.errors?.['email']">Email invalide</div>
  </div>
</div>
<button type="button" (click)="addEmail()">Ajouter</button>
```

### 5.2 FormArray de FormGroup : objets répétables

Cas : lignes de produits (commande) — chaque ligne a `productId`, `qty`, `price`.

```ts
type LineGroup = {
  productId: FormControl<string>;
  qty: FormControl<number>;
  price: FormControl<number>;
};

type OrderForm = {
  customerEmail: FormControl<string>;
  lines: FormArray<FormGroup<LineGroup>>;
};

function createLine(): FormGroup<LineGroup> {
  return new FormGroup<LineGroup>({
    productId: new FormControl('', { nonNullable: true, validators: [Validators.required] }),
    qty: new FormControl(1, { nonNullable: true, validators: [Validators.required, Validators.min(1)] }),
    price: new FormControl(0, { nonNullable: true, validators: [Validators.required, Validators.min(0)] })
  });
}

const orderForm = new FormGroup<OrderForm>({
  customerEmail: new FormControl('', { nonNullable: true, validators: [Validators.required, Validators.email] }),
  lines: new FormArray<FormGroup<LineGroup>>([createLine()])
});
```

### 5.3 Génération de formulaire par configuration

Approche : décrire un formulaire via un schéma de champs.

```ts
type FieldType = 'text' | 'email' | 'number';

interface FieldConfig {
  key: string;
  label: string;
  type: FieldType;
  required?: boolean;
  min?: number;
  validators?: ValidatorFn[];
}

function buildForm(config: FieldConfig[]): FormGroup {
  const group: Record<string, FormControl> = {};

  for (const f of config) {
    const validators: ValidatorFn[] = [];
    if (f.required) validators.push(Validators.required);
    if (f.type === 'email') validators.push(Validators.email);
    if (f.type === 'number' && typeof f.min === 'number') validators.push(Validators.min(f.min));
    if (f.validators) validators.push(...f.validators);

    group[f.key] = new FormControl('', validators);
  }

  return new FormGroup(group);
}
```

> Ce pattern est utile pour des back-offices, formulaires administrables, ou des moteurs de formulaires.

### 5.4 Mise à jour dynamique sans perdre l’état

Quand on reconstruit un formulaire, on peut perdre :

- `dirty/touched`
- `disabled`
- `async validators running`

Recommandation :

- préférer `addControl/removeControl` sur un `FormGroup` existant
- ou reconstruire en recyclant les contrôles si possible

---

## 6) Custom Form Controls (ControlValueAccessor)

### 6.1 Pourquoi ?

Créer un composant réutilisable qui se comporte comme un `<input>` Angular Forms :

- `formControlName` fonctionne
- l’état disabled et touched remonte
- la validation s’intègre

Exemples :

- `date-picker`
- `phone-input`
- `chips-input` (liste de tags)

### 6.2 Implémenter `ControlValueAccessor`

Exemple : `app-rating` (1..5).

```ts
import { Component, forwardRef, Input } from '@angular/core';
import { ControlValueAccessor, NG_VALUE_ACCESSOR } from '@angular/forms';

@Component({
  selector: 'app-rating',
  template: `
    <div class="rating" (blur)="onTouched()" tabindex="0">
      <button type="button"
              *ngFor="let s of stars"
              [class.active]="s <= value"
              (click)="set(s)"
              [disabled]="disabled">
        {{ s }}
      </button>
    </div>
  `,
  providers: [{
    provide: NG_VALUE_ACCESSOR,
    useExisting: forwardRef(() => RatingComponent),
    multi: true
  }]
})
export class RatingComponent implements ControlValueAccessor {
  @Input() max = 5;
  value = 0;
  disabled = false;

  get stars(): number[] {
    return Array.from({ length: this.max }, (_, i) => i + 1);
  }

  private onChange: (v: number) => void = () => {};
  private onTouched: () => void = () => {};

  writeValue(v: number): void {
    this.value = v ?? 0;
  }

  registerOnChange(fn: (v: number) => void): void {
    this.onChange = fn;
  }

  registerOnTouched(fn: () => void): void {
    this.onTouched = fn;
  }

  setDisabledState(isDisabled: boolean): void {
    this.disabled = isDisabled;
  }

  set(v: number) {
    if (this.disabled) return;
    this.value = v;
    this.onChange(v);
    this.onTouched();
  }
}
```

Usage :

```html
<form [formGroup]="form">
  <app-rating formControlName="score"></app-rating>
</form>
```

```ts
score: new FormControl(0, { nonNullable: true })
```

### 6.3 Ajouter une validation custom au composant

Deux options :

1. Validation portée par le `FormControl` parent (simple)
2. Le composant **fournit** un validateur via `NG_VALIDATORS`

Exemple : `app-rating` impose un minimum.

```ts
import { NG_VALIDATORS, Validator, AbstractControl, ValidationErrors } from '@angular/forms';

@Component({
  // ...
  providers: [
    // NG_VALUE_ACCESSOR ...
    {
      provide: NG_VALIDATORS,
      useExisting: forwardRef(() => RatingComponent),
      multi: true
    }
  ]
})
export class RatingComponent implements ControlValueAccessor, Validator {
  @Input() min = 1;

  validate(control: AbstractControl): ValidationErrors | null {
    const v = control.value as number;
    return v < this.min ? { ratingMin: { min: this.min, actual: v } } : null;
  }
}
```

### 6.4 Cas avancé : composant qui encapsule un sous-formulaire

Pattern : le composant interne gère un `FormGroup` et expose une valeur agrégée.

Ex : `app-date-range` qui retourne `{ start: Date; end: Date }`.

Bonnes pratiques :

- synchroniser `writeValue` → patch du sous-form
- écouter `valueChanges` avec `distinctUntilChanged` et `emitEvent: false` selon besoin
- gérer la désactivation : `group.disable({ emitEvent: false })`

---

## 7) Patterns, performance et testing

### 7.1 Désactiver/activer proprement

```ts
if (readonly) {
  form.disable({ emitEvent: false });
} else {
  form.enable({ emitEvent: false });
}
```

### 7.2 Accès aux erreurs : helper d’affichage

```ts
export function getError(control: AbstractControl, code: string): any {
  return control.errors?.[code];
}
```

Template :

```html
<small class="error" *ngIf="email.touched && getError(email, 'required')">Email requis</small>
<small class="error" *ngIf="email.touched && getError(email, 'email')">Email invalide</small>
<small class="error" *ngIf="email.touched && getError(email, 'emailTaken')">Email déjà utilisé</small>
```

### 7.3 Performance : réduire les recalculs

- utiliser `updateOn: 'blur'` si pertinent
- éviter de reconstruire le formulaire inutilement
- privilégier `patchValue` avec `emitEvent: false` si vous hydratez des données
- sur gros formulaires, découper en **sous-composants** (chaque sous-arbre gère son affichage)

### 7.4 Tests unitaires

Tester un validateur synchrone :

```ts
import { FormControl } from '@angular/forms';

describe('noWhitespaceValidator', () => {
  it('should fail on spaces only', () => {
    const c = new FormControl('   ');
    expect(noWhitespaceValidator(c)).toEqual({ whitespace: true });
  });
});
```

Tester un async validator :

```ts
import { of } from 'rxjs';

it('should mark emailTaken if api returns true', (done) => {
  const api = { isEmailTaken: () => of(true) } as any;
  const v = emailTakenValidatorDebounced(api, 0);

  v(new FormControl('a@b.com')).subscribe(res => {
    expect(res).toEqual({ emailTaken: true });
    done();
  });
});
```

---

## 8) Atelier fil rouge (projet guidé)

### Sujet
Construire un formulaire d’inscription avancé :

- Informations : email, mot de passe + confirmation
- Profil : prénom/nom (trim, règles)
- Adresse (sous-groupe)
- Liste dynamique de moyens de contact (FormArray)
- Validation asynchrone : email unique côté serveur
- Composant custom : `app-password-strength` ou `app-rating`

### Étapes proposées

1. **Structure** : créer les `FormGroup` imbriqués et typer le formulaire.
2. **Validation synchrone** : règles de format, required, cross-field password.
3. **Validation asynchrone** : vérification email unique avec debounce.
4. **Dynamique** : ajouter/supprimer des contacts (FormArray de FormGroup).
5. **Custom Control** : intégrer un composant qui expose un `FormControl`.
6. **UX** : messages d’erreur, désactivation submit, affichage pending.
7. **Tests** : validateurs unitaires + test simple de soumission.

### Exemple de squelette (formulaire)

```ts
type ContactGroup = {
  type: FormControl<'phone' | 'slack' | 'email'>;
  value: FormControl<string>;
};

type SignupForm = {
  email: FormControl<string>;
  passwordGroup: FormGroup<{
    password: FormControl<string>;
    confirm: FormControl<string>;
  }>;
  firstName: FormControl<string>;
  lastName: FormControl<string>;
  address: FormGroup<{
    street: FormControl<string>;
    zip: FormControl<string>;
    city: FormControl<string>;
  }>;
  contacts: FormArray<FormGroup<ContactGroup>>;
  score: FormControl<number>;
};

function createContact(): FormGroup<ContactGroup> {
  return new FormGroup<ContactGroup>({
    type: new FormControl('phone', { nonNullable: true }),
    value: new FormControl('', { nonNullable: true, validators: [Validators.required] })
  });
}

// Construction
const form = new FormGroup<SignupForm>({
  email: new FormControl('', {
    nonNullable: true,
    validators: [Validators.required, Validators.email],
    asyncValidators: [emailTakenValidatorDebounced(userApi, 400)],
    updateOn: 'blur'
  }),
  passwordGroup: new FormGroup({
    password: new FormControl('', { nonNullable: true, validators: [Validators.required, Validators.minLength(8)] }),
    confirm: new FormControl('', { nonNullable: true, validators: [Validators.required] })
  }, { validators: [passwordMatchValidator] }),
  firstName: new FormControl('', { nonNullable: true, validators: [Validators.required, minTrimmedLength(2)] }),
  lastName: new FormControl('', { nonNullable: true, validators: [Validators.required, minTrimmedLength(2)] }),
  address: new FormGroup({
    street: new FormControl('', { nonNullable: true }),
    zip: new FormControl('', { nonNullable: true, validators: [Validators.required] }),
    city: new FormControl('', { nonNullable: true, validators: [Validators.required] })
  }),
  contacts: new FormArray([createContact()]),
  score: new FormControl(0, { nonNullable: true })
});
```

---

## Annexes

### A) Cheatsheet — Méthodes clés

- `setValue` : nécessite tous les champs
- `patchValue` : partiel, recommandé pour hydratation
- `reset` : réinitialise valeur + état
- `markAllAsTouched` : forcer affichage erreurs au submit
- `addValidators / removeValidators / clearValidators`
- `setErrors` : à utiliser prudemment (risque d’écraser des erreurs)

### B) Anti-patterns fréquents

1. **Mettre une règle cross-field sur un FormControl** au lieu du FormGroup.
2. **Recréer** le formulaire à chaque changement (perte d’état).
3. **Async validator** qui appelle l’API à chaque frappe, sans debounce.
4. Manipuler `setErrors` sans merger : écrase d’autres erreurs.

### C) Ressources

- Docs Angular — Reactive Forms : https://angular.dev/guide/forms/reactive-forms
- API Validators : https://angular.io/api/forms/Validators
- RxJS operators : https://rxjs.dev/

---

## Évaluation (suggestion)

- Quiz court : statuts, erreurs, cross-field, async
- Exercice : implémenter un FormArray de lignes + total calculé + validation "minimum 1 ligne".
- Validation : code review sur patterns (lisibilité, maintenabilité, tests).
