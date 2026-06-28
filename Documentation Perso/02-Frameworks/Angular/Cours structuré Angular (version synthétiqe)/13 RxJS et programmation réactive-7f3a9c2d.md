# Formation Angular — RxJS et programmation réactive

> **Public** : développeurs Angular (débutant → intermédiaire)
>
> **Pré-requis** : TypeScript, bases Angular (components/services), notions d’async (Promise)
>
> **Durée conseillée** : 1 à 2 jours (adaptable)
>
> **Objectifs pédagogiques**
>
> - Comprendre la **programmation réactive** et les **Observables**
> - Savoir **créer, transformer, combiner** des flux RxJS
> - Maîtriser les opérateurs clés : **map, filter, switchMap** (et proches)
> - Intégrer RxJS proprement dans Angular : **HttpClient**, **Forms**, **routing**, **gestion d’état**
> - Éviter les pièges courants : **subscriptions**, **fuites mémoire**, **concurrence**, **erreurs**
>
---

## Plan de formation

1. **Introduction à la programmation réactive**
   - Synchrone vs asynchrone
   - Pull vs Push
   - Pourquoi RxJS dans Angular
2. **Les fondamentaux RxJS**
   - Observable, Observer, Subscription
   - Cold vs Hot observables
   - Création d’observables (of, from, interval, timer)
3. **Opérateurs de transformation**
   - map, pluck, scan
   - filter, take, skip
4. **Opérateurs de combinaison**
   - merge, concat, combineLatest, withLatestFrom, forkJoin
5. **Gestion de la concurrence et des effets**
   - switchMap, mergeMap, concatMap, exhaustMap
   - debounceTime, throttleTime
   - tap (effets)
6. **Gestion des erreurs et stratégies de reprise**
   - catchError, retry, retryWhen, finalize
7. **RxJS dans Angular (cas pratiques)**
   - HttpClient : requêtes et annulation
   - AsyncPipe et bonnes pratiques
   - Reactive Forms : valueChanges, statusChanges
   - Router : événements et résolveurs
8. **Architecture et patterns réactifs**
   - Services « façade » (facade pattern)
   - Subject, BehaviorSubject, ReplaySubject
   - Partage de flux : share, shareReplay
   - Mini-store maison
9. **Atelier de synthèse**
   - Construire un module de recherche (UI + API) avec cache, annulation et gestion d’erreur
10. **Checklist & anti-patterns**
   - Subscriptions manuelles, nested subscribes
   - shareReplay mal utilisé
   - erreurs silencieuses

---

## 1) Introduction à la programmation réactive

### 1.1 Synchrone vs asynchrone
- **Synchrone** : le code s’exécute ligne par ligne.
- **Asynchrone** : une opération prend du temps (HTTP, timers, événements UI). Le résultat arrive plus tard.

Dans Angular, l’asynchrone est partout :
- appels `HttpClient`
- événements DOM (clic, input)
- routing
- WebSockets / SSE

### 1.2 Pull vs Push
- **Pull** : le consommateur demande les données (ex : tableau + boucle)
- **Push** : le producteur pousse des événements quand ils arrivent (ex : clic, stream)

RxJS est orienté **Push**.

### 1.3 Pourquoi RxJS dans Angular
Angular expose naturellement des APIs sous forme d’**Observable** :
- `HttpClient` retourne des `Observable`
- `FormControl.valueChanges` est un `Observable`
- `Router.events` est un `Observable`

L’intérêt :
- composer les flux (transformation, combinaison)
- gérer la concurrence (annuler/ignorer les requêtes)
- centraliser la gestion d’erreur
- réduire l’état mutable

---

## 2) Les fondamentaux RxJS

### 2.1 Observable, Observer, Subscription
- **Observable** : une source de données au fil du temps
- **Observer** : ce qui consomme (callbacks `next`, `error`, `complete`)
- **Subscription** : lien actif entre observable et observer, permet `unsubscribe()`

Schéma mental :

```
Producer (Observable)  --->  Consumer (Observer)
         |                     next/error/complete
         +---- Subscription (unsubscribe)
```

Exemple minimal :

```ts
import { Observable } from 'rxjs';

const obs$ = new Observable<number>((subscriber) => {
  subscriber.next(1);
  subscriber.next(2);
  subscriber.complete();
});

const sub = obs$.subscribe({
  next: (v) => console.log('value', v),
  error: (e) => console.error(e),
  complete: () => console.log('done')
});

sub.unsubscribe();
```

