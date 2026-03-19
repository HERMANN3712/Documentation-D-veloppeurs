# Formation Angular #10 — Comparaison **Signals** et **RxJS**

> Objectif : comprendre **quand utiliser les Signals**, **quand RxJS reste indispensable**, et **comment les combiner** proprement dans une architecture Angular moderne, sans les opposer inutilement.

---

## Informations générales

- **Public visé** : développeurs Angular (intermédiaires à avancés), formateurs, leads.
- **Pré-requis** :
  - bases Angular (components, services, DI, change detection)
  - bases RxJS (Observable, Subject, operators, subscription)
  - notions d’async/await et HTTP
- **Durée conseillée** : 1 journée (6–7h) ou 2 demi-journées.
- **Version** : Angular 16+ (Signals), exemples compatibles 17+.

---

## Objectifs pédagogiques

À l’issue de la formation, vous saurez :

1. Expliquer les différences de **modèle mental** entre Signals et Observables.
2. Identifier les cas d’usage où les **Signals** excellent (état local, UI fine grained).
3. Identifier les cas d’usage où **RxJS** est le meilleur outil (asynchronisme complexe, orchestration, streaming).
4. Combiner Signals et RxJS (interop) : **fromObservable / toObservable**, effets, stratégies de stockage.
5. Concevoir une architecture avancée **hybride** (UI en Signals, flux métier en RxJS).
6. Éviter les pièges : sur-réactivité, fuites, duplication, effets non maîtrisés.

---

## Plan de formation (vue d’ensemble)

1. **Contexte et problématique** : pourquoi Angular propose Signals en plus de RxJS
2. **Rappels RxJS** : ce que RxJS fait très bien
3. **Fondamentaux Signals** : état, computed, effect, granularité
4. **Comparaison structurée** : responsabilités, performance, ergonomie, testabilité
5. **Cas d’usage concrets** : UI local vs flux asynchrones complexes
6. **Interop Signals ↔ RxJS** : patterns recommandés
7. **Architecture avancée hybride** : guidelines et anti-patterns
8. **Atelier / mini-projet** : refactor d’un écran Angular “Observable-only” vers hybride

---

# 1) Contexte : Signals et RxJS, pourquoi les deux ?

Angular utilise RxJS depuis longtemps comme base de la programmation réactive. Avec l’introduction des **Signals**, Angular apporte un outil natif pour gérer la réactivité **synchrone** et **fine-grained** au niveau UI.

### Message clé de la formation

- **Signals** : excellents pour **l’état local** et la **réactivité fine** dans l’interface.
- **RxJS** : essentiel pour **les flux asynchrones complexes**, **événements multiples**, **streaming**, **websockets**, et **orchestration de requêtes**.
- Une architecture mature **combine** Signals et RxJS, au lieu de les opposer.

---

# 2) Rappels RxJS (utile pour cadrer la comparaison)

## 2.1 Observable : push async et multi-valeurs

Un `Observable<T>` représente un flux pouvant produire :

- 0 à n valeurs dans le temps
- un `complete` (fin)
- une `error`

```ts
import { Observable } from 'rxjs';

const o$ = new Observable<number>((subscriber) => {
  subscriber.next(1);
  setTimeout(() => subscriber.next(2), 1000);
  setTimeout(() => subscriber.complete(), 2000);
});
```

## 2.2 La force de RxJS : la composition

RxJS n’est pas seulement “des Observables” : c’est un **langage** de transformation/composition de flux via des opérateurs.

- mapping : `map`, `switchMap`, `mergeMap`, `concatMap`, `exhaustMap`
- filtrage : `filter`, `distinctUntilChanged`, `debounceTime`, `throttleTime`
- combinaison : `combineLatest`, `forkJoin`, `merge`, `concat`, `race`
- multicasting : `shareReplay`, `publish`, `refCount`
- coordination : `retry`, `catchError`, `timeout`

## 2.3 Exemple express : orchestration de requêtes

```ts
search(term$: Observable<string>) {
  return term$.pipe(
    debounceTime(300),
    distinctUntilChanged(),
    switchMap(term => this.http.get(`/api/search?q=${term}`)),
    catchError(() => of([]))
  );
}
```

---

# 3) Fondamentaux Signals

