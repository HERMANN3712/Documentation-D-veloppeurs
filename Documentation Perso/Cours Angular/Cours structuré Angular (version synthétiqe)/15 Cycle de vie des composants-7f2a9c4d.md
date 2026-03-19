# Formation Angular — Cycle de vie des composants

## Informations générales
- **Public visé** : développeurs Angular (débutants à intermédiaires)
- **Pré‑requis** : TypeScript, notions de composants Angular, binding, @Input/@Output, services.
- **Durée indicative** : 2h30 à 4h (selon exercices)
- **Objectifs pédagogiques**
  - Comprendre le **cycle de vie** d’un composant Angular.
  - Savoir quand et pourquoi utiliser les hooks : `ngOnChanges`, `ngOnInit`, `ngDoCheck`, `ngAfterContentInit`, `ngAfterContentChecked`, `ngAfterViewInit`, `ngAfterViewChecked`, `ngOnDestroy`.
  - Éviter les erreurs fréquentes (double exécution, subscriptions non nettoyées, manipulations DOM trop tôt, etc.).
  - Mettre en œuvre des patterns robustes (unsubscribe, `takeUntilDestroyed`, `AsyncPipe`, détection de changements).

---

## Plan de la formation
1. **Introduction : pourquoi un cycle de vie ?**
2. **Vue globale des hooks Angular**
3. **`ngOnInit` : initialisation**
4. **`ngOnChanges` : réaction aux changements d’`@Input()`**
5. **`ngDoCheck` : détection de changements personnalisée**
6. **Hooks de contenu projeté : `AfterContent*`**
7. **Hooks de vue : `AfterView*` et `@ViewChild`**
8. **`ngOnDestroy` : nettoyage et prévention des fuites mémoire**
9. **Bonnes pratiques et pièges courants**
10. **Atelier guidé (exercices)**
11. **Checklist récapitulative**

---

## 1) Introduction : pourquoi un cycle de vie ?

Un composant Angular n’existe pas « d’un coup ». Il :
- est **instancié** (constructeur),
- reçoit potentiellement des **inputs**,
- est **initialisé**,
- s’affiche et se met à jour en fonction des changements (data, events, async),
- puis est **détruit**.

Angular expose des **hooks** (méthodes) que vous pouvez implémenter pour exécuter du code **à des moments clés**.

> Idée centrale : **le bon code au bon moment** (et uniquement là).

---

## 2) Vue globale des hooks Angular

### Ordre typique (simplifié)
1. `constructor()`
2. `ngOnChanges()` (si le composant a des `@Input()` et qu’au moins un a une valeur)
3. `ngOnInit()`
4. `ngDoCheck()`
5. `ngAfterContentInit()`
6. `ngAfterContentChecked()`
7. `ngAfterViewInit()`
8. `ngAfterViewChecked()`
9. (boucle) `ngOnChanges()` / `ngDoCheck()` / `ngAfterContentChecked()` / `ngAfterViewChecked()` à chaque cycle de détection
10. `ngOnDestroy()`

### Table de synthèse
| Hook | Quand ? | Usage principal |
|------|---------|----------------|
| `ngOnChanges(changes)` | à l’arrivée/modif des `@Input()` | réagir aux entrées, recalcul dérivé |
| `ngOnInit()` | 1 fois après le 1er `ngOnChanges` | init (fetch, config, subscriptions) |
| `ngDoCheck()` | à chaque détection | contrôle fin / collections mutables |
| `ngAfterContentInit()` | 1 fois après projection de contenu | accès au contenu projeté (`ng-content`) |
| `ngAfterContentChecked()` | après chaque check du contenu | validation, réactions (avec prudence) |
| `ngAfterViewInit()` | 1 fois après init de la vue | accès `@ViewChild`, DOM, libs UI |
| `ngAfterViewChecked()` | après chaque check de la vue | rarement, diagnostics/perf |
| `ngOnDestroy()` | avant destruction | cleanup: unsubscribe, timers, listeners |

