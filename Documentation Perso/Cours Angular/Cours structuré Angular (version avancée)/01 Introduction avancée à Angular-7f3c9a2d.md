# Formation : Introduction avancée à Angular

> Public cible : développeurs ayant déjà pratiqué Angular (bases composants/modules/routing/forms) et souhaitant monter en compétence sur l’architecture, la performance et l’industrialisation.

## Objectifs pédagogiques

À l’issue de la formation, le participant sera capable de :

- Expliquer l’architecture interne d’Angular (DI, compilation, zones, change detection).
- Choisir et appliquer une stratégie de détection de changements adaptée (Default vs OnPush) et optimiser les templates.
- Maîtriser la programmation réactive avec RxJS et éviter les pièges fréquents (fuites mémoire, multi-subscriptions).
- Mettre en place une gestion d’état (pattern maison, services, ComponentStore/Ngrx) et isoler les effets.
- Structurer un projet scalable (feature modules/standalone, dossiers, conventions).
- Implémenter le routing avancé (lazy loading, guards, resolvers, préchargement) et gérer la navigation.
- Optimiser les performances runtime et build (bundle, images, change detection, preloading).
- Industrialiser l’application (tests, lint, CI, build profiles, i18n) avec bonnes pratiques.

## Prérequis

- TypeScript (interfaces, generics, async/await).
- Angular : composants, directives, pipes, services, routing (basique), forms.
- Connaissance ES6 + notions HTML/CSS.

## Format & organisation

- Durée indicative : 2 jours (adaptable 1 à 3 jours)
- Alternance : théorie (40 %) / ateliers (60 %)
- Support : ce document + dépôt d’exercices

---

# Plan détaillé

1. **Architecture interne d’Angular**
   1. Runtime, compilation, zones
   2. Injection de dépendances (DI) avancée
   3. Modules vs Standalone, scopes et providers
2. **Change Detection & performances templates**
   1. Mécanisme de détection de changements
   2. OnPush, immutabilité, signals (panorama)
   3. TrackBy, pipes purs/impurs, async pipe
3. **RxJS avancé pour Angular**
   1. Observables, Hot/Cold, Subjects
   2. Patterns Angular (async pipe, takeUntilDestroyed)
   3. Opérateurs clés (switchMap, concatMap, exhaustMap, shareReplay)
4. **Gestion d’état (State Management)**
   1. État local vs global
   2. Pattern “smart/dumb”, façade, store léger
   3. Écosystèmes (ComponentStore/Ngrx) et bonnes pratiques
5. **Routing avancé & Lazy loading**
   1. Lazy loading de features
   2. Guards, resolvers, préchargement
   3. Navigation, params, data, erreurs
6. **Optimisation des performances**
   1. Profilage (Angular DevTools)
   2. Budgets, bundles, code splitting
   3. SSR/hydration (survol), images, fonts
7. **Industrialisation & qualité**
   1. Architecture de projet, convention, monorepo
   2. Tests unitaires & intégration
   3. CI/CD, environnements, versioning

---

# 1) Architecture interne d’Angular

## 1.1 Angular en quelques couches

Angular s’articule autour de :

- **Template + Component class** : UI déclarative et logique.
- **Change Detection** : synchronise modèle → vue.
- **Dependency Injection (DI)** : fournit et compose les services.
- **Router** : navigation, composition de pages.
- **Forms** : modèle et validation.
- **HttpClient** : communication API.

### Compilation : JIT vs AOT

- **AOT (Ahead-of-Time)** : compilation au build, plus rapide au runtime, plus sûr.
- **JIT (Just-in-Time)** : compilation au runtime (souvent dev), plus souple mais plus lourd.

Aujourd’hui, Angular CLI privilégie l’AOT en prod (et souvent même en dev selon versions).

### Zone.js (concept)

Historiquement Angular s’appuie sur **Zone.js** pour détecter les événements async (timers, XHR, promesses…). Cela déclenche la change detection.

Conséquence : trop d’événements async peuvent entraîner trop de cycles de change detection.

> Note : Angular évolue vers des APIs plus “zoneless” avec notamment les **signals** et des optimisations, mais comprendre Zone/CD reste essentiel.

---

## 1.2 Dependency Injection (DI) avancée

### Providers : portée (scope)

Un provider peut vivre dans différents scopes :

- **Root injector** : `providedIn: 'root'` → singleton global.
- **Injector de module** (quand pertinent) : singleton à l’échelle du module.
- **Injector de composant** : nouvelle instance par composant.

Exemple : service global vs service par composant.

```ts
@Injectable({ providedIn: 'root' })
export class ApiClient {}

@Component({
  selector: 'app-editor',
  providers: [DraftService],
  template: `...`
})
export class EditorComponent {}
```

Ici `DraftService` est instancié par `EditorComponent` (utile pour isoler l’état).

