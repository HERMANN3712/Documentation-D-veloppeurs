# Formation Angular (30) — Gestion de la mémoire avec RxJS

> **Objectif** : comprendre et prévenir les fuites mémoire liées aux *subscriptions* RxJS en Angular, adopter les approches recommandées (**async pipe**, **takeUntilDestroyed**, destruction explicite quand nécessaire) et concevoir une architecture réduisant les abonnements manuels.

---

## 0) Prérequis & contexte

### Prérequis techniques
- Angular >= 16 recommandé (pour **takeUntilDestroyed** et les API associées).
- Maîtrise de base de RxJS : `Observable`, `Subscription`, `Subject`, opérateurs (`map`, `switchMap`, `takeUntil`, etc.).
- Comprendre le cycle de vie Angular : `constructor`, `ngOnInit`, `ngOnDestroy`.

### Pourquoi parler de mémoire en Angular ?
- Dans une SPA, les composants s’affichent/disparaissent souvent. Si un composant s’abonne à un flux sans se désabonner, l’abonnement peut survivre à la destruction du composant.
- La fuite mémoire *la plus classique* en Angular vient de **subscriptions non nettoyées**.
- Le coût : dégradation des performances, comportements fantômes (handlers encore actifs), trafic réseau inutile.

---

## 1) Plan détaillé de la formation

1. **Notions clés** : Observable vs Subscription, complétion, hot/cold, cycle de vie Angular
2. **Où apparaissent les fuites mémoire** : subscriptions, timers, event listeners, sujets non terminés
3. **Approche n°1 (recommandée) : `async` pipe**
4. **Approche n°2 (recommandée) : `takeUntilDestroyed`**
5. **Approche n°3 : destruction explicite (quand nécessaire)**
6. **Architecture** : limiter les abonnements manuels (facades, store, container/presentational)
7. **Cas pratiques & patterns** (HTTP, route params, WebSocket)
8. **Diagnostic** : repérer les fuites (Chrome DevTools, Angular DevTools, logs)
9. **Checklist & bonnes pratiques de fin de cours**

---

## 2) Notions clés : comment “naît” et “meurt” une subscription

### Observable vs Subscription
- Un **Observable** est une description *paresseuse* d’un flux.
- Une **Subscription** est l’exécution active de ce flux (et l’attache aux ressources associées).

```ts
const obs$ = interval(1000);       // description
const sub = obs$.subscribe(console.log); // exécution

// Si sub n'est jamais unsub, l'interval continue.
```

### Complétion et nettoyage automatique
- Certaines sources complètent naturellement : `HttpClient.get()` complète après la réponse.
- Beaucoup de sources ne complètent **jamais** : `interval`, `fromEvent`, `WebSocket`, `Subject`, `router.events`, `valueChanges` (forms) …

> Règle pratique : **si ça ne complète pas tout seul, vous devez gérer la fin.**

### Hot vs Cold (impact mémoire)
- **Cold** : chaque subscription crée une exécution séparée (ex: `http.get`).
- **Hot** : la source émet indépendamment des abonnés (ex: `Subject`, `fromEvent`).

Fuites fréquentes : *hot observables* abonnés par des composants détruits.

---

## 3) Où sont les fuites mémoire classiques en Angular ?

### 3.1 Subscriptions dans `ngOnInit` sans cleanup

```ts
export class UserComponent implements OnInit {
  ngOnInit() {
    this.userService.user$.subscribe(u => {
      // ...
    });
  }
}
```

Si `user$` ne complète pas, le composant est retenu en mémoire.

### 3.2 `fromEvent`, `interval`, `timer`
```ts
fromEvent(window, 'resize').subscribe(() => this.recomputeLayout());
interval(1000).subscribe(() => this.tick++);
```
Sans désabonnement : listeners et timers persistants.

### 3.3 `FormControl.valueChanges`
```ts
this.form.controls['q'].valueChanges
  .pipe(debounceTime(300))
  .subscribe(q => this.search(q));
```
`valueChanges` ne complète pas avec le composant.

### 3.4 `router.events` et flux globaux
```ts
this.router.events.subscribe(e => { /* ... */ });
```
Flux global : si vous oubliez de couper, il vit tant que l’app vit.

---

## 4) Approche recommandée n°1 : `async` pipe

### Pourquoi `async` pipe est la solution par défaut
- Souscription **automatique**.
- Désabonnement **automatique** à la destruction du composant.
- Gestion du *change detection* (marque la vue au bon moment).

### Exemple : afficher des données

**Composant**
```ts
export class UsersComponent {
  users$ = this.userService.users$; // Observable<User[]>
}
```

**Template**
```html
<ul>
  <li *ngFor="let u of (users$ | async)">
    {{ u.name }}
  </li>
</ul>
```

