# Formation Angular — Bootstrapping moderne avec `bootstrapApplication`

> **Objectif** : maîtriser le démarrage “moderne” d’Angular basé sur `bootstrapApplication()` (standalone) et comprendre comment configurer proprement **providers**, **router**, **HttpClient** et **intercepteurs** sans `AppModule`.

- **Public** : développeurs Angular (débutants++/intermédiaires) ayant déjà utilisé `NgModule` ou souhaitant migrer vers les standalone.
- **Pré-requis** : TypeScript, notions Angular (components, DI, services), RxJS de base.
- **Durée conseillée** : 3h à 1 journée (selon profondeur et atelier migration).

---

## Plan du cours

1. **Contexte : du `NgModule` à `bootstrapApplication`**
   - Évolution d’Angular : standalone components & APIs fonctionnelles
   - Pourquoi `AppModule` devient optionnel
   - Ce que change le “bootstrapping moderne”

2. **Anatomie d’un démarrage moderne**
   - `main.ts` minimal
   - Options de `bootstrapApplication(AppComponent, { providers: [...] })`
   - Providers explicites vs agrégés dans un module

3. **Configuration des providers de façon moderne**
   - `provide*` helpers : vision d’ensemble
   - Fournir des services, tokens, config
   - Environnements (`environment.ts`) et injection

4. **Router moderne**
   - `provideRouter(routes, ...)`
   - Standalone routes + lazy loading
   - Stratégies de préchargement et features Router

5. **HttpClient moderne**
   - `provideHttpClient()`
   - `withInterceptors()`, `withInterceptorsFromDi()`
   - Bonnes pratiques : interceptors fonctionnels vs classes

6. **Intercepteurs : patterns et cas d’usage**
   - Auth/JWT, logging, erreurs globales, retry, correlation id
   - Composition et ordre des interceptors
   - Tests rapides d’interceptors

7. **Migration depuis une appli basée sur `AppModule`**
   - Stratégie progressive
   - Check-list de migration
   - Pièges fréquents

8. **Architecture & bonnes pratiques**
   - Séparer “configuration plateforme” vs “configuration métier”
   - Fichiers dédiés : `app.config.ts`, `app.routes.ts`
   - Lisibilité, maintenabilité, testabilité

9. **Atelier guidé**
   - Mettre en place `bootstrapApplication`
   - Ajouter router + HttpClient + interceptors
   - Vérifier avec un mini scénario (login simulé)

---

## 1) Contexte : du `NgModule` à `bootstrapApplication`

### 1.1. Historique rapide
Pendant longtemps, Angular s’appuyait fortement sur les **NgModules** :
- `AppModule` comme point d’entrée
- déclaration des composants/pipes/directives
- import d’autres modules (`HttpClientModule`, `RouterModule.forRoot(...)`, etc.)
- configuration des providers (interceptors, guards, etc.)

Depuis Angular 14+ (et surtout Angular 15+), Angular introduit et encourage :
- les **standalone components**
- des APIs de configuration **fonctionnelles** (`provideRouter`, `provideHttpClient`, ...)
- un démarrage via **`bootstrapApplication()`**

### 1.2. Pourquoi `bootstrapApplication` ?
Le démarrage moderne :
- **supprime la nécessité** d’un `AppModule` central
- rend la configuration **plus explicite** (tout est lisible dans une liste de providers)
- favorise une architecture alignée sur les **standalone components**
- simplifie la mise en place du **Router**, de **HttpClient** et des **intercepteurs**

### 1.3. Ce qui change concrètement
Avant (style module, simplifié) :
```ts
// main.ts
platformBrowserDynamic().bootstrapModule(AppModule);

// app.module.ts
@NgModule({
  declarations: [AppComponent],
  imports: [BrowserModule, HttpClientModule, RouterModule.forRoot(routes)],
  providers: [{ provide: HTTP_INTERCEPTORS, useClass: AuthInterceptor, multi: true }],
  bootstrap: [AppComponent]
})
export class AppModule {}
```