### Tokens d’injection & multi providers

Quand on ne veut/peut pas injecter une classe, on utilise un **InjectionToken**.

```ts
export const API_BASE_URL = new InjectionToken<string>('API_BASE_URL');

bootstrapApplication(AppComponent, {
  providers: [
    { provide: API_BASE_URL, useValue: 'https://api.example.com' }
  ]
});

@Injectable({ providedIn: 'root' })
export class UsersService {
  constructor(@Inject(API_BASE_URL) private baseUrl: string) {}
}
```

Les **multi providers** permettent l’extension (plugins).

```ts
export const APP_PLUGINS = new InjectionToken<string[]>('APP_PLUGINS');

providers: [
  { provide: APP_PLUGINS, useValue: 'audit', multi: true },
  { provide: APP_PLUGINS, useValue: 'metrics', multi: true },
]
```

### useClass / useExisting / useFactory

- `useClass`: fournit une implémentation.
- `useExisting`: alias vers un autre provider.
- `useFactory`: fabrique avec dépendances.

Bon usage : `useFactory` avec config environnement.

---

## 1.3 Modules vs Standalone (organisation moderne)

### Standalone components

Les composants standalone réduisent la boilerplate et facilitent le lazy loading.

```ts
@Component({
  standalone: true,
  selector: 'app-user-list',
  imports: [CommonModule, RouterModule],
  template: `...`
})
export class UserListComponent {}
```

### Guidance d’architecture

- **Feature-first** (recommandé) : organiser par domaine (users, orders…).
- Séparer : `feature/`, `shared/`, `core/`.
- Éviter une couche `shared` “fourre-tout” : préférer des libs de design system, UI, utilitaires.

---

### Atelier 1 (45–60 min)

**But** : construire une feature “Users” avec une façade, des services injectés au bon scope.

- Créer un `UsersApiService` (root).
- Créer `UsersFacade` fourni au niveau feature (ou composant racine de la feature).
- Ajouter un `InjectionToken` de configuration.

Livrable : feature isolée, testable, avec injection correcte.

---

# 2) Change Detection & performances templates

## 2.1 Comment Angular met à jour la vue

Angular exécute des cycles de **change detection** pour mettre à jour le DOM.

- Stratégie par défaut : vérifie l’arbre de composants.
- La CD est déclenchée par des événements async (Zone.js) et certaines actions internes.

Problème typique : composants lourds + CD fréquente ⇒ ralentissement.

---

## 2.2 OnPush : principe et contraintes

Avec `ChangeDetectionStrategy.OnPush`, Angular ne vérifie un composant que si :

- un **@Input** change *par référence* (immutabilité)
- un event DOM du composant se produit
- un observable poussé via `async` pipe émet
- on force via `markForCheck()` / `detectChanges()`

```ts
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush,
  ...
})
export class ProductCardComponent {
  @Input() product!: Product;
}
```

### Bonnes pratiques OnPush

- Favoriser l’**immutabilité** (`map`, `filter`, spread) plutôt que mutation.
- Passer des Observables au template et utiliser `async`.
- Éviter de créer des objets/fonctions inline dans le template (sauf besoin) :

```html
<!-- À éviter si recalcul fréquent -->
<div [ngClass]="{ active: isActive(user) }"></div>
```

Préférer :

```ts
readonly vm$ = this.service.vm$;
```

---

## 2.3 TrackBy, pipes, async pipe

### `trackBy` sur `*ngFor`

Limite le re-render quand la liste change.

```html
<li *ngFor="let u of users; trackBy: trackById">{{u.name}}</li>
```

```ts
trackById = (_: number, u: User) => u.id;
```

### Pipes purs vs impurs

- Pipe **pur** : recalculé quand les entrées changent (référence)
- Pipe **impur** : recalcul à chaque cycle CD → coûteux

### `async` pipe

- Souscrit/désouscrit automatiquement
- Combine bien avec OnPush

---

## 2.4 Panorama : Signals (notions)

Les **signals** introduisent une réactivité fine-grain.

- Mieux contrôler quand la vue se met à jour.
- Potentiel “zoneless”.

Objectif ici : savoir que c’est une option moderne, comprendre comment cela impacte les patterns (mais sans remplacer RxJS partout).

---

### Atelier 2 (60–75 min)

Optimiser un écran liste + recherche :

- Passer le composant en `OnPush`.
- Remplacer des subscriptions manuelles par `async`.
- Ajouter `trackBy`.
- Mesurer avec Angular DevTools (frames, CD cycles).

---

# 3) RxJS avancé pour Angular

## 3.1 Cold vs Hot, Subjects

- **Cold** : commence à émettre à la souscription (ex: `http.get`).
- **Hot** : source partagée (ex: `Subject`, events, WebSocket).

### Subjects

