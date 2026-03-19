# Préparation d’entretien Angular **Senior**

> Formation structurée (plan + contenu complet) pour se préparer à un entretien Angular avancé : change detection, stratégie **OnPush**, **RxJS**, **lazy loading**, **dependency injection**, architecture, **formulaires réactifs**, guards, intercepteurs, state management, **signals** et optimisations.

---

## Objectifs pédagogiques

À l’issue de la formation, vous serez capable de :

- Expliquer **le fonctionnement interne** d’Angular (zone.js, change detection, DI, compilation, rendering).
- Justifier et appliquer **OnPush** et des **stratégies d’optimisation** performantes.
- Utiliser **RxJS** de manière idiomatique (hot/cold, multicasting, opérateurs clés, patterns).
- Structurer un projet Angular pour un contexte **enterprise** (modules/standalone, architecture, boundaries).
- Maîtriser les **Reactive Forms** et leurs patterns (validation async, ControlValueAccessor, forms dynamiques).
- Mettre en place lazy loading, guards, intercepteurs et strategie d’erreurs.
- Comparer et implémenter des solutions de **state management** (RxJS, NgRx, ComponentStore, Signals store).
- Comprendre et utiliser **Signals** (computed, effect, interop RxJS) et stratégies réactives.
- Répondre comme un senior : **explications claires**, arbitrages, trade-offs, dette technique, performance.

---

## Pré-requis

- Angular (v14+) : composants, directives, routing, services.
- TypeScript : génériques, types utilitaires, classes.
- RXJS de base : Observable, Subscription.

---

## Public

- Développeurs Angular confirmés visant un poste **Senior**.
- Tech leads / formateurs souhaitant une trame d’entretien.

---

## Format et déroulé

- Durée suggérée : **1 à 2 jours** (adaptable)
- Pédagogie : théorie + démos + questions d’entretien + exercices
- Livrables : checklists, snippets, anti-patterns, réponses types

---

# Plan de la formation

1. **Lecture d’un entretien senior** : ce qu’on attend (communication, architecture, performance)
2. **Change Detection** : mécanismes, cycles, zones, coûts, debug
3. **OnPush** : règles, pièges, patterns et performance
4. **RxJS avancé** : mental model, opérateurs, erreurs, multicasting
5. **Lazy loading & Routing** : architecture, preload, standalone, performance
6. **Dependency Injection** : hiérarchie d’injecteurs, providers, tokens, scope
7. **Architecture de projet** : domain boundaries, hexagonal/clean, libs, standalone
8. **Formulaires réactifs** : patterns avancés, validation, dynamiques, CVA
9. **Guards & Résolveurs** : auth, permissions, data prefetch
10. **Intercepteurs HTTP** : auth, retry, cache, erreurs
11. **State management** : RxJS, NgRx, ComponentStore, signals-based
12. **Signals** : concepts, effects, computed, interop, migration
13. **Optimisation** : performance, bundle, rendering, runtime
14. **Batterie de questions** : questions types senior + grilles de réponse

---

# 1) Comprendre un entretien Angular Senior

## 1.1 Ce que l’évaluateur cherche

- **Maîtrise conceptuelle** : explication de la CD, DI, RxJS, routing.
- **Prise de décision** : pourquoi tel pattern ? quels risques ?
- **Capacité à debugger** : profiler, DevTools, logs, reproduction.
- **Qualité** : architecture, tests, lisibilité, scalabilité.
- **Performance** : OnPush, trackBy, lazy loading, budgets.

## 1.2 Méthode de réponse (structure)

Utilisez une structure simple :

1. **Définition** (ce que c’est)
2. **Pourquoi** (problème résolu)
3. **Comment** dans Angular (mécanisme)
4. **Exemple** (snippet et cas concret)
5. **Limites / trade-offs**

---

# 2) Change Detection Angular : comprendre et expliquer

## 2.1 Qu’est-ce que la Change Detection ?

La change detection (CD) est le mécanisme qui **synchronise l’état** (data) avec la **vue** (DOM). Concrètement, Angular :

- évalue les bindings (interpolations, propriétés, pipes)
- compare les valeurs
- met à jour le DOM si nécessaire

