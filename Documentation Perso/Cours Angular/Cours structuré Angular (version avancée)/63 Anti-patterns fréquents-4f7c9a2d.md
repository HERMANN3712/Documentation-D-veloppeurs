# Formation Angular — Anti-patterns fréquents

- **Public** : développeurs Angular (junior à confirmé)
- **Durée suggérée** : 1 journée (6–7h) ou 2 demi‑journées
- **Pré‑requis** : bases Angular (components, services, RxJS, HttpClient), TypeScript
- **Objectifs** :
  - Identifier les anti‑patterns les plus fréquents en Angular
  - Comprendre leurs impacts (maintenabilité, performance, bugs)
  - Savoir refactorer vers des patterns recommandés
  - Mettre en place des garde‑fous (linters, conventions, revues, architectures)

---

## Plan de la formation

1. **Introduction**
   - Pourquoi les anti‑patterns apparaissent
   - Coût réel : complexité, dette, régressions, performance
   - Stratégie globale : rendre le code *facile à lire, tester, faire évoluer*

2. **Composants trop gros (God Components)**
   - Symptômes
   - Causes fréquentes
   - Solutions (smart/dumb, facades, directives, pipes)
   - Atelier : découpage d’un composant monolithique

3. **Subscriptions manuelles non nettoyées**
   - Memory leaks et effets de bord
   - Solutions : `async` pipe, `takeUntilDestroyed`, `DestroyRef`, `shareReplay` avec prudence
   - Atelier : refactor RxJS

4. **Appels HTTP dans les composants sans abstraction**
   - Problèmes : duplication, test difficile, coupling UI/infra
   - Patterns : Repository/Service, Data Access Layer, Facade, Resolver (avec prudence)
   - Atelier : extraction d’une couche d’accès données

5. **Templates trop complexes**
   - Symptômes : logique dans le HTML, `*ngIf` imbriqués, appels de méthodes, ternaires
   - Solutions : ViewModel, pipes purs, `@if/@for`, directives structurelles, composants de présentation
   - Atelier : simplification de template

6. **Usage excessif de `any`**
   - Pourquoi `any` tue TypeScript
   - Alternatives : `unknown`, types discriminants, generics, DTOs, `zod`/validation runtime
   - Atelier : typage progressif

7. **Partage d’état implicite**
   - Le piège des singletons et variables mutables partagées
   - Flux explicites : Inputs/Outputs, services d’état dédiés, RxJS store, signals
   - Atelier : rendre l’état explicite

8. **Services fourre‑tout (God Services)**
   - Symptômes
   - Découpage par responsabilités
   - Cohésion, boundaries, naming
   - Atelier : refactor en services plus petits

9. **Checklist et garde‑fous**
   - Conventions d’équipe
   - ESLint + règles utiles
   - Revue de code
   - Mesures : taille composants, couverture tests, complexité cyclomatique

10. **Conclusion & plan d’action**

---

# 1) Introduction

## 1.1 Pourquoi les anti‑patterns apparaissent

En Angular, les anti‑patterns sont souvent le résultat de :
- **Pression de livraison** : “ça marche, on ship”.
- **Manque d’architecture initiale** : pas de règles sur où placer la logique.
- **RxJS mal maîtrisé** : subscriptions manuelles, opérateurs à l’aveugle.
- **Équipe hétérogène** : styles différents, conventions absentes.

## 1.2 Impacts concrets

- **Maintenabilité** : toute modification casse autre chose.
- **Testabilité** : logique couplée au DOM, au HTTP, aux singletons.
- **Performance** : change detection surchargée, templates lourds.
- **Fiabilité** : memory leaks, états incohérents, conditions impossibles à raisonner.

## 1.3 Règle d’or

> Un composant doit surtout **orchestrer l’UI**, pas contenir toute la logique métier, pas gérer directement l’infrastructure, et éviter de gérer manuellement la mémoire RxJS.

---

# 2) Anti‑pattern : Composants trop gros (God Components)

## 2.1 Symptômes

- Fichier `.ts` très long (200–500+ lignes).
- Nombreuses responsabilités : fetch data, mapping, validation, navigation, gestion d’état, tracking, etc.
- Multiples subscriptions et timers.
- Template complexe et fortement couplé aux détails des données.

### Indicateurs
- Trop de `private` methods “helper”
- Beaucoup de `if/else` pour des cas métiers
- Trop de dépendances injectées (5–10+)

## 2.2 Exemple (anti‑pattern)