- `Subject`: multicast, pas de valeur initiale.
- `BehaviorSubject`: conserve la dernière valeur.
- `ReplaySubject`: rejoue N valeurs.

Bon usage en Angular : `BehaviorSubject` pour un état simple local; sinon store dédié.

---

## 3.2 Patterns Angular pour RxJS

### Éviter les fuites mémoire

- Utiliser `async` pipe
- Ou `takeUntilDestroyed()` (Angular)

```ts
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';

this.route.params
  .pipe(takeUntilDestroyed())
  .subscribe(...);
```

### Eviter les subscriptions imbriquées

À la place de :

```ts
this.route.params.subscribe(p => {
  this.api.getUser(p['id']).subscribe(user => ...);
});
```

Utiliser `switchMap` :

```ts
user$ = this.route.params.pipe(
  map(p => p['id']),
  switchMap(id => this.api.getUser(id))
);
```

---

## 3.3 Opérateurs clés (quand et pourquoi)

- `switchMap` : annule la requête précédente (recherche/autocomplete).
- `concatMap` : met en file (sauvegardes séquentielles).
- `exhaustMap` : ignore tant que l’action n’est pas terminée (anti double-click login).
- `mergeMap` : parallèle (attention aux courses).
- `shareReplay(1)` : mise en cache / partage des résultats.

Exemple de cache HTTP “simple” :

```ts
users$ = this.http.get<User[]>('/api/users').pipe(
  shareReplay({ bufferSize: 1, refCount: true })
);
```

---

## 3.4 Erreurs, retry, backoff

- `catchError` pour transformer/mapper.
- `retry` / `retryWhen` pour stratégie.

Exemple backoff basique :

```ts
this.http.get('/api').pipe(
  retry({ count: 3, delay: (_, i) => timer(200 * (i + 1)) }),
  catchError(err => of({ error: true, err }))
);
```

---

### Atelier 3 (60 min)

Construire un flux “search as you type” :

- `formControl.valueChanges`
- `debounceTime`, `distinctUntilChanged`
- `switchMap` vers API
- gestion d’erreurs
- partage via `shareReplay`

---

# 4) Gestion d’état (State Management)

## 4.1 Définir l’état

On distingue :

- **État UI local** : open/close, onglet sélectionné.
- **État de feature** : liste filtrée, pagination, selection.
- **État global** : utilisateur connecté, permissions, préférences.

Critères :

- Taille et complexité
- Besoin de time-travel/debugging
- Synchronisation multi écrans

---

## 4.2 Pattern Facade + Store léger

Approche souvent suffisante :

- `Facade` expose des Observables (`vm$`, `items$`) et des commandes (`load()`, `select()`…)
- Stockage via `BehaviorSubject` ou `signal` (selon style)
- Side effects isolés

Exemple (simplifié) :

```ts
type UsersState = {
  loading: boolean;
  users: User[];
  error?: string;
};

@Injectable()
export class UsersFacade {
  private readonly state$ = new BehaviorSubject<UsersState>({
    loading: false,
    users: [],
  });

  readonly vm$ = this.state$.pipe(
    map(s => ({ loading: s.loading, users: s.users, error: s.error })),
    distinctUntilChanged()
  );

  constructor(private api: UsersApiService) {}

  load() {
    this.state$.next({ ...this.state$.value, loading: true, error: undefined });
    this.api.getUsers().pipe(
      finalize(() => this.state$.next({ ...this.state$.value, loading: false })),
      catchError(err => {
        this.state$.next({ ...this.state$.value, error: 'Load failed' });
        return EMPTY;
      })
    ).subscribe(users => {
      this.state$.next({ ...this.state$.value, users });
    });
  }
}
```

### Conseils

- Favoriser des **view models** (`vm$`) prêts pour le template.
- Séparer commande / query.
- Tester la façade comme une unité.

---

## 4.3 Quand utiliser NgRx / ComponentStore

### Indications

- Nombreuses actions/effets
- État complexe partagé entre plusieurs features
- Besoin d’outils (DevTools, time-travel, audit)

### Bonnes pratiques

- Stores “feature” plutôt que monolithiques.
- Effets pour les appels API.
- Sélecteurs memoizés.

---

### Atelier 4 (60–90 min)

Mettre en place une façade + state “feature” pour :

- sélection d’un item
- pagination
- cache de la liste

Bonus : extraire un `vm$` consommé par un composant OnPush.

---

# 5) Routing avancé & Lazy loading

## 5.1 Lazy loading (feature modules ou standalone)

### Lazy loading avec routes

```ts
export const routes: Routes = [
  {
    path: 'users',
    loadChildren: () => import('./users/users.routes').then(m => m.USERS_ROUTES)
  }
];
```

Avantages :

- Réduit le bundle initial
- Améliore le TTI (Time to Interactive)

---

