# Formation Angular (Avancé) — Cycle de vie des composants en profondeur

> Public : développeurs Angular (intermédiaire → avancé) souhaitant maîtriser précisément l’ordre d’exécution, les implications perf, et les bonnes pratiques d’utilisation des hooks.

## Objectifs pédagogiques

À l’issue de cette formation, vous saurez :

- Décrire **l’ordre exact** d’exécution des hooks Angular dans les cas courants (création, mises à jour, destruction).
- Expliquer **pourquoi** certains hooks sont appelés fréquemment (coût potentiel, change detection).
- Choisir le hook approprié selon le besoin (initialisation, synchronisation d’inputs, accès DOM, nettoyage).
- Écrire des composants **performants et prédictibles** : éviter recalculs, effets de bord, et boucles de détection.
- Structurer votre logique avec parcimonie (RxJS, signals, OnPush, async pipe).

## Pré-requis

- Angular ≥ 15 recommandé (concepts identiques depuis Angular 2, nuances selon versions).
- Connaissances : composants, @Input/@Output, templates, services, RxJS (bases).

## Matériel et contexte

- IDE (VS Code), navigateur, Angular CLI.
- Le cours se concentre sur les hooks :
  - `ngOnChanges`
  - `ngOnInit`
  - `ngDoCheck`
  - `ngAfterContentInit`
  - `ngAfterViewInit`
  - `ngOnDestroy`

---

# Plan de formation

1. **Rappels indispensables : Change Detection & zones**
2. **Vue d’ensemble : les hooks et leur rôle**
3. **Chronologie détaillée : création et premier rendu**
4. **Chronologie détaillée : mises à jour (CD cycle)**
5. **Étude approfondie des hooks**
   1. `ngOnChanges` (inputs, SimpleChanges, patterns)
   2. `ngOnInit` (initialisation idempotente)
   3. `ngDoCheck` (détection personnalisée, dangers)
   4. `ngAfterContentInit` (projection, ContentChild)
   5. `ngAfterViewInit` (ViewChild, DOM, libs)
   6. `ngOnDestroy` (nettoyage, subscriptions)
6. **Coût et performance : pièges fréquents**
7. **Bonnes pratiques avancées** (OnPush, immutabilité, async pipe, signals, memoization)
8. **Ateliers / exercices guidés**
9. **Checklist de revue de code**

---

# 1) Rappels indispensables : Change Detection & zone.js

## 1.1 Qu’est-ce que le Change Detection (CD) ?

Angular met à jour l’UI en évaluant les expressions des templates et en synchronisant la vue avec l’état du composant.

Un **cycle de change detection** (CD cycle) peut être déclenché par :

- Événements DOM (click, input…)
- Timers (setTimeout, setInterval)
- Promises / microtasks
- XHR / fetch (selon intégration)
- Émissions RxJS (souvent via async pipe)
- Manuellement via `ChangeDetectorRef` (`markForCheck`, `detectChanges`)

En mode standard (Default), Angular vérifie tous les composants concernés par la branche change-detected. En **OnPush** (ChangeDetectionStrategy.OnPush), la vérification se restreint aux cas où :

- Un `@Input` reçoit une **nouvelle référence**
- Un événement est déclenché dans le composant
- `markForCheck()` est appelé
- Un `async` pipe émet

## 1.2 Pourquoi les hooks coûtent ?

Certains hooks sont appelés **à chaque cycle de CD** (directement ou indirectement). Le coût est rarement le hook lui-même, mais :

- Les **calculs** effectués dedans
- Les accès DOM coûteux
- Les déclenchements de CD supplémentaires
- Les subscriptions non nettoyées
- Les side effects (HTTP, navigation, modifications d’inputs…) exécutés trop souvent

---

# 2) Vue d’ensemble des hooks

