# 20 — Core module, Shared module et alternatives modernes (standalone)

> **Public visé** : développeurs Angular (niveau intermédiaire) et formateurs souhaitant mettre à jour leurs pratiques.
>
> **Pré-requis** : TypeScript, DI Angular, notions de modules (NgModule), routing.
>
> **Objectif** : comprendre l’approche historique **CoreModule/SharedModule**, ses pièges, et savoir la **remplacer proprement** dans une architecture moderne basée sur les **standalone components**, les **providers centralisés**, et des **imports ciblés**.

---

## Sommaire

1. [Contexte historique et motivations](#1-contexte-historique-et-motivations)
2. [Le pattern CoreModule (historique)](#2-le-pattern-coremodule-historique)
3. [Le pattern SharedModule (historique)](#3-le-pattern-sharedmodule-historique)
4. [Problèmes fréquents avec Core/Shared en pratique](#4-problèmes-fréquents-avec-coreshared-en-pratique)
5. [Rappels modernes : standalone, DI et scoping des providers](#5-rappels-modernes--standalone-di-et-scoping-des-providers)
6. [Alternatives modernes au CoreModule](#6-alternatives-modernes-au-coremodule)
7. [Alternatives modernes au SharedModule](#7-alternatives-modernes-au-sharedmodule)
8. [Organisation recommandée d’un projet Angular moderne](#8-organisation-recommandée-dun-projet-angular-moderne)
9. [Cas pratiques guidés (pas à pas)](#9-cas-pratiques-guidés-pas-à-pas)
10. [Checklist d’audit et règles d’équipe](#10-checklist-daudit-et-règles-déquipe)
11. [FAQ et pièges](#11-faq-et-pièges)
12. [Annexes : snippets réutilisables](#12-annexes--snippets-réutilisables)

---

## 1. Contexte historique et motivations

Pendant des années, Angular (v2 → v14 environ) s’est structuré autour des **NgModules**.

Deux patterns se sont imposés :

- **CoreModule** :
  - point unique pour les **singletons** (services, interceptors, guards),
  - configuration d’app,
  - composants “shell” non réutilisables (ex: `AppComponent`, `NavbarComponent`).

- **SharedModule** :
  - composants, directives, pipes **réutilisables**,
  - ré-export de modules Angular (`CommonModule`, `FormsModule`),
  - importé dans de nombreux modules/features.

### Pourquoi cela existait ?

- Les NgModules **mélangeaient** : déclaration UI + providers + imports/ré-exports.
- On voulait éviter :
  - des imports répétitifs,
  - des providers dupliqués,
  - un code “fourre-tout” dans `AppModule`.

### Changement de paradigme

Avec l’arrivée des **standalone components** (v14+) et l’approche moderne **bootstrapApplication** (v14+), on peut :

- déclarer/importer **au plus près du besoin** ;
- fournir des services **via l’injecteur racine** (`providedIn: 'root'`) ou **au bootstrap** ;
- remplacer les “modules globaux” par des **providers explicites** (ex: `provideRouter`, `provideHttpClient`) et des **imports ciblés**.

---

## 2. Le pattern CoreModule (historique)

### 2.1. Intention

Un `CoreModule` servait à :

- regrouper les **services singletons** (Auth, Logger…),
- fournir des **interceptors** HTTP,
- déclarer des composants “layout” utilisés une fois (header, footer),
- centraliser une config (ex: tokens d’injection).

### 2.2. Exemple typique (NgModule)

```ts
// core/core.module.ts
@NgModule({
  imports: [CommonModule, HttpClientModule],
  providers: [AuthService, LoggerService],
})
export class CoreModule {
  constructor(@Optional() @SkipSelf() parent: CoreModule) {
    if (parent) {
      throw new Error('CoreModule must be imported only once (AppModule).');
    }
  }
}
```

**Objectif** : empêcher l’import multiple et donc la duplication de providers.

### 2.3. Où l’importer ?

- Dans `AppModule` uniquement (ou `AppServerModule` côté SSR).

---

## 3. Le pattern SharedModule (historique)

### 3.1. Intention

Un `SharedModule` servait à :

- déclarer des composants/pipes/directives réutilisables,
- ré-exporter des modules Angular fréquemment utilisés.

### 3.2. Exemple typique

```ts
// shared/shared.module.ts
@NgModule({
  imports: [CommonModule],
  declarations: [CapitalizePipe, ButtonComponent],
  exports: [CommonModule, CapitalizePipe, ButtonComponent],
})
export class SharedModule {}
```

**Idée** : un seul import apporte plein de choses.

---

## 4. Problèmes fréquents avec Core/Shared en pratique

### 4.1. Effet “barrel” involontaire

`SharedModule` devient souvent un **fourre-tout** :

- exports non maîtrisés,
- dépendances implicites,
- augmentation du coût cognitif.

### 4.2. Importer Shared = importer trop

Vous importez `SharedModule` pour 1 pipe, mais récupérez :

- des composants inutiles,
- des providers accidentels (si quelqu’un en a ajouté),
- des dépendances transverses.

### 4.3. Providers dupliqués / scope non voulu

- Importer `CoreModule` dans une feature lazy-loaded peut créer un **second injecteur** et provoquer :
  - double instance d’un service,
  - comportements incohérents (Auth/State).

### 4.4. Couplage caché

`SharedModule` peut masquer des besoins :

- des directives nécessaires,
- un module de traduction,
- des providers (ex: `DateAdapter`) importés sans le savoir.

### 4.5. Migration douloureuse vers standalone

Si “tout” passe par Core/Shared, le passage vers standalone force à :

- découpler les providers,
- rendre les imports explicites,
- clarifier la responsabilité de chaque bloc.

---

## 5. Rappels modernes : standalone, DI et scoping des providers

### 5.1. Standalone components

Un composant standalone déclare ses dépendances dans `imports` :

```ts
@Component({
  standalone: true,
  selector: 'app-user-card',
  templateUrl: './user-card.html',
  imports: [NgIf, DatePipe],
})
export class UserCardComponent {}
```

**Bénéfice** : dépendances explicites et localisées.

### 5.2. Où fournir des services aujourd’hui ?

Trois niveaux principaux :

1. **Racine globale** : `@Injectable({ providedIn: 'root' })`
2. **Bootstrap application** : `bootstrapApplication(AppComponent, { providers: [...] })`
3. **Route-level providers** : providers associés à une route (excellent pour lazy loading)

### 5.3. Rappels sur les injecteurs

- Chaque lazy-loaded route/module peut créer un **injecteur enfant**.
- Fournir un service dans un injecteur enfant ⇒ instance *spécifique* à ce scope.
- Fournir un singleton global ⇒ `providedIn: 'root'` ou providers du bootstrap.

---

## 6. Alternatives modernes au CoreModule

### 6.1. Le “Core” devient un **ensemble de providers**

Au lieu d’un module, on crée un fichier `app.providers.ts` (ou plusieurs) :

```ts
// app/app.providers.ts
import { provideHttpClient, withInterceptors } from '@angular/common/http';
import { provideRouter } from '@angular/router';

export const appProviders = [
  provideRouter(APP_ROUTES),
  provideHttpClient(
    withInterceptors([authInterceptor])
  ),
];
```

Puis au bootstrap :

```ts
bootstrapApplication(AppComponent, {
  providers: [...appProviders]
});
```

**Résultat** :
- plus de `CoreModule` à importer,
- les singletons sont fournis de façon centralisée et explicite.

### 6.2. Services singletons : `providedIn: 'root'`

```ts
@Injectable({ providedIn: 'root' })
export class AuthService {
  // singleton global
}
```

**Conseil** : réserver les providers “au bootstrap” pour :
- config globale dépendante d’environnement,
- interceptors,
- initializers,
- providers qui ne peuvent pas être en `providedIn`.

### 6.3. HTTP moderne : `provideHttpClient`

Remplace souvent `HttpClientModule` :

```ts
provideHttpClient(
  withInterceptors([authInterceptor, loggingInterceptor]),
  // optionnel: withFetch()
)
```

### 6.4. Router moderne : `provideRouter`

```ts
export const APP_ROUTES: Routes = [
  {
    path: '',
    loadComponent: () => import('./shell/shell.component').then(m => m.ShellComponent),
    children: [
      {
        path: 'users',
        loadChildren: () => import('./users/users.routes').then(m => m.USERS_ROUTES),
      },
    ],
  },
];

bootstrapApplication(AppComponent, {
  providers: [provideRouter(APP_ROUTES)]
});
```

### 6.5. “Core UI” : préférer un **Shell** standalone

Au lieu d’un module core qui déclare `NavbarComponent`, créer un composant `ShellComponent` standalone.

```ts
@Component({
  standalone: true,
  imports: [RouterOutlet, NavbarComponent],
  template: `
    <app-navbar />
    <router-outlet />
  `,
})
export class ShellComponent {}
```

### 6.6. Configuration globale via tokens

```ts
export interface AppConfig { apiBaseUrl: string; }
export const APP_CONFIG = new InjectionToken<AppConfig>('APP_CONFIG');

bootstrapApplication(AppComponent, {
  providers: [
    { provide: APP_CONFIG, useValue: { apiBaseUrl: '/api' } },
  ]
});
```

---

## 7. Alternatives modernes au SharedModule

### 7.1. Le “shared” devient une bibliothèque de **standalone building blocks**

Au lieu d’un `SharedModule` géant :

- `shared/ui/button/button.component.ts` (standalone)
- `shared/ui/card/card.component.ts` (standalone)
- `shared/pipes/capitalize.pipe.ts` (standalone)
- `shared/directives/autofocus.directive.ts` (standalone)

Chaque consumer importe ce dont il a besoin.

#### Exemple : pipe standalone

```ts
@Pipe({ name: 'capitalize', standalone: true })
export class CapitalizePipe implements PipeTransform {
  transform(value: string | null | undefined): string {
    if (!value) return '';
    return value.charAt(0).toUpperCase() + value.slice(1);
  }
}
```

Usage :

```ts
@Component({
  standalone: true,
  imports: [CapitalizePipe],
  template: `{{ username() | capitalize }}`,
})
export class ProfileComponent {
  username = signal('alex');
}
```

### 7.2. Imports ciblés : éviter les “ré-exports massifs”

Avec standalone, vous importez directement :

- `NgIf`, `NgFor`, `AsyncPipe`, `DatePipe` depuis `@angular/common`
- `ReactiveFormsModule` si nécessaire
- composants UI spécifiques

```ts
import { NgIf, AsyncPipe } from '@angular/common';
import { ReactiveFormsModule } from '@angular/forms';
```

### 7.3. `importProvidersFrom(...)` : une étape de migration, pas un objectif

On peut encore consommer un module existant via :

```ts
bootstrapApplication(AppComponent, {
  providers: [
    importProvidersFrom(SomeLegacyModule)
  ]
});
```

**À utiliser** :
- pour migrer progressivement,
- pour des libs tierces encore module-based.

**À éviter** :
- comme remplacement “automatique” d’un `SharedModule` omniprésent.

### 7.4. Regrouper sans sur-importer : “feature shared”

Il est parfois légitime d’avoir un *mini-shared* **par feature**.

Exemple : dans `users/`, créer `users/ui/` avec composants dédiés à la feature.

**Règle** :
- `shared/` = réutilisable cross-feature
- `users/ui/` = réutilisable *dans users seulement*

---

## 8. Organisation recommandée d’un projet Angular moderne

### 8.1. Proposition d’arborescence

```
src/
  app/
    app.component.ts           (standalone)
    app.routes.ts
    app.providers.ts

    shell/
      shell.component.ts
      navbar/
        navbar.component.ts

    shared/
      ui/
        button/button.component.ts
        card/card.component.ts
      pipes/
        capitalize.pipe.ts
      directives/
        autofocus.directive.ts

    features/
      users/
        users.routes.ts
        pages/
          users-list.page.ts
          user-details.page.ts
        data-access/
          users.service.ts
          users.api.ts
        ui/
          user-card/user-card.component.ts
```

### 8.2. Règles de dépendances (simples)

- `shared/*` ne dépend pas des features.
- `feature/*/data-access` ne dépend pas de `feature/*/pages`.
- `pages` consomme `ui` + `data-access`.
- Les providers globaux sont dans `app.providers.ts`.

---

## 9. Cas pratiques guidés (pas à pas)

### Cas pratique A — Migrer CoreModule vers providers de bootstrap

#### Situation
Vous avez :
- `CoreModule` avec interceptors + AuthService + un token de config.

#### Étapes

1) **Mettre les services en `providedIn: 'root'`** quand possible

```ts
@Injectable({ providedIn: 'root' })
export class LoggerService {}
```

2) **Créer `app.providers.ts`**

```ts
export const APP_CONFIG = new InjectionToken<AppConfig>('APP_CONFIG');

export const appProviders = [
  { provide: APP_CONFIG, useValue: { apiBaseUrl: '/api' } },
  provideHttpClient(withInterceptors([authInterceptor])),
];
```

3) **Fournir au bootstrap**

```ts
bootstrapApplication(AppComponent, {
  providers: [...appProviders]
});
```

4) **Supprimer l’import CoreModule** (et son guard anti-multi-import)

#### Critère de réussite
- une seule instance AuthService
- interceptors actifs
- plus aucun import de CoreModule

---

### Cas pratique B — Découper un SharedModule en standalone

#### Situation
`SharedModule` exporte 30 éléments, beaucoup inutilisés.

#### Étapes

1) Convertir chaque pipe/directive en standalone

```ts
@Directive({ selector: '[autofocus]', standalone: true })
export class AutofocusDirective {
  constructor(private el: ElementRef<HTMLInputElement>) {}
  ngAfterViewInit() { this.el.nativeElement.focus(); }
}
```

2) Convertir les composants en standalone

```ts
@Component({
  standalone: true,
  selector: 'app-button',
  template: `<button class="btn"><ng-content /></button>`,
})
export class ButtonComponent {}
```

3) Mettre à jour les consumers (imports ciblés)

```ts
@Component({
  standalone: true,
  imports: [ButtonComponent, AutofocusDirective, NgIf],
  template: `
    <app-button autofocus *ngIf="canSave()">Save</app-button>
  `
})
export class EditPage {}
```

4) Supprimer progressivement SharedModule

#### Critère de réussite
- plus de `SharedModule` importé
- dépendances explicites par composant

---

### Cas pratique C — Providers au niveau route (remplacement “Core” local)

#### Objectif
Créer une instance de `UsersFacade` *par navigation dans la feature*.

```ts
export const USERS_ROUTES: Routes = [
  {
    path: '',
    providers: [UsersFacade],
    children: [
      {
        path: '',
        loadComponent: () => import('./pages/users-list.page').then(m => m.UsersListPage),
      },
    ],
  },
];
```

**Résultat** :
- l’état de la feature est scoped,
- pas besoin d’un module spécifique.

---

## 10. Checklist d’audit et règles d’équipe

### 10.1. Remplacer CoreModule si…

- il sert principalement à fournir des services interceptors/config.

**Action** : déplacer vers `app.providers.ts` + `providedIn: 'root'`.

### 10.2. Supprimer SharedModule si…

- il est importé partout “par défaut”.

**Action** : passer les éléments en standalone et importer ciblé.

### 10.3. Règles recommandées

- Ne pas ré-exporter “tout” via un seul point d’entrée.
- Les providers globaux sont déclarés au bootstrap.
- Les providers “feature state” sont au niveau route.
- `shared/` doit rester **petit** et **stable**.

---

## 11. FAQ et pièges

### « Puis-je encore utiliser des NgModules ? »
Oui. Standalone est un **superset** : vous pouvez mixer. Mais évitez de recréer des “Core/Shared” globaux par habitude.

### « `providedIn: 'root'` suffit-il toujours ? »
Non. Certains providers doivent être configurés (tokens, interceptors, options). Ceux-là vont au bootstrap ou à la route.

### « Importer trop d’éléments en standalone, est-ce un problème perf ? »
En général, non : Angular et les bundlers gèrent le tree-shaking. Le vrai gain est la **clarté** et la **maîtrise des dépendances**.

### « Et pour les libs Material/PrimeNG ? »
- Si la lib est module-based : utilisez des imports ciblés par module, ou `importProvidersFrom` au bootstrap si nécessaire.
- Tendez vers le minimum : évitez un `MaterialModule` géant.

---

## 12. Annexes : snippets réutilisables

### 12.1. Template `app.providers.ts`

```ts
import { provideRouter } from '@angular/router';
import { provideHttpClient, withInterceptors } from '@angular/common/http';

import { APP_ROUTES } from './app.routes';
import { authInterceptor } from './core/http/auth.interceptor';

export const appProviders = [
  provideRouter(APP_ROUTES),
  provideHttpClient(withInterceptors([authInterceptor])),
];
```

### 12.2. Interceptor fonctionnel (Angular moderne)

```ts
import { HttpInterceptorFn } from '@angular/common/http';

export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const token = localStorage.getItem('token');
  if (!token) return next(req);

  return next(
    req.clone({
      setHeaders: { Authorization: `Bearer ${token}` },
    })
  );
};
```

### 12.3. Exemple de route lazy avec `loadComponent`

```ts
export const APP_ROUTES: Routes = [
  {
    path: 'settings',
    loadComponent: () => import('./features/settings/settings.page').then(m => m.SettingsPage),
  },
];
```

---

## Conclusion

- **CoreModule** et **SharedModule** étaient des solutions pragmatiques à l’ère des NgModules.
- En architecture moderne **standalone**, on privilégie :
  - des **providers explicites** (bootstrap / route / root),
  - des **imports ciblés** par composant,
  - une organisation par responsabilités.

Vous gagnez :
- une découpe plus claire,
- moins de magie implicite,
- une migration et une maintenance plus simples.
