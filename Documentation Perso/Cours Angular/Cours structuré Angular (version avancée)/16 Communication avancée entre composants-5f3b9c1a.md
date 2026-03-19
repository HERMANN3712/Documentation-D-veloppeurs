# Formation Angular — Communication avancée entre composants

- **Référence** : 16
- **Public** : développeurs Angular (intermédiaire → avancé)
- **Pré‑requis** : composants, `@Input()` / `@Output()`, services, DI, RxJS (bases), templates, forms (optionnel)
- **Durée conseillée** : 1 jour (7h) ou 2 demi‑journées
- **Version Angular** : 16+ (inclut Signals) — adaptable 14/15 avec variantes

---

## Objectifs pédagogiques

À l’issue de la formation, le participant saura :

1. Choisir un **mécanisme de communication** adapté (portée, couplage, testabilité).
2. Mettre en place des **services partagés** et comprendre la **portée d’injection** (root, module, composant, route).
3. Utiliser l’**injection hiérarchique** pour créer des “scopes” de state.
4. Construire un **store local** basé RxJS (Subject/BehaviorSubject/ReplaySubject) ou **Signals**.
5. Partager des **Signals** entre composants (via service, ou via provider local).
6. Appliquer des patterns recommandés : **smart/dumb**, façade, unidirectional data flow, anti‑patterns.
7. Savoir quand passer à un **state management global** (NgRx, NGXS, Akita, SignalStore, etc.).

---

## Plan (structure de la formation)

1. **Rappels & limites de `@Input()` / `@Output()`**
2. **Cartographie des options** (couplage vs portée vs complexité)
3. **Services partagés** : communication par service + RxJS
4. **Injection hiérarchique** : providers au niveau composant / route / module
5. **Stores locaux RxJS** : patterns, sélection, mutations, teardown
6. **Signals** : état partagé moderne et interop RxJS
7. **State management global** : critères, architecture, exemples
8. **Atelier fil rouge** : dashboard + filtre + liste + panneau de détails
9. **Synthèse & checklist de décision**

---

## 1) Rappels & limites de `@Input()` / `@Output()`

### 1.1 `@Input()` / `@Output()` : quand c’est parfait
- Communication **parent → enfant** (Input) et **enfant → parent** (Output)
- Faible complexité, traçabilité claire dans le template
- Data flow explicite, très testable

### 1.2 Limites fréquentes
- **Prop drilling** : passer des données à travers plusieurs niveaux de composants
- Relation **frères** (siblings) : nécessite remonter puis redescendre
- Partage d’état entre **pages** ou **routes**
- Besoin de **cache**, de **sources multiples**, de **multicast** (plusieurs observateurs)
- Gestion de cycles de vie complexes (abonnements, destruction)

### 1.3 Anti‑pattern classique
- Empiler `@Output()` pour remonter des événements “globaux” au root
- Utiliser des `EventEmitter` dans les services (préférer `Subject`)

---

## 2) Cartographie des options (choisir le bon outil)

### 2.1 Axes de décision

| Option | Portée | Couplage | Asynchronisme | Complexité | Cas typiques |
|---|---:|---:|---:|---:|---|
| `@Input`/`@Output` | arborescence | faible | faible/moyen | faible | parent/enfant |
| Service partagé (root) | globale app | moyen | moyen/fort | moyen | auth, préférences |
| Service fourni au composant (scope) | sous-arbre | faible/moyen | moyen/fort | moyen | wizard, feature isolée |
| RxJS Subjects/Store local | local/feature | faible/moyen | fort | moyen/fort | listes, filtres, cache |
| Signals partagés (via service) | local/feature/global | faible/moyen | moyen | moyen | état UI, dérivations |
| State management global | globale app | faible (via règles) | fort | fort | grandes apps, audit |

### 2.2 Règles simples
- **Même feature, plusieurs composants** → service scoped *ou* store local
- **État trans‑feature** (plusieurs domaines) → store global
- **Événement ponctuel** (toast, navigation) → bus d’événements minimal (Subject)
- **Éviter** les singletons globaux si l’état ne doit pas être global

---

## 3) Services partagés : communication via service + RxJS

### 3.1 Objectif
Créer une source de vérité et/ou un bus d’événements, consommable par plusieurs composants.

### 3.2 Exemple : filtre partagé entre composants (RxJS)