## 2.2 Déclenchement : zone.js et le « tick »

Historiquement, Angular s’appuie sur **zone.js** pour détecter des événements asynchrones et déclencher une CD :

- événements DOM (click, input…)
- timers (setTimeout, setInterval)
- Promises
- requêtes HTTP

Une fois l’événement terminé, Angular lance un cycle de CD.

> À l’entretien, on attend que vous sachiez relier : *event async → zone → tick → change detection → rendu*.

## 2.3 Stratégies : Default vs OnPush

- **Default** : à chaque tick, Angular vérifie le composant et ses descendants.
- **OnPush** : Angular ne vérifie le composant que dans certains cas (cf. section 3).

## 2.4 Arbre de composants et coût

La CD parcourt l’**arbre** ; le coût dépend :

- du nombre de composants
- du nombre de bindings
- de la fréquence des ticks
- des calculs synchrones dans templates/pipes

### Anti-patterns fréquents

- méthodes coûteuses appelées depuis le template (`{{ heavyFn() }}`)
- pipes impurs sans raison
- subscriptions multiples non maîtrisées

## 2.5 Debug et mesures

- **Angular DevTools** → profiler (CD cycles, composants re-render)
- `ChangeDetectorRef` pour comprendre/contrôler
- `console.time` / `performance.mark`

---

# 3) OnPush : règles, patterns et pièges

## 3.1 Définition

`ChangeDetectionStrategy.OnPush` réduit les vérifications : le composant n’est re-checké que si :

1. une **Input** change de **référence**
2. un **event** est déclenché dans sa vue
3. un `async` pipe émet une nouvelle valeur
4. on force via `markForCheck()` / `detectChanges()`

## 3.2 Exemple

```ts
@Component({
  selector: 'app-user-card',
  template: `{{ user.name }}`,
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class UserCardComponent {
  @Input() user!: User;
}
```

### Piège classique : mutation

```ts
// MAUVAIS avec OnPush
this.user.name = 'Alice';
```

**Correct** : créer une nouvelle référence.

```ts
this.user = { ...this.user, name: 'Alice' };
```

## 3.3 Quand l’utiliser ?

- listes, dashboards, vues à forte densité
- applications avec state centralisé immuable
- composants « presentational »

## 3.4 markForCheck vs detectChanges

- `markForCheck()` : marque pour le **prochain** cycle (préférable)
- `detectChanges()` : déclenche immédiatement sur la branche (à utiliser avec parcimonie)

## 3.5 Questions d’entretien typiques

- *Pourquoi OnPush améliore la performance ?*
- *Quels événements déclenchent la CD sur un composant OnPush ?*
- *Comment gérer un service qui pousse des données (Subject) sans async pipe ?*

---

# 4) RxJS avancé : mental model senior

## 4.1 Cold vs Hot

- **Cold Observable** : l’exécution démarre à la subscription (ex: `http.get`)
- **Hot Observable** : source partagée (ex: `Subject`, events)

## 4.2 Multicasting et partage

Problème : plusieurs subscriptions déclenchent plusieurs exécutions.

Solution : `shareReplay` (avec précaution).

```ts
const data$ = this.http.get('/api/data').pipe(
  shareReplay({ bufferSize: 1, refCount: true })
);
```

### Points à expliquer

- `refCount` évite de garder la source vivante sans abonnés
- `shareReplay` peut « cacher » des erreurs si mal utilisé

## 4.3 Opérateurs clés à maîtriser (avec cas)

- **switchMap** : annule la requête précédente (recherche)
- **mergeMap** : parallélise (batch)
- **concatMap** : séquence (ordre)
- **exhaustMap** : ignore tant que en cours (login)

Exemple recherche :

```ts
this.results$ = this.searchCtrl.valueChanges.pipe(
  debounceTime(300),
  distinctUntilChanged(),
  switchMap(q => this.api.search(q))
);
```

## 4.4 Gestion d’erreurs

```ts
source$.pipe(
  catchError(err => {
    this.logger.error(err);
    return of([]); // fallback
  })
);
```

### À l’entretien