### Éviter les abonnements “juste pour afficher”
**Anti-pattern :**
```ts
users: User[] = [];
ngOnInit() {
  this.userService.users$.subscribe(u => this.users = u);
}
```
**Correct :** exposer `users$` et consommer via `async`.

### Gestion d'états : loading / error
Utilisez une VM (ViewModel) observable :
```ts
vm$ = this.userService.users$.pipe(
  map(users => ({ users, loading: false, error: null })),
  startWith({ users: [], loading: true, error: null }),
  catchError(err => of({ users: [], loading: false, error: err }))
);
```
```html
<ng-container *ngIf="vm$ | async as vm">
  <p *ngIf="vm.loading">Chargement…</p>
  <p *ngIf="vm.error">Erreur : {{ vm.error.message }}</p>
  <ul><li *ngFor="let u of vm.users">{{u.name}}</li></ul>
</ng-container>
```

### Bonnes pratiques avec `async`
- Préférer **un seul `async`** par template “zone” avec `as vm`.
- Éviter d’appeler une méthode qui retourne un Observable directement dans le template (risque de recréations).

---

## 5) Approche recommandée n°2 : `takeUntilDestroyed`

### Principe
Angular fournit un mécanisme de “signal de destruction” via `DestroyRef`.
`takeUntilDestroyed()` coupe la subscription lorsque le contexte est détruit.

> C’est la stratégie idéale pour les subscriptions **nécessaires** côté classe (ex: effets, bridging avec API imperative).

### Exemple simple
```ts
import { Component, DestroyRef, inject } from '@angular/core';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';

@Component({
  selector: 'app-clock',
  template: '{{ time }}',
})
export class ClockComponent {
  private destroyRef = inject(DestroyRef);
  time = new Date();

  ngOnInit() {
    interval(1000)
      .pipe(takeUntilDestroyed(this.destroyRef))
      .subscribe(() => (this.time = new Date()));
  }
}
```

### Exemple avec `valueChanges`
```ts
this.form.controls['q'].valueChanges
  .pipe(
    debounceTime(300),
    distinctUntilChanged(),
    takeUntilDestroyed(this.destroyRef)
  )
  .subscribe(q => this.search(q));
```

### Exemples typiques où `takeUntilDestroyed` est utile
- Intégration d’une lib imperative (charts, maps) avec callbacks.
- Écoute d’événements globaux (`router.events`, `fromEvent`).
- Synchronisation de state local, side effects contrôlés.

### Variante : sans stocker `destroyRef`
Vous pouvez injecter inline selon votre style :
```ts
interval(1000)
  .pipe(takeUntilDestroyed(inject(DestroyRef)))
  .subscribe(...);
```

---

## 6) Approche n°3 : destruction explicite (quand nécessaire)

### Quand l’utiliser ?
- Code legacy Angular < 16.
- Cas où vous devez gérer manuellement plusieurs subscriptions.
- Besoin d’un cycle de vie custom (désabonnement avant `ngOnDestroy`).

### Pattern 1 — Stocker les subscriptions
```ts
private sub = new Subscription();

ngOnInit() {
  this.sub.add(this.router.events.subscribe(() => {}));
  this.sub.add(fromEvent(window, 'resize').subscribe(() => {}));
}

ngOnDestroy() {
  this.sub.unsubscribe();
}
```

### Pattern 2 — `Subject` + `takeUntil` (legacy)
```ts
private readonly destroy$ = new Subject<void>();

ngOnInit() {
  fromEvent(window, 'resize')
    .pipe(takeUntil(this.destroy$))
    .subscribe(() => this.recomputeLayout());
}

ngOnDestroy() {
  this.destroy$.next();
  this.destroy$.complete();
}
```

> Point critique : **ne pas oublier `complete()`** sur `destroy$`.

### Comparatif rapide
| Solution | Pro | Con |
|---|---|---|
| `async` pipe | zéro code de cleanup | limité au template |
| `takeUntilDestroyed` | simple, robuste, moderne | nécessite Angular récent |
| `Subscription` / `destroy$` | compatible legacy | plus de boilerplate, risque d’oubli |

---

## 7) Architecture : limiter les abonnements manuels

### But
Le meilleur moyen d’éviter une fuite mémoire est de **ne pas créer** d’abonnement manuel lorsqu’il n’est pas indispensable.

### 7.1 Container / Presentational
- **Container** : compose les flux, expose `vm$`.
- **Presentational** : reçoit des `@Input` et émet des `@Output`, sans subscription.

```ts
// container
vm$ = combineLatest([
  this.facade.users$,
  this.facade.loading$,
]).pipe(map(([users, loading]) => ({ users, loading })));
```

### 7.2 Facade / Service orienté Observables
Au lieu de s’abonner dans le composant, exposez un flux prêt à consommer.

