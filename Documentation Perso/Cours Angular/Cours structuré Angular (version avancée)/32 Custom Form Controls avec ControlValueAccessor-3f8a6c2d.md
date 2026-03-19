# Formation Angular — Custom Form Controls avec `ControlValueAccessor`

- **Code formation** : 32
- **Public** : développeurs Angular (niveau intermédiaire)
- **Pré-requis** : TypeScript, composants Angular, bases des **Reactive Forms** et/ou **Template-driven forms**
- **Durée conseillée** : 1 jour (6–7h) ou 2 demi-journées

---

## Objectifs pédagogiques

À la fin de la formation, vous saurez :

1. Expliquer le rôle de `ControlValueAccessor` (CVA) et son intégration dans Angular Forms.
2. Créer un **composant de formulaire personnalisé** compatible avec Reactive Forms et Template-driven.
3. Gérer correctement :
   - la **propagation de valeur** (écriture/lecture),
   - l’état **disabled**,
   - les événements **touched/blur**,
   - la **validation** (via `Validator` et/ou erreurs personnalisées).
4. Concevoir un composant CVA robuste (accessibilité, UX, edge cases, performance).

---

## Plan de la formation

1. **Rappels Angular Forms** : architecture, `AbstractControl`, `NgControl`, cycles de vie.
2. **Pourquoi et quand utiliser un Custom Form Control**.
3. **Anatomie d’un ControlValueAccessor** : `writeValue`, `registerOnChange`, `registerOnTouched`, `setDisabledState`.
4. **Implémentation guidée** : créer un composant `app-rating` (exemple complet).
5. **Intégration** : Reactive Forms, Template-driven, `formControlName`, `ngModel`.
6. **Validation** : intégration native (Validators Angular), stratégie de retour d’erreurs, `NG_VALIDATORS`.
7. **Bonnes pratiques** : accessibilité, gestion du focus, `OnPush`, tests, compatibilité UI tiers.
8. **Atelier final** : transformer un composant UI existant en CVA (checklist + corrigé).

---

# 1) Rappels : Angular Forms (le minimum utile)

Angular propose deux approches :

- **Template-driven forms** (directive `ngModel`)
- **Reactive forms** (classes `FormControl`, `FormGroup`, `FormArray`)

Dans les deux cas, Angular Forms manipule des **contrôles abstraits** :

- `AbstractControl` (base)
- `FormControl`
- `FormGroup`
- `FormArray`

Un « champ de formulaire » (input HTML ou composant) communique avec le `FormControl` via une couche d’adaptation.

## Le problème

- Les éléments natifs (`<input>`, `<select>`) sont déjà compatibles.
- Un composant maison (`<app-date-picker>`, `<app-phone-input>`, `<app-toggle>`) **ne l’est pas** par défaut.

Si vous voulez :

- `formControlName="..."`
- `[(ngModel)]` (si vous l’utilisez)
- `disabled`, `touched`, `dirty`, `valid/invalid`
- et la validation intégrée

…alors vous devez implémenter **`ControlValueAccessor`**.

---

# 2) Pourquoi / Quand utiliser `ControlValueAccessor`

Vous utilisez un CVA quand vous voulez qu’un composant se comporte comme un champ Angular Forms.

Cas typiques :

- **Composants UI internes** : select avancé, input de téléphone multi-pays, slider, rating.
- **Wrapping d’un composant tiers** : un date picker, un editor WYSIWYG, etc.
- **Valeur complexe** : l’UI manipule un modèle différent (ex. `{ countryCode, number }`), mais le form control doit recevoir un objet/valeur consolidée.

Bénéfices :

- Intégration transparente à Angular Forms
- Centralisation des règles de validation dans les formes
- UX cohérente (disabled, touched, erreurs)

---

# 3) Anatomie d’un `ControlValueAccessor`

L’interface `ControlValueAccessor` impose 4 méthodes :

```ts
export interface ControlValueAccessor {
  writeValue(obj: any): void;
  registerOnChange(fn: any): void;
  registerOnTouched(fn: any): void;
  setDisabledState?(isDisabled: boolean): void;
}
```