- différence `catchError` vs `tap({ error })`
- stratégie de retry (`retry`, `retryWhen`) avec backoff

## 4.5 Subscription management

- Favoriser `async` pipe
- `takeUntilDestroyed()` (Angular 16+)

```ts
readonly data$ = this.api.data().pipe(
  takeUntilDestroyed(this.destroyRef)
);
```

---

# 5) Lazy Loading & Routing (avancé)

## 5.1 Pourquoi

- réduire le **bundle initial**
- améliorer le **Time To Interactive**
- isoler des features

## 5.2 Lazy loading : routes

### Standalone

```ts
export const routes: Routes = [
  {
    path: 'admin',
    loadChildren: () => import('./admin/admin.routes').then(m => m.ADMIN_ROUTES)
  }
];
```

### Modules (legacy)

```ts
{
  path: 'admin',
  loadChildren: () => import('./admin/admin.module').then(m => m.AdminModule)
}
```

## 5.3 Préchargement

- `PreloadAllModules`
- stratégie custom (précharger selon profil, réseau)

## 5.4 Bonnes pratiques

- routes « feature-based »
- guards ciblés (éviter guard global lourd)
- splitter gros vendors si nécessaire

---

# 6) Dependency Injection (DI) : niveau senior

## 6.1 Concepts

DI = fournir des dépendances sans couplage fort.

- **Injector** résout des tokens → instances
- hiérarchie : root, module, component

## 6.2 Scopes de providers

- `providedIn: 'root'` : singleton app
- providers au **niveau component** : instance par composant (utile pour encapsuler état local)

```ts
@Injectable({ providedIn: 'root' })
export class ApiService {}

@Component({
  providers: [LocalStore]
})
export class FeatureComponent {
  constructor(public store: LocalStore) {}
}
```

## 6.3 Tokens et multi-providers

### InjectionToken

```ts
export const API_URL = new InjectionToken<string>('API_URL');

providers: [{ provide: API_URL, useValue: 'https://api.example.com' }]
```

### Multi-provider (ex: interceptors)

```ts
{ provide: HTTP_INTERCEPTORS, useClass: AuthInterceptor, multi: true }
```

## 6.4 useFactory et dépendances

```ts
{ 
  provide: Analytics,
  useFactory: (cfg: Config) => new Analytics(cfg.key),
  deps: [Config]
}
```

## 6.5 Pièges

- circular dependencies
- providers « cachés » au niveau component
- testabilité : préférer interfaces/tokens

---

# 7) Architecture de projet Angular (enterprise)

## 7.1 Objectifs

- découplage et maintenabilité
- limites claires (domain boundaries)
- facilité de test et de refactor

## 7.2 Découpage recommandé

- `core/` (singleton services, interceptors, guards globaux)
- `shared/` (components/pipes/directives réutilisables)
- `features/` (par domaine métier)
- `ui/` (design system)
- `data-access/` (services API, stores)

> En mono-repo, utiliser Nx (libs) est souvent un plus.

## 7.3 Smart vs Dumb components

- **container** : orchestration, state, appels services
- **presentational** : inputs/outputs, OnPush, facile à tester

## 7.4 Standalone vs Modules

- Standalone simplifie le graph (imports localisés)
- Modules restent valables pour legacy

## 7.5 Conventions

- pas de logique métier dans les templates
- facades pour isoler NgRx / API
- éviter les imports « transverses » non maîtrisés

---

# 8) Formulaires réactifs (Reactive Forms) : avancé

## 8.1 Pourquoi reactive forms

- modèle explicite en TS
- validation composable
- formulaires dynamiques

## 8.2 Validation sync & async

```ts
this.form = new FormGroup({
  email: new FormControl('', {
    validators: [Validators.required, Validators.email],
    asyncValidators: [this.emailTakenValidator()],
    updateOn: 'blur'
  })
});
```

### Validator async (ex)

```ts
emailTakenValidator(): AsyncValidatorFn {
  return (ctrl) => this.api.isEmailTaken(ctrl.value).pipe(
    map(isTaken => (isTaken ? { taken: true } : null)),
    catchError(() => of(null))
  );
}
```

## 8.3 Forms dynamiques (FormArray)

