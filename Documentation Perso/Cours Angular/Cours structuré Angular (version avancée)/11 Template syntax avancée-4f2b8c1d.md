# Formation Angular – Template syntax avancée

- **Référence** : 11
- **Thème** : Template syntax avancée
- **Public** : Développeurs Angular (intermédiaire → avancé)
- **Prérequis** :
  - Bases Angular (components, modules/standalone, binding, services)
  - Notions de RxJS (observables, async pipe) appréciées
  - Connaissances TypeScript (types, fonctions, classes)
- **Durée indicative** : 1 journée (6–7h) ou 2 demi-journées

---

## Objectifs pédagogiques

À l’issue de la formation, le participant sera capable de :

1. Maîtriser les **structural directives** et leurs variantes modernes (nouveaux *control flow blocks*).
2. Utiliser les **pipes** (built-in, custom, impures) en comprenant les impacts perf.
3. Exploiter les **template reference variables** et les APIs associées (`@ViewChild`, `exportAs`).
4. Mettre en place des **bindings conditionnels** robustes (classes, styles, attributs, propriétés).
5. Construire des **templates dynamiques** (`ng-template`, `ng-container`, `ngTemplateOutlet`, `ngComponentOutlet`).
6. Éviter les expressions coûteuses dans le template et déplacer la logique côté composant.
7. Tirer parti des **computed signals** et valeurs préparées pour des templates performants.

---

## Plan de la formation

1. **Rappels & cadre de performance du template**
2. **Structural directives avancées**
   - `*ngIf` / `@if`, `*ngFor` / `@for`, `*ngSwitch` / `@switch`
   - `trackBy`, alias, `as`, `let`, context
   - Composition et directives personnalisées
3. **Pipes avancés**
   - `async`, `date`, `currency`, `i18n` patterns
   - Custom pipes : pure vs impure
   - Caching, memoization, alternatives (computed signals)
4. **Template reference variables & interaction template/composant**
   - `#ref`, `exportAs`, accès DOM/Directive
   - `@ViewChild`, `@ViewChildren`, timing, `static`
5. **Bindings conditionnels avancés**
   - `[class]`, `[class.foo]`, `[ngClass]`
   - `[style]`, `[style.width.px]`, `[ngStyle]`
   - `[attr.*]`, `[disabled]` vs `attr.disabled`
   - Patterns sûrs (nullish coalescing, safe navigation)
6. **Templates dynamiques**
   - `ng-template`, `ng-container`
   - `ngTemplateOutlet` + context
   - Content projection & `ng-content` (rappels utiles)
   - `ngComponentOutlet` + injection
7. **Fonctions dans le template : bonnes pratiques & anti-patterns**
   - Coûts de change detection
   - Pré-calcul côté composant
   - `computed()` (signals), `toSignal()`, `effect()` (orienté affichage)
   - Patterns de refactoring
8. **Atelier de synthèse** (mise en pratique + checklist)

---

## 1) Rappels & cadre de performance du template

### 1.1 Comment Angular évalue un template

- À chaque cycle de **change detection** (CD), Angular réévalue les expressions du template.
- Même avec `ChangeDetectionStrategy.OnPush`, certains événements déclenchent une vérification (inputs, events, async pipe, signals…).
- Conséquence : une expression innocente (ex : `items.filter(...).map(...)`) peut être exécutée **très souvent**.

### 1.2 Règles d’or

- **Éviter** dans le template :
  - Appels de fonctions non mémoïsés : `getSomething()`
  - Créations d’objets/array : `{...}`, `[]`, `new Date()`
  - Chaînes de transformations : `arr.filter().sort().map()`
- **Préparer** les valeurs :
  - Calculer côté composant (dans `ngOnInit`, via setters d’input, via RxJS, via signals)
  - Utiliser des `computed()` (signals) ou des pipes **purs** si approprié

### 1.3 Préparation "data → view"

- Objectif : fournir au template des données déjà prêtes, idéalement :
  - stables par référence (pas de nouvel objet à chaque CD)
  - simples à consommer (`vm.*`, `state.*`)

Exemple de pattern **ViewModel** en composant :

```ts
type UserVm = {
  fullName: string;
  isActive: boolean;
};

@Component({
  selector: 'app-user-card',
  templateUrl: './user-card.html',
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class UserCardComponent {
  @Input({ required: true }) user!: { firstName: string; lastName: string; active: boolean };

  // Préparation d’une valeur simple
  get vm(): UserVm {
    // ⚠️ Exemple pédagogique (getter recalculé) :
    // Préférer une préparation stable (signals/computed) si cela devient coûteux.
    return {
      fullName: `${this.user.firstName} ${this.user.lastName}`,
      isActive: this.user.active,
    };
  }
}
```