### 2.2 Cold vs Hot observables
- **Cold** : chaque abonnement déclenche sa propre exécution (ex : `HttpClient.get`)
- **Hot** : source partagée (ex : événements DOM, `Subject`)

Démo conceptuelle :

```ts
import { interval, take } from 'rxjs';

const cold$ = interval(1000).pipe(take(3));

cold$.subscribe(v => console.log('A', v));
setTimeout(() => cold$.subscribe(v => console.log('B', v)), 1500);
// B démarre à 0 quand il s’abonne => cold
```

### 2.3 Création d’observables courants

- `of(...values)` : émettre une liste de valeurs
- `from(promise|array)` : convertir une Promise ou un tableau en observable
- `interval(ms)` : compteur infini
- `timer(dueTime, period?)` : délai puis intervalle

```ts
import { of, from, interval, timer } from 'rxjs';

of(1,2,3).subscribe(console.log);
from(fetch('/api')).subscribe(console.log);
interval(1000).subscribe(console.log);
timer(2000).subscribe(() => console.log('2s'));
```

---

## 3) Opérateurs de transformation

> Un opérateur est une fonction appliquée via `pipe(...)`.

### 3.1 `map` — transformer la valeur

```ts
import { of, map } from 'rxjs';

of(1, 2, 3).pipe(
  map(x => x * 10)
).subscribe(console.log); // 10, 20, 30
```

Cas typique Angular : adapter un DTO backend vers un modèle UI.

```ts
this.http.get<UserDto[]>('/api/users').pipe(
  map(dtos => dtos.map(dto => ({
    id: dto.id,
    fullName: `${dto.firstName} ${dto.lastName}`
  })))
);
```

### 3.2 `filter` — filtrer les émissions

```ts
import { from, filter } from 'rxjs';

from([1,2,3,4,5]).pipe(
  filter(x => x % 2 === 0)
).subscribe(console.log); // 2,4
```

Cas typique : ignorer les saisies trop courtes.

### 3.3 `scan` — accumulation (comme reduce mais progressif)

```ts
import { of, scan } from 'rxjs';

of(1,2,3).pipe(
  scan((acc, v) => acc + v, 0)
).subscribe(console.log); // 1, 3, 6
```

---

## 4) Opérateurs de combinaison

### 4.1 `merge` — fusionner des flux

```ts
import { merge, fromEvent, map } from 'rxjs';

const clicks$ = fromEvent(document, 'click').pipe(map(() => 'click'));
const keys$ = fromEvent(document, 'keydown').pipe(map(() => 'key'));

merge(clicks$, keys$).subscribe(console.log);
```

### 4.2 `concat` — enchaîner des flux (séquentiel)

```ts
import { concat, of, delay } from 'rxjs';

concat(
  of('A').pipe(delay(1000)),
  of('B').pipe(delay(1000))
).subscribe(console.log); // A puis B
```

### 4.3 `combineLatest` — combiner les dernières valeurs

Idéal pour une UI pilotée par plusieurs contrôles.

```ts
import { combineLatest, map } from 'rxjs';

combineLatest([this.search$, this.category$]).pipe(
  map(([text, cat]) => ({ text, cat }))
);
```

### 4.4 `forkJoin` — attendre la fin de plusieurs observables

Utile pour charger plusieurs ressources en parallèle (qui complètent).

```ts
import { forkJoin } from 'rxjs';

forkJoin({
  user: this.http.get('/api/user'),
  settings: this.http.get('/api/settings')
}).subscribe(({ user, settings }) => {
  // exécuté quand les deux ont complété
});
```

---

## 5) Concurrence & effets : `switchMap` et amis

Les opérateurs de *mapping vers observable* transforment un `Observable<A>` en `Observable<B>` en lançant un observable interne.

### 5.1 `switchMap` — basculer sur la dernière requête (annulation)
- Annule l’observable précédent quand une nouvelle valeur arrive.
- Parfait pour recherche/autocomplete.

```ts
import { debounceTime, distinctUntilChanged, switchMap } from 'rxjs';

this.searchControl.valueChanges.pipe(
  debounceTime(300),
  distinctUntilChanged(),
  switchMap(term => this.http.get(`/api/search?q=${term}`))
).subscribe(results => {
  this.results = results;
});
```

### 5.2 `mergeMap` — paralléliser
Lance toutes les requêtes, résultats dans l’ordre d’arrivée.

