# Formation Angular — Formulaires Angular (Template-driven & Reactive Forms)

> **Objectif** : maîtriser la création, la validation et la gestion d’état des formulaires Angular, en comprenant les deux approches **template-driven** et **reactive forms**, avec une préférence pratique pour les **Reactive Forms** dans les applications complexes.

---

## 1) Présentation & objectifs pédagogiques

### Public visé
- Développeurs frontend (débutant à intermédiaire en Angular)
- Formateurs/tech leads souhaitant structurer une montée en compétences

### Prérequis
- Bases TypeScript (interfaces, classes, fonctions)
- Bases Angular (modules/standalone, composants, data binding, directives)

### Objectifs pédagogiques
À la fin de la formation, l’apprenant saura :
1. Différencier **Template-driven forms** et **Reactive forms**
2. Construire un formulaire complet (UI, validation, soumission)
3. Gérer l’état : `value`, `status`, `touched`, `dirty`, `errors`
4. Écrire des validateurs **sync** et **async** (custom)
5. Gérer des formulaires dynamiques : `FormArray`, champs conditionnels
6. Implémenter des patterns robustes (typed forms, reset, patch, UX erreurs)

---

## 2) Vue d’ensemble : les deux types de formulaires Angular

Angular propose **2 familles** :

### A. Template-driven forms
- Logique de formulaire majoritairement dans le **template** (`[(ngModel)]`, directives `ngForm`, `ngModel`)
- Simple à démarrer
- Moins efficace sur : formulaires complexes, validation avancée, composition, tests

### B. Reactive forms (recommandé pour applications complexes)
- Définition du formulaire dans le **code** (TS)
- Basé sur la notion de **modèle de formulaire** : `FormControl`, `FormGroup`, `FormArray`
- Très bon contrôle, testabilité, scénarios dynamiques

**Recommandation** :
- Petits formulaires simples, prototypes → Template-driven OK
- Formulaires long/complexe, multi-étapes, validation riche, dynamique → **Reactive Forms**

---

## 3) Mise en place — imports et architecture

### 3.1 Imports Angular
- **Template-driven** → `FormsModule`
- **Reactive forms** → `ReactiveFormsModule`

#### Exemple (NgModule)
```ts
import { NgModule } from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';
import { FormsModule, ReactiveFormsModule } from '@angular/forms';

@NgModule({
  imports: [BrowserModule, FormsModule, ReactiveFormsModule],
})
export class AppModule {}
```

#### Exemple (Standalone component)
```ts
import { Component } from '@angular/core';
import { CommonModule } from '@angular/common';
import { ReactiveFormsModule } from '@angular/forms';

@Component({
  standalone: true,
  selector: 'app-form',
  imports: [CommonModule, ReactiveFormsModule],
  templateUrl: './form.component.html',
})
export class FormComponent {}
```

---

## 4) Template-driven forms — fondamentaux

### 4.1 Concept
Le formulaire est piloté par le template via :
- `<form #f="ngForm">`
- `[(ngModel)]` pour binder un champ
- Validation via attributs HTML et directives Angular

### 4.2 Exemple complet : formulaire de contact

#### TypeScript
```ts
import { Component } from '@angular/core';

interface ContactModel {
  name: string;
  email: string;
  message: string;
}

@Component({
  selector: 'app-contact-td',
  templateUrl: './contact-td.component.html'
})
export class ContactTdComponent {
  model: ContactModel = {
    name: '',
    email: '',
    message: ''
  };

  submitted = false;

  submit() {
    this.submitted = true;
    // À ce niveau, le template fournit l’état du formulaire.
    // Ici on simule un envoi.
    console.log('Submit TD:', this.model);
  }
}
```

