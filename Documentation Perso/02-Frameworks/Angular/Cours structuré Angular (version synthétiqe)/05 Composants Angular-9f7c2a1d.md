# Formation Angular — Composants Angular

**Public** : développeurs débutants à intermédiaires en Angular

**Pré‑requis** : bases TypeScript, notions HTML/CSS, npm

**Objectifs pédagogiques**
- Comprendre le rôle d’un composant dans l’architecture Angular.
- Savoir créer, configurer et organiser des composants.
- Maîtriser template, styles, data binding et communication entre composants.
- Utiliser le cycle de vie, la détection de changements et les bonnes pratiques.

**Durée indicative** : 1 jour (7h) ou 2 demi‑journées

---

## Plan de la formation

1. **Rappels et architecture Angular**
2. **Anatomie d’un composant Angular**
3. **Création et déclaration de composants**
4. **Templates : syntaxe et data binding**
5. **Styles, encapsulation et bonnes pratiques CSS**
6. **Communication entre composants : @Input/@Output, events**
7. **Cycle de vie (Lifecycle Hooks)**
8. **Composants avancés : Content Projection, ViewChild, Dynamic Components**
9. **Performance et Change Detection**
10. **Structuration, conventions et tests**
11. **Atelier guidé : construire une mini‑feature réutilisable**

---

## 1) Rappels et architecture Angular

### 1.1 Les composants comme blocs de base
Dans Angular, **l’interface utilisateur** est construite comme un arbre de **composants**. Chaque composant :
- possède une **vue** (template HTML),
- éventuellement des **styles** (CSS/SCSS),
- une **logique TypeScript** (classe),
- et s’intègre dans un **module** (ou via des **standalone components**).

> Idée clé : *un composant Angular est une unité cohérente UI + comportement*, conçue pour être **réutilisable** et **testable**.

### 1.2 Liens avec les autres briques
- **Services** : logique métier, accès aux APIs, partage d’état.
- **Directives/Pipes** : enrichissent le template.
- **Routing** : affichage de composants selon l’URL.

---

## 2) Anatomie d’un composant Angular

### 2.1 Définition via le décorateur `@Component`
Un composant est défini par une classe TypeScript décorée par `@Component`.

```ts
import { Component } from '@angular/core';

@Component({
  selector: 'app-hello',
  template: `<h1>Hello {{ name }}</h1>`,
  styles: [`h1 { color: #1976d2; }`]
})
export class HelloComponent {
  name = 'Angular';
}
```

**Propriétés majeures de `@Component`**
- `selector` : nom de la balise HTML qui instancie le composant.
- `template` ou `templateUrl` : contenu HTML.
- `styles` ou `styleUrls` : styles associés.
- `standalone` : composant autonome (Angular ≥ 14).
- `imports` : dépendances (directives, pipes, composants) pour standalone.
- `changeDetection` : stratégie de détection des changements.
- `encapsulation` : encapsulation des styles.

### 2.2 Séparation template / styles / logique
Typiquement, on préfère des fichiers séparés pour la lisibilité :

```ts
@Component({
  selector: 'app-user-card',
  templateUrl: './user-card.component.html',
  styleUrls: ['./user-card.component.scss']
})
export class UserCardComponent {}
```

---

## 3) Création et déclaration de composants

### 3.1 Générer un composant avec Angular CLI
```bash
ng generate component features/users/user-card
# ou abrégé
ng g c features/users/user-card
```

### 3.2 Déclaration dans un NgModule (approche classique)
Dans Angular « traditionnel », un composant est déclaré dans un module :

```ts
@NgModule({
  declarations: [UserCardComponent],
  imports: [CommonModule],
  exports: [UserCardComponent]
})
export class UsersModule {}
```

### 3.3 Standalone components (moderne)
```ts
import { Component } from '@angular/core';
import { CommonModule } from '@angular/common';