## Rôle de chaque méthode

### `writeValue(value)`
Appelée par Angular quand la valeur du `FormControl` change (initialisation, `setValue`, `patchValue`).

- Vous devez **mettre à jour l’UI** (état interne / affichage).
- Ne déclenchez pas `onChange` depuis `writeValue` (sinon boucle).

### `registerOnChange(fn)`
Angular vous fournit une fonction `fn`.

- Vous devez la stocker (souvent `this.onChange = fn`).
- Vous l’appelez quand l’utilisateur modifie la valeur via l’UI.

### `registerOnTouched(fn)`
Angular vous fournit une fonction `fn`.

- Stockez-la (souvent `this.onTouched = fn`).
- Appelez-la quand le champ est « touché » (souvent `blur`, ou interaction de focus).

### `setDisabledState(isDisabled)`
Angular vous informe de l’état disabled.

- Vous devez désactiver l’UI (attributs, interactions, styles).

---

# 4) Mise en place : provider `NG_VALUE_ACCESSOR`

Pour qu’Angular reconnaisse votre composant comme un champ de formulaire, il faut fournir `NG_VALUE_ACCESSOR`.

Exemple type :

```ts
import { Component, forwardRef } from '@angular/core';
import { ControlValueAccessor, NG_VALUE_ACCESSOR } from '@angular/forms';

@Component({
  selector: 'app-rating',
  templateUrl: './rating.component.html',
  providers: [{
    provide: NG_VALUE_ACCESSOR,
    useExisting: forwardRef(() => RatingComponent),
    multi: true,
  }]
})
export class RatingComponent implements ControlValueAccessor {
  // ...
}
```

Notes :

- `multi: true` est obligatoire (Angular peut enregistrer plusieurs accessors).
- `forwardRef` évite des soucis de référence circulaire.

---

# 5) Implémentation guidée : composant `app-rating`

## Objectif

Créer un composant de notation (1 à 5) utilisable comme :

```html
<form [formGroup]="form">
  <app-rating formControlName="rating"></app-rating>
</form>
```

### 5.1 API et contraintes

- Valeur : `number | null`
- `disabled` doit empêcher l’interaction
- Doit marquer `touched` quand l’utilisateur quitte le composant (ou interagit explicitement)
- Doit supporter `writeValue` (mise à jour depuis le formulaire)

---

## 5.2 Template (HTML)

```html
<div class="rating"
     [class.rating--disabled]="isDisabled"
     (blur)="markAsTouched()"
     tabindex="0">

  <button type="button"
          *ngFor="let star of stars"
          class="rating__star"
          [class.is-active]="star <= (value ?? 0)"
          [disabled]="isDisabled"
          (click)="select(star)">
    ★
  </button>
</div>
```

Notes :

- `tabindex="0"` donne un focus au conteneur (utile pour `touched` et accessibilité).
- `button type="button"` évite de soumettre un `<form>`.

---

## 5.3 Styles (CSS rapide)

```css
.rating { display: inline-flex; gap: 4px; outline: none; }
.rating__star { font-size: 24px; background: transparent; border: none; cursor: pointer; }
.rating__star.is-active { color: gold; }
.rating--disabled .rating__star { cursor: not-allowed; opacity: 0.5; }
```

---

## 5.4 Composant (TypeScript)

```ts
import { Component, forwardRef } from '@angular/core';
import { ControlValueAccessor, NG_VALUE_ACCESSOR } from '@angular/forms';

@Component({
  selector: 'app-rating',
  templateUrl: './rating.component.html',
  styleUrls: ['./rating.component.css'],
  providers: [{
    provide: NG_VALUE_ACCESSOR,
    useExisting: forwardRef(() => RatingComponent),
    multi: true,
  }]
})
export class RatingComponent implements ControlValueAccessor {
  stars = [1, 2, 3, 4, 5];

  value: number | null = null;
  isDisabled = false;

  // callbacks fournis par Angular Forms
  private onChange: (value: number | null) => void = () => {};
  private onTouched: () => void = () => {};

  // 1) Angular -> UI
  writeValue(value: number | null): void {
    this.value = value;
  }

  // 2) UI -> Angular
  registerOnChange(fn: (value: number | null) => void): void {
    this.onChange = fn;
  }

  registerOnTouched(fn: () => void): void {
    this.onTouched = fn;
  }

  setDisabledState(isDisabled: boolean): void {
    this.isDisabled = isDisabled;
  }

  select(star: number): void {
    if (this.isDisabled) return;

    // exemple : cliquer la même valeur pourrait "toggle" à null
    const next = (this.value === star) ? null : star;

    this.value = next;
    this.onChange(next);
    this.markAsTouched();
  }

  markAsTouched(): void {
    this.onTouched();
  }
}
```