#### Template
```html
<form #f="ngForm" (ngSubmit)="submit()" novalidate>
  <div>
    <label>Nom</label>
    <input
      name="name"
      [(ngModel)]="model.name"
      #name="ngModel"
      required
      minlength="2" />

    <small *ngIf="name.invalid && (name.dirty || name.touched)">
      <span *ngIf="name.errors?.['required']">Le nom est requis.</span>
      <span *ngIf="name.errors?.['minlength']">Min 2 caractères.</span>
    </small>
  </div>

  <div>
    <label>Email</label>
    <input
      name="email"
      type="email"
      [(ngModel)]="model.email"
      #email="ngModel"
      required
      email />

    <small *ngIf="email.invalid && (email.dirty || email.touched)">
      <span *ngIf="email.errors?.['required']">Email requis.</span>
      <span *ngIf="email.errors?.['email']">Email invalide.</span>
    </small>
  </div>

  <div>
    <label>Message</label>
    <textarea
      name="message"
      [(ngModel)]="model.message"
      #message="ngModel"
      required
      minlength="10"></textarea>

    <small *ngIf="message.invalid && (message.dirty || message.touched)">
      <span *ngIf="message.errors?.['required']">Message requis.</span>
      <span *ngIf="message.errors?.['minlength']">Min 10 caractères.</span>
    </small>
  </div>

  <button type="submit" [disabled]="f.invalid">Envoyer</button>

  <pre>Form status: {{ f.status }} | valid: {{ f.valid }}</pre>
  <pre>Model: {{ model | json }}</pre>
</form>
```

### 4.3 Points clés template-driven
- Simple mais la logique se disperse dans le template
- Difficile à tester finement la logique de validation
- Moins adapté aux formulaires dynamiques riches

---

## 5) Reactive Forms — concepts & avantages

### 5.1 Briques de base
- `FormControl` : un champ (valeur + état + validation)
- `FormGroup` : un groupe de contrôles (objet)
- `FormArray` : une liste de contrôles/groupe (tableau)

### 5.2 États et propriétés essentiels
- `value` : valeur courante
- `status` : `VALID | INVALID | PENDING | DISABLED`
- `errors` : map d’erreurs
- `touched/untouched` : interaction blur
- `dirty/pristine` : modification

### 5.3 Flux réactifs
Reactive forms expose des Observables :
- `valueChanges`
- `statusChanges`

Ces flux permettent :
- validation dynamique
- auto-save
- calculs dérivés
- UX plus riche

---

## 6) Reactive Forms — création d’un formulaire complet

### 6.1 Exemple : formulaire d’inscription

#### Objectif fonctionnel
- Champs : prénom, email, password, confirmation, newsletter
- Validation : required, email, minLength, match password
- Soumission : désactiver bouton tant que formulaire invalide

### 6.2 TypeScript (FormBuilder + Typed Forms)
> Angular supporte les **typed reactive forms**. L’approche ci-dessous illustre un modèle typé et des validateurs.

```ts
import { Component } from '@angular/core';
import {
  AbstractControl,
  FormBuilder,
  ReactiveFormsModule,
  ValidationErrors,
  Validators,
} from '@angular/forms';
import { CommonModule } from '@angular/common';

function passwordsMatch(group: AbstractControl): ValidationErrors | null {
  const password = group.get('password')?.value;
  const confirm = group.get('confirmPassword')?.value;
  if (!password || !confirm) return null;
  return password === confirm ? null : { passwordsMismatch: true };
}

@Component({
  standalone: true,
  selector: 'app-signup-rf',
  imports: [CommonModule, ReactiveFormsModule],
  templateUrl: './signup-rf.component.html'
})
export class SignupRfComponent {
  constructor(private fb: FormBuilder) {}

  // FormGroup typé (approche simple via inférence)
  form = this.fb.group(
    {
      firstName: this.fb.nonNullable.control('', [Validators.required, Validators.minLength(2)]),
      email: this.fb.nonNullable.control('', [Validators.required, Validators.email]),
      password: this.fb.nonNullable.control('', [Validators.required, Validators.minLength(8)]),
      confirmPassword: this.fb.nonNullable.control('', [Validators.required]),
      newsletter: this.fb.nonNullable.control(false),
    },
    {
      validators: [passwordsMatch],
      updateOn: 'blur',
    }
  );

  submitting = false;

  get f() {
    return this.form.controls;
  }

  submit() {
    // Force affichage des erreurs si l’utilisateur clique trop tôt
    this.form.markAllAsTouched();

    if (this.form.invalid) return;

    this.submitting = true;

    const payload = this.form.getRawValue();
    console.log('Submit RF payload:', payload);

    // Simule un appel API
    setTimeout(() => {
      this.submitting = false;
      this.form.reset({
        firstName: '',
        email: '',
        password: '',
        confirmPassword: '',
        newsletter: false,
      });
    }, 700);
  }
}
```