```ts
@Injectable({ providedIn: 'root' })
export class UsersFacade {
  private readonly refresh$ = new Subject<void>();

  users$ = this.refresh$.pipe(
    startWith(void 0),
    switchMap(() => this.http.get<User[]>('/api/users')),
    shareReplay({ bufferSize: 1, refCount: true })
  );

  refresh() { this.refresh$.next(); }

  constructor(private http: HttpClient) {}
}
```

- `shareReplay({ refCount: true })` aide à éviter de garder la source vivante sans abonnés.

### 7.3 Éviter les subscriptions “de transit”
**Anti-pattern :**
```ts
ngOnInit() {
  this.route.params.subscribe(p => {
    this.http.get(...).subscribe(...);
  });
}
```
**Correct :** composer avec `switchMap` et exposer un observable.
```ts
user$ = this.route.paramMap.pipe(
  map(pm => pm.get('id')!),
  switchMap(id => this.http.get<User>(`/api/users/${id}`))
);
```
Puis template via `async`.

---

## 8) Cas pratiques (avec solutions)

### Cas 1 — HTTP : pas de fuite, mais attention aux patterns
`HttpClient` complète, donc **pas de fuite** en soi.
Mais la multiplication de subscriptions (imbriquées) est à éviter.

**Bon :**
```ts
user$ = this.route.paramMap.pipe(
  map(pm => pm.get('id')!),
  switchMap(id => this.http.get<User>(`/api/users/${id}`)),
  shareReplay({ bufferSize: 1, refCount: true })
);
```

### Cas 2 — WebSocket / SSE (ne complète pas)
Si vous connectez un flux long-lived, il faut couper à la destruction.

```ts
messages$ = this.ws.messages$; // Observable<Message>

ngOnInit() {
  this.messages$
    .pipe(takeUntilDestroyed(this.destroyRef))
    .subscribe(m => this.storeMessage(m));
}
```

### Cas 3 — `fromEvent` + performance
Toujours désabonner, et pensez aux opérateurs qui réduisent la charge :
```ts
fromEvent(window, 'scroll')
  .pipe(
    throttleTime(100),
    takeUntilDestroyed(this.destroyRef)
  )
  .subscribe(() => this.onScroll());
```

---

## 9) Diagnostic : repérer et prouver une fuite

### Symptômes
- Après navigation, la mémoire ne redescend pas.
- Des logs continuent après destruction du composant.
- Plusieurs handlers identiques actifs (resize, scroll…).

### Méthode de base
1. Ajouter des logs `ngOnInit`/`ngOnDestroy`.
2. Vérifier qu’après navigation, aucune émission ne déclenche de code du composant.
3. Chrome DevTools : profiler mémoire (heap snapshots) et vérifier les rétentions.

### Indices de fuite fréquents
- Références retenues par :
  - `Subscription` active
  - event listener global
  - timer
  - store/event bus + callbacks non deregister

---

## 10) Checklist de fin

### Règles d’or
- Pour l’affichage : **`async` pipe**.
- Pour la logique impérative côté composant : **`takeUntilDestroyed`**.
- Pour legacy : `Subscription` composite ou `destroy$` + `takeUntil`.
- Éviter les `subscribe` imbriqués, préférer `switchMap/mergeMap/concatMap`.
- Préférer une architecture qui **compose** des observables plutôt que “s’abonne partout”.

### Checklist rapide
- [ ] Chaque `fromEvent/interval/router.events/valueChanges` a un cleanup.
- [ ] Les flux exposés aux templates passent par `async`.
- [ ] Les services qui font `shareReplay` utilisent `refCount` si approprié.
- [ ] Pas d’abonnements manuels “juste pour assigner une variable d’affichage”.
- [ ] Les patterns legacy (`destroy$`) complètent correctement.

---

## 11) Exercices (optionnels)

1. Refactorer un composant qui fait 3 `subscribe()` dans `ngOnInit` vers :
   - un `vm$` + `async`
   - un `takeUntilDestroyed` pour l’unique effet impératif restant
2. Transformer des subscriptions imbriquées (route -> http) en chaîne `switchMap`.
3. Ajouter un mécanisme de refresh avec `Subject` + `switchMap` et consommation via `async`.

---

## Annexes — Snippets utiles

### A) `shareReplay` : rappel
- `shareReplay(1)` sans `refCount` peut garder une source vivante indéfiniment.
- Préférez souvent :
```ts
shareReplay({ bufferSize: 1, refCount: true })
```

### B) Anti-pattern : `subscribe` dans un service sans sortie observable
Un service qui fait des subscriptions internes sans exposer de mécanisme de cleanup peut être une source de fuite.
Préférez retourner des observables composés, ou gérer explicitement le cycle de vie (rare côté service).

---

**Fin de la formation.**