| Hook | Moment | Fréquence | Usage typique | Risques |
|------|--------|-----------|---------------|--------|
| `ngOnChanges` | Avant `ngOnInit` et à chaque changement d’`@Input` | Potentiellement fréquent | Réagir aux changements d’inputs, dériver un état | Recalculs, effets de bord si non contrôlés |
| `ngOnInit` | Après premier `ngOnChanges` | 1 fois | Initialisation, lancement de flux, fetch (souvent via service) | Lancer trop de logique liée aux inputs tardifs |
| `ngDoCheck` | À chaque CD sur ce composant | Très fréquent | Cas avancés : détection custom, deep check | Gros coût/perf, rendre le code fragile |
| `ngAfterContentInit` | Après init de contenu projeté | 1 fois | Accéder à `@ContentChild`, init projection | Confusion content vs view |
| `ngAfterViewInit` | Après init de la vue & des ViewChild | 1 fois | Accès DOM, libs, mesure dimensions | ExpressionChangedAfter..., CD additionnel |
| `ngOnDestroy` | Avant destruction | 1 fois | Cleanup subscriptions, listeners, timers | Mémoire/leaks si oublié |

---

# 3) Chronologie : création et premier rendu

## 3.1 Séquence principale (simplifiée)

Sur le **premier rendu** d’un composant :

1. Construction (`constructor`)
2. Assignation des inputs
3. `ngOnChanges` (si composant a des `@Input`)
4. `ngOnInit`
5. `ngDoCheck`
6. Rendu du template + création des enfants
7. `ngAfterContentInit` (si projection)
8. `ngAfterContentChecked` *(non abordé en détail ici)*
9. `ngAfterViewInit`
10. `ngAfterViewChecked` *(non abordé en détail ici)*

> Notes :
> - Angular appelle aussi `ngAfterContentChecked` / `ngAfterViewChecked` à chaque cycle, mais votre programme de formation cible les hooks listés.
> - L’ordre exact peut varier selon l’arbre (parent/enfant), mais les grandes étapes restent : inputs → init → check → content/view init.

## 3.2 Exemple : observer l’ordre dans la console

```ts
@Component({
  selector: 'app-demo',
  template: `
    <h2>Demo</h2>
    <p>{{ value }}</p>
  `
})
export class DemoComponent
  implements OnChanges, OnInit, DoCheck, AfterContentInit, AfterViewInit, OnDestroy {

  @Input() value = 0;

  constructor() {
    console.log('constructor');
  }

  ngOnChanges(changes: SimpleChanges) {
    console.log('ngOnChanges', changes);
  }

  ngOnInit() {
    console.log('ngOnInit');
  }

  ngDoCheck() {
    console.log('ngDoCheck');
  }

  ngAfterContentInit() {
    console.log('ngAfterContentInit');
  }

  ngAfterViewInit() {
    console.log('ngAfterViewInit');
  }

  ngOnDestroy() {
    console.log('ngOnDestroy');
  }
}
```

---

# 4) Chronologie : mises à jour (CD cycles)

## 4.1 Ce qui se passe lors d’une mise à jour

Lorsqu’un événement déclenche un CD cycle :

- Angular vérifie les bindings du template
- Met à jour le DOM si nécessaire
- Appelle certains hooks selon situation

### Hooks concernés en mise à jour

- `ngOnChanges` : appelé **si** un `@Input` a changé (nouvelle valeur assignée).
- `ngDoCheck` : appelé **à chaque cycle** pour ce composant (s’il est checké).

`ngOnInit`, `ngAfterContentInit`, `ngAfterViewInit` ne se rejouent pas (sauf recréation du composant due à `*ngIf`, navigation, etc.).

## 4.2 Cas classique : parent met à jour un input

- Parent change une propriété utilisée dans `[value]="parentValue"`
- En CD cycle : l’enfant reçoit la nouvelle entrée
- `ngOnChanges` de l’enfant est appelé
- Puis `ngDoCheck` (et checks associés)

---

# 5) Étude approfondie des hooks

## 5.1 `ngOnChanges` — Réagir correctement aux @Input

### Quand est-il appelé ?

- Au **premier binding** des inputs (avant `ngOnInit`)
- À chaque fois qu’Angular assigne une nouvelle valeur à un `@Input`

### Signature et `SimpleChanges`

```ts
ngOnChanges(changes: SimpleChanges) {
  const c = changes['value'];
  if (c) {
    console.log(c.previousValue, c.currentValue, c.firstChange);
  }
}
```

