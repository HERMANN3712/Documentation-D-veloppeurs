# Formation Angular — ViewChild, ContentChild et références de template

> Public : développeurs Angular (intermédiaire → avancé)  
> Durée indicative : 3h30 à 1 journée (selon profondeur des ateliers)  
> Prérequis : composants, templates, directives, modules/standalone, RxJS de base, cycle de vie Angular

---

## Objectifs pédagogiques

À l’issue de la formation, vous saurez :

- Expliquer la différence entre **références de template**, **@ViewChild/@ViewChildren** et **@ContentChild/@ContentChildren**.
- Accéder correctement à un élément DOM, une directive ou un composant enfant en respectant le **cycle de vie**.
- Choisir le bon outil (Inputs/Outputs, services, signaux, template refs, queries) en évitant le **couplage excessif**.
- Mettre en œuvre des usages avancés : contrôle fin d’un composant enfant, intégration de bibliothèques tierces, patterns pour requêtes dynamiques.
- Identifier les pièges : `static`, changements conditionnels (`*ngIf`), projection (`ng-content`), tests, SSR/hydration.

---

## Plan (structure du cours)

1. **Introduction : pourquoi des queries ?**
2. **Références de template (`#ref`)**
3. **@ViewChild / @ViewChildren (vue du composant)**
4. **@ContentChild / @ContentChildren (contenu projeté)**
5. **Cycle de vie et option `static`**
6. **Cas d’usage recommandés (et anti-patterns)**
7. **Exemples avancés et patterns robustes**
8. **Ateliers pratiques**
9. **Checklist + récapitulatif**

---

## 1) Introduction : pourquoi des queries ?

Dans Angular, l’approche standard pour orchestrer des composants est :

- **@Input / @Output** (données et événements)
- services partagés (avec DI)
- routing / state management

Pourtant, il arrive qu’on doive **accéder directement à un élément ou composant “en dessous”** (enfant), par exemple :

- intégrer une librairie JS qui veut un **élément DOM** (éditeur WYSIWYG, chart, map…)
- déclencher une méthode d’un composant enfant (ex : `open()`, `focus()`, `reset()`)
- lire un état d’une directive (ex : `MatMenuTrigger`, `CdkScrollable`, etc.)

C’est là que les **queries** interviennent :

- `@ViewChild` / `@ViewChildren` : accès à la **vue** (template du composant)
- `@ContentChild` / `@ContentChildren` : accès au **contenu projeté** via `ng-content`
- Les **références de template** `#ref` : un identifiant local dans le template

> Idée clé : ces mécanismes sont puissants mais peuvent créer un **couplage fort** et un code fragile si on ne respecte pas le cycle de vie et les changements du DOM.

---

## 2) Références de template (`#ref`) : la base

### 2.1 Définition
Une **référence de template** est un identifiant local déclaré dans le HTML :

```html
<input #searchInput type="text" />
<button (click)="onSearch(searchInput.value)">Rechercher</button>
```

- Portée : **le template** (et ses enfants)
- Usage typique : lire une valeur, passer une référence à un handler, lier à une directive exportée.

### 2.2 Référence d’élément HTML
Sans précision, `#ref` pointe vers l’élément (en réalité un `ElementRef` ou API équivalente selon binding).

Exemple :

```html
<input #nameInput />
<p>Valeur : {{ nameInput.value }}</p>
```

### 2.3 Référence d’une directive via `exportAs`
De nombreuses directives Angular Material/CDK exposent une API via `exportAs`.

```html
<button mat-button [matMenuTriggerFor]="menu" #trigger="matMenuTrigger">
  Menu
</button>
<mat-menu #menu="matMenu"> ... </mat-menu>

<button (click)="trigger.openMenu()">Ouvrir</button>
```

Ici `trigger` référence l’instance de `MatMenuTrigger`, pas l’élément DOM.

### 2.4 Quand préférer `#ref` à @ViewChild ?

- Si la valeur est utilisée **uniquement dans le template**.
- Si vous n’avez pas besoin d’accéder depuis la classe TypeScript.

> Bon réflexe : commencez par une solution *template-only* si possible.

---

## 3) @ViewChild et @ViewChildren : interroger la vue du composant

### 3.1 Définition
`@ViewChild()` permet de récupérer une référence à :

- un **composant enfant**
- une **directive**
- un **élément DOM** (via `ElementRef`)
- un **TemplateRef** / `ViewContainerRef`

À l’échelle de la **vue** du composant (son template).

### 3.2 Exemple : récupérer un composant enfant

**Enfant :**