```ts
const items = new FormArray<FormGroup<ItemForm>>([]);
items.push(this.createItem());
```

## 8.4 ControlValueAccessor (CVA)

Permet de créer un composant de formulaire compatible Angular Forms.

Points à citer :

- `writeValue`, `registerOnChange`, `registerOnTouched`
- gérer `disabled`
- éviter les boucles (propager change proprement)

## 8.5 Pattern : présentation des erreurs

- helpers pour afficher erreurs selon `touched/dirty`
- centraliser messages d’erreurs

---

# 9) Guards & Résolveurs : sécurité et UX

## 9.1 Guards principaux

- `CanActivate` : autoriser navigation
- `CanMatch` : éviter de charger une route (utile avec lazy loading)
- `CanDeactivate` : empêcher quitter (form dirty)

## 9.2 AuthGuard (ex simplifié)

```ts
export const authGuard: CanActivateFn = () => {
  const auth = inject(AuthService);
  const router = inject(Router);

  return auth.isLoggedIn$().pipe(
    map(ok => ok || router.createUrlTree(['/login']))
  );
};
```

## 9.3 Resolver (précharger données)

- améliore UX (page chargée avec data)
- attention : peut bloquer navigation si API lente

---

# 10) Intercepteurs HTTP : cross-cutting concerns

## 10.1 Cas d’usage

- ajouter token auth
- logging
- gestion d’erreurs standardisée
- retry/backoff
- cache

## 10.2 Auth interceptor

```ts
@Injectable()
export class AuthInterceptor implements HttpInterceptor {
  intercept(req: HttpRequest<any>, next: HttpHandler) {
    const token = localStorage.getItem('token');
    const authReq = token
      ? req.clone({ setHeaders: { Authorization: `Bearer ${token}` } })
      : req;

    return next.handle(authReq);
  }
}
```

## 10.3 Error interceptor (pattern)

- mapper codes HTTP → erreurs métier
- envoyer à Sentry/Datadog
- afficher toast (via service)

## 10.4 Cache interceptor (attention)

- cache GET idempotents
- invalidation (URLs, headers)
- risque : incohérence, sécurité

---

# 11) State Management : options et arbitrages

## 11.1 Pourquoi un state management

- éviter « spaghetti subscriptions »
- centraliser règles métier
- rendre le flux de données prédictible

## 11.2 Options

### A) RxJS simple + service store

- `BehaviorSubject` / `ReplaySubject`
- sélecteurs via `map` + `distinctUntilChanged`

Avantages : léger. Inconvénients : discipline nécessaire.

### B) NgRx (Store/Effects/Entities)

- idéal grosses apps, équipe, règles strictes
- verbose mais standardisé

À expliquer : actions → reducers → store; effects pour async.

### C) ComponentStore

- state local à une feature
- moins boilerplate que Store global

### D) Signals-based store

- simple, réactif, bonne intégration CD
- attention à l’outillage/discipline (selon stack)

## 11.3 Pattern : facade

Expose une API stable au reste de l’app.

- `vm$` ou `vm` (computed signal)
- commandes (methods) : `load()`, `update()`

---

# 12) Signals : compréhension & usage

## 12.1 Concepts

- **signal** : valeur observable synchronement (lecture via `()`)
- **computed** : dérivation mémorisée
- **effect** : effet de bord déclenché par dépendances

```ts
const count = signal(0);
const double = computed(() => count() * 2);

effect(() => {
  console.log('count =', count());
});
```

## 12.2 Intégration avec Angular

- templates lisent des signals efficacement
- combinable avec OnPush

## 12.3 Interop RxJS

- convertir Observable → signal (selon APIs Angular)
- garder RxJS pour streams complexes (websocket, operators)

## 12.4 Pièges

- effets non maîtrisés (boucles)
- dépendances implicites
- rendre testable (isoler effects)

---

# 13) Stratégies d’optimisation (performance)

## 13.1 Runtime/UI

- OnPush + immutabilité
- `trackBy` dans `*ngFor`

```html
<li *ngFor="let item of items; trackBy: trackById">...</li>
```