```ts
import { from, mergeMap } from 'rxjs';

from([1,2,3]).pipe(
  mergeMap(id => this.http.get(`/api/items/${id}`))
).subscribe(console.log);
```

### 5.3 `concatMap` — séquencer
Une seule à la fois, conserve l’ordre.

### 5.4 `exhaustMap` — ignorer tant que la précédente tourne
Utile pour un bouton « submit » (éviter double clic).

```ts
import { fromEvent, exhaustMap } from 'rxjs';

fromEvent(this.saveBtn.nativeElement, 'click').pipe(
  exhaustMap(() => this.http.post('/api/save', this.form.value))
).subscribe();
```

### 5.5 `tap` — effets de bord (logging, loading state)

```ts
import { tap, finalize } from 'rxjs';

this.loading = true;

this.http.get('/api/data').pipe(
  tap(() => console.log('request started')),
  finalize(() => this.loading = false)
).subscribe();
```

---

## 6) Gestion des erreurs

### 6.1 `catchError` — capturer et reprendre

```ts
import { catchError, of } from 'rxjs';

this.http.get('/api/data').pipe(
  catchError(err => {
    // log + fallback
    console.error(err);
    return of([]);
  })
).subscribe(data => this.data = data);
```

### 6.2 `retry` — retenter

```ts
import { retry } from 'rxjs';

this.http.get('/api/data').pipe(
  retry({ count: 2, delay: 500 })
).subscribe();
```

### 6.3 `finalize` — s’exécute dans tous les cas
Pratique pour masquer un spinner, relâcher une ressource.

---

## 7) RxJS dans Angular (bonnes pratiques)

### 7.1 Préférer l’`AsyncPipe`
- Gère l’abonnement/désabonnement automatiquement
- Réduit le risque de fuites mémoire

**Component :**

```ts
results$ = this.searchControl.valueChanges.pipe(
  debounceTime(300),
  distinctUntilChanged(),
  switchMap(term => this.api.search(term))
);
```

**Template :**

```html
<ul>
  <li *ngFor="let r of results$ | async">{{ r.label }}</li>
</ul>
```

### 7.2 Éviter le « nested subscribe »
Anti-pattern :

```ts
this.a$.subscribe(a => {
  this.b$(a).subscribe(b => { /* ... */ });
});
```

Préférer :

```ts
this.a$.pipe(
  switchMap(a => this.b$(a))
).subscribe();
```

### 7.3 Désabonnement : `takeUntilDestroyed` (Angular récent)

```ts
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';

this.router.events.pipe(
  takeUntilDestroyed(this.destroyRef)
).subscribe();
```

(Alternative : `takeUntil(this.destroy$)` dans les versions plus anciennes.)

### 7.4 Reactive Forms : `valueChanges`

```ts
this.form.valueChanges.pipe(
  debounceTime(200),
  map(v => v.email?.trim()),
  filter(email => !!email && email.includes('@'))
).subscribe(validEmail => {
  // activer une action, proposer une suggestion, etc.
});
```

### 7.5 Routing : écouter `Router.events`

```ts
import { filter } from 'rxjs';
import { NavigationEnd } from '@angular/router';

this.router.events.pipe(
  filter((e): e is NavigationEnd => e instanceof NavigationEnd)
).subscribe(e => console.log('navigated', e.urlAfterRedirects));
```

---

## 8) Subjects, partage et mini-store

### 8.1 `Subject` vs `BehaviorSubject`
- `Subject` : ne garde pas de valeur courante
- `BehaviorSubject` : conserve la dernière valeur et l’émet aux nouveaux abonnés

```ts
import { BehaviorSubject } from 'rxjs';

private readonly _count$ = new BehaviorSubject<number>(0);
readonly count$ = this._count$.asObservable();

increment() {
  this._count$.next(this._count$.value + 1);
}
```

### 8.2 Partager un observable : `shareReplay`
Cas : une requête HTTP que plusieurs composants consomment.

```ts
import { shareReplay } from 'rxjs';

readonly config$ = this.http.get('/api/config').pipe(
  shareReplay({ bufferSize: 1, refCount: true })
);
```

**Attention** : sans `refCount`, le cache reste vivant même sans abonnés.

### 8.3 Pattern « Facade »
Objectif : exposer des `Observable` publics et encapsuler les `Subject`.

