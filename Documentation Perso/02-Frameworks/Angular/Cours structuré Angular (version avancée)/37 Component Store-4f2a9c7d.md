# Formation Angular — Component Store (NgRx)

> **Positionnement** : *Component Store* est une solution intermédiaire entre l’état local (simple `@Input()` / `signals` / services ad hoc) et **NgRx Store** global. Il permet de gérer l’état d’une **feature** ou d’un **composant complexe** avec une approche **réactive**, en limitant la charge d’infrastructure d’un store global.

---

## Sommaire

1. [Objectifs pédagogiques](#objectifs-pédagogiques)
2. [Pré-requis](#pré-requis)
3. [Public visé](#public-visé)
4. [Durée et format](#durée-et-format)
5. [Plan détaillé](#plan-détaillé)
6. [Cours complet (contenu détaillé)](#cours-complet-contenu-détaillé)
   1. [Quand utiliser Component Store ?](#1-quand-utiliser-component-store-)
   2. [Concepts clés](#2-concepts-clés)
   3. [Installation & setup](#3-installation--setup)
   4. [Premier store : state, selectors, updaters](#4-premier-store--state-selectors-updaters)
   5. [Effects : orchestration asynchrone](#5-effects--orchestration-asynchrone)
   6. [Architecture recommandée](#6-architecture-recommandée)
   7. [Patterns avancés](#7-patterns-avancés)
   8. [Tests unitaires](#8-tests-unitaires)
   9. [Anti-patterns & erreurs fréquentes](#9-anti-patterns--erreurs-fréquentes)
   10. [Migration depuis services locaux / mini-stores](#10-migration-depuis-services-locaux--mini-stores)
   11. [Atelier fil rouge](#11-atelier-fil-rouge)
7. [Annexes : cheat sheet](#annexes--cheat-sheet)

---

## Objectifs pédagogiques

À la fin de cette formation, vous saurez :

- Expliquer la **différence** entre état local, Component Store et NgRx Store global.
- Créer un **Component Store** pour gérer une feature/composant complexe.
- Concevoir des **selectors** performants et composables.
- Utiliser des **updaters** pour des transitions d’état déterministes.
- Implémenter des **effects** pour appels HTTP, orchestration, gestion d’erreurs.
- Structurer une feature avec un store **scopé** et testable.
- Éviter les **anti-patterns** et choisir le bon niveau de store.

---

## Pré-requis

- Angular : composants, DI, RxJS de base (`Observable`, `pipe`, `map`, `switchMap`…)
- TypeScript : types, interfaces
- HTTPClient et services Angular

---

## Public visé

Développeurs Angular (intermédiaire à avancé), tech leads, formateurs.

---

## Durée et format

- **1 jour** (7h) recommandé (ou 2×3h30)
- Alternance : théorie + démonstrations + atelier guidé

---

## Plan détaillé

1. **Positionnement & cas d’usage**
   - État local vs store global vs Component Store
   - Critères de choix
2. **Concepts Component Store**
   - State, selectors, updaters, effects
   - Immutabilité, composition, performances
3. **Mise en place**
   - Installation, création, injection, scope
4. **Selectors & ViewModel**
   - `select`, `combineLatest` interne, `vm$`
   - Optimisations (memo, `distinctUntilChanged`)
5. **Updaters & transitions d’état**
   - Mises à jour atomiques
   - Normalisation simple
6. **Effects**
   - Appels API, concurrence (`switchMap`, `exhaustMap`, `concatMap`)
   - Gestion des erreurs & loading
7. **Architecture**
   - Store par composant/feature
   - Relations avec services API
   - Standalone + route scope
8. **Tests**
   - Tests selectors / updaters / effects
9. **Patterns avancés**
   - Entités, pagination, recherche/debounce
   - Annulation, retry, cache
   - Interaction entre stores
10. **Atelier fil rouge**

---

# Cours complet (contenu détaillé)

## 1. Quand utiliser Component Store ?

### 1.1 Le problème
Dans Angular, l’état peut rapidement se disperser :

- variables locales dans le composant,
- `BehaviorSubject` dans des services,
- plusieurs flux RxJS combinés dans le composant,
- logique d’orchestration (loading, erreurs, pagination) imbriquée dans le template.

Résultat : composant difficile à tester, effets de bord, duplication.

### 1.2 Positionnement

| Solution | Portée | Avantages | Inconvénients |
|---|---:|---|---|
| État local (propriétés du composant) | composant | simple, direct | devient ingérable si complexe/asynchrone |
| Service ad hoc (Subjects) | feature | flexible | souvent non standardisé, tests + difficiles |
| **Component Store** | feature/composant complexe | pattern clair, réactif, testable | nécessite RxJS et discipline |
| NgRx Store global | application | cohérence globale, devtools, actions/reducers | plus de boilerplate, surcoût pour petites features |

### 1.3 Critères de choix
Utilisez **Component Store** quand :

- une feature a un état **non trivial** (loading, erreurs, filtres, pagination, sélection…)
- le besoin est **scopé** (pas forcément global à toute l’app)
- vous voulez un modèle **réactif** et **testable**
- l’infra NgRx Store global est trop lourde pour cette zone.

Évitez-le si :

- l’état est très simple (2–3 champs)
- il doit être partagé globalement dans toute l’application
- votre équipe n’est pas à l’aise avec RxJS (ou sans temps d’acculturation)

---

## 2. Concepts clés

Component Store (de `@ngrx/component-store`) fournit une classe `ComponentStore<State>` avec :

1. **State** : source unique de vérité (immutabilité)
2. **Selectors** : dérivation de state → données UI
3. **Updaters** : fonctions pures qui transforment l’état
4. **Effects** : orchestration de side effects (HTTP, logs…), déclenchés par des inputs.

### 2.1 State : la forme de l’état
Bonnes pratiques :

- **minimal** : ne stockez que ce qui est nécessaire
- **dérivez** le reste via selectors (ex: `filteredItems$`)
- gardez l’état **sérialisable** autant que possible.

### 2.2 Immutabilité
On ne modifie pas l’état en place. On **retourne un nouvel objet**.

- ✅ `return { ...state, loading: true }`
- ❌ `state.loading = true; return state;`

### 2.3 Selectors
Selectors = Observables dérivés via `this.select(...)`.

- améliorer la lisibilité du composant (un seul `vm$`)
- mieux contrôler les recalculs (`distinctUntilChanged`)

### 2.4 Updaters
Updaters = transitions d’état **pures** et **atomiques**.

- appel : `this.setLoading(true)`
- garantit : `state => newState` sans side effect.

### 2.5 Effects
Effects = pipeline RxJS qui reçoit un flux d’inputs et exécute du code asynchrone.

- gère concurrence et annulation (`switchMap`, `exhaustMap`…)
- centralise loading/erreurs
- améliore la testabilité (stubber services API)

---

## 3. Installation & setup

### 3.1 Installation

```bash
npm i @ngrx/component-store
```

> Component Store peut être utilisé **sans** NgRx Store global.

### 3.2 Créer un store
On crée un service qui étend `ComponentStore<State>`.

#### Exemple d’état

```ts
export interface TodosState {
  todos: Todo[];
  filter: 'all' | 'active' | 'done';
  loading: boolean;
  error: string | null;
}

const initialState: TodosState = {
  todos: [],
  filter: 'all',
  loading: false,
  error: null,
};
```

#### Classe store

```ts
import { Injectable } from '@angular/core';
import { ComponentStore } from '@ngrx/component-store';

@Injectable()
export class TodosStore extends ComponentStore<TodosState> {
  constructor() {
    super(initialState);
  }
}
```

### 3.3 Scope (important)
Component Store doit être **scopé** :

- `providers: [TodosStore]` sur un composant de feature
- ou sur une route (avec `providers` dans la config de route)
- ou dans un provider d’un conteneur.

Exemple sur un composant standalone :

```ts
@Component({
  standalone: true,
  selector: 'app-todos-page',
  templateUrl: './todos-page.html',
  providers: [TodosStore],
})
export class TodosPageComponent {
  readonly vm$ = this.store.vm$;
  constructor(public readonly store: TodosStore) {}
}
```

---

## 4. Premier store : state, selectors, updaters

### 4.1 Selectors

```ts
readonly todos$ = this.select((s) => s.todos);
readonly filter$ = this.select((s) => s.filter);
readonly loading$ = this.select((s) => s.loading);
readonly error$ = this.select((s) => s.error);

readonly filteredTodos$ = this.select(
  this.todos$,
  this.filter$,
  (todos, filter) => {
    switch (filter) {
      case 'active':
        return todos.filter(t => !t.done);
      case 'done':
        return todos.filter(t => t.done);
      default:
        return todos;
    }
  }
);
```

> `this.select(a$, b$, projector)` combine les flux et applique un projector.

### 4.2 Le pattern `vm$`
Un *ViewModel* agrège tout ce que le template consomme.

```ts
readonly vm$ = this.select({
  todos: this.filteredTodos$,
  loading: this.loading$,
  error: this.error$,
  filter: this.filter$,
});
```

Dans le template :

```html
<ng-container *ngIf="vm$ | async as vm">
  <app-spinner *ngIf="vm.loading" />
  <app-error *ngIf="vm.error" [message]="vm.error" />

  <app-filter
    [value]="vm.filter"
    (valueChange)="store.setFilter($event)"
  />

  <app-todo-list
    [todos]="vm.todos"
    (toggle)="store.toggleDone($event)"
  />
</ng-container>
```

### 4.3 Updaters

```ts
readonly setFilter = this.updater<TodosState['filter']>((state, filter) => ({
  ...state,
  filter,
}));

readonly setLoading = this.updater<boolean>((state, loading) => ({
  ...state,
  loading,
}));

readonly setError = this.updater<string | null>((state, error) => ({
  ...state,
  error,
}));

readonly setTodos = this.updater<Todo[]>((state, todos) => ({
  ...state,
  todos,
}));

readonly toggleDone = this.updater<Todo['id']>((state, id) => ({
  ...state,
  todos: state.todos.map(t => t.id === id ? { ...t, done: !t.done } : t),
}));
```

**Règles** :

- un updater ne fait que **mettre à jour** l’état
- pas d’appel HTTP, pas de `subscribe` dans un updater.

---

## 5. Effects : orchestration asynchrone

### 5.1 Service API

```ts
@Injectable({ providedIn: 'root' })
export class TodosApi {
  constructor(private http: HttpClient) {}

  list(): Observable<Todo[]> {
    return this.http.get<Todo[]>('/api/todos');
  }

  update(id: string, patch: Partial<Todo>): Observable<Todo> {
    return this.http.patch<Todo>(`/api/todos/${id}`, patch);
  }
}
```

### 5.2 Effect de chargement

```ts
constructor(private api: TodosApi) {
  super(initialState);
}

readonly loadTodos = this.effect<void>((trigger$) =>
  trigger$.pipe(
    // on démarre le loading
    tap(() => {
      this.setLoading(true);
      this.setError(null);
    }),
    switchMap(() =>
      this.api.list().pipe(
        tap({
          next: (todos) => this.setTodos(todos),
          error: (err) => this.setError(this.formatError(err)),
        }),
        finalize(() => this.setLoading(false))
      )
    )
  )
);

private formatError(err: unknown): string {
  return err instanceof Error ? err.message : 'Erreur inconnue';
}
```

**Pourquoi `switchMap` ?**

- si le user re-déclenche `loadTodos()` (navigation, refresh), on annule le précédent appel.

### 5.3 Déclencher un effect

Dans le composant (ou `ngOnInit`) :

```ts
ngOnInit() {
  this.store.loadTodos();
}
```

> `loadTodos()` est une fonction (créée par `effect`) qui pousse une valeur dans `trigger$`.

### 5.4 Pattern : effect paramétré

Ex : persister `toggleDone` côté serveur.

```ts
readonly toggleDoneAndPersist = this.effect<Todo['id']>((id$) =>
  id$.pipe(
    withLatestFrom(this.todos$),
    switchMap(([id, todos]) => {
      const todo = todos.find(t => t.id === id);
      if (!todo) return EMPTY;

      // optimistic update
      this.toggleDone(id);

      return this.api.update(id, { done: !todo.done }).pipe(
        catchError((err) => {
          // rollback naïf
          this.toggleDone(id);
          this.setError(this.formatError(err));
          return EMPTY;
        })
      );
    })
  )
);
```

### 5.5 Concurrence : choisir le bon opérateur

| Opérateur | Comportement | Cas typique |
|---|---|---|
| `switchMap` | annule l’ancien | recherche, reload |
| `exhaustMap` | ignore si déjà en cours | login, double-submit |
| `concatMap` | met en queue | batch d’actions séquentielles |
| `mergeMap` | parallèle | tâches indépendantes |

---

## 6. Architecture recommandée

### 6.1 Découpage

- `feature/`
  - `todos-page.component.ts` (container)
  - `todos.store.ts`
  - `todos.api.ts`
  - `components/` (présentation : list, filter…)

### 6.2 Store proche de l’UI, API dans un service

- **Store** : logique de state, orchestration, mapping UI
- **API service** : détails HTTP
- **Composants presentation** : reçoivent des inputs, émettent des outputs.

### 6.3 Router-level providers
Scoper le store à la route :

```ts
export const TODOS_ROUTES: Routes = [
  {
    path: '',
    providers: [TodosStore],
    loadComponent: () => import('./todos-page.component').then(m => m.TodosPageComponent),
  }
];
```

---

## 7. Patterns avancés

### 7.1 Gérer un state « request » (loading + error)
Pattern courant :

```ts
interface RequestState {
  loading: boolean;
  error: string | null;
}
```

Puis le réutiliser dans plusieurs stores.

### 7.2 Debounce de recherche

```ts
readonly setQuery = this.updater<string>((s, query) => ({ ...s, query }));

readonly search = this.effect<string>((query$) =>
  query$.pipe(
    debounceTime(300),
    distinctUntilChanged(),
    tap(() => this.setLoading(true)),
    switchMap(q => this.api.search(q).pipe(
      tap({ next: r => this.setResults(r), error: e => this.setError(this.formatError(e)) }),
      finalize(() => this.setLoading(false))
    ))
  )
);
```

> On choisit `switchMap` pour annuler les requêtes précédentes.

### 7.3 Pagination simple
Stocker dans le state : `page`, `pageSize`, `total`.

- selectors : `pageCount$`, `canNext$`, `canPrev$`
- effect : `loadPage`.

### 7.4 Interaction entre stores
Deux stratégies :

1. **Injecter un store dans un autre** (à utiliser avec parcimonie)
2. **Remonter** l’interaction au composant container (souvent plus propre)

Règle : éviter la dépendance circulaire et garder des scopes clairs.

### 7.5 Cache local
Component Store peut faire un cache simple par feature (ex: dernières données) tant que :

- le scope de vie du store correspond au besoin, et
- on ne doit pas partager l’état globalement.

---

## 8. Tests unitaires

### 8.1 Tester les updaters

```ts
describe('TodosStore updaters', () => {
  let store: TodosStore;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [TodosStore],
    });
    store = TestBed.inject(TodosStore);
  });

  it('should set filter', (done) => {
    store.setFilter('done');

    store.filter$.subscribe(filter => {
      expect(filter).toBe('done');
      done();
    });
  });
});
```

### 8.2 Tester un effect (avec service mock)

Approche simple : mock `TodosApi` et vérifier les updaters appelés via state final.

```ts
const apiMock = {
  list: () => of([{ id: '1', title: 'A', done: false }]),
};

TestBed.configureTestingModule({
  providers: [
    TodosStore,
    { provide: TodosApi, useValue: apiMock },
  ],
});

store.loadTodos();

store.todos$.subscribe(todos => {
  expect(todos.length).toBe(1);
});
```

> Pour des tests plus fins (marble tests), on peut injecter un `TestScheduler` et contrôler le temps RxJS.

---

## 9. Anti-patterns & erreurs fréquentes

### 9.1 Mettre de la logique métier lourde dans le composant
Le composant doit être principalement :

- binding UI
- déclenchement d’intentions (`store.loadTodos()`, `store.setFilter(...)`)

### 9.2 Faire des `subscribe` manuels partout
Préférez :

- `vm$ | async` dans le template
- effets dans le store
- `tap` dans les effects plutôt que `subscribe`.

### 9.3 Stocker des données dérivables
Ex : stocker `filteredTodos` **et** `todos`. Préférez :

- stocker `todos` et `filter`
- dériver `filteredTodos$`.

### 9.4 Mauvais scope
Si vous fournissez le store au mauvais niveau, vous pouvez :

- perdre l’état (store recréé trop souvent)
- ou le rendre trop global (state accidentellement partagé)

---

## 10. Migration depuis services locaux / mini-stores

### 10.1 Symptômes d’un service "Subject"
Vous avez :

- `private state$ = new BehaviorSubject<State>(...)`
- beaucoup de `next({...})`
- du code asynchrone dispersé.

### 10.2 Migration progressive

1. Définir `State` + `initialState`
2. Remplacer `BehaviorSubject` par `ComponentStore`
3. Convertir `next()` en **updaters**
4. Déplacer les appels API vers **effects**
5. Créer `vm$` et simplifier le composant.

---

## 11. Atelier fil rouge

### Objectif
Construire une page "Todos" avec :

- chargement initial
- filtres
- toggle + persistance
- gestion loading + erreurs
- *optimistic update* + rollback

### Étapes (guidées)

1. Créer le state `TodosState`
2. Écrire selectors + `vm$`
3. Ajouter updaters (`setFilter`, `setTodos`, `toggleDone`)
4. Ajouter `loadTodos` effect
5. Ajouter `toggleDoneAndPersist` effect
6. Écrire 2 tests : un updater et un effect

### Livrable attendu

- code structuré `feature/`
- composant view simplifié
- store testé.

---

# Annexes : cheat sheet

## API ComponentStore

- `select(projector)` ou `select(a$, b$, projector)`
- `updater((state, value) => newState)`
- `effect((origin$) => origin$.pipe(...))`
- `setState(newState)` ou `patchState(partial)` (selon versions)

## Checklist qualité

- [ ] state minimal
- [ ] selectors pour dériver
- [ ] `vm$` unique pour le template
- [ ] updaters purs
- [ ] effects pour asynchrone
- [ ] gestion erreurs + loading cohérente
- [ ] scope providers correct

---

## Références

- NgRx Component Store (docs) : https://ngrx.io/guide/component-store
- RxJS operators : https://rxjs.dev