#### Service (façade)
```ts
import { Injectable } from '@angular/core';
import { BehaviorSubject, distinctUntilChanged, map } from 'rxjs';

export interface ProductFilter {
  query: string;
  onlyAvailable: boolean;
}

const defaultFilter: ProductFilter = { query: '', onlyAvailable: false };

@Injectable({ providedIn: 'root' })
export class ProductFilterService {
  private readonly filterSubject = new BehaviorSubject<ProductFilter>(defaultFilter);

  readonly filter$ = this.filterSubject.asObservable().pipe(
    distinctUntilChanged((a, b) => a.query === b.query && a.onlyAvailable === b.onlyAvailable)
  );

  readonly query$ = this.filter$.pipe(map(f => f.query));

  setFilter(patch: Partial<ProductFilter>) {
    const current = this.filterSubject.value;
    this.filterSubject.next({ ...current, ...patch });
  }

  reset() {
    this.filterSubject.next(defaultFilter);
  }
}
```

#### Composant A : barre de recherche
```ts
import { Component } from '@angular/core';
import { ProductFilterService } from './product-filter.service';

@Component({
  selector: 'app-filter-bar',
  template: `
    <input
      [value]="(filterService.query$ | async) ?? ''"
      (input)="filterService.setFilter({ query: $any($event.target).value })" />

    <label>
      <input type="checkbox"
        (change)="filterService.setFilter({ onlyAvailable: $any($event.target).checked })" />
      Disponible
    </label>
  `
})
export class FilterBarComponent {
  constructor(public readonly filterService: ProductFilterService) {}
}
```

#### Composant B : liste
```ts
import { Component } from '@angular/core';
import { combineLatest, map } from 'rxjs';
import { ProductFilterService } from './product-filter.service';
import { ProductsApi } from './products.api';

@Component({
  selector: 'app-product-list',
  template: `
    <ul>
      <li *ngFor="let p of (vm$ | async)">{{ p.name }}</li>
    </ul>
  `
})
export class ProductListComponent {
  readonly vm$ = combineLatest([
    this.api.products$,
    this.filterService.filter$
  ]).pipe(
    map(([products, filter]) => products
      .filter(p => !filter.onlyAvailable || p.available)
      .filter(p => p.name.toLowerCase().includes(filter.query.toLowerCase()))
    )
  );

  constructor(
    private readonly api: ProductsApi,
    private readonly filterService: ProductFilterService
  ) {}
}
```

### 3.3 Points d’attention
- **Éviter** les effets de bord dans les `map` : utiliser des méthodes dédiées.
- Contrôler la **portée** du service (root vs feature vs composant).
- `BehaviorSubject` = état courant; `Subject` = événements; `ReplaySubject` = buffer.

---

## 4) Injection hiérarchique : créer des scopes d’état

Angular DI est hiérarchique : un provider déclaré à un niveau “enfant” **masque** le provider parent.

### 4.1 Pourquoi c’est clé pour la communication avancée
- Même service, **plusieurs instances** selon le contexte
- Idéal pour : wizards, onglets, drawers, composants réutilisables

### 4.2 Exemple : store de filtre limité à une page

#### `ProductsPageComponent` fournit le service
```ts
import { Component } from '@angular/core';
import { ProductFilterService } from './product-filter.service';

@Component({
  selector: 'app-products-page',
  template: `
    <app-filter-bar />
    <app-product-list />
  `,
  providers: [ProductFilterService]
})
export class ProductsPageComponent {}
```

- Ici, `FilterBarComponent` et `ProductListComponent` partagent **la même instance** de `ProductFilterService`.
- En naviguant vers une autre page, l’état est **recréé** (utile pour éviter les fuites d’état global).

### 4.3 Variante : provider au niveau route (Angular Router)
- Via `providers` dans la configuration de route (Angular 15+).
- Permet un scope par route sans wrapper component.

---

## 5) Stores locaux RxJS : structurer l’état sans store global

### 5.1 Pourquoi un “store local”
- Encapsuler : **state + sélecteurs + actions**
- Réduction du couplage : les composants ne manipulent pas des Subjects directement
- Meilleure testabilité

### 5.2 Pattern minimal : state immuable + sélecteurs + actions

