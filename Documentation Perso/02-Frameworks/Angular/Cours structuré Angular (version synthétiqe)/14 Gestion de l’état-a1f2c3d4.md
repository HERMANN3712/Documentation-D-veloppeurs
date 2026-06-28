# Formation Angular — Gestion de l’état (State Management)

**Référence**: 14  
**Public**: développeurs Angular (débutant avancé → intermédiaire)  
**Pré-requis**: TypeScript, Angular (components, services, RxJS de base), notions HTTP et routing  
**Durée conseillée**: 1 à 2 jours (selon profondeur NgRx)  

---

## Objectifs pédagogiques

À la fin de cette formation, vous serez capable de :

- Expliquer ce qu’est “l’état” d’une application Angular et pourquoi il devient difficile à gérer dans des applications complexes.
- Choisir une stratégie de gestion d’état adaptée : local (component), partagé (services), centralisé (store type NgRx).
- Mettre en place un **Service Store** simple basé sur RxJS (Subject/BehaviorSubject) et le consommer avec `async` pipe.
- Comprendre les principes d’une architecture **unidirectionnelle** (actions → reducers → store → vues).
- Implémenter une gestion d’état **centralisée avec NgRx** (actions, reducers, selectors, effects) sur un cas concret.
- Appliquer des bonnes pratiques : immutabilité, normalisation des données, performance, testabilité.

---

## Plan de la formation

1. **Introduction : comprendre l’état**
   - Définition et typologie (UI state, domain state, server state)
   - Symptômes d’une mauvaise gestion
   - Notions clés : source of truth, immutabilité, flux unidirectionnel

2. **Gestion d’état “native” Angular : local & services**
   - État local de composant
   - Partage via Input/Output, services, DI
   - RxJS : BehaviorSubject, Observables, async pipe

3. **Services partagés : construire un mini-store RxJS**
   - Modèle de state
   - Méthodes de mise à jour (set/patch)
   - Sélecteurs (streams dérivés)
   - Gestion des effets “à la main” (HTTP, errors)

4. **Pourquoi (et quand) centraliser : limites des services “maison”**
   - Scalabilité, conventions, traçabilité
   - Complexité des effets asynchrones
   - Debug, time-travel, outils

5. **NgRx : concepts fondamentaux**
   - Store, actions, reducers
   - Selectors et memoization
   - Effects
   - Entity (optionnel mais recommandé)
   - DevTools

6. **Atelier fil rouge : Todo / Products / Users**
   - Mise en place progressive : service store → NgRx
   - CRUD, liste + détail, loading/error
   - Optimisations : OnPush, trackBy

7. **Bonnes pratiques & tests**
   - Immutabilité, normalisation
   - Patterns (feature store, facade)
   - Tests reducers/selectors/effects

8. **Synthèse : matrice de décision**
   - Quand rester simple
   - Quand passer à NgRx
   - Checklist d’architecture

---

# 1) Introduction : comprendre l’état

## 1.1 Définition

L’**état (state)** d’une application est l’ensemble des **données** qui décrivent *ce que l’utilisateur voit* et *comment l’application se comporte* à un instant T.

Dans Angular, l’état peut être :

- **Local** : propre à un composant (ex. valeur d’un champ de formulaire).
- **Partagé** : utilisé par plusieurs composants (ex. l’utilisateur connecté).
- **Persisté** : stocké côté client (LocalStorage) ou côté serveur.

## 1.2 Typologie pratique

1. **UI State** (état d’interface)
   - Exemples : `isSidebarOpen`, `selectedTab`, `formTouched`, `isModalOpen`.

2. **Domain State** (état métier)
   - Exemples : `cartItems`, `selectedProductId`, `currentUser`.

3. **Server State** (état venant du serveur)
   - Exemples : `products`, `orders`, `userProfile`.
   - Souvent asynchrone (loading/error/success), potentiellement “stale”, cache.

## 1.3 Symptômes d’un problème de gestion d’état

- Données dupliquées dans plusieurs composants.
- Synchronisation fragile (un composant met à jour, un autre n’est pas informé).
- Prop drilling (passage d’Input en cascade, Output partout).
- Bugs intermittents liés à l’asynchrone.
- Difficulté de test, de debug et de refactoring.

