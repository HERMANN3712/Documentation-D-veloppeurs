# Formation — RxJS fondamentaux avancés (Angular)

**Public** : développeurs Angular ayant déjà pratiqué RxJS (Observables, pipe, opérateurs usuels).

**Objectifs pédagogiques**
- Distinguer *streams froids* et *chauds* et en déduire les implications sur la consommation.
- Maîtriser **Subject** (Subject, BehaviorSubject, ReplaySubject, AsyncSubject) et le **multicasting**.
- Appliquer des stratégies robustes de **gestion d’erreurs** et de **retry**.
- Comprendre le **scheduling** (micro/macro-tâches, queues), et l’impact des schedulers RxJS.
- Composer des workflows asynchrones avancés (patterns de composition, annulation, concurrence, backpressure).
- Produire du code RxJS lisible, testable et performant dans Angular.

**Pré-requis**
- TypeScript (types, generics), ES2015+.
- Angular (services, HttpClient, composants, change detection).
- Bases RxJS : Observable/Observer, subscribe, pipe, map/filter/tap, switchMap.

**Format** : 1–2 jours (adaptable) — alternance théorie + ateliers.

---

## Plan (vue d’ensemble)

1. **Rappels essentiels & modèle mental RxJS**
2. **Streams froids vs chauds**
3. **Subjects et variantes**
4. **Multicasting, partage, caching**
5. **Gestion d’erreurs et stratégies de reprise**
6. **Scheduling RxJS et concurrence**
7. **Patterns avancés de composition asynchrone**
8. **Intégration Angular : services, composants, async pipe, interop signals**
9. **Tests RxJS (marble testing) & pièges courants**
10. **Atelier de synthèse : architecture réactive d’un module**

---

# 1) Rappels essentiels & modèle mental RxJS

## 1.1 Observable : pull vs push, synchro vs asynchro
Un **Observable** représente un flux d’événements dans le temps.
- Un flux peut émettre `next`, puis `complete`, ou `error`.
- Un Observable peut être **synchrone** (ex. `of(1,2,3)`) ou **asynchrone** (ex. `interval(1000)`, `HttpClient.get`).

### Contrat de base
- Après `error` ou `complete` : plus aucun `next` ne doit être émis.
- Le consommateur reçoit via un `Subscriber`.

```ts
import { Observable } from 'rxjs';

const obs$ = new Observable<number>(subscriber => {
  subscriber.next(1);
  subscriber.next(2);
  subscriber.complete();

  // Teardown (cleanup)
  return () => console.log('unsubscribed');
});

const sub = obs$.subscribe({
  next: v => console.log(v),
  complete: () => console.log('done')
});

sub.unsubscribe();
```

## 1.2 Teardown, unsubscribe et cycle de vie
- Un subscribe retourne un **Subscription**.
- Le **teardown** est la fonction de nettoyage appelée lors du `unsubscribe`.
- Important en Angular : éviter les fuites mémoire (surtout dans composants/services long-lived).

### Patterns Angular
- **async pipe** : gère automatiquement l’abonnement.
- `takeUntilDestroyed()` (Angular >= 16 + rxjs-interop) : abonnement lié au cycle de vie.

---

# 2) Streams froids vs chauds

## 2.1 Définitions
### Stream froid (*cold*)
- L’Observable **démarre** et **produit** ses valeurs **pour chaque abonné**.
- Chaque subscriber a sa propre exécution.

Exemples : `HttpClient.get()`, `defer(() => fetch(...))`, `of(...)` (en pratique).

### Stream chaud (*hot*)
- La source existe **indépendamment** des abonnés.
- Les abonnés « s’accrochent » au flux existant et peuvent **rater** des valeurs.

Exemples : événements DOM (`fromEvent`), WebSocket partagé, `Subject`.

## 2.2 Implications pratiques
- Cold : duplications de side effects (plusieurs requêtes HTTP), isolation, determinisme.
- Hot : partage naturel, diffusion à plusieurs, mais nécessite souvent buffer/replay.

## 2.3 Démonstration : duplication d’appel HTTP

```ts
const data$ = this.http.get('/api/users'); // cold

data$.subscribe(); // 1 requête
data$.subscribe(); // 2 requêtes
```

Solution : **partager** (voir section 4).