`SimpleChange` contient :
- `previousValue`
- `currentValue`
- `firstChange`

### Bon usage

- **Dériver un état interne** depuis les inputs (ex : normaliser, pré-calculer)
- Lancer une logique **pure** ou idempotente
- Déclencher un flux quand l’input change (mais attention aux effets de bord)

### Anti-pattern : effets de bord répétitifs

Éviter dans `ngOnChanges` :
- Appels HTTP non contrôlés
- Écritures dans des services partagés sans garde
- Logique lourde à chaque changement

### Pattern recommandé : garde + comparaison

```ts
ngOnChanges(changes: SimpleChanges) {
  const q = changes['query'];
  if (!q || q.firstChange) return;

  if (q.previousValue === q.currentValue) return;

  this.search(q.currentValue);
}
```

### Pattern avancé : input setter vs `ngOnChanges`

- **Setter `@Input()`** : pratique pour un seul input, logique localisée
- **`ngOnChanges`** : utile pour corréler plusieurs inputs et gérer `firstChange`

```ts
private _config!: Config;
@Input() set config(v: Config) {
  this._config = v;
  this.recomputeDerivedState(v);
}
get config() { return this._config; }
```

> En OnPush, changer une propriété interne à la volée ne déclenchera pas forcément de CD ailleurs : privilégiez immutabilité et streams.

---

## 5.2 `ngOnInit` — Initialisation unique et prévisible

### Quand ?

- Appelé **une seule fois**, après le premier `ngOnChanges`.

### Que mettre dedans ?

- Initialisation qui ne dépend pas de l’arrivée future d’inputs
- Mise en place de streams RxJS, abonnements (ou mieux : `async` pipe)
- Récupération de données initiales (souvent via service)

### Exemple : initialiser un stream plutôt que recalculer

```ts
readonly items$ = this.store.items$; // exposer un observable au template

ngOnInit() {
  this.store.loadOnce();
}
```

### Piège : initialiser à partir d’inputs tardifs

Si un parent fournit un input asynchrone (ex : via `*ngIf="data$ | async as d"`), l’enfant sera créé seulement quand `d` existe — donc `ngOnInit` verra déjà la valeur.

Mais si l’input peut changer après, alors la logique doit être dans `ngOnChanges` ou dans un stream.

---

## 5.3 `ngDoCheck` — Puissant, rarement nécessaire

### Quand ?

- Appelé à chaque CD cycle pour ce composant (s’il est vérifié).

### À quoi ça sert ?

- Implémenter une détection personnalisée quand `ngOnChanges` ne suffit pas.
- Surveiller des mutations internes (objets modifiés en place) — mais c’est souvent le symptôme d’un design à améliorer.

### Coût potentiel

Très élevé si :
- Vous faites un deep compare
- Vous appelez des fonctions lourdes
- Vous déclenchez du CD supplémentaire

### Exemple : détection d’une mutation (à éviter si possible)

```ts
private lastSnapshot = '';

ngDoCheck() {
  const snapshot = JSON.stringify(this.config);
  if (snapshot !== this.lastSnapshot) {
    this.lastSnapshot = snapshot;
    this.recompute();
  }
}
```

> Cet exemple est volontairement coûteux : il illustre pourquoi `ngDoCheck` doit être une exception.

### Alternatives modernes

- **Immutabilité** : remplacer l’objet par une nouvelle référence → `ngOnChanges`/OnPush fonctionne.
- **RxJS** : traiter les changements au fil de l’eau (operators).
- **Signals** (Angular ≥ 16) : recalculs fins via `computed`/`effect`.

---

## 5.4 `ngAfterContentInit` — Contenu projeté (Content Projection)

### Définitions

- **Content** : ce qui est projeté dans `<ng-content>` depuis le parent.
- **View** : le template interne du composant.

### Quand ?

- Une seule fois, quand la projection est initialisée.

### Usage

- Accéder à `@ContentChild` / `@ContentChildren`
- Initialiser une logique basée sur le contenu projeté