---

## 3) `ngOnInit` : initialisation

### À retenir
- Exécuté **une seule fois**.
- Vous pouvez considérer `ngOnInit` comme « le vrai point d’entrée » du composant.
- Le constructeur doit rester léger : **injection** + initialisation de champs simples.

### Exemple
```ts
import { Component, OnInit } from '@angular/core';

@Component({
  selector: 'app-user-list',
  template: `...`
})
export class UserListComponent implements OnInit {
  users: string[] = [];

  constructor(/* injection */) {
    // Pas de logique métier lourde ici
  }

  ngOnInit(): void {
    // Initialisation : appels HTTP, subscriptions, configuration
    this.users = ['Ada', 'Grace', 'Linus'];
  }
}
```

### Cas d’usage recommandés
- Déclencher un **chargement initial** (HTTP via service).
- Monter une **subscription** (mais prévoir un nettoyage — voir `ngOnDestroy`).
- Initialiser des données dérivées **après** que les inputs existent.

---

## 4) `ngOnChanges` : réaction aux changements d’`@Input()`

### À retenir
- Appelé **à chaque changement** d’un `@Input()` (y compris à l’initialisation).
- Donne accès à `SimpleChanges` : ancienne valeur, nouvelle valeur, `firstChange`.

### Exemple
```ts
import { Component, Input, OnChanges, SimpleChanges } from '@angular/core';

@Component({
  selector: 'app-user-card',
  template: `
    <h3>{{ fullName }}</h3>
  `
})
export class UserCardComponent implements OnChanges {
  @Input() firstName = '';
  @Input() lastName = '';

  fullName = '';

  ngOnChanges(changes: SimpleChanges): void {
    // Recalculer une donnée dérivée à partir des inputs
    this.fullName = `${this.firstName} ${this.lastName}`.trim();

    if (changes['firstName']?.firstChange) {
      // logique spécifique à la première affectation
    }
  }
}
```

### Bonnes pratiques
- Utiliser `ngOnChanges` quand une propriété **dépend** des inputs.
- Éviter d’y déclencher des opérations coûteuses à chaque changement sans garde (debounce/memoization).

### Piège courant : mutation vs remplacement
`ngOnChanges` se déclenche sur **changement de référence**.
- Si un parent **mute** un objet (`obj.x = 2`) sans remplacer la référence, le hook peut ne pas être déclenché (selon contexte).
- Préférer des mises à jour immuables : `this.obj = { ...this.obj, x: 2 }`.

---

## 5) `ngDoCheck` : détection de changements personnalisée

### À retenir
- Appelé très fréquemment (à chaque cycle de change detection).
- Permet de gérer des cas où `ngOnChanges` ne suffit pas (ex. collections mutées).

### Exemple : détecter une mutation dans un tableau
```ts
import { Component, DoCheck, Input } from '@angular/core';

@Component({
  selector: 'app-tags',
  template: `
    <p>Nombre de tags: {{tagCount}}</p>
  `
})
export class TagsComponent implements DoCheck {
  @Input() tags: string[] = [];

  tagCount = 0;
  private previousLength = 0;

  ngDoCheck(): void {
    if (this.tags.length !== this.previousLength) {
      this.previousLength = this.tags.length;
      this.tagCount = this.tags.length;
    }
  }
}
```

### Attention performance
- `ngDoCheck` peut vite devenir coûteux.
- Garder la logique **minimaliste**.
- Préférer l’approche immuable + `ngOnChanges` quand possible.

---

## 6) Hooks de contenu projeté : `AfterContent*`

Quand vous utilisez la projection via `ng-content`, Angular distingue :
- **contenu projeté** (Content) : fourni par le parent,
- **vue du composant** (View) : template du composant.

### `ngAfterContentInit()`
- Appelé **une fois** après que le contenu projeté a été initialisé.