```ts
@Component({
  selector: 'app-orders',
  templateUrl: './orders.component.html',
})
export class OrdersComponent implements OnInit, OnDestroy {
  orders: Order[] = [];
  isLoading = false;
  error?: string;

  private sub = new Subscription();

  constructor(
    private http: HttpClient,
    private router: Router,
    private analytics: AnalyticsService,
    private toast: ToastService,
  ) {}

  ngOnInit(): void {
    this.isLoading = true;

    const s = this.http.get<OrderDto[]>('/api/orders')
      .subscribe({
        next: (dtos) => {
          this.orders = dtos.map(d => ({
            id: d.id,
            total: d.total_cents / 100,
            createdAt: new Date(d.created_at),
          }));
          this.analytics.track('orders_loaded', { count: this.orders.length });
          this.isLoading = false;
        },
        error: () => {
          this.error = 'Erreur de chargement';
          this.toast.error(this.error);
          this.isLoading = false;
        }
      });

    this.sub.add(s);
  }

  open(order: Order) {
    this.router.navigate(['/orders', order.id]);
  }

  ngOnDestroy(): void {
    this.sub.unsubscribe();
  }
}
```

## 2.3 Problèmes

- Le composant gère :
  - l’accès aux données (HTTP)
  - le mapping DTO→UI
  - orchestration UI (loaders, erreurs)
  - analytics et toasts
- Test unitaire difficile : il faut mocker trop de choses.
- Le refactor est risqué : une simple modif sur le mapping peut casser l’UI.

## 2.4 Refactor : séparation des responsabilités

### Option A — **Facade** (recommandé pour pages)

- Le composant consomme un **ViewModel**.
- La facade orchestre : chargements, mapping, erreurs.

```ts
@Injectable({ providedIn: 'root' })
export class OrdersFacade {
  private readonly http = inject(HttpClient);

  private readonly _state = new BehaviorSubject<{
    loading: boolean;
    error?: string;
    orders: Order[];
  }>({ loading: false, orders: [] });

  readonly vm$ = this._state.asObservable();

  load(): void {
    this._state.next({ ...this._state.value, loading: true, error: undefined });

    this.http.get<OrderDto[]>('/api/orders').pipe(
      map(dtos => dtos.map(mapOrderDtoToOrder)),
      catchError(() => {
        this._state.next({ ...this._state.value, loading: false, error: 'Erreur de chargement' });
        return EMPTY;
      })
    ).subscribe(orders => {
      this._state.next({ loading: false, orders });
    });
  }
}

function mapOrderDtoToOrder(d: OrderDto): Order {
  return {
    id: d.id,
    total: d.total_cents / 100,
    createdAt: new Date(d.created_at),
  };
}
```

Composant :

```ts
@Component({
  selector: 'app-orders',
  templateUrl: './orders.component.html',
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class OrdersComponent {
  private readonly facade = inject(OrdersFacade);

  readonly vm$ = this.facade.vm$;

  ngOnInit() {
    this.facade.load();
  }
}
```

Template :

```html
<ng-container *ngIf="vm$ | async as vm">
  <app-spinner *ngIf="vm.loading"></app-spinner>
  <app-error *ngIf="vm.error" [message]="vm.error"></app-error>

  <app-orders-list *ngIf="!vm.loading && !vm.error" [orders]="vm.orders"></app-orders-list>
</ng-container>
```

### Option B — split Smart/Presentational

- `OrdersPageComponent` (smart) : flux, VM
- `OrdersListComponent` (dumb) : affichage purement

## 2.5 Règles pratiques

- Un composant de page : orchestration uniquement.
- Déplacer :
  - mapping et transformations → **mappers / adapters**
  - logique métier → **domain services**
  - state UI → **facade / store**
- **Limiter les injections** : si un composant injecte 8 services, c’est un signal.

---

# 3) Anti‑pattern : Subscriptions manuelles non nettoyées

## 3.1 Symptômes

- `subscribe()` dans des composants/services sans `unsubscribe()`.
- Utilisation de `Subscription` agrégées mais oubli de cleanup.
- `setInterval`, `fromEvent` sans désinscription.

## 3.2 Pourquoi c’est dangereux

- **Fuites mémoire** : composants détruits mais flux actifs.
- **Bugs fantômes** : handlers exécutés plusieurs fois.
- **Performance** : accumulation de listeners.

## 3.3 Anti‑pattern (classique)

```ts
export class SearchComponent implements OnInit {
  results: Item[] = [];

  constructor(private api: ApiService) {}

  ngOnInit(): void {
    this.api.results$().subscribe(r => this.results = r);
  }
}
```

## 3.4 Refactors recommandés

### A) Privilégier `async` pipe

```ts
export class SearchComponent {
  readonly results$ = this.api.results$();
  constructor(private api: ApiService) {}
}
```

```html
<ul>
  <li *ngFor="let r of results$ | async">{{ r.name }}</li>
</ul>
```

### B) `takeUntilDestroyed` (Angular 16+)