Après (style moderne) :
```ts
// main.ts
import { bootstrapApplication } from '@angular/platform-browser';
import { AppComponent } from './app/app.component';
import { appConfig } from './app/app.config';

bootstrapApplication(AppComponent, appConfig)
  .catch(err => console.error(err));
```

La configuration (router, http, providers) est déplacée vers des **providers** dans une config : `appConfig`.

---

## 2) Anatomie d’un démarrage moderne

### 2.1. `main.ts` minimal (recommandé)
Objectif : un point d’entrée lisible et stable.

```ts
import { bootstrapApplication } from '@angular/platform-browser';
import { AppComponent } from './app/app.component';
import { appConfig } from './app/app.config';

bootstrapApplication(AppComponent, appConfig)
  .catch(console.error);
```

### 2.2. Créer `app.config.ts`
Angular fournit le type `ApplicationConfig` pour structurer la configuration.

```ts
// app/app.config.ts
import { ApplicationConfig } from '@angular/core';
import { provideRouter } from '@angular/router';
import { provideHttpClient } from '@angular/common/http';

import { routes } from './app.routes';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes),
    provideHttpClient(),
  ]
};
```

### 2.3. Les providers comme “contrat de démarrage”
Le tableau `providers` devient la “liste de ce que l’app active” :
- routing
- http
- interceptors
- animations
- hydratation SSR, etc.

Cela encourage :
- la **lisibilité**
- le **contrôle explicite**
- la **réduction de magie** des imports de modules

---

## 3) Configuration moderne des providers

### 3.1. Fournir un service
Si un service est déjà décoré avec `@Injectable({ providedIn: 'root' })`, **aucune action** n’est nécessaire.

Sinon, vous pouvez l’ajouter dans `appConfig` :
```ts
import { ApplicationConfig } from '@angular/core';
import { SomeService } from './core/some.service';

export const appConfig: ApplicationConfig = {
  providers: [SomeService]
};
```

### 3.2. Fournir un token de configuration
Pattern typique : exposer une config d’API.

```ts
import { InjectionToken } from '@angular/core';

export interface ApiConfig {
  baseUrl: string;
}

export const API_CONFIG = new InjectionToken<ApiConfig>('API_CONFIG');
```

Puis provider :
```ts
import { ApplicationConfig } from '@angular/core';
import { API_CONFIG } from './core/api-config.token';
import { environment } from '../environments/environment';

export const appConfig: ApplicationConfig = {
  providers: [
    {
      provide: API_CONFIG,
      useValue: { baseUrl: environment.apiBaseUrl }
    }
  ]
};
```

### 3.3. Centraliser proprement
Bonne pratique : séparer configuration “plateforme” et “métier”.
- `app.config.ts` : providers globaux
- `core/` : tokens, interceptors, services techniques
- `features/` : routes lazy, composants standalone

---

## 4) Router moderne : `provideRouter`

### 4.1. Définir les routes
Créer un fichier `app.routes.ts` :

```ts
import { Routes } from '@angular/router';

export const routes: Routes = [
  {
    path: '',
    loadComponent: () => import('./features/home/home.component').then(m => m.HomeComponent)
  },
  {
    path: 'admin',
    loadChildren: () => import('./features/admin/admin.routes').then(m => m.ADMIN_ROUTES)
  },
  { path: '**', redirectTo: '' }
];
```

### 4.2. Activer le router au démarrage
Dans `app.config.ts` :
```ts
import { ApplicationConfig } from '@angular/core';
import { provideRouter, withComponentInputBinding } from '@angular/router';
import { routes } from './app.routes';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(
      routes,
      withComponentInputBinding() // pratique: mappe les params vers @Input()
    )
  ]
};
```

### 4.3. Router-outlet dans un composant standalone

```ts
import { Component } from '@angular/core';
import { RouterOutlet } from '@angular/router';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [RouterOutlet],
  template: `
    <h1>Demo</h1>
    <router-outlet />
  `
})
export class AppComponent {}
```

---

## 5) HttpClient moderne : `provideHttpClient`

### 5.1. Activer HttpClient