```ts
@Component({
  selector: 'app-card',
  template: `
    <div class="card">
      <ng-content></ng-content>
    </div>
  `
})
export class CardComponent implements AfterContentInit {
  @ContentChild('title') titleEl?: ElementRef<HTMLElement>;

  ngAfterContentInit() {
    // À ce moment, le contenu projeté est résolu
    console.log('Projected title:', this.titleEl);
  }
}
```

Parent :

```html
<app-card>
  <h3 #title>Bonjour</h3>
</app-card>
```

### Pièges

- Vouloir mesurer/accéder au DOM complet : préférez `ngAfterViewInit` si cela concerne la *view* du composant.
- Attention aux contenus conditionnels (`*ngIf`) : la présence de ContentChild peut varier.

---

## 5.5 `ngAfterViewInit` — Accès ViewChild / DOM / librairies tierces

### Quand ?

- Une seule fois, lorsque la *vue* (template) du composant est initialisée, et que les `@ViewChild` sont résolus.

### Cas d’usage

- Accéder à un élément DOM via `ViewChild`
- Initialiser une librairie (chart, editor, map)
- Mesurer des dimensions (après rendu)

```ts
@Component({
  template: `
    <canvas #canvas></canvas>
  `
})
export class ChartComponent implements AfterViewInit {
  @ViewChild('canvas', { static: true }) canvas!: ElementRef<HTMLCanvasElement>;

  ngAfterViewInit() {
    const ctx = this.canvas.nativeElement.getContext('2d');
    // init chart...
  }
}
```

### Risque : `ExpressionChangedAfterItHasBeenCheckedError`

Si vous modifiez une propriété bindée au template dans `ngAfterViewInit`, Angular peut détecter un changement après vérification.

Solutions courantes :

- Déplacer la logique dans `ngOnInit` si possible
- Utiliser un microtask (dernier recours) :

```ts
ngAfterViewInit() {
  queueMicrotask(() => {
    this.ready = true;
  });
}
```

- Ou injecter `ChangeDetectorRef` et appeler `detectChanges()` après avoir modifié l’état (à utiliser avec parcimonie)

```ts
constructor(private cdr: ChangeDetectorRef) {}

ngAfterViewInit() {
  this.ready = true;
  this.cdr.detectChanges();
}
```

> `detectChanges()` force un passage de CD local — utile, mais peut masquer un problème de conception s’il est surutilisé.

---

## 5.6 `ngOnDestroy` — Nettoyage et prévention des fuites mémoire

### Quand ?

- Juste avant que le composant soit détruit (navigation, `*ngIf` qui passe à false, etc.).

### Ce qu’il faut nettoyer

- subscriptions RxJS (si pas gérées par `async` pipe)
- timers
- listeners DOM (`addEventListener`)
- ressources externes (websocket, chart instances)

### Pattern recommandé : `takeUntilDestroyed` (Angular 16+)

```ts
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';
import { DestroyRef, inject } from '@angular/core';

export class DemoComponent {
  private destroyRef = inject(DestroyRef);

  ngOnInit() {
    this.service.stream$
      .pipe(takeUntilDestroyed(this.destroyRef))
      .subscribe(v => {
        // ...
      });
  }
}
```

### Pattern classique : Subject `destroy$`

```ts
private readonly destroy$ = new Subject<void>();

ngOnInit() {
  this.service.stream$
    .pipe(takeUntil(this.destroy$))
    .subscribe();
}

ngOnDestroy() {
  this.destroy$.next();
  this.destroy$.complete();
}
```

---

# 6) Coût et performance : pièges fréquents

## 6.1 Le vrai problème : logique appelée trop souvent

- `ngDoCheck` (et les checked hooks) peuvent tourner énormément.
- `ngOnChanges` peut être appelé souvent si des inputs changent fréquemment.

### Signaux d’alerte

- Scroll saccadé
- CPU élevé
- Trop de logs de hooks dans la console
- Requêtes réseau en rafale au moindre changement

## 6.2 Anti-patterns communs

1. **Calculs lourds dans les hooks**
   - Exemple : tri/filtre d’un gros tableau à chaque CD.
2. **Functions in template**
   - Chaque CD évalue les fonctions.