## 3.1 Le modèle : état synchrone + dépendances trackées

Un Signal est une valeur réactive *synchronisée* en mémoire.

- `signal(initialValue)` : état mutable contrôlé
- `computed(() => ...)` : valeur dérivée mémoïsée
- `effect(() => ...)` : effet déclenché quand ses dépendances changent

```ts
import { signal, computed, effect } from '@angular/core';

const count = signal(0);
const doubled = computed(() => count() * 2);

effect(() => {
  console.log('count=', count(), 'doubled=', doubled());
});

count.set(1);
count.update(v => v + 1);
```

## 3.2 Granularité : mise à jour fine du template

Les Signals permettent à Angular d’optimiser les mises à jour du DOM en ne recalculant/rendant que ce qui dépend réellement de la valeur.

Exemple template :

```html
<button (click)="count.set(count() + 1)">+</button>
<p>Count: {{ count() }}</p>
<p>Doubled: {{ doubled() }}</p>
```

## 3.3 Principes importants (à garder en tête)

- Les Signals sont **synchrones** : la lecture `count()` retourne la valeur immédiatement.
- Les dépendances sont trackées automatiquement **à l’exécution**.
- `computed` est **lazy** (calculé à la demande) et **mémoïsé**.
- `effect` est pour les **side effects** (log, persistance, DOM non Angular, appels conditionnels *avec prudence*).

---

# 4) Comparaison structurée Signals vs RxJS

## 4.1 Tableau comparatif (synthèse)

| Critère | Signals | RxJS |
|---|---|---|
| Nature | Valeur réactive synchrone | Flux asynchrone/multi-événements |
| Modèle mental | **State → derive → render** | **Stream → transform → subscribe** |
| Force principale | État local UI, dérivations simples, granularité | Orchestration async, composition complexe, streaming |
| Gestion d’erreur | Pas “native” (à implémenter) | `catchError`, `retry`, `materialize`… |
| Annulation | Pas le concept principal | `switchMap` annule, `takeUntil`, `AbortSignal` via interop |
| Multiplexing | Non (plutôt état) | Oui (`merge`, `combineLatest`, `withLatestFrom`…) |
| Backpressure/throttling | Non natif | `throttleTime`, `buffer`, `window`, etc. |
| Debug | simple (valeurs) | plus complexe (streams) |
| Test | simple (valeurs synchrones) | nécessite marbles / tests async |
| Intégration template | directe (`{{ sig() }}`) | `async` pipe / `toSignal` |

## 4.2 En bref : quand choisir quoi ?

### Signals conviennent particulièrement à :

- l’**état local** d’un composant (UI state)
- les **états dérivés** (computed) sans asynchronisme complexe
- la **réactivité fine** du template et de l’UI
- remplacer certains usages de `BehaviorSubject` “juste pour stocker une valeur”

### RxJS reste essentiel pour :

- flux **asynchrones** complexes
- gestion de **plusieurs sources** d’événements
- **streaming** (SSE), **websockets**, notifications, temps réel
- orchestration de requêtes et politiques de retry/backoff
- scénarios de concurrence : `switchMap`, `mergeMap` etc.

---

# 5) Cas d’usage concrets

## 5.1 Cas 1 — État local UI : Signals gagnent en simplicité

**Problème** : gérer l’ouverture/fermeture d’un panneau, un filtre local, une sélection.

### Variante RxJS (souvent surdimensionnée)

```ts
readonly opened$ = new BehaviorSubject(false);

toggle() { this.opened$.next(!this.opened$.value); }
```

### Variante Signals (préférée)

```ts
opened = signal(false);

toggle() { this.opened.update(v => !v); }
```

Pourquoi c’est mieux ici :
- pas de subscription
- pas d’async pipe
- état local naturellement synchronisé

## 5.2 Cas 2 — Formulaire + validations + UI réactive

Signals pour l’état de présentation (loading, errors affichés) et RxJS pour l’orchestration (debounce, cancel, concurrent).

```ts
loading = signal(false);
error = signal<string | null>(null);

results$ = this.form.controls.q.valueChanges.pipe(
  debounceTime(250),
  distinctUntilChanged(),
  tap(() => { this.loading.set(true); this.error.set(null); }),
  switchMap(q => this.http.get(`/api/search?q=${q}`).pipe(
    catchError(err => { this.error.set('Erreur de recherche'); return of([]); }),
    finalize(() => this.loading.set(false))
  ))
);
```