```ts
@Component({
  selector: 'app-counter',
  template: `
    <p>Compteur: {{ value }}</p>
    <button (click)="inc()">+</button>
  `,
  standalone: true
})
export class CounterComponent {
  value = 0;
  inc() { this.value++; }
  reset() { this.value = 0; }
}
```

**Parent :**

```ts
@Component({
  selector: 'app-parent',
  template: `
    <app-counter />
    <button (click)="resetChild()">Reset enfant</button>
  `,
  imports: [CounterComponent],
  standalone: true
})
export class ParentComponent {
  @ViewChild(CounterComponent) counter?: CounterComponent;

  resetChild() {
    this.counter?.reset();
  }
}
```

Notes :
- `CounterComponent` est typé ; très pratique.
- `counter` devient disponible **après** l’initialisation de la vue (cf. cycle de vie).

### 3.3 Exemple : récupérer un élément DOM (ElementRef)

```html
<input #inputEl type="text" />
<button (click)="focus()">Focus</button>
```

```ts
export class ParentComponent {
  @ViewChild('inputEl') input?: ElementRef<HTMLInputElement>;

  focus() {
    this.input?.nativeElement.focus();
  }
}
```

**Attention sécurité/SSR :** manipuler le DOM directement est parfois incompatible SSR/hydration. 
Préférez `Renderer2` ou CDK quand c’est possible.

### 3.4 @ViewChildren : QueryList et collections

```html
<app-counter />
<app-counter />
<app-counter />
<button (click)="resetAll()">Reset tous</button>
```

```ts
import { QueryList, ViewChildren } from '@angular/core';

export class ParentComponent {
  @ViewChildren(CounterComponent) counters!: QueryList<CounterComponent>;

  resetAll() {
    this.counters.forEach(c => c.reset());
  }
}
```

`QueryList` est une liste **vivante** qui se met à jour quand la vue change (ex. `*ngIf`, `*ngFor`).

---

## 4) @ContentChild et @ContentChildren : interroger du contenu projeté

### 4.1 Projection : rappel

```html
<app-card>
  <h2>Mon titre</h2>
  <p>Mon contenu</p>
</app-card>
```

Dans `app-card`, on a :

```html
<div class="card">
  <ng-content></ng-content>
</div>
```

Ici, le DOM “à l’intérieur” de `<app-card>` n’appartient pas à la **vue** de `app-card`: c’est du **contenu projeté**.

### 4.2 Exemple : récupérer une directive projetée

Supposons une directive marker :

```ts
@Directive({
  selector: '[appCardTitle]',
  standalone: true
})
export class CardTitleDirective {}
```

Le composant carte :

```ts
@Component({
  selector: 'app-card',
  template: `
    <div class="card">
      <div class="header">
        <ng-content select="[appCardTitle]"></ng-content>
      </div>
      <div class="body">
        <ng-content></ng-content>
      </div>
    </div>
  `,
  standalone: true
})
export class CardComponent {
  @ContentChild(CardTitleDirective) title?: CardTitleDirective;
}
```

Usage :

```html
<app-card>
  <h2 appCardTitle>Titre projeté</h2>
  <p>Contenu projeté…</p>
</app-card>
```

`@ContentChild` permet à `CardComponent` de détecter si un titre projected existe.

### 4.3 Exemple : récupérer un TemplateRef projeté
Pattern classique : laisser l’utilisateur fournir un template.

```html
<app-table>
  <ng-template #row let-item>
    <strong>{{ item.name }}</strong>
  </ng-template>
</app-table>
```

Côté composant :

```ts
@Component({
  selector: 'app-table',
  template: `
    <div *ngFor="let item of data">
      <ng-container
        *ngTemplateOutlet="rowTpl; context: { $implicit: item }">
      </ng-container>
    </div>
  `,
  standalone: true
})
export class TableComponent {
  @ContentChild('row', { read: TemplateRef }) rowTpl!: TemplateRef<unknown>;

  data = [{ name: 'A' }, { name: 'B' }];
}
```

---

## 5) Cycle de vie et option `static` (critique en pratique)

### 5.1 Hooks à connaître

- `ngOnInit()` : inputs init ; la vue n’est pas forcément prête.
- `ngAfterContentInit()` : contenu projeté initialisé → queries **ContentChild** disponibles.
- `ngAfterViewInit()` : vue initialisée → queries **ViewChild** disponibles.
- `ngAfterContentChecked()` / `ngAfterViewChecked()` : appelé à chaque cycle de détection (à éviter pour logique lourde).

**Règle simple :**