```ts
import { ApplicationConfig } from '@angular/core';
import { provideHttpClient } from '@angular/common/http';

export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient()
  ]
};
```

### 5.2. Utiliser HttpClient dans un service

```ts
import { inject, Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { API_CONFIG } from './api-config.token';

@Injectable({ providedIn: 'root' })
export class UserApi {
  private http = inject(HttpClient);
  private config = inject(API_CONFIG);

  getMe() {
    return this.http.get(`${this.config.baseUrl}/me`);
  }
}
```

---

## 6) Intercepteurs : configuration et patterns

Angular moderne permet :
- des intercepteurs **fonctionnels** (recommandé)
- des intercepteurs **classiques** (compatibilité / DI)

### 6.1. Intercepteur fonctionnel (recommandé)
Exemple : ajout d’un header `Authorization` (token simulé).

```ts
import { HttpInterceptorFn } from '@angular/common/http';

export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const token = localStorage.getItem('token');

  if (!token) {
    return next(req);
  }

  const cloned = req.clone({
    setHeaders: { Authorization: `Bearer ${token}` }
  });

  return next(cloned);
};
```

### 6.2. Enregistrer l’intercepteur avec `withInterceptors`

```ts
import { ApplicationConfig } from '@angular/core';
import { provideHttpClient, withInterceptors } from '@angular/common/http';
import { authInterceptor } from './core/http/auth.interceptor';

export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(
      withInterceptors([authInterceptor])
    )
  ]
};
```

### 6.3. Intercepteur de logging

```ts
import { HttpInterceptorFn } from '@angular/common/http';
import { tap } from 'rxjs/operators';

export const loggingInterceptor: HttpInterceptorFn = (req, next) => {
  const started = performance.now();

  return next(req).pipe(
    tap({
      next: () => {
        const ms = performance.now() - started;
        console.debug(`[HTTP] ${req.method} ${req.urlWithParams} (${ms.toFixed(1)}ms)`);
      },
      error: (err) => {
        const ms = performance.now() - started;
        console.warn(`[HTTP] ERROR ${req.method} ${req.urlWithParams} (${ms.toFixed(1)}ms)`, err);
      }
    })
  );
};
```

Composition :
```ts
provideHttpClient(withInterceptors([loggingInterceptor, authInterceptor]))
```

> **Ordre** : l’intercepteur en premier dans le tableau voit la requête en premier.

### 6.4. Compatibilité : intercepteurs “DI” (legacy)
Si vous avez déjà des intercepteurs en classes :

```ts
import { Injectable } from '@angular/core';
import { HttpInterceptor, HttpRequest, HttpHandler, HttpEvent } from '@angular/common/http';
import { Observable } from 'rxjs';

@Injectable()
export class LegacyAuthInterceptor implements HttpInterceptor {
  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    return next.handle(req);
  }
}
```

Vous pouvez :
1) les fournir via DI, puis
2) activer la collecte avec `withInterceptorsFromDi()`.

```ts
import { provideHttpClient, withInterceptorsFromDi, HTTP_INTERCEPTORS } from '@angular/common/http';

export const appConfig: ApplicationConfig = {
  providers: [
    {
      provide: HTTP_INTERCEPTORS,
      useClass: LegacyAuthInterceptor,
      multi: true
    },
    provideHttpClient(withInterceptorsFromDi())
  ]
};
```

---

## 7) Migration depuis une application avec `AppModule`

### 7.1. Stratégie progressive (recommandée)
1. Convertir `AppComponent` en standalone (`standalone: true`)
2. Remplacer `bootstrapModule(AppModule)` par `bootstrapApplication(AppComponent, config)`
3. Déplacer les imports “module” vers des providers :
   - `RouterModule.forRoot` → `provideRouter(routes)`
   - `HttpClientModule` → `provideHttpClient(...)`
   - Interceptors → `withInterceptors(...)` ou `withInterceptorsFromDi()`
4. Convertir progressivement les feature modules vers `loadComponent` / `loadChildren` standalone