> Pour les cas non triviaux, on préférera `computed()` ou des flux RxJS transformés.

---

## 2) Structural directives avancées

### 2.1 `*ngIf` avancé : `else`, `then`, `as`

```html
<div *ngIf="user$ | async as user; else loading">
  <h2>{{ user.name }}</h2>
  <p *ngIf="user.isAdmin; else notAdmin">Administrateur</p>
  <ng-template #notAdmin>Utilisateur</ng-template>
</div>

<ng-template #loading>Chargement…</ng-template>
```

Points clés :

- `as user` évite de répéter `user$ | async`.
- Les branches `else`/`then` déplacent l’affichage conditionnel dans des templates réutilisables.

### 2.2 Nouveaux control flow blocks (Angular moderne)

Selon votre version d’Angular, vous pouvez utiliser : `@if`, `@for`, `@switch`.

```html
@if (user(); as u) {
  <h2>{{ u.name }}</h2>
} @else {
  <p>Chargement…</p>
}
```

Avec `@for` + `track` :

```html
@for (item of items(); track item.id) {
  <app-item [item]="item" />
} @empty {
  <p>Aucun élément</p>
}
```

> **Bon réflexe** : toujours définir un `track` / `trackBy` sur les listes non triviales.

### 2.3 `*ngFor` avancé : variables locales, `trackBy`

```html
<li *ngFor="let p of products; let i = index; let last = last; trackBy: trackById">
  {{ i + 1 }} - {{ p.name }} <span *ngIf="last">(dernier)</span>
</li>
```

```ts
trackById(index: number, item: { id: string }) {
  return item.id;
}
```

Pourquoi `trackBy` :

- Évite la recréation inutile de DOM quand la liste change.
- Réduit le risque de perte d’état local (focus, champs en cours de saisie, animations).

### 2.4 `*ngSwitch` / `@switch`

```html
<div [ngSwitch]="status">
  <p *ngSwitchCase="'draft'">Brouillon</p>
  <p *ngSwitchCase="'published'">Publié</p>
  <p *ngSwitchDefault>Inconnu</p>
</div>
```

### 2.5 Directives structurelles personnalisées

Une structural directive transforme la structure DOM via `TemplateRef` + `ViewContainerRef`.

Exemple : directive `*appUnless` (inverse de `*ngIf`) :

```ts
@Directive({ selector: '[appUnless]' })
export class UnlessDirective {
  private hasView = false;

  constructor(
    private tpl: TemplateRef<unknown>,
    private vcr: ViewContainerRef
  ) {}

  @Input() set appUnless(condition: boolean) {
    if (!condition && !this.hasView) {
      this.vcr.createEmbeddedView(this.tpl);
      this.hasView = true;
    } else if (condition && this.hasView) {
      this.vcr.clear();
      this.hasView = false;
    }
  }
}
```

Usage :

```html
<p *appUnless="isLoggedIn">Veuillez vous connecter</p>
```

---

## 3) Pipes avancés

### 3.1 Pipes built-in utiles en "avancé"

- `async` : souscription/désouscription auto + CD
- `date`, `currency`, `percent`, `decimal`
- `json` (debug)
- `slice`, `uppercase/lowercase/titlecase`

Exemple combinant `async` + `as` :

```html
<ng-container *ngIf="vm$ | async as vm">
  <h3>{{ vm.title }}</h3>
  <p>{{ vm.amount | currency: 'EUR' }}</p>
</ng-container>
```

### 3.2 Pipe custom : pure vs impure

- **Pipe pure** (par défaut) : recalculée seulement si la **référence** des entrées change.
- **Pipe impure** (`pure: false`) : recalculée à chaque CD → à éviter sauf besoin (ex : `Date.now()` relatif, collection mutable legacy…)

Exemple pipe pure :

```ts
@Pipe({ name: 'fullName' })
export class FullNamePipe implements PipeTransform {
  transform(user: { firstName: string; lastName: string } | null | undefined): string {
    if (!user) return '';
    return `${user.firstName} ${user.lastName}`;
  }
}
```

Usage :

```html
<p>{{ user | fullName }}</p>
```

### 3.3 Anti-pattern : transformer des listes dans le template

À éviter :

```html
<!-- ❌ coûteux, recrée des arrays et déclenche du travail à chaque CD -->
<li *ngFor="let u of users.filter(isActive).sort(sortByName)">
  {{ u.name }}
</li>
```

Préférer :