## 1.4 Principes clés

- **Single Source of Truth** : une source unique pour un type de données.
- **Immutabilité** : on ne modifie pas l’objet existant, on crée une nouvelle valeur.
- **Flux unidirectionnel** : l’UI déclenche des événements → l’état évolue → l’UI se met à jour.

---

# 2) Gestion d’état “native” Angular : local & services

## 2.1 État local de composant

Cas d’usage : l’état ne concerne que le composant (ou un petit sous-arbre).

```ts
@Component({
  selector: 'app-counter',
  template: `
    <button (click)="dec()">-</button>
    <span>{{ count }}</span>
    <button (click)="inc()">+</button>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class CounterComponent {
  count = 0;

  inc() { this.count++; }
  dec() { this.count--; }
}
```

**Avantages** : simple, direct.  
**Limites** : partage difficile, état parfois dispersé.

## 2.2 Partage d’état via Input/Output

Approche efficace mais qui peut devenir verbeuse dans les grands arbres de composants.

- Parent = source de vérité
- Enfant reçoit via `@Input()`
- Enfant notifie via `@Output()`

## 2.3 Partage via services (Dependency Injection)

Un service singleton (au niveau module ou root) peut stocker de l’état partagé.

### Exemple minimal

```ts
@Injectable({ providedIn: 'root' })
export class SessionService {
  private _user?: User;

  setUser(user: User) { this._user = user; }
  get user() { return this._user; }
}
```

**Problème** : ce modèle n’est pas réactif. Les composants ne sont pas notifiés automatiquement.

## 2.4 RxJS : la base du state réactif

Le pattern courant est d’exposer :

- un **Observable** en lecture (`state$`)
- des **méthodes** pour écrire (`setX()`, `loadX()`, etc.)

Outils courants :

- `BehaviorSubject` : garde la dernière valeur et l’émet aux nouveaux abonnés.
- `Subject` : ne garde pas d’historique.
- `async` pipe : gère l’abonnement/désabonnement automatiquement côté template.

---

# 3) Services partagés : construire un mini-store RxJS

Objectif : centraliser l’état d’une feature avec un service, sans framework.

## 3.1 Modéliser l’état

Prenons une feature **Products**.

```ts
export interface ProductsState {
  items: Product[];
  selectedId: string | null;
  loading: boolean;
  error: string | null;
}

export const initialProductsState: ProductsState = {
  items: [],
  selectedId: null,
  loading: false,
  error: null,
};
```

## 3.2 Créer un service store

```ts
@Injectable({ providedIn: 'root' })
export class ProductsStore {
  private readonly _state$ = new BehaviorSubject<ProductsState>(initialProductsState);

  // Lecture: observable public
  readonly state$ = this._state$.asObservable();

  // Sélecteurs (streams dérivés)
  readonly items$ = this.state$.pipe(map(s => s.items));
  readonly loading$ = this.state$.pipe(map(s => s.loading));
  readonly error$ = this.state$.pipe(map(s => s.error));
  readonly selectedProduct$ = this.state$.pipe(
    map(s => s.items.find(p => p.id === s.selectedId) ?? null)
  );

  constructor(private readonly api: ProductsApi) {}

  // Utilitaire d’update immutable
  private setState(partial: Partial<ProductsState>) {
    const current = this._state$.value;
    this._state$.next({ ...current, ...partial });
  }

  select(id: string | null) {
    this.setState({ selectedId: id });
  }