### 6.3 Template (liaison avec formGroup / formControlName)
```html
<form [formGroup]="form" (ngSubmit)="submit()" novalidate>
  <div>
    <label>Prénom</label>
    <input type="text" formControlName="firstName" />
    <small *ngIf="f.firstName.invalid && (f.firstName.touched || f.firstName.dirty)">
      <span *ngIf="f.firstName.errors?.['required']">Prénom requis.</span>
      <span *ngIf="f.firstName.errors?.['minlength']">Min 2 caractères.</span>
    </small>
  </div>

  <div>
    <label>Email</label>
    <input type="email" formControlName="email" />
    <small *ngIf="f.email.invalid && (f.email.touched || f.email.dirty)">
      <span *ngIf="f.email.errors?.['required']">Email requis.</span>
      <span *ngIf="f.email.errors?.['email']">Email invalide.</span>
    </small>
  </div>

  <div>
    <label>Mot de passe</label>
    <input type="password" formControlName="password" />
    <small *ngIf="f.password.invalid && (f.password.touched || f.password.dirty)">
      <span *ngIf="f.password.errors?.['required']">Mot de passe requis.</span>
      <span *ngIf="f.password.errors?.['minlength']">Min 8 caractères.</span>
    </small>
  </div>

  <div>
    <label>Confirmation</label>
    <input type="password" formControlName="confirmPassword" />
    <small *ngIf="f.confirmPassword.invalid && (f.confirmPassword.touched || f.confirmPassword.dirty)">
      <span *ngIf="f.confirmPassword.errors?.['required']">Confirmation requise.</span>
    </small>
  </div>

  <small *ngIf="form.errors?.['passwordsMismatch'] && (f.confirmPassword.touched || f.confirmPassword.dirty)">
    Les mots de passe ne correspondent pas.
  </small>

  <div>
    <label>
      <input type="checkbox" formControlName="newsletter" />
      S’inscrire à la newsletter
    </label>
  </div>

  <button type="submit" [disabled]="form.invalid || submitting">
    {{ submitting ? 'Envoi...' : 'Créer un compte' }}
  </button>

  <hr />
  <pre>Status: {{ form.status }}</pre>
  <pre>Value: {{ form.value | json }}</pre>
</form>
```

### 6.4 Bonnes pratiques UX sur les erreurs
- Ne pas afficher les erreurs dès le chargement : utiliser `touched`/`dirty`
- Au `submit` : `markAllAsTouched()` pour guider l’utilisateur
- Centraliser l’affichage d’erreurs via un composant (voir section 11)

---

## 7) Validations — built-in, custom, sync & async

### 7.1 Validateurs built-in (Angular)
- `Validators.required`
- `Validators.email`
- `Validators.minLength(n)` / `maxLength(n)`
- `Validators.pattern(regex)`

### 7.2 Validateur custom (sync)
Exemple : refuser un prénom dans une blacklist.

```ts
import { AbstractControl, ValidationErrors, ValidatorFn } from '@angular/forms';

export function forbiddenNames(names: string[]): ValidatorFn {
  const set = new Set(names.map(n => n.toLowerCase()));
  return (control: AbstractControl): ValidationErrors | null => {
    const value = (control.value ?? '').toString().toLowerCase().trim();
    if (!value) return null;
    return set.has(value) ? { forbiddenName: true } : null;
  };
}
```