```ts
@Injectable({ providedIn: 'root' })
export class UsersFacade {
  private readonly refresh$ = new Subject<void>();

  readonly users$ = this.refresh$.pipe(
    startWith(void 0),
    switchMap(() => this.http.get<User[]>('/api/users')),
    shareReplay({ bufferSize: 1, refCount: true })
  );

  reload() { this.refresh$.next(); }

  constructor(private http: HttpClient) {}
}
```

---

## 9) Atelier — Recherche avec annulation, cache et erreurs

### 9.1 Énoncé
Construire une recherche avec :
- saisie utilisateur (Reactive Forms)
- `debounceTime` (300ms)
- annulation des requêtes via `switchMap`
- gestion d’erreur avec message UI
- cache simple par terme (`Map`)

### 9.2 Service API

```ts
@Injectable({ providedIn: 'root' })
export class SearchApi {
  constructor(private http: HttpClient) {}

  search(term: string) {
    return this.http.get<SearchResult[]>(`/api/search?q=${encodeURIComponent(term)}`);
  }
}
```

### 9.3 Facade avec cache

```ts
@Injectable({ providedIn: 'root' })
export class SearchFacade {
  private cache = new Map<string, SearchResult[]>();

  search(term: string) {
    const key = term.trim().toLowerCase();
    const cached = this.cache.get(key);
    if (cached) return of(cached);

    return this.api.search(key).pipe(
      tap(results => this.cache.set(key, results))
    );
  }

  constructor(private api: SearchApi) {}
}
```

### 9.4 Component (flux complet)

```ts
readonly control = new FormControl('', { nonNullable: true });

readonly vm$ = this.control.valueChanges.pipe(
  startWith(this.control.value),
  map(v => v.trim()),
  debounceTime(300),
  distinctUntilChanged(),
  switchMap(term => {
    if (term.length < 2) {
      return of({ term, results: [], error: null as string | null });
    }

    return this.facade.search(term).pipe(
      map(results => ({ term, results, error: null as string | null })),
      catchError(() => of({ term, results: [], error: 'Erreur de recherche' }))
    );
  }),
  shareReplay({ bufferSize: 1, refCount: true })
);
```

Template :

```html
<input [formControl]="control" placeholder="Rechercher..." />

<ng-container *ngIf="vm$ | async as vm">
  <p *ngIf="vm.error" class="error">{{ vm.error }}</p>

  <ul>
    <li *ngFor="let r of vm.results">{{ r.label }}</li>
  </ul>
</ng-container>
```

---

## 10) Checklist & anti-patterns

### À faire
- Utiliser `AsyncPipe` dès que possible
- Nommer les Observables avec suffixe `$` (`users$`, `vm$`)
- Centraliser appels HTTP dans services/facades
- Gérer les erreurs (au moins `catchError` + UX)
- Gérer la concurrence (`switchMap` en recherche)

### À éviter
- **Nested subscribe** (complexité + fuites)
- Stocker des valeurs asynchrones dans trop de variables mutables
- `shareReplay(1)` sans réfléchir au cycle de vie (utiliser `refCount`)
- Laisser un flux mourir sur erreur si on veut une UI résiliente

---

## Annexes

### A) Aide-mémoire opérateurs essentiels

| Catégorie | Opérateurs | Usage |
|---|---|---|
| Transformation | `map`, `scan` | transformer, accumuler |
| Filtrage | `filter`, `take`, `first` | ignorer / limiter |
| Concurrence | `switchMap`, `mergeMap`, `concatMap`, `exhaustMap` | annuler/paralléliser/séquencer |
| Combinaison | `combineLatest`, `withLatestFrom`, `forkJoin`, `merge` | composer des sources |
| Temps | `debounceTime`, `throttleTime`, `delay` | UX + réseau |
| Erreurs | `catchError`, `retry`, `finalize` | robustesse |

### B) Glossaire
- **Observable** : flux de valeurs dans le temps
- **Subscription** : connexion active à un observable
- **Operator** : fonction de composition via `pipe`
- **Subject** : observable « hot » avec émission manuelle (`next`)

---

## Conclusion
RxJS est le cœur de la gestion de l’asynchrone dans Angular. Savoir manipuler les Observables, comprendre la concurrence (`switchMap` et amis) et adopter les bonnes pratiques (`AsyncPipe`, facades, gestion d’erreurs) permet de construire des applications plus **prévisibles**, **maintenables** et **réactives**.

