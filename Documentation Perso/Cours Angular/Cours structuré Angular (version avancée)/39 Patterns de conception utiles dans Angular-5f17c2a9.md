# Formation — Patterns de conception utiles dans Angular

- **Public** : développeurs Angular (intermédiaire)
- **Durée suggérée** : 1 journée (6–7h) ou 2 demi-journées
- **Pré-requis** : TypeScript, bases d’Angular (components, services, DI), bases de RxJS
- **Objectifs pédagogiques** :
  - Identifier des **patterns** pertinents dans une application Angular.
  - Structurer l’accès à une *feature* via le **Facade pattern**.
  - Isoler la transformation de données via l’**Adapter pattern**.
  - Rendre un comportement interchangeable avec **Strategy**.
  - Maîtriser l’**Observer** (RxJS) pour orchestrer flux et état.
  - Appliquer la **Dependency Inversion** grâce à l’injection de dépendances.

---

## Plan de la formation

1. **Introduction : pourquoi des patterns dans Angular ?**
   - Patterns vs “bonnes pratiques”
   - Composition Angular (components/services/modules) et points d’extension
   - Principes associés : SOLID, séparation des responsabilités, testabilité

2. **Facade Pattern : simplifier l’accès à une feature**
   - Intention et bénéfices
   - Anti-patterns fréquents (component qui orchestre tout)
   - Mise en pratique : *Feature Facade* (state + appels API + mapping)
   - Tests : isoler composants et façade

3. **Adapter Pattern : transformer les données d’API**
   - DTO vs Domain Model
   - Mapping, normalisation, compatibilité de versions
   - Où placer l’adapter (data layer) ?
   - Erreurs courantes : mapping dispersé

4. **Strategy Pattern : changer de comportement dynamiquement**
   - Famille d’algorithmes interchangeables
   - Sélection à l’exécution (runtime)
   - Variante Angular : multi-providers, tokens, injection conditionnelle

5. **Observer Pattern via RxJS : événements, flux, état**
   - Observables/Observers/Subscriptions
   - Subjects (BehaviorSubject/ReplaySubject) et usages
   - Composition (map/switchMap/shareReplay)
   - Patterns RxJS utiles : service de store léger, destroy lifecycle

6. **Dependency Inversion avec l’injection de dépendances Angular**
   - Dépendre d’abstractions (interfaces/tokens)
   - InjectionToken, providers, useClass/useFactory/useValue
   - “Ports & Adapters” (Hexagonal) à petite échelle

7. **Atelier final : combiner les patterns**
   - Cas complet : feature “Produits”
   - Objectifs : façade + adapters + stratégies + RxJS + DIP

8. **Checklists & conclusion**
   - Quand utiliser quel pattern ?
   - Impact sur la maintenance et les tests

---

# 1) Introduction : pourquoi des patterns dans Angular ?

Angular fournit déjà une structure (composants, services, DI, RxJS). Les **patterns de conception** apportent :

- **Lisibilité** : un langage commun dans l’équipe (“on passe par la façade”, “on adapte les DTOs”).
- **Évolutivité** : ajouter des fonctionnalités sans casser l’existant.
- **Testabilité** : remplacer facilement une dépendance, isoler le code.
- **Robustesse** : réduction des effets “spaghetti” (mapping partout, logique métier dans les composants, etc.).

### Pattern vs architecture
- Un **pattern** = solution réutilisable à un problème récurrent (ex. Strategy).
- Une **architecture** = organisation globale (ex. Clean Architecture / Hexagonal).

### Règle pratique
> Un pattern est utile si, et seulement si, il **réduit la complexité** à long terme (maintenance, tests, changements).

---

# 2) Facade Pattern dans Angular

## 2.1 Intention
Le **Facade pattern** fournit une *API simplifiée* vers un sous-système complexe.

Dans Angular, une **feature facade** sert souvent à :
- centraliser les **appels API**,
- gérer un **état** (loading, data, error),
- exposer des **Observables** simples aux composants,
- encapsuler la composition RxJS (combineLatest, switchMap, etc.),
- masquer la complexité (cache, rafraîchissement, pagination).

## 2.2 Symptômes indiquant qu’une façade est nécessaire
- Les composants :
  - injectent plusieurs services (API, store, router, logger…),
  - contiennent trop de logique RxJS,
  - déclenchent de multiples effets (toast, navigation, tracking).