```ts
import { Injectable, OnDestroy } from '@angular/core';
import { BehaviorSubject, Subject, map, takeUntil } from 'rxjs';

export interface ProductsState {
  loading: boolean;
  items: Array<{ id: string; name: string; available: boolean }>;
  selectedId: string | null;
}

const initialState: ProductsState = {
  loading: false,
  items: [],
  selectedId: null,
};

@Injectable()
export class ProductsStore implements OnDestroy {
  private readonly destroy$ = new Subject<void>();

  private readonly stateSubject = new BehaviorSubject<ProductsState>(initialState);
  readonly state$ = this.stateSubject.asObservable();

  // selectors
  readonly loading$ = this.state$.pipe(map(s => s.loading));
  readonly items$ = this.state$.pipe(map(s => s.items));
  readonly selectedId$ = this.state$.pipe(map(s => s.selectedId));

  // state updater (interne)
  private setState(patch: Partial<ProductsState>) {
    this.stateSubject.next({ ...this.stateSubject.value, ...patch });
  }

  // actions
  load(items: ProductsState['items']) {
    this.setState({ items, loading: false });
  }

  setLoading(loading: boolean) {
    this.setState({ loading });
  }

  select(id: string | null) {
    this.setState({ selectedId: id });
  }

  connectToApi(api$: any) {
    this.setLoading(true);
    api$.pipe(takeUntil(this.destroy$)).subscribe({
      next: (items: any) => this.load(items),
      error: () => this.setLoading(false),
    });
  }

  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
    this.stateSubject.complete();
  }
}
```

### 5.3 Intégration via providers (scope feature)
```ts
@Component({
  selector: 'app-products-page',
  template: `
    <app-products-list />
    <app-product-details />
  `,
  providers: [ProductsStore]
})
export class ProductsPageComponent {
  constructor(store: ProductsStore, api: ProductsApi) {
    store.connectToApi(api.products$);
  }
}
```

### 5.4 Bonnes pratiques RxJS
- Favoriser des **streams publics en lecture seule** (`items$`, etc.).
- Éviter d’exposer le Subject.
- Ajouter `distinctUntilChanged` sur sélecteurs lourds.
- Teardown : `takeUntilDestroyed()` (Angular 16) ou `destroy$`.

---

## 6) Signals : état partagé moderne + dérivations

### 6.1 Rappels Signals (Angular 16+)
- `signal()` : état mutable contrôlé
- `computed()` : dérivation pure
- `effect()` : effets (IO)
- Très bon pour l’état UI local et lisible.

### 6.2 Store local en Signals (via service)

```ts
import { Injectable, computed, signal } from '@angular/core';

export interface UiState {
  query: string;
  onlyAvailable: boolean;
}

@Injectable()
export class ProductsSignalsStore {
  private readonly _query = signal('');
  private readonly _onlyAvailable = signal(false);

  readonly query = this._query.asReadonly();
  readonly onlyAvailable = this._onlyAvailable.asReadonly();

  readonly filter = computed(() => ({
    query: this._query(),
    onlyAvailable: this._onlyAvailable(),
  }));

  setQuery(v: string) { this._query.set(v); }
  setOnlyAvailable(v: boolean) { this._onlyAvailable.set(v); }
  reset() {
    this._query.set('');
    this._onlyAvailable.set(false);
  }
}
```

#### Fournir le store au bon niveau (scope)
```ts
@Component({
  selector: 'app-products-page',
  template: `
    <app-filter-bar />
    <app-product-list />
  `,
  providers: [ProductsSignalsStore]
})
export class ProductsPageComponent {}
```

#### Consommer dans un composant (template)
```ts
import { Component, inject } from '@angular/core';
import { ProductsSignalsStore } from './products-signals.store';

@Component({
  selector: 'app-filter-bar',
  template: `
    <input [value]="store.query()" (input)="store.setQuery($any($event.target).value)" />
    <label>
      <input type="checkbox" [checked]="store.onlyAvailable()"
             (change)="store.setOnlyAvailable($any($event.target).checked)" />
      Disponible
    </label>
  `
})
export class FilterBarComponent {
  readonly store = inject(ProductsSignalsStore);
}
```

### 6.3 Interop RxJS ↔ Signals (quand nécessaire)
- RxJS reste utile pour : websockets, streams asynchrones, annulation, composition.
- Approche : garder RxJS côté API, convertir en signal côté UI.