### Points clés

- `writeValue` met seulement l’état interne.
- `onChange` est appelé **uniquement** sur action utilisateur.
- `onTouched` est appelé quand le composant est « touché ».

---

# 6) Utilisation dans Reactive Forms

## Exemple complet

```ts
import { Component } from '@angular/core';
import { FormBuilder, Validators } from '@angular/forms';

@Component({
  selector: 'app-demo-form',
  templateUrl: './demo-form.component.html'
})
export class DemoFormComponent {
  form = this.fb.group({
    rating: [null as number | null, [Validators.required, Validators.min(1)]],
  });

  constructor(private fb: FormBuilder) {}

  submit(): void {
    if (this.form.invalid) {
      this.form.markAllAsTouched();
      return;
    }

    console.log(this.form.value);
  }
}
```

```html
<form [formGroup]="form" (ngSubmit)="submit()">
  <label>Note</label>
  <app-rating formControlName="rating"></app-rating>

  <div class="error" *ngIf="form.controls.rating.touched && form.controls.rating.invalid">
    <div *ngIf="form.controls.rating.errors?.['required']">La note est requise.</div>
    <div *ngIf="form.controls.rating.errors?.['min']">La note doit être ≥ 1.</div>
  </div>

  <button type="submit">Envoyer</button>
</form>
```

---

# 7) Utilisation dans Template-driven (optionnel)

Même CVA, autre consommation :

```html
<app-rating [(ngModel)]="rating" name="rating" required></app-rating>
<p>Valeur: {{ rating }}</p>
```

Remarques :

- Le composant CVA fonctionne aussi avec `ngModel`.
- Attention aux règles de style d’équipe: Angular recommande Reactive Forms pour les projets complexes.

---

# 8) Brancher la validation au composant (CVA + Validator)

## Pourquoi ajouter `Validator` ?

Parfois, vous voulez que le **composant** expose sa propre validation (ex. valeur minimale, format interne, contraintes UI).

Deux approches :

1. **Validation côté formulaire** (souvent préférable)
2. **Validation côté composant** via `NG_VALIDATORS`

### Quand valider dans le composant ?

- Le composant encapsule une logique interne difficile à exprimer depuis l’extérieur.
- Vous wrappez un composant tiers et voulez exposer des erreurs standardisées.

---

## Implémenter `Validator`

Angular permet d’ajouter :

- `validate(control: AbstractControl): ValidationErrors | null`
- `registerOnValidatorChange(fn: () => void)` (optionnel)

### Exemple : `app-rating` impose une valeur entre 1 et 5 quand non-null