- `@ViewChild` → utilisez la valeur en général dans `ngAfterViewInit`.
- `@ContentChild` → utilisez la valeur en général dans `ngAfterContentInit`.

### 5.2 L’option `static`

```ts
@ViewChild('inputEl', { static: true }) input!: ElementRef<HTMLInputElement>;
```

- `static: true` : disponible dès `ngOnInit`.
- `static: false` (par défaut) : disponible en `ngAfterViewInit`.

Quand utiliser `static: true` ?

- Quand l’élément est **toujours présent** dans le template (pas dans un `*ngIf`, `*ngFor`, etc).
- Quand on en a besoin **avant** `ngAfterViewInit` (rare).

**Piège fréquent :**

```html
<input *ngIf="show" #inputEl />
```

Ici `static: true` est faux : l’élément n’est pas garanti au moment des queries statiques.

### 5.3 Queries dynamiques : prise en compte des changements
Si la vue change (toggle `*ngIf`, mise à jour `*ngFor`), `@ViewChild` peut passer de `undefined` → défini (ou inverse). 
Pour `@ViewChildren`, la `QueryList` émet des changements :

```ts
@ViewChildren(CounterComponent) counters!: QueryList<CounterComponent>;

ngAfterViewInit() {
  this.counters.changes.subscribe(list => {
    console.log('nouvelle liste', list.toArray());
  });
}
```

Pensez désabonnement (ou utilisez `takeUntilDestroyed`).

---

## 6) Cas d’usage recommandés (et anti-patterns)

### 6.1 Cas d’usage recommandés

1. **Intégration de bibliothèques tierces DOM**
   - ex : initialiser un carousel, un diagramme, une map
   - nécessite souvent un `HTMLElement`

2. **Contrôle fin d’un composant enfant**
   - ex : `open()`, `close()`, `focusFirstInvalid()`
   - utile pour composants UI techniques (modales, accordéons, champs)

3. **Interop avec directives du framework/ecosystème**
   - Angular Material, CDK, directives personnalisées

4. **Templates projetés (ContentChild)**
   - ex : API de composants “framework interne” (table, card, modal)

### 6.2 Anti-patterns (à éviter)

- Remplacer systématiquement `@Input`/`@Output` par `ViewChild`.
  - vous créez un composant parent qui “pilote” l’enfant au lieu d’un contrat de composants.

- Naviguer le DOM (ex : `nativeElement.querySelector(...)`) de manière extensive.
  - fragile, non typé, casse lors des refactors, moins compatible SSR.

- Déclencher `detectChanges()` en boucle via `AfterViewChecked`.
  - souvent signe d’un problème de conception.

- Exposer des détails internes d’un composant enfant.
  - mieux : concevoir une API publique claire.

---

## 7) Exemples avancés et patterns robustes

### 7.1 Pattern : wrapper d’une librairie tierce (DOM)

But : intégrer une librairie qui attend un élément container.

```html
<div #host class="chart-host"></div>
```

```ts
import { AfterViewInit, Component, ElementRef, OnDestroy, ViewChild } from '@angular/core';

@Component({
  selector: 'app-chart-wrapper',
  templateUrl: './chart-wrapper.component.html',
  standalone: true
})
export class ChartWrapperComponent implements AfterViewInit, OnDestroy {
  @ViewChild('host', { static: false }) host!: ElementRef<HTMLElement>;
  private chart: any;

  ngAfterViewInit() {
    // pseudo-code : initialisation librairie
    this.chart = createChart(this.host.nativeElement, { /* options */ });
  }

  ngOnDestroy() {
    this.chart?.destroy?.();
  }
}
```

Bonnes pratiques :
- init dans `ngAfterViewInit`
- cleanup dans `ngOnDestroy`
- éviter de stocker l’élément ailleurs ; encapsuler.

### 7.2 Pattern : composant enfant contrôlable mais peu couplé
Au lieu d’appeler une méthode privée, exposez une API stable :

```ts
export class ModalComponent {
  open() {}
  close() {}
}

export class HostComponent {
  @ViewChild(ModalComponent) modal?: ModalComponent;
  openModal() { this.modal?.open(); }
}
```

Alternative souvent meilleure : déclencher via un `@Input()`/signal (state-driven) :

```html
<app-modal [open]="isOpen" (openChange)="isOpen = $event"></app-modal>
```

`ViewChild` reste pertinent si :
- vous devez faire un `focus()` immédiat
- vous avez une séquence imperative (ex : `open()` puis `focus()`)

### 7.3 `read` : lire un token différent
Vous pouvez demander une autre “lecture” (ex : obtenir `ElementRef` plutôt que composant).