Utilisation :
```ts
firstName: this.fb.nonNullable.control('', [Validators.required, forbiddenNames(['admin', 'root'])])
```

### 7.3 Validateur async (ex : email déjà utilisé)
> Un validateur async retourne un `Observable<ValidationErrors | null>` (ou une Promise).

```ts
import { AbstractControl, AsyncValidatorFn, ValidationErrors } from '@angular/forms';
import { Observable, of, timer } from 'rxjs';
import { map, switchMap } from 'rxjs/operators';

function emailTakenValidator(fakeApi: (email: string) => Observable<boolean>): AsyncValidatorFn {
  return (control: AbstractControl): Observable<ValidationErrors | null> => {
    const email = (control.value ?? '').toString();
    if (!email) return of(null);

    return timer(300).pipe(
      switchMap(() => fakeApi(email)),
      map(isTaken => (isTaken ? { emailTaken: true } : null))
    );
  };
}
```

Utilisation :
```ts
email: this.fb.nonNullable.control('', {
  validators: [Validators.required, Validators.email],
  asyncValidators: [emailTakenValidator(this.checkEmail.bind(this))],
  updateOn: 'blur',
})

checkEmail(email: string) {
  // simulation
  return of(email.toLowerCase().includes('test'));
}
```

### 7.4 Comprendre `PENDING`
Lorsqu’un async validator s’exécute, le contrôle passe en `PENDING`.
- Désactiver le bouton : `form.pending || form.invalid`

---

## 8) Formulaires dynamiques avec FormArray

### Cas d’usage
- Liste de compétences, téléphones, participants
- Ajout/suppression à la volée

### Exemple : liste de compétences

#### TypeScript
```ts
import { Component } from '@angular/core';
import { FormArray, FormBuilder, ReactiveFormsModule, Validators } from '@angular/forms';
import { CommonModule } from '@angular/common';

@Component({
  standalone: true,
  selector: 'app-skills',
  imports: [CommonModule, ReactiveFormsModule],
  templateUrl: './skills.component.html'
})
export class SkillsComponent {
  constructor(private fb: FormBuilder) {}

  form = this.fb.group({
    skills: this.fb.array([
      this.fb.nonNullable.control('Angular', [Validators.required, Validators.minLength(2)])
    ])
  });

  get skills(): FormArray {
    return this.form.get('skills') as FormArray;
  }

  addSkill() {
    this.skills.push(this.fb.nonNullable.control('', [Validators.required, Validators.minLength(2)]));
  }

  removeSkill(i: number) {
    this.skills.removeAt(i);
  }

  submit() {
    this.form.markAllAsTouched();
    if (this.form.invalid) return;
    console.log(this.form.value);
  }
}
```

#### Template
```html
<form [formGroup]="form" (ngSubmit)="submit()">
  <h3>Compétences</h3>

  <div formArrayName="skills">
    <div *ngFor="let ctrl of skills.controls; let i = index">
      <input [formControlName]="i" placeholder="Compétence" />
      <button type="button" (click)="removeSkill(i)">Supprimer</button>

      <small *ngIf="ctrl.invalid && (ctrl.dirty || ctrl.touched)">
        <span *ngIf="ctrl.errors?.['required']">Requis.</span>
        <span *ngIf="ctrl.errors?.['minlength']">Min 2 caractères.</span>
      </small>
    </div>
  </div>

  <button type="button" (click)="addSkill()">Ajouter</button>
  <button type="submit" [disabled]="form.invalid">Enregistrer</button>

  <pre>{{ form.value | json }}</pre>
</form>
```

---

## 9) Gestion avancée : enable/disable, setValue/patchValue, updateOn

### 9.1 disable/enable
```ts
this.form.controls.newsletter.disable();
this.form.controls.newsletter.enable();
```