@Component({
  selector: 'app-user-card',
  standalone: true,
  imports: [CommonModule],
  templateUrl: './user-card.component.html',
  styleUrl: './user-card.component.scss'
})
export class UserCardComponent {}
```

**Avantages** : moins de boilerplate, dépendances explicites, simplification.

### 3.4 Utiliser un composant
Dans un template parent :

```html
<app-user-card></app-user-card>
```

---

## 4) Templates : syntaxe et data binding

### 4.1 Interpolation
Afficher une valeur :

```html
<p>Bonjour {{ userName }} !</p>
```

### 4.2 Property binding
Lier une propriété DOM/attribut :

```html
<img [src]="avatarUrl" [alt]="userName" />
<button [disabled]="isSaving">Enregistrer</button>
```

### 4.3 Event binding
Réagir à un événement :

```html
<button (click)="increment()">+</button>
```

```ts
count = 0;
increment() {
  this.count++;
}
```

### 4.4 Two-way binding (`[(ngModel)]`)
Nécessite `FormsModule`.

```html
<input [(ngModel)]="userName" />
<p>Valeur: {{ userName }}</p>
```

### 4.5 Directives structurelles : `*ngIf`, `*ngFor`

```html
<p *ngIf="isAdmin">Accès administrateur</p>

<ul>
  <li *ngFor="let u of users; trackBy: trackById">{{ u.name }}</li>
</ul>
```

```ts
trackById(index: number, u: { id: number }) {
  return u.id;
}
```

### 4.6 `ng-container` et `ng-template`
- `ng-container` : groupe logique sans élément DOM.
- `ng-template` : template différé/réutilisable.

```html
<ng-container *ngIf="users.length; else empty">
  <app-user-card *ngFor="let u of users" [user]="u"></app-user-card>
</ng-container>

<ng-template #empty>
  <p>Aucun utilisateur.</p>
</ng-template>
```

---

## 5) Styles, encapsulation et bonnes pratiques CSS

### 5.1 Styles locaux
Chaque composant peut avoir ses styles :

```scss
:host {
  display: block;
}

.card {
  border: 1px solid #eee;
  padding: 1rem;
  border-radius: 8px;
}
```

### 5.2 `:host` et `:host-context`
- `:host` cible l’élément hôte du composant.
- `:host-context(.theme-dark)` applique selon un contexte parent.

### 5.3 Encapsulation
Par défaut : `ViewEncapsulation.Emulated` (scoping CSS via attributs).

```ts
import { ViewEncapsulation } from '@angular/core';

@Component({
  /* ... */
  encapsulation: ViewEncapsulation.Emulated
})
export class UserCardComponent {}
```

Autres options :
- `None` : styles globaux.
- `ShadowDom` : Shadow DOM natif (selon support navigateurs).

### 5.4 Bonnes pratiques
- BEM ou conventions similaires.
- Éviter les styles globaux inutiles.
- Penser accessibilité (contrastes, focus, etc.).

---

## 6) Communication entre composants

### 6.1 `@Input()` : passer des données du parent vers l’enfant

**Enfant** :
```ts
import { Component, Input } from '@angular/core';

@Component({
  selector: 'app-user-card',
  templateUrl: './user-card.component.html'
})
export class UserCardComponent {
  @Input() user!: { id: number; name: string; role: string };
}
```

**Template enfant** :
```html
<div class="card">
  <h3>{{ user.name }}</h3>
  <p>Rôle: {{ user.role }}</p>
</div>
```

**Parent** :
```html
<app-user-card [user]="selectedUser"></app-user-card>
```

#### Inputs typés, valeurs par défaut
```ts
@Input() compact = false;
```

### 6.2 `@Output()` : remonter un événement

**Enfant** :
```ts
import { Component, EventEmitter, Input, Output } from '@angular/core';

@Component({
  selector: 'app-user-card',
  template: `
    <div class="card">
      <h3>{{ user.name }}</h3>
      <button (click)="select()">Sélectionner</button>
    </div>
  `
})
export class UserCardComponent {
  @Input() user!: { id: number; name: string };
  @Output() selected = new EventEmitter<number>();