## 2.4 Exercice
- Observer (console/logs) : 2 abonnements sur un cold observable vs un hot.

---

# 3) Subjects et variantes

## 3.1 Subject (base)
**Subject** = à la fois **Observable** et **Observer**.
- On peut `next()` dedans.
- Tous les abonnés reçoivent les valeurs en temps réel.

```ts
import { Subject } from 'rxjs';

const bus$ = new Subject<string>();

bus$.subscribe(v => console.log('A', v));
bus$.next('hello');

bus$.subscribe(v => console.log('B', v));
bus$.next('world');
```

Ici, **B** ne reçoit pas `hello`.

### Quand l’utiliser ?
- Bus d’événements, pont vers callbacks.
- Déclencheurs impératifs (clic, refresh manuel) combinés avec des streams.

### Anti-pattern
- Sur-utiliser `Subject` pour « tout » (on perd la nature déclarative).

## 3.2 BehaviorSubject
- Stocke la **dernière valeur** et la réémet au nouvel abonné.
- Nécessite une valeur initiale.

```ts
import { BehaviorSubject } from 'rxjs';

const state$ = new BehaviorSubject<number>(0);
state$.subscribe(v => console.log('A', v)); // 0
state$.next(1);
state$.subscribe(v => console.log('B', v)); // 1
```

Usage : état applicatif simple, selections, paramètres.

## 3.3 ReplaySubject
- Réémet les **N dernières valeurs** (ou sur une fenêtre de temps).

```ts
import { ReplaySubject } from 'rxjs';

const replay$ = new ReplaySubject<number>(2);
replay$.next(1);
replay$.next(2);
replay$.next(3);

replay$.subscribe(v => console.log(v)); // 2, 3
```

Usage : cache court, synchronisation tardive.

## 3.4 AsyncSubject
- N’émet que la **dernière valeur** lors du **complete**.

```ts
import { AsyncSubject } from 'rxjs';

const a$ = new AsyncSubject<number>();
a$.subscribe(v => console.log('A', v));

a$.next(1);
a$.next(2);
a$.complete();

// A reçoit 2 uniquement
```

Usage : pont vers API « promise-like », résultat final.

## 3.5 Bonnes pratiques
- Exposer un `Observable` en public, garder le Subject en privé.

```ts
private readonly refreshSubject = new Subject<void>();
readonly refresh$ = this.refreshSubject.asObservable();

triggerRefresh() { this.refreshSubject.next(); }
```

---

# 4) Multicasting, partage, caching

## 4.1 Le problème : side effects multipliés
Un Observable cold refait l’exécution par abonné.

## 4.2 `share()` — multicast sans replay
- Partage une seule souscription à la source.
- Attention : le comportement au *refCount* (démarre à 1 abonné, stop à 0).

```ts
import { share } from 'rxjs/operators';

const shared$ = coldSource$.pipe(share());
```

Cas d’usage : `fromEvent`, calcul coûteux, WebSocket.

## 4.3 `shareReplay()` — partage + replay/cache
- Très utilisé pour **HTTP caching**.
- Rejoue la dernière (ou N dernières) émissions.

```ts
import { shareReplay } from 'rxjs/operators';

const users$ = this.http.get<User[]>('/api/users').pipe(
  shareReplay({ bufferSize: 1, refCount: true })
);
```

### Choix `refCount`
- `refCount: true` : la source se désabonne quand plus personne n’écoute (libère ressources) ; attention aux ré-abonnements (nouvel appel HTTP).
- `refCount: false` : la source reste « vivante » (cache persistant) ; attention à la mémoire et aux flux infinis.

## 4.4 `connectable`, `publish`, `refCount` (concepts)
Historique RxJS : `publish()` + `connect()` ; aujourd’hui via `connectable`.
Objectif : contrôler le moment où la source démarre.

Idée :
- *cold* → rendre *hot* via multicasting.
- Démarrer sur `connect()` plutôt que sur le 1er subscribe.

## 4.5 Patterns de cache HTTP dans Angular
### Cache simple dans un service
```ts
@Injectable({ providedIn: 'root' })
export class UsersService {
  private readonly users$ = this.http.get<User[]>('/api/users').pipe(
    shareReplay({ bufferSize: 1, refCount: false })
  );

  constructor(private http: HttpClient) {}

  getUsers(): Observable<User[]> {
    return this.users$;
  }
}
```