- Du code dupliqué pour charger/rafraîchir une même donnée.

## 2.3 Exemple : Feature “Produits” (version simple)

### 2.3.1 Contrats
- API renvoie des `ProductDto`
- UI manipule des `Product` (modèle “métier” ou “domaine”)

```ts
// product.model.ts
export type ProductId = string;

export interface Product {
  id: ProductId;
  name: string;
  price: number;
  inStock: boolean;
}

export interface ProductDto {
  product_id: string;
  product_name: string;
  price_cents: number;
  stock: 'Y' | 'N';
}
```

### 2.3.2 Service API (data access)
```ts
// product-api.service.ts
import { HttpClient } from '@angular/common/http';
import { Injectable } from '@angular/core';
import { Observable } from 'rxjs';
import { ProductDto } from './product.model';

@Injectable({ providedIn: 'root' })
export class ProductApiService {
  constructor(private http: HttpClient) {}

  list(): Observable<ProductDto[]> {
    return this.http.get<ProductDto[]>('/api/products');
  }
}
```

### 2.3.3 Façade : état minimal et API simple
```ts
// product.facade.ts
import { Injectable } from '@angular/core';
import { BehaviorSubject, Observable, catchError, map, of, shareReplay, tap } from 'rxjs';
import { ProductApiService } from './product-api.service';
import { Product, ProductDto } from './product.model';

@Injectable({ providedIn: 'root' })
export class ProductFacade {
  private readonly reload$ = new BehaviorSubject<void>(undefined);

  private readonly loadingSubject = new BehaviorSubject<boolean>(false);
  readonly loading$ = this.loadingSubject.asObservable();

  private readonly errorSubject = new BehaviorSubject<string | null>(null);
  readonly error$ = this.errorSubject.asObservable();

  // Flux de données "produits"
  readonly products$: Observable<Product[]> = this.reload$.pipe(
    tap(() => {
      this.loadingSubject.next(true);
      this.errorSubject.next(null);
    }),
    // switchMap serait pertinent si rechargement rapide; ici map->api simple
    // On garde simple pour l’exemple
    map(() => null),
    // on appelle l’API via un stream séparé pour lisibilité
    // (dans une vraie app, utilisez switchMap)
    //
    // Pour l’exemple, remplaçons par switchMap :
  );

  // Version corrigée avec switchMap
  readonly products2$: Observable<Product[]> = this.reload$.pipe(
    tap(() => {
      this.loadingSubject.next(true);
      this.errorSubject.next(null);
    }),
    // recharge déclenche un nouvel appel
    // (si un reload intervient avant la fin, switchMap annule l’appel précédent)
    // eslint-disable-next-line rxjs/no-implicit-any-catch
    // (la règle dépend de vos configs)
    //
    // Importez switchMap :
  );

  constructor(private api: ProductApiService) {
    // Implémentation réelle dans le constructeur pour éviter doublon dans l’extrait.
    this.products2$ = this.reload$.pipe(
      tap(() => {
        this.loadingSubject.next(true);
        this.errorSubject.next(null);
      }),
      // lazy import dans snippet
      // @ts-ignore
      switchMap(() => this.api.list()),
      map((dtos: ProductDto[]) => dtos.map(this.toProduct)),
      tap(() => this.loadingSubject.next(false)),
      catchError((err) => {
        this.loadingSubject.next(false);
        this.errorSubject.next('Erreur de chargement des produits');
        return of([] as Product[]);
      }),
      // partage la dernière valeur entre plusieurs composants
      shareReplay({ bufferSize: 1, refCount: true })
    );
  }

  reload(): void {
    this.reload$.next();
  }

  private toProduct(dto: ProductDto): Product {
    return {
      id: dto.product_id,
      name: dto.product_name,
      price: dto.price_cents / 100,
      inStock: dto.stock === 'Y'
    };
  }
}
```

> Remarque pédagogique : l’extrait ci-dessus introduit `switchMap`. En production, on évite les “@ts-ignore” et on organise les imports proprement. L’idée clé : **le composant n’a pas à connaître le détail des appels**.