  select() {
    this.selected.emit(this.user.id);
  }
}
```

**Parent** :
```html
<app-user-card
  *ngFor="let u of users"
  [user]="u"
  (selected)="onUserSelected($event)"
></app-user-card>
```

```ts
onUserSelected(userId: number) {
  this.selectedUserId = userId;
}
```

### 6.3 Pattern « Présentation vs Container »
- **Presentational component** : reçoit des données, émet des events, pas de dépendances lourdes.
- **Container component** : gère récupération données, services, routing.

### 6.4 Communication via service (siblings / état partagé)
Un service injectable peut centraliser l’état ou diffuser via RxJS `Subject`.

---

## 7) Cycle de vie (Lifecycle Hooks)

### 7.1 Les hooks principaux
- `ngOnChanges(changes)` : quand un `@Input` change.
- `ngOnInit()` : initialisation après construction.
- `ngAfterViewInit()` : vue et `@ViewChild` prêts.
- `ngOnDestroy()` : nettoyage (unsubscribe, timers, listeners).

Exemple :
```ts
import { Component, Input, OnChanges, SimpleChanges, OnDestroy } from '@angular/core';

@Component({
  selector: 'app-user-card',
  template: `...`
})
export class UserCardComponent implements OnChanges, OnDestroy {
  @Input() userId!: number;

  ngOnChanges(changes: SimpleChanges) {
    if (changes['userId']) {
      // réagir à un changement d'entrée
    }
  }

  ngOnDestroy() {
    // cleanup ici
  }
}
```

### 7.2 Nettoyage RxJS
Utiliser `takeUntilDestroyed` (Angular récent) ou un `Subject`.

---

## 8) Composants avancés

### 8.1 Projection de contenu (Content Projection)
Permet d’insérer du contenu dans un composant via `<ng-content>`.

**Composant** :
```html
<div class="panel">
  <h2><ng-content select="[panel-title]"></ng-content></h2>
  <div class="body">
    <ng-content></ng-content>
  </div>
</div>
```

**Utilisation** :
```html
<app-panel>
  <span panel-title>Utilisateurs</span>
  <p>Contenu libre projeté.</p>
</app-panel>
```

### 8.2 `@ViewChild` et accès au template
Accéder à un élément ou composant enfant.

```ts
import { Component, ElementRef, ViewChild, AfterViewInit } from '@angular/core';

@Component({
  selector: 'app-search',
  template: `<input #q /><button (click)="focus()">Focus</button>`
})
export class SearchComponent implements AfterViewInit {
  @ViewChild('q') q!: ElementRef<HTMLInputElement>;

  ngAfterViewInit() {
    this.q.nativeElement.focus();
  }

  focus() {
    this.q.nativeElement.focus();
  }
}
```

### 8.3 Composants dynamiques (aperçu)
Aujourd’hui, on privilégie des patterns avec `ngComponentOutlet` ou des portails/CDK.

```html
<ng-container *ngComponentOutlet="componentType"></ng-container>
```

---

## 9) Performance et Change Detection

### 9.1 Comment Angular met à jour la vue
Angular exécute une **détection de changements** lors de plusieurs événements (click, HTTP, timers, etc.).

### 9.2 `ChangeDetectionStrategy.OnPush`
Recommandé pour composants présentations.

```ts
import { ChangeDetectionStrategy, Component, Input } from '@angular/core';