Observation : le flux “métier async” reste RxJS, l’état UI est en Signals.

## 5.3 Cas 3 — Websocket / streaming : RxJS au centre

```ts
const socket$ = webSocket<{ type: string; payload: any }>('wss://example');

const notifications$ = socket$.pipe(
  filter(msg => msg.type === 'notification'),
  map(msg => msg.payload),
  shareReplay({ bufferSize: 1, refCount: true })
);
```

Ici, Signals ne remplacent pas RxJS : ce sont deux niveaux différents.

---

# 6) Interop : combiner Signals et RxJS

Angular fournit des passerelles utiles (à partir d’Angular 16+ via `@angular/core/rxjs-interop`).

## 6.1 Convertir Observable → Signal : `toSignal()`

Pour consommer un flux RxJS dans le template et dériver via `computed`.

```ts
import { toSignal } from '@angular/core/rxjs-interop';

readonly user$ = this.http.get<User>('/api/me').pipe(shareReplay(1));
readonly user = toSignal(this.user$, { initialValue: null });

readonly displayName = computed(() => this.user()?.name ?? 'Invité');
```

Bonnes pratiques :
- fournir `initialValue` (ou gérer `undefined`) pour éviter les trous de valeur
- conserver l’Observable comme source si vous avez besoin d’opérateurs RxJS

## 6.2 Convertir Signal → Observable : `toObservable()`

Utile si vous avez un état local en signal, mais devez le brancher sur une chaîne RxJS.

```ts
import { toObservable } from '@angular/core/rxjs-interop';

term = signal('');
term$ = toObservable(this.term);

results$ = this.term$.pipe(
  debounceTime(300),
  switchMap(q => this.http.get(`/api/search?q=${q}`))
);
```

## 6.3 Pattern recommandé : “RxJS pour l’IO, Signals pour la vue”

- **Services data** : exposent des `Observable` (HTTP, websocket, events)
- **Composants** :
  - consomment via `toSignal()` quand ils ont besoin d’une valeur synchronisée
  - utilisent `computed` pour dériver des états d’affichage
  - gardent RxJS pour les scénarios d’orchestration

---

# 7) Architecture avancée : comment les combiner sans conflit

## 7.1 Principes d’architecture

1. **Ne pas tout convertir** : garder RxJS lorsque le problème est un flux.
2. **Éviter la double source de vérité** : une entité “source” (Observable *ou* Signal), l’autre est dérivée.
3. **État local UI en Signals** : ouvert/fermé, sélection, tri local, flags de vue.
4. **État applicatif partagé** :
   - soit en RxJS (store observable) si vous avez déjà une architecture stream
   - soit en Signals (signal store) si l’état est principalement synchronisé et localement dérivable
5. **Effets maîtrisés** : `effect()` doit rester prévisible, éviter les appels HTTP non contrôlés dans un effect sans garde.

## 7.2 Anti-patterns fréquents

### Anti-pattern A — Transformer toute donnée en Signal “par principe”
Conséquence : perte des opérateurs RxJS, annulation et orchestration difficiles.

### Anti-pattern B — Dupliquer l’état : `items$` + `itemsSignal` mutables séparément
Conséquence : divergences, bugs intermittents.

### Anti-pattern C — Mettre de l’IO dans `effect()` sans stratégie d’annulation
Un `effect` peut se relancer souvent et provoquer des requêtes multiples.

**Préférer** orchestrer les appels via RxJS (`switchMap`) ou des APIs contrôlées.

## 7.3 Recommandations pragmatiques

- **Un composant = des Signals** pour :
  - état d’interaction
  - dérivations UI
  - mapping de données en view-model
- **Un service = RxJS** pour :
  - appels HTTP, websockets
  - caching partagé
  - coordination multi-sources

---

# 8) Atelier guidé — Refactor vers une approche hybride

## 8.1 Énoncé

Vous avez un composant “liste produits” qui :
- lit un terme de recherche
- appelle une API
- gère un loading, une erreur, et une vue filtrée