### `ngAfterContentChecked()`
- Appelé après **chaque vérification** du contenu projeté.

### Exemple (conceptuel)
**Composant wrapper**
```html
<!-- wrapper.component.html -->
<div class="panel">
  <ng-content></ng-content>
</div>
```

Si vous devez interagir avec des éléments projetés (rare, mais possible via `@ContentChild` / `@ContentChildren`), cela se fait typiquement dans `AfterContentInit`.

---

## 7) Hooks de vue : `AfterView*` et `@ViewChild`

### `ngAfterViewInit()`
- Appelé **une fois** lorsque la vue (template) et les `@ViewChild` sont prêts.
- Idéal pour : initialiser une lib UI nécessitant la présence du DOM, mesurer un élément, etc.

### Exemple avec `@ViewChild`
```ts
import { AfterViewInit, Component, ElementRef, ViewChild } from '@angular/core';

@Component({
  selector: 'app-measure',
  template: `
    <h2 #title>Mesure</h2>
  `
})
export class MeasureComponent implements AfterViewInit {
  @ViewChild('title') titleEl!: ElementRef<HTMLHeadingElement>;

  ngAfterViewInit(): void {
    const width = this.titleEl.nativeElement.getBoundingClientRect().width;
    // Utiliser la mesure, déclencher une animation, etc.
    console.log('Title width', width);
  }
}
```

### `ngAfterViewChecked()`
- Appelé après **chaque** vérification de la vue.
- À éviter pour modifier des valeurs boundées : risque de `ExpressionChangedAfterItHasBeenCheckedError`.

> Règle : si vous devez « corriger » une valeur après check, utilisez plutôt un flux (Observable), `ChangeDetectorRef`, ou repensez la source de vérité.

---

## 8) `ngOnDestroy` : nettoyage et prévention des fuites mémoire

### À retenir
- Appelé juste avant que le composant soit retiré du DOM.
- Indispensable pour :
  - **unsubscribe** des Observables (si pas `AsyncPipe`),
  - annuler timers (`setInterval`, `setTimeout`),
  - retirer event listeners manuels.

### Exemple : pattern `takeUntilDestroyed` (Angular récent)
```ts
import { Component, DestroyRef, inject, OnInit } from '@angular/core';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';
import { interval } from 'rxjs';

@Component({
  selector: 'app-ticker',
  template: `...`
})
export class TickerComponent implements OnInit {
  private destroyRef = inject(DestroyRef);

  ngOnInit(): void {
    interval(1000)
      .pipe(takeUntilDestroyed(this.destroyRef))
      .subscribe(v => console.log('tick', v));
  }
}
```

### Exemple : pattern classique `Subject` + `takeUntil`
```ts
import { Component, OnDestroy, OnInit } from '@angular/core';
import { Subject, interval, takeUntil } from 'rxjs';

@Component({
  selector: 'app-ticker',
  template: `...`
})
export class TickerComponent implements OnInit, OnDestroy {
  private destroyed$ = new Subject<void>();

  ngOnInit(): void {
    interval(1000)
      .pipe(takeUntil(this.destroyed$))
      .subscribe(v => console.log('tick', v));
  }

  ngOnDestroy(): void {
    this.destroyed$.next();
    this.destroyed$.complete();
  }
}
```

### Alternative recommandée : `AsyncPipe`
Quand possible, éviter de s’abonner manuellement.
```html
<p>{{ data$ | async }}</p>
```

---

## 9) Bonnes pratiques et pièges courants

### 9.1 Constructeur vs `ngOnInit`
- **Constructeur** : injection + initialisation simple.
- **`ngOnInit`** : logique d’initialisation (HTTP, subscriptions, calculs).

### 9.2 Ne pas modifier des bindings dans `AfterViewChecked`
Risque d’erreur et de boucles de change detection.

### 9.3 Inputs : préférer immutabilité
Facilite la détection de changements et la performance (surtout avec `ChangeDetectionStrategy.OnPush`).