```ts
import {
  Component, forwardRef, Input
} from '@angular/core';
import {
  AbstractControl,
  ControlValueAccessor,
  NG_VALIDATORS,
  NG_VALUE_ACCESSOR,
  ValidationErrors,
  Validator
} from '@angular/forms';

@Component({
  selector: 'app-rating',
  templateUrl: './rating.component.html',
  styleUrls: ['./rating.component.css'],
  providers: [
    {
      provide: NG_VALUE_ACCESSOR,
      useExisting: forwardRef(() => RatingComponent),
      multi: true,
    },
    {
      provide: NG_VALIDATORS,
      useExisting: forwardRef(() => RatingComponent),
      multi: true,
    }
  ]
})
export class RatingComponent implements ControlValueAccessor, Validator {
  @Input() min = 1;
  @Input() max = 5;

  stars = [1, 2, 3, 4, 5];
  value: number | null = null;
  isDisabled = false;

  private onChange: (value: number | null) => void = () => {};
  private onTouched: () => void = () => {};
  private onValidatorChange: () => void = () => {};

  writeValue(value: number | null): void {
    this.value = value;
  }
  registerOnChange(fn: (value: number | null) => void): void {
    this.onChange = fn;
  }
  registerOnTouched(fn: () => void): void {
    this.onTouched = fn;
  }
  setDisabledState(isDisabled: boolean): void {
    this.isDisabled = isDisabled;
  }

  validate(_: AbstractControl): ValidationErrors | null {
    if (this.value == null) return null; // laissez "required" être géré par le form si besoin
    if (this.value < this.min) return { ratingMin: { min: this.min, actual: this.value } };
    if (this.value > this.max) return { ratingMax: { max: this.max, actual: this.value } };
    return null;
  }

  registerOnValidatorChange(fn: () => void): void {
    this.onValidatorChange = fn;
  }

  select(star: number): void {
    if (this.isDisabled) return;
    const next = (this.value === star) ? null : star;

    this.value = next;
    this.onChange(next);
    this.onValidatorChange();
    this.onTouched();
  }
}
```

Points importants :

- L’input `min/max` peut changer la validation ; appelez `onValidatorChange` lors de changements significatifs.
- La validation « `required` » est souvent mieux portée par le formulaire (via `Validators.required`) pour rester cohérent.

---

# 9) Accès à `NgControl` (patterns avancés)

Parfois, vous voulez lire l’état du contrôle (`invalid`, `touched`) depuis le composant (afin d’afficher une bordure rouge / message).

Vous pouvez injecter `NgControl` :

```ts
import { Component, Optional, Self } from '@angular/core';
import { NgControl } from '@angular/forms';

@Component({
  selector: 'app-something',
  template: `...`
})
export class SomethingComponent {
  constructor(@Optional() @Self() public ngControl: NgControl) {
    if (this.ngControl) {
      // Permet au composant de se déclarer comme value accessor sans providers explicites
      // (pattern alternatif)
      this.ngControl.valueAccessor = this as any;
    }
  }
}
```

### Quand utiliser ce pattern ?

- Composants partagés très génériques.
- Besoin d’accéder au control depuis le composant.

Inconvénient :

- Moins explicite que le provider `NG_VALUE_ACCESSOR`.

---

# 10) Gestion correcte de `touched`, focus et accessibilité

## Stratégie `touched`

- Un champ devient `touched` quand il a reçu le focus puis l’a perdu.
- Beaucoup de composants custom n’ont pas de `blur` natif.

Bonnes pratiques :

- Fournir un élément focusable (`tabindex="0"`) sur le conteneur.
- Mapper `blur` vers `onTouched`.
- Appeler `onTouched` lors de la première interaction si le composant est essentiellement « cliquable ».

## Accessibilité

- Donnez un rôle ou une sémantique correcte (`button`, `aria-pressed`, `role="slider"`, etc.).
- Supportez clavier (ex. flèches gauche/droite, espace/entrée).
- Exposez `aria-disabled` si nécessaire.

---

# 11) Edge cases fréquents

## 1) Boucle de valeur (writeValue → onChange)

Ne faites pas :

- Appeler `onChange` dans `writeValue`.

`writeValue` est **descendant** (Form → UI), `onChange` est **montant** (UI → Form).

## 2) Valeurs `null` / `undefined`

- Normalisez (souvent `null` est préférable).
- Documentez la valeur “vide”.

## 3) `setDisabledState` oublié

Résultat : le formulaire peut être disabled, mais votre UI reste interactive.

## 4) Composants à valeur complexe

Exemple : un input téléphone avec valeur objet.

- Le CVA manipule l’objet final : `{ country: string, number: string }`.
- L’UI peut contenir plusieurs inputs internes.

Approche :

- Maintenir un `model` interne.
- Sur chaque modification interne, construire la valeur finale et appeler `onChange(finalValue)`.