```ts
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';

export class SearchComponent {
  private readonly destroyRef = inject(DestroyRef);

  ngOnInit(): void {
    this.api.results$().pipe(
      takeUntilDestroyed(this.destroyRef)
    ).subscribe(r => {
      // side effects contrôlés
    });
  }
}
```

### C) Éviter `subscribe` si possible

- `subscribe` est légitime pour :
  - navigation
  - analytics
  - trigger d’un `Subject` interne
  - bridging vers une lib non Rx

Mais dans ce cas, encapsuler et nettoyer.

## 3.5 Piège : `shareReplay` mal utilisé

```ts
const data$ = http.get('/api').pipe(shareReplay(1));
```

- Si la source ne complète pas (ex: `interval`, `fromEvent`), `shareReplay` peut retenir une référence.
- Préférer :
  - `shareReplay({ bufferSize: 1, refCount: true })` quand pertinent
  - ou des patterns store/signals.

---

# 4) Anti‑pattern : Appels HTTP dans les composants sans abstraction

## 4.1 Symptômes

- `HttpClient` injecté dans des composants.
- URLs, headers, params gérés dans l’UI.
- Mapping DTO fait dans le composant.

## 4.2 Problèmes

- **Couplage** UI ↔ API
- **Duplication** des endpoints
- **Tests** plus lourds (HttpTestingController dans tests de composants)
- **Évolution** difficile (changement d’API = refactor massif)

## 4.3 Pattern recommandé : couche Data Access

Créer un service dédié par ressource :

```ts
@Injectable({ providedIn: 'root' })
export class OrdersApi {
  private readonly http = inject(HttpClient);

  list$(): Observable<OrderDto[]> {
    return this.http.get<OrderDto[]>('/api/orders');
  }

  getById$(id: string): Observable<OrderDto> {
    return this.http.get<OrderDto>(`/api/orders/${id}`);
  }
}
```

Puis un **adapter** :

```ts
@Injectable({ providedIn: 'root' })
export class OrdersRepository {
  private readonly api = inject(OrdersApi);

  list$(): Observable<Order[]> {
    return this.api.list$().pipe(map(dtos => dtos.map(mapOrderDtoToOrder)));
  }
}
```

Composant :

```ts
export class OrdersComponent {
  readonly orders$ = this.repo.list$();
  constructor(private repo: OrdersRepository) {}
}
```

## 4.4 Bonus : Interceptors et configuration

- Auth headers, correlationId, baseUrl → **HttpInterceptors**
- Endpoints → `environment`/config

---

# 5) Anti‑pattern : Templates trop complexes

## 5.1 Symptômes

- Conditions imbriquées : `*ngIf` dans `*ngIf`.
- Ternaires multiplies.
- Appels de fonctions dans le template.
- Beaucoup de `| async` empilés.

## 5.2 Anti‑pattern : méthode appelée dans template

```html
<div *ngFor="let item of items">
  {{ computeLabel(item) }}
</div>
```

```ts
computeLabel(item: Item) {
  return expensiveTransform(item);
}
```

Problèmes :
- Réexécution à chaque cycle de change detection.
- Résultats instables s’il y a des effets.

## 5.3 Refactors

### A) Pré-calcul côté VM

```ts
readonly vm$ = this.items$.pipe(
  map(items => items.map(i => ({
    ...i,
    label: expensiveTransform(i)
  })))
);
```

```html
<div *ngFor="let item of (vm$ | async)">
  {{ item.label }}
</div>
```

### B) Utiliser des **pipes purs**

```ts
@Pipe({ name: 'itemLabel', pure: true })
export class ItemLabelPipe {
  transform(item: Item): string {
    return expensiveTransform(item);
  }
}
```

```html
{{ item | itemLabel }}
```

### C) Factoriser en composants

- Extraire une section UI répétée en `ItemRowComponent`.
- Le template parent reste lisible.

### D) Utiliser `@if` / `@for` (Angular récent)

- Plus lisible, scope plus clair, évite certains pièges.

---

# 6) Anti‑pattern : usage excessif de `any`

## 6.1 Symptômes

- `any` utilisé pour “aller plus vite”.
- Types manquants sur les retours d’API.
- Méthodes utilitaires reçoivent/retournent `any`.

## 6.2 Pourquoi c’est coûteux

- Perte de l’autocomplétion et refactors dangereux.
- Bugs runtime (champ absent, format inattendu).
- Propagation : un `any` contamine le code.

## 6.3 Alternatives

### A) `unknown` + narrowing

```ts
function parse(x: unknown): number {
  if (typeof x !== 'number') throw new Error('Expected number');
  return x;
}
```

### B) DTO typés

```ts
export interface OrderDto {
  id: string;
  total_cents: number;
  created_at: string;
}
```

### C) Types discriminants

```ts
type ApiResult<T> =
  | { kind: 'ok'; data: T }
  | { kind: 'error'; message: string };
```