3. **Mutations d’objets** en place avec OnPush
   - OnPush ne voit pas le changement si la référence ne change pas.
4. **Subscriptions manuelles** sans unsubscribe

## 6.3 Mesurer et profiler

- Angular DevTools : profiler change detection
- Chrome Performance : timeline + long tasks
- Ajout ponctuel de logs + compteurs (à retirer ensuite)

---

# 7) Bonnes pratiques avancées (parcimonie + prédictibilité)

## 7.1 Préférer des modèles réactifs

- Exposer des observables au template et utiliser `async` pipe.
- Centraliser les transformations avec RxJS (map, distinctUntilChanged, shareReplay).

```ts
readonly vm$ = combineLatest([
  this.query$,
  this.filters$,
]).pipe(
  map(([q, f]) => buildViewModel(q, f)),
  distinctUntilChanged((a, b) => deepEqual(a, b)),
  shareReplay({ bufferSize: 1, refCount: true })
);
```

## 7.2 OnPush + immutabilité

- Utiliser `ChangeDetectionStrategy.OnPush`
- Changer les références (`array = [...array, newItem]`) au lieu de muter (`push`).

## 7.3 Mémoïsation / caching local

- Cacher le résultat d’un calcul si les entrées n’ont pas changé.
- Attention à ne pas créer de cache global non nettoyé.

## 7.4 Side effects : les isoler

- Éviter les side effects dans `ngOnChanges` et surtout `ngDoCheck`.
- Déclencher les effets via streams, effects (signals), ou handlers explicites.

---

# 8) Ateliers / exercices guidés

## Exercice 1 — Observer l’ordre d’appel

- Créer un composant parent + enfant avec `@Input`.
- Ajouter des logs aux hooks.
- Déclencher :
  1. création via `*ngIf`
  2. changement d’input via un bouton
  3. destruction via `*ngIf`

Objectif : écrire la chronologie observée et l’expliquer.

## Exercice 2 — Corriger un composant trop coûteux

Scénario :
- Un composant filtre une liste dans `ngDoCheck`.

Attendu :
- Remplacer par un flux (observable) ou une stratégie immuable.

## Exercice 3 — Nettoyage des subscriptions

- Créer un interval observable.
- Vérifier que la destruction stoppe l’activité.
- Implémenter `takeUntilDestroyed`.

## Exercice 4 — ViewChild et ExpressionChanged…

- Accéder à une dimension DOM en `ngAfterViewInit`.
- Mettre à jour une propriété bindée.
- Corriger l’erreur via une approche appropriée.

---

# 9) Checklist de revue de code (spéciale lifecycle)

- [ ] Y a-t-il des **effets de bord** dans `ngOnChanges`/`ngDoCheck` ?
- [ ] Les `@Input` sont-ils **immutables** (nouvelles références) ?
- [ ] Le composant utilise-t-il **OnPush** quand pertinent ?
- [ ] Les subscriptions sont-elles gérées par `async` pipe ou nettoyées (`takeUntilDestroyed`/`ngOnDestroy`) ?
- [ ] Les calculs lourds sont-ils memoïsés / déplacés dans des streams ?
- [ ] Les accès DOM sont-ils strictement dans `ngAfterViewInit` (ou via directives dédiées) ?
- [ ] Pas de `detectChanges()` « pansement » répété sans justification ?

---

# Annexes

## A) Résumé ultra-court (à garder sous la main)

- `ngOnChanges` : réagir aux **inputs** (avec garde, logique pure).
- `ngOnInit` : initialiser **une fois** (setup streams, init).
- `ngDoCheck` : rare, à manier avec **extrême parcimonie**.
- `ngAfterContentInit` : contenu **projeté** disponible.
- `ngAfterViewInit` : vue et **ViewChild/DOM** disponibles.
- `ngOnDestroy` : **nettoyer** pour éviter leaks.

## B) Mini-glossaire

- **Content projection** : insérer du markup enfant dans un composant via `<ng-content>`.
- **OnPush** : stratégie de CD optimisée, exige immutabilité et/ou signals/streams.
- **Side effect** : action non pure (HTTP, update service, navigation, DOM direct).

---

*Fin de la formation.*