  loadAll() {
    this.setState({ loading: true, error: null });

    this.api.getAll().subscribe({
      next: items => this.setState({ items, loading: false }),
      error: err => this.setState({ error: String(err), loading: false })
    });
  }
}
```

### Points importants

- **Immutabilité** : `next({ ...current, ...partial })`
- **Sélecteurs** : on évite que les composants connaissent la structure interne en profondeur.
- **Single source of truth** : `ProductsStore` devient la référence.

## 3.3 Consommer le store dans un composant

```ts
@Component({
  selector: 'app-products',
  template: `
    <ng-container *ngIf="loading$ | async; else content">
      <p>Chargement…</p>
    </ng-container>

    <ng-template #content>
      <p *ngIf="(error$ | async) as err" class="error">{{ err }}</p>

      <ul>
        <li *ngFor="let p of (items$ | async); trackBy: trackById"
            (click)="select(p.id)">
          {{ p.name }}
        </li>
      </ul>

      <section *ngIf="(selected$ | async) as selected">
        <h3>Détail</h3>
        <pre>{{ selected | json }}</pre>
      </section>
    </ng-template>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class ProductsComponent {
  readonly items$ = this.store.items$;
  readonly loading$ = this.store.loading$;
  readonly error$ = this.store.error$;
  readonly selected$ = this.store.selectedProduct$;

  constructor(private readonly store: ProductsStore) {}

  ngOnInit() {
    this.store.loadAll();
  }

  select(id: string) {
    this.store.select(id);
  }

  trackById(_: number, p: Product) { return p.id; }
}
```

## 3.4 Gérer les effets asynchrones proprement (sans fuite mémoire)

Le `subscribe()` dans le store fonctionne, mais il faut :

- éviter de multiplier les subscriptions non gérées
- gérer l’annulation (ex. recherche)

Une approche plus “RxJS” consiste à piloter via un **Subject d’actions**.

```ts
@Injectable({ providedIn: 'root' })
export class ProductsStoreRx {
  private readonly actions$ = new Subject<
    | { type: 'loadAll' }
    | { type: 'select'; id: string | null }
  >();

  private readonly state$ = this.actions$.pipe(
    startWith({ type: 'loadAll' } as const),
    scan((state: ProductsState, action) => {
      switch (action.type) {
        case 'select':
          return { ...state, selectedId: action.id };
        case 'loadAll':
          return { ...state, loading: true, error: null };
        default:
          return state;
      }
    }, initialProductsState),
    shareReplay({ bufferSize: 1, refCount: true })
  );

  // Exposer des selectors...
}
```

Ce pattern se rapproche d’un store, mais devient vite complexe : c’est précisément le terrain de NgRx.

---

# 4) Pourquoi (et quand) centraliser : limites des services “maison”

Les services partagés fonctionnent très bien… jusqu’à une certaine taille.

## 4.1 Limites typiques

- **Conventions** : chaque équipe code son “store” différemment.
- **Traçabilité** : difficile de savoir “qui a modifié quoi et quand”.
- **Asynchrone** : gestion fine des effets (annulation, retries, concat vs switch, etc.) répétitive.
- **DevTools** : pas de time-travel/debug standard.
- **Tests** : possible, mais manque de structure.

## 4.2 Indicateurs justifiant NgRx

Vous devriez considérer NgRx si :

- Plusieurs vues manipulent les mêmes données et doivent rester synchronisées.
- Vous avez des règles de mise à jour complexes (beaucoup de cas/événements).
- Vous voulez standardiser une architecture à l’échelle équipe.
- Vous avez des besoins forts de debug / audit / relecture d’événements.

---

# 5) NgRx : concepts fondamentaux

NgRx implémente une architecture inspirée de Redux :

1. **UI** dispatch une **action**
2. Le **reducer** calcule le nouvel état (synchrone, pur)
3. Le **Store** publie l’état
4. L’UI lit via des **selectors**
5. Les **effects** gèrent l’asynchrone et dispatchent des actions de résultat

## 5.1 Installation (indicatif)

```bash
ng add @ngrx/store
ng add @ngrx/effects
ng add @ngrx/store-devtools
```

> Le détail exact dépend de votre version Angular/NgRx et de votre setup (standalone, modules, etc.).

## 5.2 Modèle de feature state

```ts
export interface ProductsFeatureState {
  items: Product[];
  loading: boolean;
  error: string | null;
  selectedId: string | null;
}

export const initialState: ProductsFeatureState = {
  items: [],
  loading: false,
  error: null,
  selectedId: null,
};
```

## 5.3 Actions

Les actions décrivent un événement (intention) :

```ts
import { createActionGroup, props, emptyProps } from '@ngrx/store';

export const ProductsActions = createActionGroup({
  source: 'Products',
  events: {
    'Load All': emptyProps(),
    'Load All Success': props<{ items: Product[] }>(),
    'Load All Failure': props<{ error: string }>(),

    'Select': props<{ id: string | null }>(),
  }
});
```

## 5.4 Reducer

Un reducer doit être :

- **pur** (pas de side effects)
- **synchrone**
- **immutable**

```ts
import { createReducer, on } from '@ngrx/store';

export const productsReducer = createReducer(
  initialState,

  on(ProductsActions.loadAll, state => ({
    ...state,
    loading: true,
    error: null,
  })),

  on(ProductsActions.loadAllSuccess, (state, { items }) => ({
    ...state,
    items,
    loading: false,
  })),

  on(ProductsActions.loadAllFailure, (state, { error }) => ({
    ...state,
    loading: false,
    error,
  })),

  on(ProductsActions.select, (state, { id }) => ({
    ...state,
    selectedId: id,
  }))
);
```

## 5.5 Selectors

Les selectors sont des fonctions (souvent memoizées) pour lire l’état.

```ts
import { createFeatureSelector, createSelector } from '@ngrx/store';

export const selectProductsState = createFeatureSelector<ProductsFeatureState>('products');

export const selectItems = createSelector(selectProductsState, s => s.items);
export const selectLoading = createSelector(selectProductsState, s => s.loading);
export const selectError = createSelector(selectProductsState, s => s.error);
export const selectSelectedId = createSelector(selectProductsState, s => s.selectedId);

export const selectSelectedProduct = createSelector(
  selectItems,
  selectSelectedId,
  (items, id) => items.find(p => p.id === id) ?? null
);
```

## 5.6 Effects (asynchrone)

Les effects écoutent les actions et déclenchent des opérations asynchrones (HTTP…).

```ts
@Injectable()
export class ProductsEffects {
  loadAll$ = createEffect(() =>
    this.actions$.pipe(
      ofType(ProductsActions.loadAll),
      switchMap(() =>
        this.api.getAll().pipe(
          map(items => ProductsActions.loadAllSuccess({ items })),
          catchError(err => of(ProductsActions.loadAllFailure({ error: String(err) })))
        )
      )
    )
  );

  constructor(
    private readonly actions$: Actions,
    private readonly api: ProductsApi
  ) {}
}
```

### Stratégies RxJS utiles

- `switchMap` : annule la requête précédente (recherche, autocomplete).
- `concatMap` : enchaîne (file d’attente) — utile si l’ordre compte.
- `mergeMap` : parallèle.
- `exhaustMap` : ignore les déclenchements tant que le précédent n’est pas fini (login).

## 5.7 Intégration dans l’application (exemples)

### Avec modules

```ts
@NgModule({
  imports: [
    StoreModule.forFeature('products', productsReducer),
    EffectsModule.forFeature([ProductsEffects]),
  ],
})
export class ProductsStateModule {}
```

### Avec standalone (indicatif)

```ts
export const PRODUCTS_STATE_PROVIDERS = [
  provideState({ name: 'products', reducer: productsReducer }),
  provideEffects(ProductsEffects),
];
```

## 5.8 Consommer dans un composant

```ts
@Component({
  selector: 'app-products',
  template: `
    <button (click)="reload()">Recharger</button>
    <p *ngIf="(loading$ | async)">Chargement…</p>

    <ul>
      <li *ngFor="let p of (items$ | async); trackBy: trackById"
          (click)="select(p.id)">
        {{ p.name }}
      </li>
    </ul>

    <pre *ngIf="(selected$ | async) as s">{{ s | json }}</pre>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class ProductsComponent {
  readonly items$ = this.store.select(selectItems);
  readonly loading$ = this.store.select(selectLoading);
  readonly selected$ = this.store.select(selectSelectedProduct);

  constructor(private readonly store: Store) {}

  ngOnInit() {
    this.store.dispatch(ProductsActions.loadAll());
  }

  reload() {
    this.store.dispatch(ProductsActions.loadAll());
  }

  select(id: string) {
    this.store.dispatch(ProductsActions.select({ id }));
  }

  trackById(_: number, p: Product) { return p.id; }
}
```

## 5.9 NgRx Entity (optionnel mais très utile)

Quand vous manipulez des listes importantes (CRUD), `@ngrx/entity` aide à :

- normaliser (`ids` + `entities`)
- générer des reducers efficaces
- créer des selectors standard

Cela améliore performance et simplifie les mises à jour.

---

# 6) Atelier fil rouge (proposé)

## 6.1 Énoncé

Construire une page “Catalogue produits” :

- Liste de produits
- Sélection d’un produit
- États `loading` et `error`
- Bouton “recharger”

## 6.2 Étapes

1. Version A : service simple (non réactif) → constater les limites.
2. Version B : service store RxJS (BehaviorSubject) → meilleure synchro.
3. Version C : NgRx
   - actions/reducer/selectors
   - effect HTTP
   - DevTools

## 6.3 Critères de réussite

- Aucune duplication de logique entre composants.
- Les composants consomment l’état via Observables + `async`.
- State transitions claires : `loadAll → success|failure`.
- Code testable (reducers/effects).

---

# 7) Bonnes pratiques & tests

## 7.1 Immutabilité

Toujours retourner de nouveaux objets/arrays :

- ✅ `items: [...state.items, newItem]`
- ❌ `state.items.push(newItem)`

## 7.2 Normalisation

Pour des collections : stocker par `id` évite des mises à jour coûteuses.

## 7.3 Facade pattern (recommandé)

Au lieu d’injecter `Store` partout, créer une facade :

```ts
@Injectable({ providedIn: 'root' })
export class ProductsFacade {
  readonly items$ = this.store.select(selectItems);
  readonly loading$ = this.store.select(selectLoading);
  readonly selected$ = this.store.select(selectSelectedProduct);

  constructor(private readonly store: Store) {}

  init() { this.store.dispatch(ProductsActions.loadAll()); }
  select(id: string) { this.store.dispatch(ProductsActions.select({ id })); }
}
```

**Avantages** : découplage, tests, refactoring, API stable.

## 7.4 Tests

- **Reducers** : tests unitaires purs (faciles)
- **Selectors** : tester la projection
- **Effects** : tests RxJS (marble tests) ou tests plus simples avec `provideMockActions`

Exemple reducer test (simplifié) :

```ts
it('should set loading true on loadAll', () => {
  const state = productsReducer(initialState, ProductsActions.loadAll());
  expect(state.loading).toBeTrue();
});
```

---

# 8) Synthèse : matrice de décision

## 8.1 Choisir la bonne stratégie

- **État local** :
  - petit composant
  - non partagé
  - faible complexité

- **Services partagés (RxJS)** :
  - partagé par quelques composants
  - logique simple à moyenne
  - besoin d’une API claire, sans lourdeur de framework

- **NgRx** :
  - application complexe
  - nombreuses interactions (actions)
  - asynchrone riche
  - besoin de conventions, robustesse, DevTools, testabilité

## 8.2 Checklist de mise en place

- [ ] Identifier les sources d’état (UI vs domain vs server)
- [ ] Définir un modèle d’état clair (interfaces)
- [ ] Éviter la duplication
- [ ] Exposer des selectors/observables (lecture)
- [ ] Centraliser les écritures (méthodes ou actions)
- [ ] Gérer loading/error systématiquement
- [ ] Assurer l’immutabilité
- [ ] Ajouter tests sur reducers/effects

---

## Annexes

### A) Glossaire

- **Store** : conteneur central d’état.
- **Action** : événement décrivant une intention.
- **Reducer** : fonction pure qui calcule le nouvel état.
- **Selector** : fonction de lecture dérivée, souvent memoizée.
- **Effect** : gestion de side effects (HTTP, storage, routing…).

### B) Erreurs courantes

- Mettre de la logique métier dans le composant au lieu du store/facade.
- Muter l’état (bugs de change detection, selectors incohérents).
- Multiplier les subscriptions manuelles au lieu d’utiliser `async` pipe.
- Faire des appels HTTP dans le reducer (interdit : side effect).

---

**Fin de la formation.**