- Pré-calcul dans le composant avec un `computed()` (signals)
- ou pipeline RxJS (`map`) avant d’arriver à la vue

Exemple (signals) :

```ts
import { computed, signal } from '@angular/core';

type User = { id: string; name: string; active: boolean };

export class UsersComponent {
  readonly users = signal<User[]>([]);

  readonly activeUsersSorted = computed(() =>
    this.users()
      .filter(u => u.active)
      .toSorted((a, b) => a.name.localeCompare(b.name))
  );

  trackUser = (_: number, u: User) => u.id;
}
```

```html
@for (u of activeUsersSorted(); track u.id) {
  <li>{{ u.name }}</li>
}
```

---

## 4) Template reference variables & interaction template/composant

### 4.1 Références locales : `#ref`

- Déclarer : `#myInput`
- Pointer un élément DOM, une directive ou un composant

```html
<input #searchInput type="text" (input)="onInput(searchInput.value)" />
<button (click)="searchInput.focus()">Focus</button>
```

### 4.2 `exportAs` (référencer une directive)

Une directive peut exposer une API via `exportAs`.

```ts
@Directive({
  selector: '[appTooltip]',
  exportAs: 'appTooltip'
})
export class TooltipDirective {
  open() {}
  close() {}
}
```

```html
<button appTooltip #tt="appTooltip" (mouseenter)="tt.open()" (mouseleave)="tt.close()">
  Survole-moi
</button>
```

### 4.3 `@ViewChild` / `@ViewChildren`

```ts
export class SearchComponent {
  @ViewChild('searchInput', { static: true })
  searchInput!: ElementRef<HTMLInputElement>;

  ngOnInit() {
    this.searchInput.nativeElement.focus();
  }
}
```

Bonnes pratiques :

- Préférer manipulations DOM via directives / composants.
- `static: true` si utilisé dans `ngOnInit`, sinon `false` + `ngAfterViewInit`.

---

## 5) Bindings conditionnels avancés

### 5.1 Classes : `[class.*]`, `[class]`, `[ngClass]`

```html
<div
  [class.is-active]="isActive"
  [class.is-disabled]="disabled"
>
  ...
</div>
```

`[ngClass]` utile pour des cas plus complexes :

```html
<div [ngClass]="{ 'is-active': isActive, 'has-error': hasError }"></div>
```

> Éviter de construire l’objet inline si cela varie souvent et devient coûteux ; préférer une valeur préparée.

### 5.2 Styles : `[style.*]`, unités, `[ngStyle]`

```html
<div [style.width.px]="width" [style.opacity]="disabled ? 0.5 : 1"></div>
```

### 5.3 Attributs : `[attr.*]` vs propriétés

- Propriété DOM : `[disabled]="true"` (recommandé)
- Attribut HTML : `[attr.aria-label]="label"` (accessibilité)

```html
<button
  [disabled]="isSubmitting"
  [attr.aria-busy]="isSubmitting"
  [attr.aria-label]="buttonLabel"
>
  Enregistrer
</button>
```

### 5.4 Safe navigation & nullish coalescing

```html
<p>{{ user?.address?.city ?? 'Ville inconnue' }}</p>
```

---

## 6) Templates dynamiques

### 6.1 `ng-container` et `ng-template`

- `ng-container` : groupement sans élément réel dans le DOM
- `ng-template` : fragment de template instanciable

```html
<ng-container *ngIf="isReady; else skeleton">
  <app-dashboard />
</ng-container>

<ng-template #skeleton>
  <app-skeleton />
</ng-template>
```

### 6.2 `ngTemplateOutlet` + context

Permet d’injecter un template avec des variables.

```html
<ng-template #row let-item let-i="index">
  <div class="row">
    <span>#{{ i }}</span>
    <strong>{{ item.name }}</strong>
  </div>
</ng-template>

<ng-container
  *ngFor="let item of items; let i = index"
  [ngTemplateOutlet]="row"
  [ngTemplateOutletContext]="{ $implicit: item, index: i }"
></ng-container>
```

Rappels :

- `$implicit` correspond à `let-item`.
- Les propriétés nommées correspondent à `let-i="index"`.

### 6.3 Projection de contenu (utile en templates avancés)

```html
<!-- composant wrapper -->
<app-card>
  <h3 cardTitle>Titre</h3>
  <p>Contenu</p>
</app-card>
```

Dans `app-card` :

```html
<div class="card">
  <div class="card__title">
    <ng-content select="[cardTitle]"></ng-content>
  </div>
  <div class="card__body">
    <ng-content></ng-content>
  </div>
</div>
```

### 6.4 `ngComponentOutlet` (composants dynamiques)

