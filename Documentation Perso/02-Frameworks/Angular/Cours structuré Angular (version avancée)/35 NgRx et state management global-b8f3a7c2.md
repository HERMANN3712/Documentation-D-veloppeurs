# Formation Angular — NgRx et state management global

- **Référence** : 35
- **Public** : développeurs Angular (intermédiaire → avancé)
- **Pré-requis** : TypeScript, RxJS (Observables, operators de base), Angular (Modules, DI, HttpClient, Router)
- **Durée indicative** : 2 jours (14h) — adaptable (1 à 3 jours)
- **Objectif** : maîtriser le **state management global** avec **NgRx** (Store, Actions, Reducers, Selectors, Effects) et savoir l’implémenter proprement dans une application **volumineuse** et **multi-écrans**.

---

## Table des matières

1. [Introduction : pourquoi un état global ?](#1-introduction--pourquoi-un-état-global-)
2. [Architecture NgRx : concepts et flux de données](#2-architecture-ngrx--concepts-et-flux-de-données)
3. [Mise en place : installation et premiers fichiers](#3-mise-en-place--installation-et-premiers-fichiers)
4. [Actions : l’intention métier](#4-actions--lintention-métier)
5. [Reducers : la pureté et l’immuabilité](#5-reducers--la-pureté-et-limmuabilité)
6. [Selectors : lire efficacement l’état](#6-selectors--lire-efficacement-létat)
7. [Effects : orchestrer l’asynchrone et les side effects](#7-effects--orchestrer-lasynchrone-et-les-side-effects)
8. [Entity : gérer des collections à grande échelle](#8-entity--gérer-des-collections-à-grande-échelle)
9. [Router Store : synchroniser la navigation et l’état](#9-router-store--synchroniser-la-navigation-et-létat)
10. [Components vs Store : patterns de responsabilité](#10-components-vs-store--patterns-de-responsabilité)
11. [Gestion du cache, chargements et erreurs](#11-gestion-du-cache-chargements-et-erreurs)
12. [Normalisation, dérivés et performance](#12-normalisation-dérivés-et-performance)
13. [Traçabilité, debug et tests](#13-traçabilité-debug-et-tests)
14. [Structuration d’un projet NgRx (scalable)](#14-structuration-dun-projet-ngrx-scalable)
15. [Exercices guidés (fil rouge)](#15-exercices-guidés-fil-rouge)
16. [Checklist de mise en production](#16-checklist-de-mise-en-production)

---

## 1) Introduction : pourquoi un état global ?

### 1.1 Problème
Dans une application Angular qui grandit (multi-écrans, features interconnectées, travail en équipe), l’état local dans les composants et les services devient :

- difficile à **partager** (props drilling, services « fourre-tout »)
- difficile à **tracer** (qui modifie quoi ? quand ?)
- difficile à **tester** (effets de bord, coupling)
- difficile à **raisonner** (mutations implicites, abonnements dispersés)

### 1.2 Objectif du state management global
Prioriser :

- **centralisation** : une source de vérité (single source of truth)
- **prévisibilité** : transitions d’état déterministes (réducteurs purs)
- **traçabilité** : journaliser les actions, time-travel (devtools)
- **scalabilité** : règles de structuration, patterns répétables

### 1.3 Quand NgRx devient pertinent ?
NgRx est particulièrement adapté lorsque :

- l’app est **volumineuse** (beaucoup de features)
- l’app est **multi-écrans** (navigation + état partagé)
- il existe des workflows **collaboratifs** et des règles métiers exigeantes
- la **traçabilité** et la **prévisibilité** priment

> Pour des apps simples, un service + signals / RxJS suffit parfois. NgRx a un coût (boilerplate, conventions) mais apporte une gouvernance.

---

## 2) Architecture NgRx : concepts et flux de données

### 2.1 Les briques
NgRx applique des principes proches de Redux :

- **Store** : conteneur de l’état global (UI + données), read-only côté application
- **Actions** : événements/intention (ce qui s’est passé / ce qu’on veut faire)
- **Reducers** : fonctions pures qui calculent le nouvel état
- **Selectors** : fonctions de lecture, mémorisées (memoization)
- **Effects** : effets de bord (HTTP, storage, router, websockets…), transforment des actions en d’autres actions

### 2.2 Flux unidirectionnel (unidirectional data flow)

1. Le composant déclenche une **action**
2. Les **reducers** mettent à jour l’état dans le **store**
3. Les **selectors** exposent des vues de l’état (observables)
4. Les **effects** écoutent certaines actions et accomplissent l’asynchrone/side effects, puis dispatchent de nouvelles actions

### 2.3 État : données vs UI
- **Données** : entités, listes, détails, références (souvent normalisées)
- **UI state** : filtres, tri, pagination, sélections, flags de chargement, erreurs

> Séparer clairement « state de données » et « state d’interface » évite la confusion et améliore la lisibilité.

---

## 3) Mise en place : installation et premiers fichiers

### 3.1 Installation
```bash
npm i @ngrx/store @ngrx/effects @ngrx/store-devtools
npm i @ngrx/entity
```
Optionnel :
```bash
npm i @ngrx/router-store
```

### 3.2 Génération (Angular CLI)
Si vous utilisez `@ngrx/schematics` :
```bash
ng add @ngrx/store
ng add @ngrx/effects
```

### 3.3 Structure cible (feature-first)
Exemple :
```
src/app/
  core/                # services, interceptors, config
  shared/              # composants partagés, pipes, utils
  state/               # root store (si vous centralisez)
  features/
    projects/
      data-access/     # actions, reducer, selectors, effects, services API
      feature/         # pages et composants smart
      ui/              # composants dumb/presentational
```

> Le dossier `data-access` regroupe tout ce qui touche au state et aux appels API. Les composants `feature` consomment selectors + dispatch.

---

## 4) Actions : l’intention métier

### 4.1 Rôle
Une action décrit :
- **un type** (string) unique
- un **payload** (données nécessaires)

Conceptuellement : « quelque chose s’est produit ».

### 4.2 Conventions
- Préfixe `[Feature]` dans le type
- Triptyque courant : `Load / Load Success / Load Failure`
- Préférer des noms métier : `CreateProject`, `AssignUser`, etc.

### 4.3 Création d’actions (createAction)
Exemple fil rouge : gestion de **Projects**.

```ts
// features/projects/data-access/projects.actions.ts
import { createAction, props } from '@ngrx/store';
import { Project } from './projects.models';

export const loadProjects = createAction(
  '[Projects] Load Projects'
);

export const loadProjectsSuccess = createAction(
  '[Projects] Load Projects Success',
  props<{ projects: Project[] }>()
);

export const loadProjectsFailure = createAction(
  '[Projects] Load Projects Failure',
  props<{ error: unknown }>()
);

export const selectProject = createAction(
  '[Projects] Select Project',
  props<{ projectId: string | null }>()
);
```

### 4.4 Actions « UI » vs « API »
Une approche claire :
- Actions **UI** (intention utilisateur) : `Load Projects`, `Select Project`
- Actions **API** (résultat asynchrone) : `Load Projects Success/Failure`

Cela rend la console des devtools plus lisible et la testabilité meilleure.

---

## 5) Reducers : la pureté et l’immuabilité

### 5.1 Rôle
Le reducer :
- reçoit `(state, action)`
- retourne **un nouvel état** (immutable)
- ne fait **pas** d’HTTP, pas de `Date.now()`, pas de side effects

### 5.2 Modèle d’état
```ts
// features/projects/data-access/projects.models.ts
export interface Project {
  id: string;
  name: string;
  status: 'active' | 'archived';
}

export interface ProjectsState {
  projects: Project[];
  selectedProjectId: string | null;
  loading: boolean;
  error: unknown | null;
}

export const initialProjectsState: ProjectsState = {
  projects: [],
  selectedProjectId: null,
  loading: false,
  error: null,
};
```

### 5.3 Création du reducer (createReducer + on)
```ts
// features/projects/data-access/projects.reducer.ts
import { createReducer, on } from '@ngrx/store';
import * as ProjectsActions from './projects.actions';
import { initialProjectsState } from './projects.models';

export const projectsFeatureKey = 'projects';

export const projectsReducer = createReducer(
  initialProjectsState,
  on(ProjectsActions.loadProjects, (state) => ({
    ...state,
    loading: true,
    error: null,
  })),
  on(ProjectsActions.loadProjectsSuccess, (state, { projects }) => ({
    ...state,
    projects,
    loading: false,
  })),
  on(ProjectsActions.loadProjectsFailure, (state, { error }) => ({
    ...state,
    loading: false,
    error,
  })),
  on(ProjectsActions.selectProject, (state, { projectId }) => ({
    ...state,
    selectedProjectId: projectId,
  }))
);
```

### 5.4 Bonnes pratiques reducers
- Rester **simple** : pas de logique métier trop complexe (sinon extraire)
- Assurer l’**immuabilité** (spread, `map`, `filter`, ou entity adapter)
- Garder un état minimal, éviter la duplication (préférer selectors)

---

## 6) Selectors : lire efficacement l’état

### 6.1 Rôle
Les selectors :
- encapsulent l’accès à l’état
- offrent des vues dérivées (tri, filtrage, mapping)
- sont **mémorisés** si basés sur `createSelector`

### 6.2 Feature selector
```ts
// features/projects/data-access/projects.selectors.ts
import { createFeatureSelector, createSelector } from '@ngrx/store';
import { ProjectsState } from './projects.models';
import { projectsFeatureKey } from './projects.reducer';

export const selectProjectsState = createFeatureSelector<ProjectsState>(
  projectsFeatureKey
);
```

### 6.3 Selectors de base
```ts
export const selectAllProjects = createSelector(
  selectProjectsState,
  (state) => state.projects
);

export const selectLoading = createSelector(
  selectProjectsState,
  (state) => state.loading
);

export const selectError = createSelector(
  selectProjectsState,
  (state) => state.error
);

export const selectSelectedProjectId = createSelector(
  selectProjectsState,
  (state) => state.selectedProjectId
);
```

### 6.4 Selector dérivé : projet sélectionné
```ts
export const selectSelectedProject = createSelector(
  selectAllProjects,
  selectSelectedProjectId,
  (projects, selectedId) =>
    selectedId ? projects.find(p => p.id === selectedId) ?? null : null
);
```

### 6.5 Erreurs fréquentes
- Faire des transformations coûteuses sans memoization
- Multiplier les abonnements `.subscribe()` dans les composants (préférer `async` pipe)

---

## 7) Effects : orchestrer l’asynchrone et les side effects

### 7.1 Rôle
Les effects :
- écoutent le flux d’actions (`Actions` observable)
- exécutent des traitements asynchrones (HTTP, localStorage, router, websockets)
- dispatchent de nouvelles actions de succès/échec

### 7.2 Service API (exemple)
```ts
// features/projects/data-access/projects.api.ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';
import { Project } from './projects.models';

@Injectable({ providedIn: 'root' })
export class ProjectsApi {
  constructor(private http: HttpClient) {}

  getAll(): Observable<Project[]> {
    return this.http.get<Project[]>('/api/projects');
  }
}
```

### 7.3 Effect de chargement
```ts
// features/projects/data-access/projects.effects.ts
import { Injectable } from '@angular/core';
import { Actions, createEffect, ofType } from '@ngrx/effects';
import { catchError, map, mergeMap, of } from 'rxjs';
import * as ProjectsActions from './projects.actions';
import { ProjectsApi } from './projects.api';

@Injectable()
export class ProjectsEffects {
  loadProjects$ = createEffect(() =>
    this.actions$.pipe(
      ofType(ProjectsActions.loadProjects),
      mergeMap(() =>
        this.api.getAll().pipe(
          map((projects) => ProjectsActions.loadProjectsSuccess({ projects })),
          catchError((error) => of(ProjectsActions.loadProjectsFailure({ error })))
        )
      )
    )
  );

  constructor(private actions$: Actions, private api: ProjectsApi) {}
}
```

### 7.4 Choisir le bon opérateur RxJS
- `mergeMap` : parallélise (plusieurs requêtes possibles)
- `switchMap` : conserve la dernière (annule la précédente) → idéal pour recherche
- `exhaustMap` : ignore les déclenchements pendant une requête → idéal bouton « submit »
- `concatMap` : met en file d’attente

### 7.5 Effects non-dispatch (log, navigation)
```ts
import { tap } from 'rxjs';

navigateOnSelect$ = createEffect(
  () => this.actions$.pipe(
    ofType(ProjectsActions.selectProject),
    tap(({ projectId }) => {
      // ex: router.navigate...
    })
  ),
  { dispatch: false }
);
```

---

## 8) Entity : gérer des collections à grande échelle

### 8.1 Pourquoi @ngrx/entity
Quand on manipule beaucoup d’objets :
- listes
- mises à jour partielles
- suppression, upsert

`@ngrx/entity` fournit :
- structure normalisée `ids + entities`
- méthodes immuables (adapter)
- selectors utilitaires

### 8.2 State avec Entity
```ts
import { EntityState, createEntityAdapter } from '@ngrx/entity';

export interface ProjectsState extends EntityState<Project> {
  selectedProjectId: string | null;
  loading: boolean;
  error: unknown | null;
}

export const adapter = createEntityAdapter<Project>();

export const initialProjectsState: ProjectsState = adapter.getInitialState({
  selectedProjectId: null,
  loading: false,
  error: null,
});
```

### 8.3 Reducer avec adapter
```ts
export const projectsReducer = createReducer(
  initialProjectsState,
  on(ProjectsActions.loadProjects, (state) => ({
    ...state,
    loading: true,
    error: null,
  })),
  on(ProjectsActions.loadProjectsSuccess, (state, { projects }) =>
    adapter.setAll(projects, {
      ...state,
      loading: false,
    })
  ),
  on(ProjectsActions.loadProjectsFailure, (state, { error }) => ({
    ...state,
    loading: false,
    error,
  }))
);
```

### 8.4 Selectors entity
```ts
const { selectAll, selectEntities } = adapter.getSelectors();

export const selectAllProjects = createSelector(
  selectProjectsState,
  selectAll
);

export const selectProjectEntities = createSelector(
  selectProjectsState,
  selectEntities
);

export const selectSelectedProject = createSelector(
  selectProjectEntities,
  selectSelectedProjectId,
  (entities, id) => (id ? entities[id] ?? null : null)
);
```

---

## 9) Router Store : synchroniser la navigation et l’état

### 9.1 Pourquoi
- tracer la navigation comme une action
- lire des params URL via selectors
- créer des effets basés sur changement de route

### 9.2 Installation
```bash
npm i @ngrx/router-store
```

### 9.3 Principes d’utilisation
- Ajouter le `StoreRouterConnectingModule`
- Définir un serializer si besoin (pour réduire l’état)

Exemples d’usage :
- `selectUrl`, `selectRouteParams`, `selectQueryParams`
- effect : si route `/projects/:id` alors dispatch `selectProject({id})`

---

## 10) Components vs Store : patterns de responsabilité

### 10.1 Smart vs Dumb
- **Smart components (containers)** :
  - dispatchent des actions
  - consomment des selectors
  - orchestrent la page

- **Dumb components (presentational)** :
  - reçoivent des `@Input()`
  - émettent des `@Output()`
  - pas de Store direct

### 10.2 Exemple de composant page
```ts
import { Component, ChangeDetectionStrategy } from '@angular/core';
import { Store } from '@ngrx/store';
import * as ProjectsActions from '../data-access/projects.actions';
import * as ProjectsSelectors from '../data-access/projects.selectors';

@Component({
  selector: 'app-projects-page',
  template: `
    <section>
      <header>
        <h1>Projects</h1>
        <button (click)="reload()">Reload</button>
      </header>

      <ng-container *ngIf="loading$ | async; else content">Loading...</ng-container>
      <ng-template #content>
        <app-projects-list
          [projects]="projects$ | async"
          (selected)="select($event)"
        />
      </ng-template>

      <pre class="error" *ngIf="(error$ | async) as err">{{ err | json }}</pre>
    </section>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class ProjectsPageComponent {
  projects$ = this.store.select(ProjectsSelectors.selectAllProjects);
  loading$ = this.store.select(ProjectsSelectors.selectLoading);
  error$ = this.store.select(ProjectsSelectors.selectError);

  constructor(private store: Store) {
    this.store.dispatch(ProjectsActions.loadProjects());
  }

  reload() {
    this.store.dispatch(ProjectsActions.loadProjects());
  }

  select(projectId: string) {
    this.store.dispatch(ProjectsActions.selectProject({ projectId }));
  }
}
```

### 10.3 Règles pratiques
- Éviter la logique métier dans les composants
- Préférer la composition de selectors
- Standardiser les noms (ex: `loadX`, `loadXSuccess`, etc.)

---

## 11) Gestion du cache, chargements et erreurs

### 11.1 Patterns de chargement
- `loading: boolean` (simple)
- `loading: 'idle' | 'loading' | 'success' | 'error'` (plus expressif)
- `loadedAt: number | null` pour invalidation cache

Exemple d’état :
```ts
export interface ProjectsState {
  // ...
  loadingStatus: 'idle' | 'loading' | 'success' | 'error';
  loadedAt: number | null;
}
```

### 11.2 Cache « simple »
Principe : si `loadedAt` récent, ne pas recharger.
- se fait souvent dans un **effect** ou un **guard**
- attention à ne pas surcharger le reducer

### 11.3 Gestion d’erreurs
- `error` typé (idéalement) : `{ message, code, details }`
- notifier via un effect non-dispatch (snackbar/toast)

---

## 12) Normalisation, dérivés et performance

### 12.1 Normalisation
Éviter de stocker des objets imbriqués en profondeur si :
- relations multiples
- mises à jour fréquentes

Approche :
- un slice par entité (ex: `projects`, `users`)
- référencer par ID

### 12.2 State minimal
Ne pas stocker ce qui se recalculera en selector (ex : `filteredProjects`).

### 12.3 Performance UI
- `OnPush` + `async` pipe
- selectors mémoïsés
- éviter de créer des selectors dans les templates (ou dans méthodes)

---

## 13) Traçabilité, debug et tests

### 13.1 Store DevTools
Ajout classique :
```ts
import { StoreDevtoolsModule } from '@ngrx/store-devtools';

StoreDevtoolsModule.instrument({
  maxAge: 50,
  logOnly: false,
})
```

Ce que vous gagnez :
- inspection action par action
- diff de state
- time travel

### 13.2 Tests reducers
Les reducers sont des fonctions pures → tests simples.

```ts
import { projectsReducer } from './projects.reducer';
import * as ProjectsActions from './projects.actions';
import { initialProjectsState } from './projects.models';

describe('projectsReducer', () => {
  it('should set loading true on loadProjects', () => {
    const state = projectsReducer(initialProjectsState, ProjectsActions.loadProjects());
    expect(state.loading).toBeTrue();
  });
});
```

### 13.3 Tests selectors
Tester les fonctions de projection ou utiliser un état mock.

### 13.4 Tests effects
- `provideMockActions` + marble testing (rxjs-marbles / jasmine-marbles)
- ou tests plus simples : déclencher action, mock api observable

---

## 14) Structuration d’un projet NgRx (scalable)

### 14.1 Approche recommandée
- **Feature-first** (par domaine)
- Au sein de chaque feature :
  - `data-access` (state + API)
  - `feature` (pages)
  - `ui` (composants)

### 14.2 Nommage et découpage
- 1 reducer par feature
- selectors regroupés
- actions regroupées
- effects regroupés (un ou plusieurs selon complexité)

### 14.3 Root store vs feature store
- **Root store** : état global transversal (auth, layout, settings)
- **Feature store** : domaine (projects, billing, admin)

---

## 15) Exercices guidés (fil rouge)

### Exercice 1 — Mise en place
Objectif : intégrer `StoreModule.forRoot/forFeature` et `EffectsModule`.
- Créer le feature `projects`
- Enregistrer reducer et effects

### Exercice 2 — Listing de projets
Objectif : `loadProjects` + affichage.
- Action `loadProjects`
- Effect HTTP vers `GET /api/projects`
- Reducer success/failure
- Selector `selectAllProjects`

### Exercice 3 — Sélection
Objectif : stocker une sélection.
- Action `selectProject({projectId})`
- Selector `selectSelectedProject`

### Exercice 4 — Refactor vers Entity
Objectif : améliorer les perfs et les updates.
- Migrer `projects: Project[]` → `EntityState<Project>`
- Mettre à jour reducers + selectors

### Exercice 5 — Gestion de statuts
Objectif : fiabiliser le rendu.
- Ajouter `loadingStatus` et gérer `idle/loading/success/error`

### Exercice 6 — Tests
Objectif : écrire 1 test reducer + 1 test effect.

---

## 16) Checklist de mise en production

- [ ] Conventions d’actions (UI vs API) respectées
- [ ] Reducers purs, sans side effects
- [ ] Selectors centralisés, memoization utilisée
- [ ] Effects : opérateur RxJS choisi correctement (`switchMap`, `exhaustMap`, ...)
- [ ] Gestion d’erreurs consistante (UI + logs)
- [ ] Chargements et cache définis (status, loadedAt)
- [ ] Structure projet stable (feature-first)
- [ ] Devtools activés seulement en dev, `logOnly` en prod si nécessaire
- [ ] Tests essentiels (reducers/effects) en place

---

## Annexes

### A) Notes de style et anti-patterns
- **Anti-pattern** : stocker des Observables dans le state
- **Anti-pattern** : subscribe manuel partout (risque de fuites)
- **Anti-pattern** : dupliquer des données (source unique)
- **Conseil** : documenter l’état (interfaces) et écrire des selectors « officiels »

### B) Glossaire rapide
- **Action** : événement intentionnel
- **Reducer** : fonction pure de transition d’état
- **Selector** : lecture/dérivation de state
- **Effect** : side effect asynchrone
- **Entity** : gestion normalisée des collections