### Cache + invalidation (refresh manuel)
```ts
private readonly refresh$ = new Subject<void>();

readonly users$ = this.refresh$.pipe(
  startWith(void 0),
  switchMap(() => this.http.get<User[]>('/api/users')),
  shareReplay({ bufferSize: 1, refCount: true })
);

refresh() { this.refresh$.next(); }
```

## 4.6 Exercice
- Construire un service qui cache un endpoint + bouton refresh.
- Vérifier (DevTools) le nombre de requêtes.

---

# 5) Gestion d’erreurs et stratégies de reprise

## 5.1 Le contrat d’erreur
- `error` termine le stream.
- Après une erreur, plus aucun `next` ne passe.

## 5.2 `catchError` : récupérer, transformer, ou relancer

```ts
import { catchError, of, throwError } from 'rxjs';

source$.pipe(
  catchError(err => {
    // Option A: fallback
    return of([]);

    // Option B: enrichir et relancer
    // return throwError(() => new Error('...'));
  })
);
```

### Attention
- `catchError` change le type : prévoir un fallback compatible.

## 5.3 `retry` vs `retryWhen`
### `retry(n)`
- Rejoue la source en cas d’erreur jusqu’à `n` fois.

```ts
http$.pipe(retry(2));
```

### `retryWhen` / `retry` avec config (RxJS récent)
Utiliser un **backoff** progressif.

```ts
import { retry, timer } from 'rxjs';

http$.pipe(
  retry({
    count: 3,
    delay: (_err, retryCount) => timer(200 * retryCount)
  })
);
```

## 5.4 `finalize` — cleanup quel que soit le scénario

```ts
source$.pipe(
  finalize(() => console.log('cleanup'))
);
```

Usage : masquer loader, libérer ressources.

## 5.5 Patterns d’erreurs en UI (Angular)
### Exposer un ViewModel
- Un flux `data$` et un flux `error$` (ou une union discriminée).

Exemple union :
```ts
type RemoteData<T> =
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: unknown };

const vm$: Observable<RemoteData<User[]>> = this.refresh$.pipe(
  startWith(void 0),
  switchMap(() =>
    this.http.get<User[]>('/api/users').pipe(
      map(data => ({ status: 'success', data } as const)),
      startWith({ status: 'loading' } as const),
      catchError(error => of({ status: 'error', error } as const))
    )
  )
);
```

## 5.6 Exercice
- Ajouter retry + backoff sur un endpoint instable.
- Remonter un état UI *loading/success/error*.

---

# 6) Scheduling RxJS et concurrence

## 6.1 Pourquoi parler de scheduling ?
- Comprendre **quand** et **sur quelle queue** les notifications sont émises.
- Résoudre des comportements inattendus (ordre d’exécution, UI freeze, tests).

## 6.2 Rappels JavaScript event loop
- **Call stack**
- **Microtasks** (Promises)
- **Macrotasks** (setTimeout, events)

## 6.3 Schedulers RxJS (principaux)
- `asyncScheduler` : macrotask (setTimeout / setInterval style).
- `queueScheduler` : exécution synchronisée en FIFO (mais peut planifier).
- `asapScheduler` : proche microtask (optimisé, après l’exécution courante).
- `animationFrameScheduler` : rendu UI.

## 6.4 `observeOn` vs `subscribeOn`
- `subscribeOn(scheduler)` : affecte **le moment** où la souscription à la source s’exécute.
- `observeOn(scheduler)` : affecte **l’émission** des notifications vers l’observateur.

```ts
import { of, asyncScheduler } from 'rxjs';
import { observeOn, subscribeOn, tap } from 'rxjs/operators';

of(1,2,3).pipe(
  tap(v => console.log('sync', v)),
  observeOn(asyncScheduler),
  tap(v => console.log('async observed', v)),
);
```

## 6.5 Concurrence : mergeMap, concatMap, switchMap, exhaustMap
- `mergeMap` : concurrence (par défaut infinie) — attention charge serveur.
- `concatMap` : séquentiel (file d’attente).
- `switchMap` : annule la requête précédente (plus précisément : unsubscribe de l’inner observable).
- `exhaustMap` : ignore les triggers tant que l’inner n’est pas terminé.