### 2.3.4 Utilisation côté composant
```ts
// product-list.component.ts
import { ChangeDetectionStrategy, Component } from '@angular/core';
import { ProductFacade } from './product.facade';

@Component({
  selector: 'app-product-list',
  template: `
    <button (click)="facade.reload()">Recharger</button>

    <p *ngIf="(facade.loading$ | async)">Chargement...</p>
    <p *ngIf="(facade.error$ | async) as error">{{ error }}</p>

    <ul>
      <li *ngFor="let p of (facade.products2$ | async)">
        {{ p.name }} — {{ p.price | currency:'EUR' }}
      </li>
    </ul>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class ProductListComponent {
  constructor(public facade: ProductFacade) {
    this.facade.reload();
  }
}
```

### 2.3.5 Bénéfices
- Le composant : simple, testable, orienté UI.
- La façade : point d’entrée unique, facilement moquable en tests.
- Le mapping et l’état : centralisés.

### 2.3.6 Tests (idée)
- Test du composant : fournir un *mock* de façade.
- Test de façade : utiliser `HttpTestingController` au niveau du service API, ou mocker `ProductApiService`.

---

# 3) Adapter Pattern : transformer les données d’API

## 3.1 Intention
Un **Adapter** convertit une interface en une autre attendue par le client.

Dans Angular :
- convertir des **DTO** côté API (noms, formats, types) en modèles utilisés par la UI/domaine.
- isoler la logique de transformation (dates, monnaie, enums, champs optionnels).

## 3.2 Pourquoi c’est important
- Votre UI ne doit pas dépendre d’un format API instable.
- Vous évitez la propagation du mapping dans 10 composants.
- Facilite les migrations API (v1 → v2) : on modifie l’adapter, pas toute l’app.

## 3.3 Mise en place

### 3.3.1 Adapter dédié
```ts
// product.adapter.ts
import { Injectable } from '@angular/core';
import { Product, ProductDto } from './product.model';

@Injectable({ providedIn: 'root' })
export class ProductAdapter {
  fromDto(dto: ProductDto): Product {
    return {
      id: dto.product_id,
      name: dto.product_name,
      price: dto.price_cents / 100,
      inStock: dto.stock === 'Y'
    };
  }

  toDto(model: Product): ProductDto {
    return {
      product_id: model.id,
      product_name: model.name,
      price_cents: Math.round(model.price * 100),
      stock: model.inStock ? 'Y' : 'N'
    };
  }
}
```

### 3.3.2 Utiliser l’adapter dans la façade
```ts
// product.facade.ts (extrait)
constructor(private api: ProductApiService, private adapter: ProductAdapter) {
  this.products$ = this.reload$.pipe(
    tap(() => this.loadingSubject.next(true)),
    // @ts-ignore
    switchMap(() => this.api.list()),
    map(dtos => dtos.map(d => this.adapter.fromDto(d))),
    tap(() => this.loadingSubject.next(false)),
    shareReplay({ bufferSize: 1, refCount: true })
  );
}
```

## 3.4 Bonnes pratiques
- **Un adapter par ressource** (UserAdapter, ProductAdapter…).
- S’assurer que l’adapter est **pur** (pas d’effet de bord) → test unitaire facile.
- Documenter les règles de mapping (ex. cents → euros).

## 3.5 Exercice
- Ajouter un champ `created_at` (ISO string) dans DTO → `createdAt: Date` dans modèle.
- Ajouter la conversion dans l’adapter + tests.

---

# 4) Strategy Pattern : changer de comportement dynamiquement

## 4.1 Intention
Le pattern **Strategy** encapsule des algorithmes interchangeables derrière une même interface.

Dans Angular, stratégies typiques :
- différents modes de tri/filtre,
- choix d’un backend (mock vs réel),
- calcul de prix selon pays / règles,
- formatage ou validation variable.

## 4.2 Exemple : stratégies de tri des produits

### 4.2.1 Contrat
```ts
// sort.strategy.ts
import { Product } from './product.model';

export interface ProductSortStrategy {
  key: 'price-asc' | 'price-desc' | 'name-asc';
  sort(items: Product[]): Product[];
}
```

### 4.2.2 Implémentations
```ts
// sort-by-price-asc.strategy.ts
import { Injectable } from '@angular/core';
import { Product } from './product.model';
import { ProductSortStrategy } from './sort.strategy';