### D) Validation runtime (optionnel)

- `zod`, `io-ts`, `valibot` pour valider les payloads d’API.

---

# 7) Anti‑pattern : partage d’état implicite

## 7.1 Symptômes

- Services singletons utilisés comme “global variables”.
- Mutations directes : `service.items.push(...)`.
- Composants qui se synchronisent via effets cachés.

## 7.2 Problèmes

- Ordre d’exécution critique.
- Effets de bord : un composant change l’état d’un autre sans contract.
- Tests instables (état qui fuit entre tests).

## 7.3 Pattern : rendre l’état explicite

### A) Flux Input/Output

- L’état appartient au parent, le child émet des actions.

### B) Store dédié (RxJS)

```ts
@Injectable()
export class CartStore {
  private readonly _items = new BehaviorSubject<CartItem[]>([]);
  readonly items$ = this._items.asObservable();

  add(item: CartItem) {
    this._items.next([...this._items.value, item]);
  }
}
```

Fournir au bon niveau (feature) :

```ts
@Component({
  providers: [CartStore]
})
export class CartPageComponent {}
```

### C) Signals (Angular)

```ts
@Injectable()
export class CartSignalsStore {
  readonly items = signal<CartItem[]>([]);
  add(item: CartItem) {
    this.items.update(v => [...v, item]);
  }
}
```

Guidelines :
- **Éviter les mutations** invisibles
- Préférer : `next([...old, new])`, `update` immuable
- Scoper l’état : feature > page > composant

---

# 8) Anti‑pattern : services fourre‑tout (God Services)

## 8.1 Symptômes

- `AppService`, `CommonService`, `UtilsService`.
- Mélange : HTTP + state + formatting + navigation.
- Fichier qui grossit à mesure que l’app grandit.

## 8.2 Problèmes

- Faible cohésion, dépendances en chaîne.
- Tests compliqués (beaucoup de mocks).
- Tout devient “central”, bloquant.

## 8.3 Stratégies de découpage

### A) Découpage par responsabilité

- `OrdersApi` : calls HTTP
- `OrdersRepository` : mapping, cache
- `OrdersFacade` : orchestration UI
- `OrdersAnalytics` : tracking

### B) Naming rules

- `*Api` = infrastructure HTTP
- `*Repository` = abstractions data
- `*Facade` / `*Store` = state/ViewModel
- `*Mapper` = transformations

### C) Dependency direction

- UI dépend de Facade/Store
- Facade dépend de Repository
- Repository dépend de Api
- Api dépend de HttpClient

---

# 9) Checklist & garde‑fous

## 9.1 Checklist revue de code

- [ ] Le composant dépasse-t-il ~150–200 lignes ? Peut-on extraire ?
- [ ] Y a-t-il des `subscribe()` ? Sont-ils nécessaires ? Cleanup ?
- [ ] `HttpClient` est-il injecté dans un composant ? Pourquoi ?
- [ ] Le template contient-il de la logique lourde ?
- [ ] `any` est-il évitable ?
- [ ] L’état est-il explicite, immuable, et bien scopé ?
- [ ] Un service fait-il “trop de choses” ?

## 9.2 ESLint/TS config utiles

- `@typescript-eslint/no-explicit-any`
- `rxjs/no-ignored-subscription`
- `rxjs/no-exposed-subjects`
- strict TS :
  - `"strict": true`
  - `"noImplicitAny": true`

## 9.3 Métriques pragmatiques

- Taille max composant (soft limit)
- Nombre max d’injections
- Complexité du template (sections/conditions)

---

# 10) Conclusion & plan d’action

## Plan d’action recommandé (équipe)

1. **Activer TypeScript strict** et réduire progressivement `any`.
2. **Standardiser** :
   - pas de HTTP dans les composants
   - `async` pipe par défaut
   - facades/stores pour pages
3. **Mettre une checklist** de revue de code (section 9.1).
4. **Refactor itératif** : 1 anti‑pattern par sprint.

## Résultat attendu

- Composants plus petits, plus lisibles
- Moins de bugs liés aux subscriptions
- Templates maintenables
- Architecture plus testable et scalable

---

## Annexes — mini ateliers (énoncés)

### Atelier 1 — Découper un God Component
- Prendre une page “liste + filtre + détails”.
- Extraire :
  - un composant présentational
  - une facade
  - un mapper DTO→VM

### Atelier 2 — RxJS cleanup
- Identifier 3 subscriptions manuelles.
- Remplacer 2 par `async` pipe.
- Remplacer 1 par `takeUntilDestroyed` (side effect légitime).

### Atelier 3 — Simplifier un template
- Supprimer les appels de méthodes.
- Remplacer les ternaires imbriqués par un VM `status`.
- Extraire un composant `EmptyState`.