@Component({
  selector: 'app-user-card',
  templateUrl: './user-card.component.html',
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class UserCardComponent {
  @Input() user!: { id: number; name: string };
}
```

**Règles OnPush (simplifiées)**
- update si `@Input` change de **référence**,
- ou si un event vient du composant,
- ou si on déclenche manuellement via `ChangeDetectorRef`.

### 9.3 `trackBy` sur `*ngFor`
Réduit les recréations DOM.

### 9.4 Éviter les fonctions lourdes dans le template
Préférer des propriétés calculées, pipes purs.

---

## 10) Structuration, conventions et tests

### 10.1 Organisation conseillée
- `features/` : composants liés à une fonctionnalité.
- `shared/` : composants réutilisables (UI générique).
- `core/` : services singleton, interceptors.

### 10.2 Conventions
- Suffixe `Component`.
- Sélecteurs `app-...`.
- Inputs/Outputs : noms clairs (`user`, `selected`, `closed`).

### 10.3 Tests unitaires (Jasmine/Karma ou Jest)
Test de rendu simple :

```ts
import { TestBed } from '@angular/core/testing';
import { UserCardComponent } from './user-card.component';

describe('UserCardComponent', () => {
  it('should render user name', async () => {
    await TestBed.configureTestingModule({
      declarations: [UserCardComponent]
    }).compileComponents();

    const fixture = TestBed.createComponent(UserCardComponent);
    fixture.componentInstance.user = { id: 1, name: 'Ada', role: 'admin' };
    fixture.detectChanges();

    expect(fixture.nativeElement.textContent).toContain('Ada');
  });
});
```

---

## 11) Atelier guidé — Mini feature « Liste + Carte utilisateur »

### Objectif
Créer :
- un composant container `UsersPageComponent` qui récupère des données,
- un composant de présentation `UserCardComponent` réutilisable,
- un composant `UsersListComponent` qui orchestre l’affichage et relaie les événements.

### Étape 1 — Modèle
```ts
export interface User {
  id: number;
  name: string;
  role: 'admin' | 'user';
}
```

### Étape 2 — `UserCardComponent` (présentation)
```ts
@Component({
  selector: 'app-user-card',
  templateUrl: './user-card.component.html',
  styleUrls: ['./user-card.component.scss'],
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class UserCardComponent {
  @Input({ required: true }) user!: User;
  @Output() selected = new EventEmitter<User>();

  select() {
    this.selected.emit(this.user);
  }
}
```

```html
<!-- user-card.component.html -->
<article class="card" (click)="select()" tabindex="0" role="button">
  <h3>{{ user.name }}</h3>
  <small>Rôle : {{ user.role }}</small>
</article>
```

### Étape 3 — `UsersListComponent`
```ts
@Component({
  selector: 'app-users-list',
  templateUrl: './users-list.component.html'
})
export class UsersListComponent {
  @Input() users: User[] = [];
  @Output() selected = new EventEmitter<User>();

  trackById = (_: number, u: User) => u.id;
}
```

```html
<!-- users-list.component.html -->
<app-user-card
  *ngFor="let u of users; trackBy: trackById"
  [user]="u"
  (selected)="selected.emit($event)"
></app-user-card>
```

### Étape 4 — `UsersPageComponent` (container)
Simuler une source :

```ts
@Component({
  selector: 'app-users-page',
  template: `
    <h2>Utilisateurs</h2>
    <app-users-list
      [users]="users"
      (selected)="onSelected($event)"
    ></app-users-list>

    <p *ngIf="selectedUser">Sélection : {{ selectedUser.name }}</p>
  `
})
export class UsersPageComponent {
  users: User[] = [
    { id: 1, name: 'Ada Lovelace', role: 'admin' },
    { id: 2, name: 'Alan Turing', role: 'user' }
  ];

  selectedUser?: User;

  onSelected(u: User) {
    this.selectedUser = u;
  }
}
```

### Points de validation
- Les données circulent via `@Input`.
- Les actions remontent via `@Output`.
- Les composants sont testables et réutilisables.

---

## Synthèse
- Un composant Angular associe **template HTML**, **styles** et **logique TypeScript** via `@Component`.
- La maîtrise du **binding**, de la **communication** parent/enfant et des **hooks** structure une application robuste.
- L’optimisation passe par `OnPush`, `trackBy`, et une architecture « container/presentation ».

---

## Annexes — Checklist rapide
- [ ] Le composant est-il réutilisable ?
- [ ] Inputs/Outputs bien typés ?
- [ ] `OnPush` pertinent ?
- [ ] `trackBy` sur les listes ?
- [ ] Cleanup dans `ngOnDestroy` ?
- [ ] Tests unitaires de base ?