### Limiter la concurrence
```ts
clicks$.pipe(
  mergeMap(() => httpCall$(), 3) // max 3 en parallèle
);
```

## 6.6 Exercice
- Implémenter une recherche typeahead : `debounceTime`, `distinctUntilChanged`, `switchMap`.
- Variante : `exhaustMap` pour un bouton « submit » anti-double clic.

---

# 7) Patterns avancés de composition asynchrone

## 7.1 Déclencheurs, état et dérivations
Approche :
- **Sources** (events, http, web socket)
- **State streams** (BehaviorSubject, store)
- **Derived streams** (combineLatest, withLatestFrom)

### `combineLatest` vs `withLatestFrom`
- `combineLatest([a$, b$])` émet quand **a ou b** émet (après que chacun ait émis au moins une fois).
- `withLatestFrom(b$)` émet quand **a$** émet, en prenant le dernier `b$`.

```ts
searchSubmit$.pipe(
  withLatestFrom(filters$),
  switchMap(([query, filters]) => api.search(query, filters))
);
```

## 7.2 Fenêtrage et buffer
- `bufferTime`, `bufferCount` : regrouper.
- `windowTime` : produire des Observables de fenêtres.

Usage : batching, analytics, réduction du bruit.

## 7.3 Backpressure (gestion de débit)
RxJS ne force pas une stratégie unique, mais propose des outils :
- `throttleTime`, `auditTime`, `sampleTime`
- `bufferTime` + `mergeMap` avec concurrence limitée

Exemple : limiter les envois d’events
```ts
events$.pipe(
  bufferTime(1000),
  filter(batch => batch.length > 0),
  mergeMap(batch => http.post('/api/events', batch))
);
```

## 7.4 `defer` : création paresseuse
- Crée l’Observable **au moment de l’abonnement**.
- Important pour capter un état courant.

```ts
const token$ = defer(() => of(localStorage.getItem('token')));
```

## 7.5 `iif` : branchement conditionnel
```ts
import { iif, of } from 'rxjs';

const result$ = iif(
  () => isLoggedIn(),
  this.http.get('/api/private'),
  of(null)
);
```

## 7.6 `race`, `timeout` et annulation
- `race` : prend le premier flux qui émet.
- `timeout` : erreur si dépassement.

```ts
import { timeout } from 'rxjs/operators';

http$.pipe(timeout({ first: 3000 }));
```

## 7.7 Exercice
- Construire un pipeline : typeahead + timeout + fallback + cancel.

---

# 8) Intégration Angular : services, composants, async pipe, interop signals

## 8.1 Services : séparation des responsabilités
- `Service API` : appels HTTP (retourne des Observables cold).
- `Facade/Store` : orchestration, cache, état.

## 8.2 Composant : ViewModel et `async`
### Recommandation
- Construire un `vm$` (Observable d’un objet) et afficher via `async`.
- Minimiser les `subscribe()` manuels.

```ts
readonly vm$ = combineLatest({
  users: this.usersFacade.users$,
  loading: this.usersFacade.loading$,
  error: this.usersFacade.error$,
});
```

## 8.3 Interop avec Angular Signals (si Angular >= 16)
- `toSignal(observable$)` : exposer un signal dérivé.
- `toObservable(signal)` : réactivité bidirectionnelle.

> Point clé : conserver RxJS pour orchestration asynchrone et streams, utiliser Signals pour état local synchronisé UI.

## 8.4 Patterns de déclenchement
- `refresh$` Subject + `switchMap`.
- `route.params` + `switchMap` pour charger des données.
- `form.valueChanges` + `debounceTime`.

## 8.5 Exercice
- Page détail : `route.paramMap` → `switchMap` vers API → `shareReplay(1)` et UI state.

---

# 9) Tests RxJS (marble testing) & pièges

## 9.1 Pourquoi les marbles ?
- Tester la logique temporelle et la composition.
- Déterministe, rapide.