## 5.2 Guards et resolvers

### Guards

- `CanActivate`: autorisation d’accès
- `CanMatch`: éviter même le matching (utile pour role-based lazy)

### Resolvers

Préchargent des données avant l’activation de route (attention UX si trop lourd).

Bon pattern : afficher skeleton + charger côté composant (souvent préférable) sauf données critiques.

---

## 5.3 Préchargement

- `PreloadAllModules`
- Stratégie custom : précharger selon connexion, rôle, heuristique.

---

### Atelier 5 (45–60 min)

- Ajouter une feature lazy-loaded
- Mettre un guard d’auth
- Précharger conditionnellement la feature “admin”

---

# 6) Optimisation des performances

## 6.1 Mesure et diagnostic

Outils :

- **Angular DevTools** : CD cycles, profiler composants.
- Chrome Performance tab : main thread, layout thrashing.

Approche :

1. Identifier l’écran lent.
2. Vérifier CD excessive.
3. Vérifier re-render list (trackBy).
4. Vérifier heavy computations dans template.

---

## 6.2 Optimisation runtime

- `OnPush` + `async` pipe
- Découper composants
- Virtual scroll (CDK) pour grandes listes
- Déporter calculs vers services / web workers si besoin

### Virtual scroll (idée)

Utiliser `@angular/cdk/scrolling` pour rendre 20–50 items instead of 2000.

---

## 6.3 Optimisation build

- Budgets dans `angular.json`
- Code splitting via lazy loading
- Analyse bundle (source-map-explorer / webpack-bundle-analyzer selon config)

### Conseils

- Éviter dépendances lourdes
- Préférer imports ciblés (tree-shaking)

---

## 6.4 SSR, hydration (survol)

- SSR améliore LCP/SEO
- Hydration réduit le coût de re-render client

Quand l’envisager :

- app publique, SEO important
- métriques Web Vitals prioritaires

---

### Atelier 6 (45–60 min)

- Mesurer un écran avant/après
- Appliquer OnPush + trackBy + virtual scroll (si pertinent)
- Vérifier la réduction des re-renders

---

# 7) Industrialisation & qualité

## 7.1 Structure projet & conventions

Recommandations :

- `core/` : singleton services, layout global, interceptors.
- `shared/` : composants UI réutilisables, pipes/patterns (limité).
- `features/<domain>/` : composants, routes, façade, state, api.

Choisir une convention de nommage :

- `*.component.ts`, `*.service.ts`, `*.routes.ts`, `*.facade.ts`.

---

## 7.2 Tests

### Unit tests

- Composants (shallow si possible)
- Services (HttpClientTestingModule)
- Facades (test de flux)

### Tests d’intégration / e2e

- Cypress / Playwright selon stack
- Scénarios critiques (login, checkout)

Conseils :

- Tester le comportement, pas l’implémentation.
- Stabiliser les sélecteurs (data-testid).

---

## 7.3 CI/CD et environnements

- Lint + tests + build en pipeline
- Build par environnement (dev/staging/prod)
- Versioning (semver)

Checklist release :

- Budgets OK
- Source maps contrôlées
- Logs/monitoring (Sentry, OpenTelemetry)

---

## 7.4 Sécurité (essentiels)

- XSS : Angular template sanitize, éviter bypass.
- CSRF : gérer cookies/token selon backend.
- JWT : stockage sécurisé (souvent cookie httpOnly).

---

### Atelier 7 (45 min)

- Ajouter un interceptor logging + auth
- Ajouter une règle de lint (ex: éviter `any`)
- Ajouter un test de façade

---

# Conclusion & suite

## Récapitulatif

Vous avez couvert :

- Les mécanismes internes (DI, compilation, zones)
- La change detection et les optimisations (OnPush, async, trackBy)
- RxJS avancé (mapping, cache, erreurs)
- La gestion d’état (façade/store, bonnes pratiques)
- Router: lazy loading, guards, préchargement
- Industrialisation (tests, CI/CD, conventions)

## Pistes de progression

- Approfondir Signals et interop RxJS
- Mettre en place SSR/hydration si besoin produit
- Standardiser une architecture “clean” multi-features (monorepo Nx)

---

# Annexes

## A) Cheatsheet opérateurs RxJS

- Recherche : `debounceTime` + `distinctUntilChanged` + `switchMap`
- Sauvegarde séquentielle : `concatMap`
- Anti double-submit : `exhaustMap`
- Cache : `shareReplay(1)`

## B) Checklist performance Angular

- [ ] Composants lourds en OnPush
- [ ] `trackBy` sur grosses listes
- [ ] Pas de fonctions inline coûteuses dans template
- [ ] `async` pipe (moins de subscriptions manuelles)
- [ ] Lazy loading des features
- [ ] Virtual scroll si grandes listes
- [ ] Budgets build configurés