```ts
trackById = (_: number, x: { id: string }) => x.id;
```

- éviter fonctions dans templates
- pipes **purs**
- virtual scroll (CDK) pour grandes listes

## 13.2 Bundling

- lazy loading
- budgets Angular
- analyse : `source-map-explorer`, `webpack-bundle-analyzer`

## 13.3 Network & data

- cache HTTP (ETag, Cache-Control)
- pagination
- `shareReplay` avec prudence pour éviter requêtes multiples

## 13.4 Rendering & SSR

- SSR/SSG si SEO & performance perçue
- hydration (si stack Angular compatible)

## 13.5 Tests de perf

- Lighthouse
- Web Vitals (LCP/INP/CLS)
- profiler Angular DevTools

---

# 14) Questions d’entretien Senior (banque) + réponses attendues

## 14.1 Change detection

1. **Explique le cycle de change detection.**
   - attendu : tree traversal, triggers (zone/events), checks, DOM updates

2. **Différence entre Default et OnPush.**
   - attendu : conditions de re-check, immutabilité, async pipe

3. **Quand utiliser detectChanges ?**
   - attendu : cas exceptionnels, risque de performance, ExpressionChanged

## 14.2 RxJS

1. **switchMap vs mergeMap vs concatMap vs exhaustMap**
   - attendu : cancellation/parallélisme/ordre

2. **Hot vs Cold + shareReplay**
   - attendu : multicasting, refCount, mémoire

3. **Stratégie erreur globale**
   - attendu : interceptor + catchError local

## 14.3 DI

1. **Hiérarchie des injecteurs**
2. **providedIn root vs providers component**
3. **InjectionToken et multi providers**

## 14.4 Routing

1. **CanMatch vs CanActivate**
2. **Préchargement**
3. **Resolver : avantages/risques**

## 14.5 Forms

1. **CVA** : comment rendre un composant compatible forms ?
2. **Validation async** : éviter appels multiples ? (`updateOn`, `debounceTime`)
3. **FormArray** : cas d’usage

## 14.6 State management

1. **Quand NgRx est justifié ?**
2. **Façade pattern**
3. **State local vs global**

## 14.7 Signals

1. **computed vs effect**
2. **interop RxJS**
3. **patterns de store signal**

---

# Exercices (optionnels mais recommandés)

1. **Refactor OnPush** : passer un écran en OnPush + trackBy + pipes purs
2. **RxJS kata** : implémenter search + cancel + retry backoff
3. **Architecture** : proposer un découpage feature/data-access/ui
4. **Forms** : composant datepicker via CVA + validation async
5. **State** : mini store (NgRx ou signals) + facade + effects

---

# Checklists de préparation (à réviser la veille)

## Checklist technique

- [ ] Déclencheurs de CD + OnPush conditions
- [ ] switchMap/mergeMap/concatMap/exhaustMap (cas typiques)
- [ ] shareReplay : risques mémoire + refCount
- [ ] Lazy loading + preloading
- [ ] DI : scopes, tokens, useFactory
- [ ] Guards : CanMatch/CanActivate/CanDeactivate
- [ ] Interceptors : auth + erreurs + retry
- [ ] Reactive Forms : async validator + FormArray + CVA
- [ ] Signals : signal/computed/effect + interop
- [ ] Optimisations : trackBy, virtual scroll, budgets

## Checklist “communication senior”

- [ ] Toujours donner les **trade-offs**
- [ ] Mentionner testabilité, observabilité (logs), monitoring
- [ ] Donner un exemple concret issu d’un projet

---

## Annexe : snippets utiles

### takeUntilDestroyed

```ts
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';
```

### Facade minimaliste (RxJS)

```ts
@Injectable({ providedIn: 'root' })
export class UsersFacade {
  private readonly state = new BehaviorSubject<{ users: User[] }>({ users: [] });

  readonly users$ = this.state.asObservable().pipe(
    map(s => s.users),
    distinctUntilChanged()
  );

  load() {
    return this.api.getUsers().pipe(
      tap(users => this.state.next({ users }))
    ).subscribe();
  }

  constructor(private api: ApiService) {}
}
```

---

# Fin de la formation