@Injectable()
export class SortByPriceAscStrategy implements ProductSortStrategy {
  key: 'price-asc' = 'price-asc';
  sort(items: Product[]): Product[] {
    return [...items].sort((a, b) => a.price - b.price);
  }
}

@Injectable()
export class SortByPriceDescStrategy implements ProductSortStrategy {
  key: 'price-desc' = 'price-desc';
  sort(items: Product[]): Product[] {
    return [...items].sort((a, b) => b.price - a.price);
  }
}

@Injectable()
export class SortByNameAscStrategy implements ProductSortStrategy {
  key: 'name-asc' = 'name-asc';
  sort(items: Product[]): Product[] {
    return [...items].sort((a, b) => a.name.localeCompare(b.name));
  }
}
```

### 4.2.3 Fournir plusieurs stratégies (multi-provider)
```ts
// sort.tokens.ts
import { InjectionToken } from '@angular/core';
import { ProductSortStrategy } from './sort.strategy';

export const PRODUCT_SORT_STRATEGIES = new InjectionToken<ProductSortStrategy[]>(
  'PRODUCT_SORT_STRATEGIES'
);
```

```ts
// product-feature.providers.ts
import { Provider } from '@angular/core';
import { PRODUCT_SORT_STRATEGIES } from './sort.tokens';
import { SortByNameAscStrategy, SortByPriceAscStrategy, SortByPriceDescStrategy } from './sort-by-price-asc.strategy';

export const PRODUCT_FEATURE_PROVIDERS: Provider[] = [
  SortByPriceAscStrategy,
  SortByPriceDescStrategy,
  SortByNameAscStrategy,
  { provide: PRODUCT_SORT_STRATEGIES, useFactory: (
      a: SortByPriceAscStrategy,
      b: SortByPriceDescStrategy,
      c: SortByNameAscStrategy
    ) => [a,b,c],
    deps: [SortByPriceAscStrategy, SortByPriceDescStrategy, SortByNameAscStrategy]
  }
];
```

### 4.2.4 Sélecteur de stratégie
```ts
// product-sort.service.ts
import { Inject, Injectable } from '@angular/core';
import { PRODUCT_SORT_STRATEGIES } from './sort.tokens';
import { ProductSortStrategy } from './sort.strategy';
import { Product } from './product.model';

@Injectable({ providedIn: 'root' })
export class ProductSortService {
  constructor(@Inject(PRODUCT_SORT_STRATEGIES) private strategies: ProductSortStrategy[]) {}

  sort(key: ProductSortStrategy['key'], items: Product[]): Product[] {
    const strat = this.strategies.find(s => s.key === key);
    return strat ? strat.sort(items) : items;
  }
}
```

### 4.2.5 Intégration avec RxJS (UI dynamique)
```ts
// product-list.component.ts (extrait)
import { BehaviorSubject, combineLatest, map } from 'rxjs';
import { ProductSortService } from './product-sort.service';
import { ProductFacade } from './product.facade';

sortKey$ = new BehaviorSubject<'price-asc'|'price-desc'|'name-asc'>('name-asc');

vm$ = combineLatest([
  this.facade.products2$,
  this.sortKey$
]).pipe(
  map(([products, key]) => ({
    products: this.sortService.sort(key, products),
    sortKey: key
  }))
);

constructor(public facade: ProductFacade, private sortService: ProductSortService) {}
```

## 4.3 Bénéfices
- Ajout d’un nouveau mode de tri = nouvelle classe, pas de `switch` géant.
- Testabilité : chaque stratégie se teste isolément.
- Extensible via DI : stratégies ajoutées progressivement.

---

# 5) Observer Pattern via RxJS

## 5.1 Intention
Le pattern **Observer** décrit une relation *publisher/subscriber* :
- un *Observable* (source) émet des valeurs,
- un *Observer* (abonné) réagit.

Dans Angular, RxJS est partout :
- `HttpClient` retourne des Observables,
- `FormControl.valueChanges`,
- `@Output()` s’apparente à un flux d’événements,
- état et événements UI.

## 5.2 Rappels RxJS utiles
- **Cold Observable** : l’exécution démarre à la souscription (HTTP).
- **Hot Observable** : source partagée (Subject, événements).

### Operators essentiels
- `map` : projection
- `switchMap` : annule le flux précédent lors d’un nouveau trigger
- `mergeMap` : parallèle
- `concatMap` : séquentiel
- `catchError` : gestion d’erreurs
- `shareReplay` : cache + partage (attention au refCount)

## 5.3 Pattern “Store léger” via BehaviorSubject
Cas d’usage : gérer un état local de feature sans lib externe.

```ts
// feature-store.ts
import { Injectable } from '@angular/core';
import { BehaviorSubject } from 'rxjs';