Objectif :
- garder RxJS pour la recherche (debounce/cancel)
- passer l’état de vue (loading, error, sélection) en Signals
- exposer au template des valeurs synchrones (signals) pour réduire les async pipes

## 8.2 Base RxJS (avant)

```ts
term$ = new Subject<string>();
loading$ = new BehaviorSubject(false);
error$ = new BehaviorSubject<string | null>(null);

products$ = this.term$.pipe(
  debounceTime(300),
  distinctUntilChanged(),
  tap(() => { this.loading$.next(true); this.error$.next(null); }),
  switchMap(term => this.http.get<Product[]>(`/api/products?q=${term}`).pipe(
    catchError(() => { this.error$.next('Erreur'); return of([]); }),
    finalize(() => this.loading$.next(false))
  )),
  shareReplay(1)
);
```

## 8.3 Version hybride (après)

```ts
import { signal, computed } from '@angular/core';
import { toSignal } from '@angular/core/rxjs-interop';
import { debounceTime, distinctUntilChanged, switchMap, catchError, of, tap, finalize, shareReplay } from 'rxjs';

term = signal('');
loading = signal(false);
error = signal<string | null>(null);

private products$ = toObservable(this.term).pipe(
  debounceTime(300),
  distinctUntilChanged(),
  tap(() => { this.loading.set(true); this.error.set(null); }),
  switchMap(term => this.http.get<Product[]>(`/api/products?q=${term}`).pipe(
    catchError(() => { this.error.set('Erreur'); return of([]); }),
    finalize(() => this.loading.set(false))
  )),
  shareReplay({ bufferSize: 1, refCount: true })
);

products = toSignal(this.products$, { initialValue: [] as Product[] });

selectedId = signal<string | null>(null);
selected = computed(() => this.products().find(p => p.id === this.selectedId()) ?? null);
```

Template :

```html
<input [value]="term()" (input)="term.set($any($event.target).value)" />

@if (loading()) {
  <p>Chargement…</p>
}
@if (error()) {
  <p class="error">{{ error() }}</p>
}

<ul>
  @for (p of products(); track p.id) {
    <li (click)="selectedId.set(p.id)">
      {{ p.name }}
    </li>
  }
</ul>

@if (selected()) {
  <section>
    <h3>Détail</h3>
    <pre>{{ selected() | json }}</pre>
  </section>
}
```

Points à discuter :
- `products$` conserve l’orchestration RxJS
- `products` en signal simplifie la conso template
- états UI (`loading`, `error`, `selectedId`) sont en signals

---

# 9) Checklist décisionnelle (à réutiliser en projet)

## Si la donnée est…

### …un état local synchronisé (UI)
- ✅ Signals

### …une valeur dérivée de plusieurs états locaux
- ✅ `computed` Signals

### …un flux async multi-événements (websocket, scroll, typeahead, events)
- ✅ RxJS

### …une orchestration (annulation, concurrence, retries, timeouts)
- ✅ RxJS

### …une donnée RxJS à afficher dans la vue
- ✅ `toSignal()` (ou `async` pipe si plus simple)

### …un signal à utiliser dans une chaîne RxJS
- ✅ `toObservable()`

---

# 10) Conclusion

- Les **Signals** apportent une solution élégante pour l’**état local** et la **réactivité fine** de l’interface.
- **RxJS** reste incontournable pour la gestion des **flux asynchrones** complexes : multiples événements, streaming, websockets, orchestration.
- L’approche avancée consiste à **combiner** :
  - RxJS comme “moteur de flux/IO”
  - Signals comme “moteur d’état UI et de dérivations synchrones”

---

## Annexes

### A) Glossaire

- **Signal** : valeur réactive synchronisée, lisible via `sig()`.
- **Computed** : signal dérivé mémoïsé.
- **Effect** : exécute un effet de bord quand les dépendances changent.
- **Observable** : flux de valeurs asynchrones.
- **Subject/BehaviorSubject** : observable “chaud” (multicast), avec ou sans valeur initiale.

### B) Sujets de discussion pour approfondir

- Signal Store vs RxJS Store (NgRx, Akita, etc.)
- Stratégies de caching : `shareReplay`, cache HTTP, invalidation
- Gestion des erreurs au niveau UI (énumérations d’état)
- Tests : tests synchrones pour signals vs marbles pour RxJS