---

# 12) Atelier final (avec checklist)

## Consigne

Vous disposez d’un composant `app-toggle` (switch On/Off). Votre objectif :

- Le rendre compatible avec `formControlName`.
- Supporter `disabled`.
- Marquer `touched` correctement.
- Exposer une validation optionnelle : si `requiredTrue`.

### Spécifications

- Valeur : `boolean`
- UI : clic sur une zone, clavier (espace)

---

## Squelette proposé

```ts
@Component({
  selector: 'app-toggle',
  template: `
    <div class="toggle"
         [class.is-on]="value"
         [class.is-disabled]="isDisabled"
         tabindex="0"
         (click)="toggle()"
         (keydown.space)="toggle()"
         (blur)="markAsTouched()"
         [attr.aria-checked]="value"
         role="switch">
      <span class="thumb"></span>
    </div>
  `,
  providers: [{
    provide: NG_VALUE_ACCESSOR,
    useExisting: forwardRef(() => ToggleComponent),
    multi: true,
  }]
})
export class ToggleComponent implements ControlValueAccessor {
  value = false;
  isDisabled = false;

  private onChange: (v: boolean) => void = () => {};
  private onTouched: () => void = () => {};

  writeValue(v: boolean): void {
    this.value = !!v;
  }
  registerOnChange(fn: (v: boolean) => void): void {
    this.onChange = fn;
  }
  registerOnTouched(fn: () => void): void {
    this.onTouched = fn;
  }
  setDisabledState(isDisabled: boolean): void {
    this.isDisabled = isDisabled;
  }

  toggle(): void {
    if (this.isDisabled) return;
    this.value = !this.value;
    this.onChange(this.value);
    this.markAsTouched();
  }

  markAsTouched(): void {
    this.onTouched();
  }
}
```

### Vérifications

- `formControl.disable()` rend le toggle non cliquable
- Le `FormControl` reflète la valeur au clic
- `touched` passe à `true` après blur / interaction

---

# 13) Tests (stratégie)

## Unit tests (Jasmine/Karma ou Jest)

À couvrir :

- `writeValue` met bien à jour l’UI
- `select/toggle` appelle `onChange` avec la bonne valeur
- `setDisabledState` empêche l’interaction
- `onTouched` est appelé au blur

Pseudo-exemple :

```ts
it('should propagate value to form control', () => {
  const fixture = TestBed.createComponent(HostComponent);
  fixture.detectChanges();

  const rating = fixture.debugElement.query(By.directive(RatingComponent)).componentInstance as RatingComponent;
  rating.select(3);

  expect(fixture.componentInstance.form.value.rating).toBe(3);
});
```

---

# 14) Bonnes pratiques récapitulatives

## Checklist CVA

- [ ] Fournir `NG_VALUE_ACCESSOR` (`multi: true`)
- [ ] Implémenter `writeValue` sans appel à `onChange`
- [ ] Appeler `onChange` uniquement sur action utilisateur
- [ ] Appeler `onTouched` lorsque pertinent (blur ou première interaction)
- [ ] Implémenter `setDisabledState`
- [ ] Gérer `null` / état vide
- [ ] Supporter accessibilité (clavier, ARIA)
- [ ] (Optionnel) Implémenter `Validator` via `NG_VALIDATORS`

---

## Annexes

### Glossaire

- **CVA** : ControlValueAccessor
- **Accessor** : adaptateur entre Angular Forms et l’UI
- **touched** : contrôle ayant été focus puis blur
- **dirty** : contrôle dont la valeur a changé depuis l’initialisation

### Liens utiles (docs)

- Angular — Forms overview
- API `ControlValueAccessor`
- API `NG_VALUE_ACCESSOR`, `NG_VALIDATORS`

---

## Fin de formation — Prochaines étapes

- Créer un composant plus complexe (date range picker, select multi, autocomplete) en appliquant la checklist.
- Ajouter `OnPush` et maîtriser les implications (immutabilité, `markForCheck`).
- Wrap d’un composant tiers (Material, PrimeNG, etc.) en CVA propre et testable.