export interface FeatureState {
  loading: boolean;
  error: string | null;
}

@Injectable({ providedIn: 'root' })
export class FeatureStore {
  private stateSubject = new BehaviorSubject<FeatureState>({ loading: false, error: null });
  readonly state$ = this.stateSubject.asObservable();

  patch(partial: Partial<FeatureState>): void {
    this.stateSubject.next({ ...this.stateSubject.value, ...partial });
  }
}
```

> Une façade peut englober ce store ou l’inverse : l’important est de **centraliser l’état**.

## 5.4 Gestion du cycle de vie : éviter les fuites
Angular propose `DestroyRef` et `takeUntilDestroyed`.

```ts
import { Component, DestroyRef, inject } from '@angular/core';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';

@Component({
  selector: 'app-demo',
  template: `...`
})
export class DemoComponent {
  private destroyRef = inject(DestroyRef);

  ngOnInit() {
    // someObservable$
    //   .pipe(takeUntilDestroyed(this.destroyRef))
    //   .subscribe();
  }
}
```

## 5.5 Exercice
- Ajouter une recherche `query$` (BehaviorSubject).
- Utiliser `debounceTime`, `distinctUntilChanged`, `switchMap` pour interroger l’API.

---

# 6) Dependency Inversion (DIP) grâce à l’injection de dépendances

## 6.1 Intention (SOLID)
Le principe d’inversion des dépendances :
- Les modules de haut niveau ne dépendent pas de modules de bas niveau.
- Les deux dépendent d’**abstractions**.

Dans Angular, on met en œuvre le DIP via :
- interfaces (au niveau TypeScript) + `InjectionToken`
- providers configurables (`useClass`, `useFactory`, `useValue`)

## 6.2 Exemple : abstraction d’un “port” de produits

### 6.2.1 Définir un contrat (port)
```ts
// product.port.ts
import { Observable } from 'rxjs';
import { Product } from './product.model';

export interface ProductPort {
  list(): Observable<Product[]>;
}
```

### 6.2.2 Créer un token d’injection
```ts
// product.tokens.ts
import { InjectionToken } from '@angular/core';
import { ProductPort } from './product.port';

export const PRODUCT_PORT = new InjectionToken<ProductPort>('PRODUCT_PORT');
```

### 6.2.3 Implémentation “API” (adapter sortant)
```ts
// product-api.adapter.ts
import { Injectable } from '@angular/core';
import { Observable, map } from 'rxjs';
import { ProductPort } from './product.port';
import { Product } from './product.model';
import { ProductApiService } from './product-api.service';
import { ProductAdapter } from './product.adapter';

@Injectable()
export class ProductApiAdapter implements ProductPort {
  constructor(private api: ProductApiService, private adapter: ProductAdapter) {}

  list(): Observable<Product[]> {
    return this.api.list().pipe(
      map(dtos => dtos.map(d => this.adapter.fromDto(d)))
    );
  }
}
```

### 6.2.4 Implémentation “Mock” (tests / dev)
```ts
// product-mock.adapter.ts
import { Injectable } from '@angular/core';
import { Observable, of } from 'rxjs';
import { ProductPort } from './product.port';
import { Product } from './product.model';

@Injectable()
export class ProductMockAdapter implements ProductPort {
  list(): Observable<Product[]> {
    return of([
      { id: 'p1', name: 'Mock A', price: 10, inStock: true },
      { id: 'p2', name: 'Mock B', price: 20, inStock: false }
    ]);
  }
}
```

### 6.2.5 Fournir l’implémentation via configuration
```ts
// app.config.ts ou product.providers.ts
import { Provider } from '@angular/core';
import { PRODUCT_PORT } from './product.tokens';
import { ProductApiAdapter } from './product-api.adapter';
import { ProductMockAdapter } from './product-mock.adapter';

export function productPortFactory(): any {
  // Exemple : basé sur une variable d’environnement
  const useMock = false;
  return useMock ? new ProductMockAdapter() : new ProductApiAdapter(undefined as any, undefined as any);
}