### 7.2. Check-list
- [ ] `main.ts` utilise `bootstrapApplication`
- [ ] `AppComponent` est standalone
- [ ] Les routes sont dans `app.routes.ts`
- [ ] `provideRouter` est présent
- [ ] `provideHttpClient` est présent
- [ ] Interceptors migrés ou activés via `withInterceptorsFromDi`
- [ ] Imports de modules inutiles supprimés

### 7.3. Pièges fréquents
- Oublier d’importer `RouterOutlet` dans un composant standalone
- Surdéclarer des providers (doublons) entre config et composants
- Mélanger `HttpClientModule` et `provideHttpClient()` (éviter : rester cohérent)
- Conflits d’ordre d’interceptors

---

## 8) Architecture & bonnes pratiques

### 8.1. Séparer la configuration
Structure recommandée :
```
src/
  main.ts
  app/
    app.component.ts
    app.config.ts
    app.routes.ts
    core/
      http/
        auth.interceptor.ts
        logging.interceptor.ts
      api-config.token.ts
      user-api.service.ts
    features/
      home/
        home.component.ts
      admin/
        admin.routes.ts
```

### 8.2. Garder `main.ts` “boring”
- pas de logique métier
- pas de configuration dispersée
- juste le démarrage

### 8.3. Prefer “function-based APIs”
- `provideRouter` et options `with...`
- `provideHttpClient` et options `with...`

Résultat : configuration plus déclarative et modulaire.

---

## 9) Atelier guidé (pas à pas)

### 9.1. Objectif de l’atelier
Mettre en place :
- bootstrap moderne
- navigation Home / Admin
- HttpClient + interceptors (auth + logging)

### 9.2. Étapes

#### Étape A — Passer en standalone et `bootstrapApplication`
1. Vérifier que `AppComponent` est standalone
2. Créer `app.config.ts` et `app.routes.ts`
3. Adapter `main.ts`

#### Étape B — Router
1. Créer une `HomeComponent` standalone
2. Ajouter une route `admin` lazy via `loadChildren`
3. Placer `RouterOutlet` dans `AppComponent`

#### Étape C — HttpClient + intercepteurs
1. Créer `API_CONFIG` + `environment.apiBaseUrl`
2. Fournir `provideHttpClient(withInterceptors([...]))`
3. Implémenter :
   - `loggingInterceptor`
   - `authInterceptor`

#### Étape D — Vérification
- Déclencher un appel HTTP (ex : `getMe()`)
- Observer :
  - le header `Authorization`
  - les logs de durée
  - le workflow router

---

## Annexes

### A) Exemple complet minimal (récap)

**`main.ts`**
```ts
import { bootstrapApplication } from '@angular/platform-browser';
import { AppComponent } from './app/app.component';
import { appConfig } from './app/app.config';

bootstrapApplication(AppComponent, appConfig)
  .catch(console.error);
```

**`app.config.ts`**
```ts
import { ApplicationConfig } from '@angular/core';
import { provideRouter, withComponentInputBinding } from '@angular/router';
import { provideHttpClient, withInterceptors } from '@angular/common/http';

import { routes } from './app.routes';
import { authInterceptor } from './core/http/auth.interceptor';
import { loggingInterceptor } from './core/http/logging.interceptor';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes, withComponentInputBinding()),
    provideHttpClient(withInterceptors([loggingInterceptor, authInterceptor]))
  ]
};
```

**`app.component.ts`**
```ts
import { Component } from '@angular/core';
import { RouterOutlet, RouterLink } from '@angular/router';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [RouterOutlet, RouterLink],
  template: `
    <nav>
      <a routerLink="/">Home</a> |
      <a routerLink="/admin">Admin</a>
    </nav>
    <router-outlet />
  `
})
export class AppComponent {}
```

---

## Résultats attendus
À la fin de cette formation, vous saurez :
- démarrer une app Angular sans `AppModule` via `bootstrapApplication`
- structurer une configuration claire et maintenable (`app.config.ts`)
- configurer le router moderne (`provideRouter`) avec lazy loading
- configurer HttpClient moderne (`provideHttpClient`) et composer des interceptors
- migrer progressivement une application existante vers une architecture standalone