```ts
@ViewChild(CounterComponent, { read: ElementRef })
el?: ElementRef<HTMLElement>;
```

Cas : vous avez besoin du host element du composant.

### 7.4 Gérer `*ngIf` et les queries

```html
<app-counter *ngIf="show" />
<button (click)="show = !show">toggle</button>
```

```ts
export class HostComponent {
  show = true;
  @ViewChild(CounterComponent) counter?: CounterComponent;

  ngAfterViewInit() {
    // peut être défini ou non selon show
  }
}
```

Si vous avez besoin de réagir aux apparitions/disparitions, préférez `@ViewChildren` + `changes`, ou un setter :

```ts
private _counter?: CounterComponent;
@ViewChild(CounterComponent)
set counter(c: CounterComponent | undefined) {
  this._counter = c;
  if (c) {
    // vient d’apparaître : initialisation
  }
}
```

### 7.5 ContentChild et API “slot” (design system)
Créer des slots via directives : `appModalTitle`, `appModalActions`, etc.

```html
<app-modal>
  <h2 appModalTitle>Supprimer</h2>
  <div appModalActions>
    <button>Annuler</button>
    <button>Confirmer</button>
  </div>
</app-modal>
```

Dans le composant :

```ts
@ContentChild(ModalTitleDirective) title?: ModalTitleDirective;
@ContentChild(ModalActionsDirective) actions?: ModalActionsDirective;
```

Usage : activer/désactiver des sections selon présence.

---

## 8) Ateliers pratiques (avec énoncés)

### Atelier 1 — Focus automatique contrôlé (ViewChild)
**But :** au clic sur “Éditer”, afficher un champ et le focus.

Contraintes :
- le champ est dans un `*ngIf`
- ne pas faire de `setTimeout` arbitraire

Pistes :
- `@ViewChild` setter ou `@ViewChildren` + `changes`
- focus dans le bon moment du cycle

### Atelier 2 — Wrapper d’une librairie DOM
**But :** intégrer une librairie qui attend un container : init `ngAfterViewInit`, destroy `ngOnDestroy`.

Livrable :
- composant `ChartWrapperComponent`
- une méthode `setData(data)` (ou input) qui met à jour le chart sans réinstancier.

### Atelier 3 — Composant `Card` avec “slot” titre (ContentChild)
**But :**
- si un titre est fourni via `[appCardTitle]`, afficher un header
- sinon, ne pas rendre la zone header

### Atelier 4 — QueryList et listes dynamiques
**But :** gérer un `*ngFor` de composants enfant et appeler `reset()` sur tous.

Bonus :
- écouter `changes` et logger le nombre d’enfants.

---

## 9) Checklist + récapitulatif

### 9.1 Choisir le bon outil

- Besoin uniquement dans le template ? → `#ref`
- Besoin de contrôler un enfant dans le template du parent ? → `@ViewChild`
- Besoin de lire du contenu projeté ? → `@ContentChild`
- Besoin de gérer une collection dynamique ? → `@ViewChildren/@ContentChildren` + `QueryList`

### 9.2 Règles de cycle de vie

- `ViewChild` : fiable à partir de `ngAfterViewInit`
- `ContentChild` : fiable à partir de `ngAfterContentInit`
- `static: true` seulement si l’élément est **toujours** présent

### 9.3 Bonnes pratiques

- Éviter la logique métier dans les hooks `After*Checked`
- Limiter les accès DOM directs ; préférer CDK/Renderer2
- Concevoir une API enfant stable pour réduire le couplage
- Nettoyer les abonnements (`changes`) et les ressources tierces (`destroy`)

---

## Annexes — Aide-mémoire (syntaxe)

### Template ref
```html
<input #i />
<button (click)="save(i.value)">OK</button>
```

### ViewChild
```ts
@ViewChild('i') i?: ElementRef<HTMLInputElement>;
@ViewChild(ChildComponent) child?: ChildComponent;
```

### ViewChildren
```ts
@ViewChildren(ChildComponent) children!: QueryList<ChildComponent>;
```

### ContentChild
```ts
@ContentChild('row', { read: TemplateRef }) rowTpl?: TemplateRef<any>;
@ContentChild(SomeDirective) projected?: SomeDirective;
```

---

### Fin de formation — Questions de validation

1. Différence entre `ViewChild` et `ContentChild` ?
2. Quand utiliser `static: true` ?
3. Pourquoi `ngAfterViewInit` est le hook typique de `ViewChild` ?
4. Comment gérer une liste dynamique d’enfants et détecter les changements ?
5. Citer 2 cas d’usage légitimes et 2 anti-patterns.
