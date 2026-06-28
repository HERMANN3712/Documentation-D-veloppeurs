# Formation Angular — Selectors, reducers et effects (NgRx)

> Objectif : maîtriser la triptyque **reducers / selectors / effects** pour construire des applications Angular complexes **plus lisibles, testables et performantes**.

---

## Sommaire

1. [Pré-requis et contexte](#1-pré-requis-et-contexte)
2. [Architecture globale NgRx](#2-architecture-globale-ngrx)
3. [Reducers](#3-reducers)
4. [Selectors](#4-selectors)
5. [Effects](#5-effects)
6. [Atelier fil rouge : gestion de produits](#6-atelier-fil-rouge--gestion-de-produits)
7. [Tests : reducers, selectors, effects](#7-tests--reducers-selectors-effects)
8. [Bonnes pratiques et anti-patterns](#8-bonnes-pratiques-et-anti-patterns)
9. [Checklists et récapitulatif](#9-checklists-et-récapitulatif)

---

## 1. Pré-requis et contexte

### Public visé
Développeurs Angular souhaitant structurer efficacement l’état applicatif avec **NgRx Store** (ou un équivalent Redux-like).

### Pré-requis
- Angular (components, services, DI, RxJS de base)
- TypeScript (interfaces, generics)
- RxJS : `Observable`, `pipe`, `map`, `switchMap`, `catchError`

### Résultats attendus
À l’issue de la formation, vous saurez :
- implémenter des **reducers** purement fonctionnels pour mettre à jour l’état
- écrire des **selectors** performants et composables
- gérer les effets secondaires via des **effects** robustes
- tester chaque couche de manière isolée

---

## 2. Architecture globale NgRx

### Les rôles
- **Actions** : événements métier (ex. `loadProducts`, `addToCart`) 
- **Reducers** : mise à jour **pure** de l’état à partir des actions
- **Selectors** : extraction/derivation de données (vues) depuis l’état
- **Effects** : gestion des **side effects** (HTTP, navigation, stockage local…)

### Flux de données simplifié

```text
UI -> dispatch(action)
  -> reducer -> new state -> UI (via selectors)
  -> effect (optionnel) -> appelle API -> dispatch(success/failure)
```

### Pourquoi séparer reducers / selectors / effects ?
- **Lisibilité** : chaque fichier a un rôle clair
- **Testabilité** : reducers/selectors = fonctions pures -> tests simples
- **Performance** : selectors mémoïsés -> recalcul minimal
- **Robustesse** : effects gèrent les erreurs et la concurrence RxJS

---

## 3. Reducers

### 3.1. Définition
Un **reducer** est une fonction pure :

```ts
(state, action) => newState
```

Propriétés essentielles :
- **sans effets de bord** (pas de HTTP, pas de `Date.now()` si cela change le résultat)
- **déterministe** (même entrée -> même sortie)
- **immuable** (ne pas modifier `state` directement)

### 3.2. Structure d’état (feature state)
Exemple : feature `products`.

```ts
export interface ProductsState {
  entities: Record<string, Product>;
  ids: string[];
  loading: boolean;
  error: string | null;
  selectedId: string | null;
}

export const initialProductsState: ProductsState = {
  entities: {},
  ids: [],
  loading: false,
  error: null,
  selectedId: null,
};
```

> Note : on stocke souvent une liste sous forme normalisée (`entities` + `ids`) pour faciliter les mises à jour et éviter les doublons.

### 3.3. Actions associées

```ts
import { createAction, props } from '@ngrx/store';

export const loadProducts = createAction('[Products] Load');
export const loadProductsSuccess = createAction(
  '[Products] Load Success',
  props<{ products: Product[] }>()
);
export const loadProductsFailure = createAction(
  '[Products] Load Failure',
  props<{ error: string }>()
);

export const selectProduct = createAction(
  '[Products] Select',
  props<{ id: string }>()
);
```

### 3.4. Implémenter un reducer avec `createReducer`

```ts
import { createReducer, on } from '@ngrx/store';
import * as ProductsActions from './products.actions';

export const productsReducer = createReducer(
  initialProductsState,

  on(ProductsActions.loadProducts, (state) => ({
    ...state,
    loading: true,
    error: null,
  })),

  on(ProductsActions.loadProductsSuccess, (state, { products }) => {
    const entities = products.reduce<Record<string, Product>>((acc, p) => {
      acc[p.id] = p;
      return acc;
    }, {});

    const ids = products.map(p => p.id);

    return {
      ...state,
      entities,
      ids,
      loading: false,
      error: null,
    };
  }),

  on(ProductsActions.loadProductsFailure, (state, { error }) => ({
    ...state,
    loading: false,
    error,
  })),

  on(ProductsActions.selectProduct, (state, { id }) => ({
    ...state,
    selectedId: id,
  }))
);
```

### 3.5. Règles d’or (immutabilité)
- ✅ Copier avec l’opérateur spread : `{ ...state, x: newX }`
- ✅ Copier les collections : `state.ids.concat(id)` / `[...state.ids, id]`
- ❌ Ne jamais faire : `state.loading = true; return state;`

### 3.6. Quand utiliser l’Entity Adapter ?
À partir du moment où vous manipulez :
- des collections
- des opérations CRUD
- des performances de lookup importantes

`@ngrx/entity` fournit :
- `addOne`, `setAll`, `updateOne`, `removeOne`
- des selectors génériques (`selectAll`, `selectEntities`)

---

## 4. Selectors

### 4.1. Définition
Un **selector** est une fonction qui **lit** l’état et renvoie :
- soit un morceau de l’état
- soit une **donnée dérivée** (tri, filtrage, agrégation)

Avec NgRx, les selectors via `createSelector` sont **mémoïsés** :
- si les entrées ne changent pas (références identiques), le résultat est réutilisé

### 4.2. Selecteur de feature

```ts
import { createFeatureSelector, createSelector } from '@ngrx/store';

export const selectProductsState =
  createFeatureSelector<ProductsState>('products');
```

### 4.3. Selectors de lecture simple

```ts
export const selectProductsLoading = createSelector(
  selectProductsState,
  (state) => state.loading
);

export const selectProductsError = createSelector(
  selectProductsState,
  (state) => state.error
);
```

### 4.4. Selectors composés (données dérivées)

```ts
export const selectProductIds = createSelector(
  selectProductsState,
  (state) => state.ids
);

export const selectProductEntities = createSelector(
  selectProductsState,
  (state) => state.entities
);

export const selectAllProducts = createSelector(
  selectProductIds,
  selectProductEntities,
  (ids, entities) => ids.map(id => entities[id])
);
```

### 4.5. Selector paramétré
NgRx recommande d’éviter les selectors « factories » dans les templates, mais ils sont utiles dans certains cas.

```ts
export const selectProductById = (id: string) => createSelector(
  selectProductEntities,
  (entities) => entities[id]
);
```

Alternative : stocker `selectedId`, puis composer :

```ts
export const selectSelectedId = createSelector(
  selectProductsState,
  (state) => state.selectedId
);

export const selectSelectedProduct = createSelector(
  selectProductEntities,
  selectSelectedId,
  (entities, selectedId) => (selectedId ? entities[selectedId] : null)
);
```

### 4.6. Performance : règles pratiques
- Préférez de **petits selectors** (réutilisables), puis composez
- Attention aux transformations qui créent toujours de nouvelles références (ex. `map`/`filter`), ce qui peut déclencher du change detection
- Centralisez les données dérivées dans des selectors plutôt que dans les composants

---

## 5. Effects

### 5.1. Définition
Un **effect** écoute un flux d’actions, effectue un traitement asynchrone/impur, et peut dispatcher de nouvelles actions.

Exemples de side effects :
- appels HTTP
- navigation (router)
- logs
- stockage local

### 5.2. Mise en place
Installer NgRx (si ce n’est pas déjà fait) :

```bash
ng add @ngrx/store
ng add @ngrx/effects
```

Déclarer les effects :

```ts
import { EffectsModule } from '@ngrx/effects';

@NgModule({
  imports: [
    EffectsModule.forRoot([ProductsEffects])
  ],
})
export class AppModule {}
```

### 5.3. Exemple d’effect HTTP

```ts
import { Injectable } from '@angular/core';
import { Actions, createEffect, ofType } from '@ngrx/effects';
import { catchError, map, switchMap, of } from 'rxjs';
import * as ProductsActions from './products.actions';
import { ProductsApi } from '../api/products.api';

@Injectable()
export class ProductsEffects {
  loadProducts$ = createEffect(() =>
    this.actions$.pipe(
      ofType(ProductsActions.loadProducts),
      switchMap(() =>
        this.api.getAll().pipe(
          map(products => ProductsActions.loadProductsSuccess({ products })),
          catchError(err =>
            of(ProductsActions.loadProductsFailure({ error: this.toMessage(err) }))
          )
        )
      )
    )
  );

  constructor(private actions$: Actions, private api: ProductsApi) {}

  private toMessage(err: unknown): string {
    return err instanceof Error ? err.message : 'Unknown error';
  }
}
```

### 5.4. Choisir le bon opérateur de concurrence
Selon le cas d’usage :
- `switchMap` : annule la requête précédente si une nouvelle arrive (recherche live)
- `concatMap` : met en file (créations successives)
- `mergeMap` : parallèle (batch, peu de collisions)
- `exhaustMap` : ignore les nouvelles tant que la précédente n’est pas terminée (bouton “refresh” anti-spam)

### 5.5. Effects sans dispatch
Utile pour logs, navigation, notifications.

```ts
import { tap } from 'rxjs';

navigateOnSelect$ = createEffect(
  () => this.actions$.pipe(
    ofType(ProductsActions.selectProduct),
    tap(({ id }) => {
      // ex: this.router.navigate(['/products', id]);
      console.log('Selected product', id);
    })
  ),
  { dispatch: false }
);
```

### 5.6. Gestion des erreurs
Principes :
- capturer l’erreur dans l’effect (`catchError`)
- retourner une action *Failure*
- afficher l’erreur via selector (UI), et/ou notifications (effect sans dispatch)

---

## 6. Atelier fil rouge : gestion de produits

### 6.1. Objectif
Construire un module `Products` qui :
- charge la liste des produits
- affiche un état `loading`
- affiche une erreur éventuelle
- sélectionne un produit

### 6.2. Composant : consommer les selectors

```ts
import { Component, ChangeDetectionStrategy } from '@angular/core';
import { Store } from '@ngrx/store';
import * as ProductsActions from './state/products.actions';
import * as ProductsSelectors from './state/products.selectors';

@Component({
  selector: 'app-products',
  template: `
    <button (click)="reload()">Reload</button>

    <p *ngIf="loading$ | async">Loading...</p>
    <p *ngIf="(error$ | async) as err">Error: {{ err }}</p>

    <ul>
      <li *ngFor="let p of (products$ | async)" (click)="select(p.id)">
        {{ p.name }} — {{ p.price | currency }}
      </li>
    </ul>

    <ng-container *ngIf="(selected$ | async) as selected">
      <h3>Selected</h3>
      <pre>{{ selected | json }}</pre>
    </ng-container>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class ProductsComponent {
  products$ = this.store.select(ProductsSelectors.selectAllProducts);
  loading$ = this.store.select(ProductsSelectors.selectProductsLoading);
  error$ = this.store.select(ProductsSelectors.selectProductsError);
  selected$ = this.store.select(ProductsSelectors.selectSelectedProduct);

  constructor(private store: Store) {
    this.store.dispatch(ProductsActions.loadProducts());
  }

  reload() {
    this.store.dispatch(ProductsActions.loadProducts());
  }

  select(id: string) {
    this.store.dispatch(ProductsActions.selectProduct({ id }));
  }
}
```

### 6.3. Points pédagogiques
- le composant ne connaît **pas** l’API
- le composant ne calcule **pas** de dérivés : il consomme des selectors
- le composant déclenche des intentions via des actions

---

## 7. Tests : reducers, selectors, effects

### 7.1. Tester un reducer
Reducer = fonction pure -> test simple.

```ts
import { productsReducer } from './products.reducer';
import * as ProductsActions from './products.actions';
import { initialProductsState } from './products.state';

describe('productsReducer', () => {
  it('should set loading on loadProducts', () => {
    const state = productsReducer(initialProductsState, ProductsActions.loadProducts());
    expect(state.loading).toBeTrue();
    expect(state.error).toBeNull();
  });

  it('should store entities on loadProductsSuccess', () => {
    const products = [{ id: '1', name: 'A', price: 10 } as any];
    const state = productsReducer(
      { ...initialProductsState, loading: true },
      ProductsActions.loadProductsSuccess({ products })
    );

    expect(state.loading).toBeFalse();
    expect(state.ids).toEqual(['1']);
    expect(state.entities['1'].name).toBe('A');
  });
});
```

### 7.2. Tester un selector
On teste la projection pure (ou la composition).

```ts
import * as Selectors from './products.selectors';

describe('products selectors', () => {
  it('selectAllProducts should map ids to entities', () => {
    const ids = ['1', '2'];
    const entities: any = {
      '1': { id: '1', name: 'A' },
      '2': { id: '2', name: 'B' },
    };

    const result = Selectors.selectAllProducts.projector(ids, entities);
    expect(result.map(p => p.name)).toEqual(['A', 'B']);
  });
});
```

### 7.3. Tester un effect
Utiliser `provideMockActions` et un mock d’API.

```ts
import { TestBed } from '@angular/core/testing';
import { provideMockActions } from '@ngrx/effects/testing';
import { Observable, of, throwError } from 'rxjs';
import { ProductsEffects } from './products.effects';
import * as ProductsActions from './products.actions';

describe('ProductsEffects', () => {
  let actions$: Observable<any>;
  let effects: ProductsEffects;
  const api = {
    getAll: jasmine.createSpy(),
  };

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [
        ProductsEffects,
        provideMockActions(() => actions$),
        { provide: 'ProductsApi', useValue: api },
      ],
    });

    effects = TestBed.inject(ProductsEffects);
  });

  it('should dispatch loadProductsSuccess on success', (done) => {
    api.getAll.and.returnValue(of([{ id: '1', name: 'A', price: 10 }]));

    actions$ = of(ProductsActions.loadProducts());

    effects.loadProducts$.subscribe(action => {
      expect(action.type).toBe(ProductsActions.loadProductsSuccess.type);
      done();
    });
  });

  it('should dispatch loadProductsFailure on error', (done) => {
    api.getAll.and.returnValue(throwError(() => new Error('boom')));
    actions$ = of(ProductsActions.loadProducts());

    effects.loadProducts$.subscribe(action => {
      expect(action.type).toBe(ProductsActions.loadProductsFailure.type);
      done();
    });
  });
});
```

> Remarque : dans un vrai projet, fournissez l’API via un token Angular (`InjectionToken`) plutôt que la chaîne `'ProductsApi'`.

---

## 8. Bonnes pratiques et anti-patterns

### Bonnes pratiques
- **Actions** : nommage clair et orienté domaine (`[Products] Load Success`)
- **Reducers** : petits, focalisés, sans logique asynchrone
- **Selectors** : centraliser la logique de dérivation, mémoïsation naturelle
- **Effects** : une responsabilité par effect, gérer erreurs et concurrence
- **State** : normaliser les collections (entities), stocker le minimum, dériver le reste

### Anti-patterns fréquents
- Mettre du HTTP dans un reducer (interdit)
- Faire de l’agrégation/tri dans le composant au lieu d’un selector
- Mettre des objets “UI state” instables dans le store sans nécessité
- Multiplier les selectors factories appelés dans le template (création répétée)

---

## 9. Checklists et récapitulatif

### Checklist reducers
- [ ] pas d’effets de bord
- [ ] immutabilité respectée
- [ ] état initial défini
- [ ] actions `Success/Failure` cohérentes

### Checklist selectors
- [ ] feature selector
- [ ] selectors simples + composition
- [ ] dérivés dans selectors, pas dans UI
- [ ] vigilance sur création de nouvelles références

### Checklist effects
- [ ] opérateur de concurrence adapté (`switchMap/concatMap/...`)
- [ ] `catchError` renvoie une action failure
- [ ] effects sans dispatch pour navigation/logs

### Conclusion
- **Reducers** = mise à jour pure de l’état
- **Selectors** = vues dérivées performantes
- **Effects** = orchestration des effets secondaires

Bien conçus, ils rendent une application complexe **plus testable** et **plus lisible**.
