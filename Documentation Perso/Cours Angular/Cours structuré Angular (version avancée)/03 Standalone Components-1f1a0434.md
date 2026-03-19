# Formation Angular — Standalone Components (Angular 14+)

> **Public** : développeurs Angular (niveau débutant++ à confirmé)  
> **Pré-requis** : TypeScript, notions de composants, services, routing Angular  
> **Durée suggérée** : 1 jour (6–7h) ou 2 demi-journées  
> **Objectif** : concevoir une application Angular moderne **sans dépendre systématiquement de NgModules**, en s’appuyant sur les **Standalone Components** pour simplifier l’architecture, améliorer le lazy loading et réduire le couplage.

---

## Sommaire (plan de formation)

1. [Contexte et motivations](#1-contexte-et-motivations)
2. [Notions clés des Standalone Components](#2-notions-clés-des-standalone-components)
3. [Créer un composant standalone](#3-créer-un-composant-standalone)
4. [Imports : composants, directives, pipes, modules…](#4-imports--composants-directives-pipes-modules)
5. [Bootstrapping d’une application standalone](#5-bootstrapping-dune-application-standalone)
6. [Routing en mode standalone](#6-routing-en-mode-standalone)
7. [Lazy loading et performance](#7-lazy-loading-et-performance)
8. [Dépendances & injection : providers, scopes, configuration](#8-dépendances--injection--providers-scopes-configuration)
9. [Interopérabilité avec NgModules (migration progressive)](#9-interopérabilité-avec-ngmodules-migration-progressive)
10. [Patterns d’architecture modernes (feature-oriented, découplage)](#10-patterns-darchitecture-modernes-feature-oriented-découplage)
11. [Tests avec Standalone Components](#11-tests-avec-standalone-components)
12. [Atelier fil rouge : mini-app 100% standalone](#12-atelier-fil-rouge--mini-app-100-standalone)
13. [Checklist, erreurs fréquentes, bonnes pratiques](#13-checklist-erreurs-fréquentes-bonnes-pratiques)
14. [Annexes : commandes, snippets, anti-patterns](#14-annexes--commandes-snippets-anti-patterns)

---

## 1. Contexte et motivations

### 1.1. Pourquoi Angular a introduit les Standalone Components ?
Historiquement, Angular structure une application via des **NgModules** (AppModule, FeatureModule, SharedModule…). Cette approche a des avantages (encapsulation, compilation, organisation), mais introduit également :

- **Boilerplate** : déclaration/exports/imports de modules, duplication de configuration.
- **Couplage implicite** : un composant dépend de son module et de sa graph d’import.
- **Complexité de lazy loading** : chargement de modules de fonctionnalité parfois surdimensionnés.
- **Courbe de prise en main** : comprendre `declarations`, `imports`, `exports`, etc.

Les **Standalone Components** simplifient l’architecture en rendant le composant **auto-suffisant** :
- Il déclare ses dépendances (directives, pipes, autres composants) via `imports`.
- Il peut être routé et lazy-loadé directement.
- Il devient un **bloc de construction** autonome.

### 1.2. Résultat recherché
- Un **point d’entrée** moderne via `bootstrapApplication()`.
- Un routing basé sur `loadComponent` / `loadChildren` (routes) en standalone.
- Un projet plus lisible, plus facile à découper et à migrer.

---

## 2. Notions clés des Standalone Components

### 2.1. Définition
Un **standalone component** est un composant Angular annoté avec :

```ts
@Component({
  standalone: true,
  ...
})
export class MyComponent {}
```

Il **n’a pas besoin** d’être déclaré dans un `NgModule` (`declarations`).

### 2.2. Standalone vs NgModule (résumé)
| Sujet | NgModule | Standalone |
|---|---|---|
| Déclaration d’un composant | `declarations` | n/a |
| Dépendances (directives, pipes, composants) | `imports` du module | `imports` du composant |
| Bootstrap | `platformBrowserDynamic().bootstrapModule(AppModule)` | `bootstrapApplication(AppComponent)` |
| Routing lazy | `loadChildren: () => import(...).then(m => m.FeatureModule)` | `loadComponent` ou routes standalone |
| Migration | nécessite des modules | progressive et incrémentale |

### 2.3. Standalone building blocks
- **Standalone Components**
- **Standalone Directives**
- **Standalone Pipes**
- **Standalone Routes** (via `provideRouter`)

---

## 3. Créer un composant standalone

### 3.1. CLI
Selon la version du CLI, vous pouvez générer un composant standalone :

```bash
ng g c features/todos/todo-list --standalone
```

> Si votre workspace utilise déjà le mode standalone par défaut, l’option `--standalone` peut être implicite.

### 3.2. Exemple minimal

```ts
import { Component } from '@angular/core';

@Component({
  selector: 'app-hello',
  standalone: true,
  template: `
    <h1>Hello Standalone</h1>
  `
})
export class HelloComponent {}
```

### 3.3. Point essentiel : la gestion des dépendances
Un composant standalone doit **importer explicitement** ce qu’il consomme dans son template.

Exemple : si vous utilisez `*ngIf`, `*ngFor`, pipes `date`, etc., vous devez importer `CommonModule` (ou des alternatives plus ciblées selon le cas).

---

## 4. Imports : composants, directives, pipes, modules…

### 4.1. Importer CommonModule (cas le plus fréquent)

```ts
import { Component } from '@angular/core';
import { CommonModule } from '@angular/common';

@Component({
  selector: 'app-dashboard',
  standalone: true,
  imports: [CommonModule],
  template: `
    <section *ngIf="ready">
      <p>Chargé !</p>
    </section>
  `
})
export class DashboardComponent {
  ready = true;
}
```

### 4.2. Importer d’autres composants standalone

```ts
@Component({
  selector: 'app-shell',
  standalone: true,
  imports: [DashboardComponent],
  template: `<app-dashboard />`
})
export class ShellComponent {}
```

### 4.3. Importer RouterModule vs RouterLink (approches)
1) Approche simple : importer `RouterModule`

```ts
import { RouterModule } from '@angular/router';

@Component({
  standalone: true,
  imports: [RouterModule],
  template: `<a routerLink="/home">Home</a>`
})
export class HeaderComponent {}
```

2) Approche plus fine (selon version) : importer directement les directives `RouterLink`, `RouterOutlet` (si disponibles dans votre version), ou utiliser `RouterModule` pour compatibilité.

### 4.4. Peut-on importer des NgModules ?
Oui. Les standalone components peuvent importer :
- des composants/directives/pipes standalone
- **des NgModules existants** (utile en migration)

```ts
imports: [CommonModule, LegacySharedModule]
```

> Recommandation : à long terme, préférez des dépendances plus **granulaires** (standalone) pour réduire le couplage.

---

## 5. Bootstrapping d’une application standalone

### 5.1. main.ts : `bootstrapApplication`

```ts
import { bootstrapApplication } from '@angular/platform-browser';
import { AppComponent } from './app/app.component';

bootstrapApplication(AppComponent)
  .catch(err => console.error(err));
```

### 5.2. Ajouter des providers globaux (HTTP, Router, etc.)
Via `providers` dans `bootstrapApplication` :

```ts
import { bootstrapApplication } from '@angular/platform-browser';
import { provideHttpClient } from '@angular/common/http';
import { provideRouter } from '@angular/router';
import { AppComponent } from './app/app.component';
import { routes } from './app/app.routes';

bootstrapApplication(AppComponent, {
  providers: [
    provideHttpClient(),
    provideRouter(routes)
  ]
});
```

### 5.3. Comparaison avec AppModule
- Avant : `AppModule` importait `BrowserModule`, `HttpClientModule`, `AppRoutingModule`…
- Maintenant : ces dépendances sont plutôt **injectées** via `provideX()` au bootstrap.

---

## 6. Routing en mode standalone

### 6.1. Définir des routes
Créez `app.routes.ts` :

```ts
import { Routes } from '@angular/router';

export const routes: Routes = [
  {
    path: '',
    pathMatch: 'full',
    redirectTo: 'home'
  },
  {
    path: 'home',
    loadComponent: () => import('./features/home/home.component')
      .then(m => m.HomeComponent)
  },
  {
    path: 'todos',
    loadChildren: () => import('./features/todos/todos.routes')
      .then(r => r.TODOS_ROUTES)
  },
  { path: '**', redirectTo: 'home' }
];
```

### 6.2. Utiliser RouterOutlet

```ts
import { Component } from '@angular/core';
import { RouterOutlet } from '@angular/router';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [RouterOutlet],
  template: `
    <h1>My Standalone App</h1>
    <router-outlet />
  `
})
export class AppComponent {}
```

### 6.3. Routes de feature en standalone
Exemple `features/todos/todos.routes.ts` :

```ts
import { Routes } from '@angular/router';

export const TODOS_ROUTES: Routes = [
  {
    path: '',
    loadComponent: () => import('./todo-list/todo-list.component')
      .then(m => m.TodoListComponent)
  },
  {
    path: ':id',
    loadComponent: () => import('./todo-detail/todo-detail.component')
      .then(m => m.TodoDetailComponent)
  }
];
```

**Bénéfice** : vous lazy-loadez une feature via un simple fichier de routes, sans module.

---

## 7. Lazy loading et performance

### 7.1. `loadComponent` : lazy loading au niveau composant
- Plus granulaire qu’un `FeatureModule` complet.
- Réduit le JavaScript initial.

### 7.2. `loadChildren` avec routes standalone
- Permet une feature “packagée” sous forme de tableau `Routes`.
- Alternative à `FeatureModule` lazy.

### 7.3. Stratégies utiles
- Préchargement (selon besoin) via le Router (ex. `withPreloading(...)` selon version).
- Réduire les dépendances importées dans les composants.

> Bon réflexe : dans un composant standalone, importez **le strict nécessaire**.

---

## 8. Dépendances & injection : providers, scopes, configuration

### 8.1. Providers au bootstrap (scope application)
Idéal pour :
- HTTP (`provideHttpClient()`)
- Router (`provideRouter(routes)`)
- interceptors / configs globales

### 8.2. Providers au niveau route
Vous pouvez définir des `providers` sur une route pour isoler un scope de dépendances (pattern “feature scope”).

```ts
{
  path: 'admin',
  providers: [/* services admin */],
  loadComponent: () => import('./admin/admin.component').then(m => m.AdminComponent)
}
```

### 8.3. Providers au niveau composant
Toujours valable en standalone :

```ts
@Component({
  standalone: true,
  providers: [TodosFacade],
  ...
})
export class TodoListComponent {}
```

**Usage typique** : isoler une façade par écran pour éviter des singletons globaux non désirés.

---

## 9. Interopérabilité avec NgModules (migration progressive)

### 9.1. Stratégie de migration réaliste
1. Garder l’existant (modules) et introduire des standalone sur de nouvelles features.
2. Convertir progressivement les composants :
   - `standalone: true`
   - déplacer les dépendances du module vers `imports`
3. Migrer le routing (optionnel dans un second temps).
4. Migrer le bootstrap (`bootstrapApplication`) quand vous êtes à l’aise.

### 9.2. Consommer un module existant
Un standalone component peut importer un `SharedModule` legacy :

```ts
imports: [LegacySharedModule]
```

### 9.3. Exposer un composant standalone à un NgModule
Un module peut importer un composant standalone via `imports`.

```ts
@NgModule({
  imports: [HelloComponent]
})
export class SomeModule {}
```

---

## 10. Patterns d’architecture modernes (feature-oriented, découplage)

### 10.1. Centrer l’architecture autour des features
Exemple de structure :

```
src/app/
  app.component.ts
  app.routes.ts
  core/
    auth/
    http/
  shared/
    ui/
    utils/
  features/
    todos/
      todo-list/
      todo-detail/
      todos.routes.ts
      data-access/
```

- `shared/ui` : composants UI réutilisables (souvent standalone)
- `features/*` : pages, routes, data-access
- `core/*` : providers globaux (auth, logging, interceptor)

### 10.2. Réduire le couplage : import explicite
Avantage majeur : lorsqu’un composant importe ses dépendances, on “voit” ce qu’il consomme.

**Anti-pattern** : importer un “gros SharedModule” qui exporte trop de choses.

### 10.3. “Smart/Dumb components”
- **Smart** (container) : fait les appels services, compose l’écran, gère le state.
- **Dumb** (presentational) : inputs/outputs, UI, réutilisable.

En standalone, cela devient naturel : un dumb component importe seulement `CommonModule` + quelques UI components.

---

## 11. Tests avec Standalone Components

### 11.1. TestBed avec imports directs

```ts
import { TestBed } from '@angular/core/testing';
import { TodoListComponent } from './todo-list.component';

describe('TodoListComponent', () => {
  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [TodoListComponent]
    }).compileComponents();
  });

  it('should create', () => {
    const fixture = TestBed.createComponent(TodoListComponent);
    expect(fixture.componentInstance).toBeTruthy();
  });
});
```

**Bénéfice** : moins de tests “module-based”, vous importez directement le sujet.

### 11.2. Mock de providers
Toujours identique :

```ts
providers: [{ provide: TodosService, useValue: mockTodosService }]
```

---

## 12. Atelier fil rouge : mini-app 100% standalone

### 12.1. Objectif
Construire une mini-application :
- `HomeComponent` (page simple)
- `Todos` feature lazy-loadée (`/todos`)
- `TodoListComponent` + `TodoDetailComponent`
- `TodosService` (mock local) + HTTP optionnel

### 12.2. Étapes
1. Créer l’app (si besoin) et basculer vers bootstrap standalone.
2. Créer `AppComponent` + `app.routes.ts`.
3. Ajouter `HomeComponent` en `loadComponent`.
4. Ajouter feature `todos` via `loadChildren` -> `TODOS_ROUTES`.
5. Implémenter `TodoListComponent` et navigation vers `TodoDetailComponent`.
6. Ajouter un provider de feature (scopé route) pour `TodosService`.

### 12.3. Exemple : TodoList (standalone)

```ts
import { CommonModule } from '@angular/common';
import { Component, inject } from '@angular/core';
import { RouterModule } from '@angular/router';
import { TodosService } from '../data-access/todos.service';

@Component({
  selector: 'app-todo-list',
  standalone: true,
  imports: [CommonModule, RouterModule],
  template: `
    <h2>Todos</h2>

    <ul>
      <li *ngFor="let t of todos">
        <a [routerLink]="['/todos', t.id]">{{ t.title }}</a>
      </li>
    </ul>
  `
})
export class TodoListComponent {
  private todosService = inject(TodosService);
  todos = this.todosService.getTodos();
}
```

### 12.4. Exemple : service simple

```ts
import { Injectable } from '@angular/core';

export interface Todo {
  id: number;
  title: string;
}

@Injectable()
export class TodosService {
  getTodos(): Todo[] {
    return [
      { id: 1, title: 'Découvrir Standalone Components' },
      { id: 2, title: 'Mettre en place le routing standalone' }
    ];
  }

  getTodo(id: number): Todo | undefined {
    return this.getTodos().find(t => t.id === id);
  }
}
```

### 12.5. Provider au niveau des routes `todos`

```ts
import { Routes } from '@angular/router';
import { TodosService } from './data-access/todos.service';

export const TODOS_ROUTES: Routes = [
  {
    path: '',
    providers: [TodosService],
    children: [
      {
        path: '',
        loadComponent: () => import('./todo-list/todo-list.component')
          .then(m => m.TodoListComponent)
      },
      {
        path: ':id',
        loadComponent: () => import('./todo-detail/todo-detail.component')
          .then(m => m.TodoDetailComponent)
      }
    ]
  }
];
```

---

## 13. Checklist, erreurs fréquentes, bonnes pratiques

### 13.1. Checklist
- [ ] Les composants sont `standalone: true`
- [ ] Les templates n’utilisent que des dépendances présentes dans `imports`
- [ ] Le bootstrap est fait via `bootstrapApplication`
- [ ] Le routing est fourni avec `provideRouter(routes)`
- [ ] Les features sont lazy-loadées via `loadComponent` / `loadChildren`
- [ ] Les providers globaux sont au bootstrap (ou core)
- [ ] Les providers de feature sont posés au niveau route/composant selon les besoins

### 13.2. Erreurs fréquentes
1. **Oublier `CommonModule`** et utiliser `*ngIf/*ngFor` → template error.
2. Importer un **SharedModule trop large** (revient à l’ancien problème).
3. Mettre trop de providers au bootstrap et créer des singletons non désirés.
4. Mélanger NgModule et standalone sans stratégie claire (risque de confusion).

### 13.3. Bonnes pratiques
- Imports minimalistes et explicites.
- Features isolées par routes + providers de scope.
- UI components réutilisables en standalone.
- Favoriser `loadComponent` pour des pages simples et `loadChildren` pour des features multi-routes.

---

## 14. Annexes : commandes, snippets, anti-patterns

### 14.1. Commandes utiles
```bash
# Générer un composant standalone
ng g c shared/ui/button --standalone

# Générer une directive standalone
ng g d shared/directives/auto-focus --standalone

# Générer un pipe standalone
ng g p shared/pipes/capitalize --standalone
```

### 14.2. Snippet “page standalone”

```ts
import { CommonModule } from '@angular/common';
import { Component } from '@angular/core';

@Component({
  selector: 'app-page',
  standalone: true,
  imports: [CommonModule],
  template: `
    <h2>Ma page</h2>
  `,
})
export class PageComponent {}
```

### 14.3. Anti-pattern : SharedModule « fourre-tout »
Symptômes :
- Exporte trop de composants/pipes.
- Injecte des providers globaux.
- Devient importé partout, gonflant le graphe.

Alternative :
- Découper en `shared/ui/*` standalone.
- Importer uniquement les pièces nécessaires.

---

## Conclusion
Les **Standalone Components** permettent de construire une application Angular moderne **sans dépendre systématiquement des NgModules classiques**. Cette approche :
- **simplifie** la structure du projet,
- **facilite le lazy loading** via `loadComponent` et routes standalone,
- **réduit le couplage** grâce aux imports explicites,
- et devient souvent le **point d’entrée principal** des nouvelles applications (bootstrap standalone).

> Prochaine étape recommandée : appliquer ces principes à une feature réelle de votre projet et mesurer l’impact sur la lisibilité, le découpage et le bundle initial.