## 9.2 `TestScheduler` (principes)
Exemple simplifié (pseudo marbles) :
```ts
import { TestScheduler } from 'rxjs/testing';
import { map } from 'rxjs/operators';

describe('map', () => {
  it('should map values', () => {
    const scheduler = new TestScheduler((a, e) => expect(a).toEqual(e));

    scheduler.run(({ cold, expectObservable }) => {
      const source = cold(' -a-b-|', { a: 1, b: 2 });
      const result = source.pipe(map(x => x * 10));
      expectObservable(result).toBe(' -a-b-|', { a: 10, b: 20 });
    });
  });
});
```

## 9.3 Pièges fréquents
- **Nested subscribe** (subscribe dans subscribe) : préférer `switchMap/mergeMap`.
- `shareReplay` + erreurs : un `shareReplay` peut « mémoriser » une erreur selon la config ; prévoir invalidation/retry.
- Oublier l’annulation : `switchMap` pour les requêtes dépendantes de l’input.
- Concurrence non limitée : `mergeMap` sans limite sur flux intense.

---

# 10) Atelier de synthèse : mini-architecture réactive

## Objectif
Construire un module Angular « Users » :
- Liste paginée + filtre + search.
- Cache de liste.
- Détail utilisateur.
- UI states (loading/error).
- Gestion de concurrence (annulation sur search).

## Étapes proposées
1. **API service** : `getUsers(params)`, `getUser(id)`.
2. **Facade** :
   - `filters$` (BehaviorSubject)
   - `refresh$` (Subject)
   - `users$` = combineLatest(filters, refresh startWith) → `switchMap` API → `shareReplay(1)`
   - `vm$` = RemoteData
3. **Composant** :
   - FormControl + `valueChanges` → `debounceTime` → `distinctUntilChanged` → setFilters
   - Template : `vm$ | async`
4. **Bonus** : retry avec backoff + timeout.

### Exemple de squelette (Facade)
```ts
@Injectable({ providedIn: 'root' })
export class UsersFacade {
  private readonly refreshSubject = new Subject<void>();
  readonly refresh$ = this.refreshSubject.asObservable();

  private readonly filtersSubject = new BehaviorSubject<{ q: string; page: number }>({ q: '', page: 1 });
  readonly filters$ = this.filtersSubject.asObservable();

  readonly usersVm$ = combineLatest([
    this.filters$,
    this.refresh$.pipe(startWith(void 0)),
  ]).pipe(
    map(([filters]) => filters),
    switchMap(filters =>
      this.api.getUsers(filters).pipe(
        map(data => ({ status: 'success', data } as const)),
        startWith({ status: 'loading' } as const),
        retry({ count: 2, delay: (_e, i) => timer(200 * i) }),
        catchError(error => of({ status: 'error', error } as const))
      )
    ),
    shareReplay({ bufferSize: 1, refCount: true })
  );

  constructor(private api: UsersApi) {}

  setQuery(q: string) {
    const curr = this.filtersSubject.value;
    this.filtersSubject.next({ ...curr, q, page: 1 });
  }

  setPage(page: number) {
    const curr = this.filtersSubject.value;
    this.filtersSubject.next({ ...curr, page });
  }

  refresh() {
    this.refreshSubject.next();
  }
}
```

---

# Annexes

## A) Cheat sheet opérateurs (sélection)
- **Transformation** : `map`, `scan`, `reduce`
- **Filtrage** : `filter`, `take`, `takeUntil`, `first`, `distinctUntilChanged`
- **Timing** : `debounceTime`, `throttleTime`, `auditTime`, `delay`
- **Combinaison** : `combineLatest`, `zip`, `forkJoin`, `withLatestFrom`, `merge`
- **Higher-order** : `switchMap`, `mergeMap`, `concatMap`, `exhaustMap`
- **Partage** : `share`, `shareReplay`
- **Erreurs** : `catchError`, `retry`, `timeout`, `finalize`

## B) Recommandations de style
- Éviter `subscribe()` dans les services : retourner des Observables.
- Regrouper la logique RxJS dans facades/services plutôt que dans les templates.
- Utiliser des noms explicites (`users$`, `usersVm$`, `refresh$`).
- Centraliser la stratégie de retry/timeout selon les endpoints.

## C) Suggestions de déroulé (1 jour)
- Matin : sections 1–4 + ateliers.
- Après-midi : sections 5–8 + atelier de synthèse.
- Fin : tests (section 9) selon le temps.