Exemple (schématique) :
```ts
import { toSignal } from '@angular/core/rxjs-interop';

readonly items = toSignal(this.api.products$, { initialValue: [] });
```

---

## 7) RxJS Subjects : bus d’événements (avec discipline)

### 7.1 Quand utiliser un Subject
- Événement “fire-and-forget” : toast, refresh global, invalidation cache
- Communication transverse sans besoin d’état courant

### 7.2 Exemple : Event bus minimal
```ts
import { Injectable } from '@angular/core';
import { Subject } from 'rxjs';

export type AppEvent =
  | { type: 'REFRESH_PRODUCTS' }
  | { type: 'SHOW_TOAST'; message: string };

@Injectable({ providedIn: 'root' })
export class AppEvents {
  private readonly eventsSubject = new Subject<AppEvent>();
  readonly events$ = this.eventsSubject.asObservable();

  emit(event: AppEvent) {
    this.eventsSubject.next(event);
  }
}
```

### 7.3 Risques et garde‑fous
- Risque : “spaghetti events” si trop utilisé.
- Limiter les types, documenter, centraliser.
- Préférer un store si vous avez besoin de **rejouer** l’état courant.

---

## 8) State management global : quand et comment

### 8.1 Indicateurs qu’un store global devient pertinent
- Plusieurs features partagent des données et règles
- Besoin d’**audit**, time-travel, devtools
- Accès concurrent et logique métier complexe
- Normalisation, cache, pagination, offline

### 8.2 Options courantes
- **NgRx Store/Effects/Entity** : robuste, verbose mais très scalable
- **NGXS** : plus simple, approche orientée classes
- **Akita** : stores/entities pragmatiques
- **@ngrx/signals** / **SignalStore** : patterns modernes autour des signals

### 8.3 Bonnes pratiques d’architecture
- Pattern **façade** : composants ↔ façade ↔ store
- Séparer **UI state** (local) vs **domain state** (global)
- Actions et sélecteurs stables, tests unitaires sur reducers/logic

---

## 9) Atelier fil rouge (guidé)

### 9.1 Énoncé
Construire une page “Produits” composée de :
- `FilterBarComponent` (recherche + checkbox)
- `ProductListComponent` (liste filtrée)
- `ProductDetailsComponent` (détails du produit sélectionné)

Contraintes :
- Pas de prop drilling
- Possibilité d’avoir **deux pages Produits** dans l’app (ex: `/products` et `/admin/products`) sans partager l’état

### 9.2 Étapes proposées
1. Mettre en place un **store local** (RxJS ou Signals) fourni au niveau `ProductsPageComponent`.
2. Exposer : `filter`, `items`, `selectedId`.
3. Connecter un flux API (observable) et gérer `loading`.
4. Consommer le store dans les 3 composants via `inject()`.
5. Ajouter un event bus (optionnel) `REFRESH_PRODUCTS`.

### 9.3 Critères de réussite
- Le filtre modifie la liste en temps réel.
- La sélection met à jour les détails.
- Changer de route réinitialise l’état (scope DI).

---

## 10) Synthèse : checklist de décision

1. **C’est strictement parent/enfant ?** → `@Input/@Output`.
2. **État partagé dans une feature (même page) ?** → service/stores **scopés** via providers.
3. **Besoin d’état courant rejouable ?** → `BehaviorSubject` ou `signal`.
4. **Événements transverses ponctuels ?** → `Subject` (event bus minimal).
5. **État global, règles métiers, devtools ?** → store global (NgRx/SignalStore).

---

## Annexes

### A) `takeUntilDestroyed()` (Angular 16)
```ts
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';
import { DestroyRef, inject } from '@angular/core';

const destroyRef = inject(DestroyRef);
source$.pipe(takeUntilDestroyed(destroyRef)).subscribe();
```

### B) Glossaire rapide
- **Portée/Scope** : durée de vie et zone où un état est partagé.
- **Couplage** : dépendance structurelle entre composants.
- **Unidirectional data flow** : flux de données dans un seul sens (actions → state → vue).

---

## Exercices supplémentaires (optionnel)
1. Refactor : passer d’un bus d’événements global à un store local.
2. Ajouter une feature : pagination + tri (et mesurer l’impact sur l’API du store).
3. Tests : tester le store (RxJS) avec des marbles ou tests synchrones; tester store signals via calls.