```html
<ng-container *ngComponentOutlet="componentType; injector: componentInjector"></ng-container>
```

Cas d’usage :

- moteur de formulaires dynamique
- rendu de widgets selon configuration

Bonnes pratiques :

- maîtriser l’injection (tokens, providers)
- garder le contrat du composant dynamique stable (inputs/outputs)

---

## 7) Fonctions dans le template : bonnes pratiques & anti-patterns

### 7.1 Pourquoi c’est un problème

Chaque fois que le template est vérifié :

- `{{ compute() }}` est réévalué
- `[ngClass]="getClasses()"` renvoie potentiellement un nouvel objet → diff coûteux

Exemple problématique :

```html
<!-- ❌ -->
<p>{{ formatPrice(product.price) }}</p>
<div [ngClass]="computeClasses(product)"></div>
```

### 7.2 Solutions recommandées

#### A) Pré-calculer une propriété (stable)

```ts
export class ProductComponent {
  @Input({ required: true }) product!: { price: number; inStock: boolean };

  formattedPrice = '';
  classes: Record<string, boolean> = {};

  ngOnChanges() {
    this.formattedPrice = new Intl.NumberFormat('fr-FR', {
      style: 'currency',
      currency: 'EUR',
    }).format(this.product.price);

    this.classes = {
      'is-in-stock': this.product.inStock,
      'is-out': !this.product.inStock,
    };
  }
}
```

```html
<p>{{ formattedPrice }}</p>
<div [ngClass]="classes"></div>
```

#### B) Utiliser `computed()` (signals)

```ts
import { computed, input } from '@angular/core';

export class ProductSignalComponent {
  readonly product = input.required<{ price: number; inStock: boolean }>();

  readonly formattedPrice = computed(() =>
    new Intl.NumberFormat('fr-FR', {
      style: 'currency',
      currency: 'EUR',
    }).format(this.product().price)
  );

  readonly classes = computed(() => ({
    'is-in-stock': this.product().inStock,
    'is-out': !this.product().inStock,
  }));
}
```

```html
<p>{{ formattedPrice() }}</p>
<div [ngClass]="classes()"></div>
```

#### C) Pipes pures (si transformation réutilisable)

- À privilégier pour une transformation **pure**, réutilisable, testable.
- Éviter les pipes impures sauf nécessité.

### 7.3 Checklist "template performant"

- [ ] Aucune fonction appelée dans le template pour des calculs non triviaux
- [ ] Aucune création d’objet/array inline dans le template (ou justifiée)
- [ ] Listes : `trackBy` / `track` présent
- [ ] `async` pipe utilisé plutôt que `.subscribe()` manuel côté composant
- [ ] Valeurs préparées via `computed()`/RxJS/inputs setters + `OnPush`

---

## 8) Atelier de synthèse (proposé)

### 8.1 Énoncé

On fournit une page `UsersAdminComponent` qui affiche :

- un chargement conditionnel
- une liste filtrée/triée
- un badge de statut
- un template de ligne personnalisable

Problèmes initiaux :

- fonctions dans le template (`getFilteredUsers()`, `formatDate(...)`)
- `users.filter().sort()` dans le template
- pas de `trackBy`

### 8.2 Objectifs de refactoring

1. Ajouter `OnPush`
2. Déplacer la préparation des données dans :
   - `computed()` (signals) ou
   - transformations RxJS (`map`)
3. Remplacer rendu conditionnel par `@if`/`*ngIf` + `else`
4. Remplacer la personnalisation de ligne par `ngTemplateOutlet`

### 8.3 Critères de réussite

- Template lisible, déclaratif
- Pas de transformations de liste inline
- Pas d’appels de fonctions non mémoïsées
- `trackBy` correct

---

## Annexes

### A) Exemples de patterns succincts

#### A.1 `async as` + `ng-container`

```html
<ng-container *ngIf="data$ | async as data">
  <app-widget [data]="data" />
</ng-container>
```

#### A.2 Gestion des états (loading/error/empty)

```html
@if (error()) {
  <app-error [error]="error()" />
} @else if (loading()) {
  <app-spinner />
} @else if ((items()?.length ?? 0) === 0) {
  <p>Aucun résultat</p>
} @else {
  @for (it of items(); track it.id) {
    <app-item [item]="it" />
  }
}
```

### B) Références (à adapter à votre stack)

- Documentation Angular : Templates, Control Flow, Pipes, Signals
- Guides perf : Change Detection, OnPush, trackBy

---

## Livrables

- Support markdown (ce document)
- Exemples de snippets prêts à intégrer
- Checklist perf template