### 9.2 setValue vs patchValue
- `setValue(...)` : exige toutes les clés
- `patchValue(...)` : partiel

```ts
this.form.patchValue({ firstName: 'Sarah' });
```

### 9.3 updateOn
- `change` (par défaut)
- `blur` (utile pour async/ éviter bruit)
- `submit` (validation au moment de soumettre)

```ts
this.fb.group({...}, { updateOn: 'submit' })
```

---

## 10) Patterns conseillés en applications réelles

### 10.1 Centraliser la logique de formulaire
- Construire le `FormGroup` dans le composant (ou un service/factory)
- Éviter d’éparpiller des conditions complexes dans le template

### 10.2 Composant d’affichage d’erreurs (approche simple)

#### error-messages.component.ts
```ts
import { Component, Input } from '@angular/core';
import { AbstractControl } from '@angular/forms';

@Component({
  selector: 'app-error-messages',
  template: `
    <ng-container *ngIf="control && control.invalid && (control.touched || control.dirty)">
      <small *ngIf="control.errors?.['required']">Champ requis.</small>
      <small *ngIf="control.errors?.['email']">Email invalide.</small>
      <small *ngIf="control.errors?.['minlength']">
        Longueur minimale: {{ control.errors?.['minlength']?.requiredLength }}
      </small>
      <small *ngIf="control.errors?.['emailTaken']">Email déjà utilisé.</small>
    </ng-container>
  `
})
export class ErrorMessagesComponent {
  @Input() control: AbstractControl | null = null;
}
```

#### Utilisation
```html
<input formControlName="email" />
<app-error-messages [control]="f.email"></app-error-messages>
```

### 10.3 Soumission robuste
- `markAllAsTouched()`
- Désactiver pendant l’envoi (`submitting`)
- Gérer les erreurs API et les remonter dans le formulaire (`setErrors`, erreurs globales)

Ex :
```ts
this.form.setErrors({ server: 'Une erreur est survenue.' });
```

---

## 11) Exercice fil rouge (atelier)

### Énoncé
Créer un formulaire **Reactive** “Profil utilisateur” avec :
- `firstName`, `lastName` (required, minLength 2)
- `email` (required, email, async: emailTaken)
- `address` (FormGroup) : `street`, `zip`, `city`
- `phones` (FormArray) : au moins 1 téléphone, pattern
- Bouton “Ajouter téléphone”, “Supprimer”
- Affichage des erreurs propre
- Bouton submit désactivé si `invalid` ou `pending`

### Critères de réussite
- Données JSON correctes au submit
- Validations visibles au bon moment
- Formulaire maintenable (code clair, pas de logique excessive dans le HTML)

---

## 12) Récapitulatif & check-list

### Choisir Template-driven si
- Formulaire très simple
- Peu de règles métier
- Peu de dynamique

### Choisir Reactive Forms si
- Formulaire complexe / dynamique
- Validation riche (sync/async)
- Besoin de testabilité et contrôle fin

### Check-list “formulaire prod-ready”
- [ ] UX erreurs : `touched/dirty`, `markAllAsTouched()`
- [ ] `updateOn` adapté
- [ ] Validations sync/async
- [ ] Gestion `pending`, désactivation du submit
- [ ] Réutilisation d’un composant pour messages d’erreurs
- [ ] FormArray pour listes dynamiques

---

## Annexes — aide-mémoire (API)

### Principales classes
- `FormControl(value, validators)`
- `FormGroup({ ...controls }, options)`
- `FormArray([ ...controls ], validators)`

### Méthodes utiles
- `setValue`, `patchValue`, `reset`
- `markAsTouched`, `markAllAsTouched`
- `setErrors`, `updateValueAndValidity`
- `disable`, `enable`

---

### Fin de formation
Pour aller plus loin :
- Multi-step forms (wizard)
- Dynamic forms driven by JSON schema
- Custom ControlValueAccessor (composants de champ réutilisables)