### 9.4 Cleanup systématique
- Subscriptions manuelles → cleanup (`ngOnDestroy` / `takeUntilDestroyed`).
- Timers → `clearInterval/clearTimeout`.
- Listeners → `removeEventListener`.

### 9.5 Diagnostic : comprendre “qui relance la détection”
Événements, XHR/HTTP, timers, Observables, input changes…

---

## 10) Atelier guidé (exercices)

### Exercice 1 — Observer l’ordre d’exécution
1. Créez un composant `LifecycleDemoComponent`.
2. Implémentez plusieurs hooks et loggez-les.

```ts
import {
  AfterContentChecked,
  AfterContentInit,
  AfterViewChecked,
  AfterViewInit,
  Component,
  DoCheck,
  Input,
  OnChanges,
  OnDestroy,
  OnInit,
  SimpleChanges,
} from '@angular/core';

@Component({
  selector: 'app-lifecycle-demo',
  template: `
    <p>Demo lifecycle: {{value}}</p>
  `
})
export class LifecycleDemoComponent
  implements
    OnChanges,
    OnInit,
    DoCheck,
    AfterContentInit,
    AfterContentChecked,
    AfterViewInit,
    AfterViewChecked,
    OnDestroy {

  @Input() value = 0;

  constructor() {
    console.log('constructor');
  }

  ngOnChanges(changes: SimpleChanges): void {
    console.log('ngOnChanges', changes);
  }

  ngOnInit(): void {
    console.log('ngOnInit');
  }

  ngDoCheck(): void {
    console.log('ngDoCheck');
  }

  ngAfterContentInit(): void {
    console.log('ngAfterContentInit');
  }

  ngAfterContentChecked(): void {
    console.log('ngAfterContentChecked');
  }

  ngAfterViewInit(): void {
    console.log('ngAfterViewInit');
  }

  ngAfterViewChecked(): void {
    console.log('ngAfterViewChecked');
  }

  ngOnDestroy(): void {
    console.log('ngOnDestroy');
  }
}
```

**Questions**
- Quels logs apparaissent au chargement ?
- Quels logs apparaissent quand le parent modifie `value` ?
- Quels logs apparaissent quand le composant est retiré (ex. `*ngIf`) ?

### Exercice 2 — `@ViewChild` et timing
- Accéder à un élément avec `@ViewChild` et constater que la référence n’est prête qu’en `ngAfterViewInit`.

### Exercice 3 — Nettoyage d’une subscription
- Implémenter un `interval` qui logge toutes les secondes.
- Vérifier qu’en naviguant / masquant le composant, le log s’arrête.

---

## 11) Checklist récapitulative

- [ ] Logique lourde hors constructeur.
- [ ] `ngOnInit` pour initialiser.
- [ ] `ngOnChanges` pour réagir aux `@Input()`.
- [ ] `ngAfterViewInit` pour `@ViewChild`/DOM.
- [ ] `ngDoCheck` seulement si nécessaire (perf!).
- [ ] `ngOnDestroy` pour cleanup.
- [ ] Favoriser `AsyncPipe` / `takeUntilDestroyed` pour éviter les fuites.

---

## Annexes — Mini mémo

### Squelette minimal
```ts
import { Component, OnInit, OnDestroy, Input, OnChanges, SimpleChanges } from '@angular/core';

@Component({
  selector: 'app-sample',
  template: `...`
})
export class SampleComponent implements OnInit, OnDestroy, OnChanges {
  @Input() data: unknown;

  ngOnChanges(changes: SimpleChanges): void {
    // quand data change
  }

  ngOnInit(): void {
    // init
  }

  ngOnDestroy(): void {
    // cleanup
  }
}
```

### Raccourci mental
- **Init** → `ngOnInit`
- **Inputs** → `ngOnChanges`
- **DOM / ViewChild** → `ngAfterViewInit`
- **Cleanup** → `ngOnDestroy`