// ⚠️ Préférez deps + useClass/useFactory correctement.
// Exemple propre :
export const PRODUCT_PORT_PROVIDER: Provider = {
  provide: PRODUCT_PORT,
  useClass: ProductApiAdapter
  // en dev, remplacez par ProductMockAdapter
};
```

### 6.2.6 Consommer le port (dans la façade)
```ts
import { Inject, Injectable } from '@angular/core';
import { PRODUCT_PORT } from './product.tokens';
import { ProductPort } from './product.port';

@Injectable({ providedIn: 'root' })
export class ProductFacade2 {
  constructor(@Inject(PRODUCT_PORT) private productPort: ProductPort) {}

  // list() délègue au port : la façade dépend d’une abstraction
}
```

## 6.3 Bénéfices
- Remplacement backend/mock sans modifier la logique.
- Tests simplifiés : injecter un faux port.
- Découplage : l’UI ne dépend pas d’HttpClient.

---

# 7) Atelier final : combiner les patterns

## 7.1 Objectif
Construire une feature “Produits” qui :
- expose une **façade** (`ProductFacade`),
- récupère les données via un **port** (`ProductPort`) (DIP),
- transforme via **adapters** (`ProductAdapter`),
- propose des comportements via **strategies** (tri),
- orchestre l’ensemble via **RxJS**.

## 7.2 Étapes guidées
1. Créer `ProductDto`/`Product`.
2. Créer `ProductAdapter`.
3. Créer `ProductPort` + `PRODUCT_PORT`.
4. Implémenter `ProductApiAdapter` et le provider.
5. Créer `ProductFacade` :
   - `loading$`, `error$`, `products$`
   - méthode `reload()`
6. Ajouter `ProductSortStrategy` + stratégies.
7. Dans `ProductListComponent` construire un `vm$` via `combineLatest`.

## 7.3 Définition d’un ViewModel (recommandé)
```ts
export interface ProductListVm {
  loading: boolean;
  error: string | null;
  products: Product[];
  sortKey: 'price-asc'|'price-desc'|'name-asc';
}
```

> Pattern complémentaire : **ViewModel** (ou Presenter) — pas demandé explicitement, mais très utile pour rendre le template minimal.

---

# 8) Checklists & conclusion

## 8.1 Quand utiliser quel pattern ?

### Facade
✅ Quand plusieurs composants consomment les mêmes données/état, ou quand l’orchestration RxJS devient complexe.

Éviter si : feature triviale avec très peu de logique (risque de surcouche inutile).

### Adapter
✅ Dès qu’il existe une différence DTO ↔ modèle de l’app (noms, types, formats).

Éviter si : API et modèle sont strictement identiques et stables (rare).

### Strategy
✅ Quand un `switch/case` ou une logique conditionnelle se multiplie et que vous anticipez de nouveaux comportements.

### Observer/RxJS
✅ Pour synchroniser événements, requêtes, et état, et éviter les callbacks/logique dispersée.

### Dependency Inversion (DI)
✅ Quand vous voulez isoler le code “haut niveau” (façade/feature) des détails (HTTP, storage, APIs externes).

## 8.2 Règles d’or
- Garder les composants orientés **présentation**.
- Centraliser le *mapping* (Adapter) et l’orchestration (Facade).
- Rendre les comportements interchangeables (Strategy) plutôt que conditionnels.
- Maintenir une DI propre (DIP) : dépendre d’abstractions.

---

## Annexes : suggestions d’organisation (folders)

Exemple de structure :

```
/products
  /data-access
    product-api.service.ts
    product-api.adapter.ts
    product.mock.adapter.ts
    product.adapter.ts
    product.port.ts
    product.tokens.ts
  /feature
    product.facade.ts
    product-feature.providers.ts
  /ui
    product-list.component.ts
  /domain
    product.model.ts
  /strategies
    sort.strategy.ts
    sort.tokens.ts
    sort-by-*.strategy.ts
```

---

### Notes de qualité (à discuter en formation)
- `shareReplay` : attention aux comportements de cache et aux souscriptions.
- Préférer `switchMap` pour les rechargements afin d’éviter des réponses hors ordre.
- Pour la production : ajouter logs, typed errors, gestion empty states.
