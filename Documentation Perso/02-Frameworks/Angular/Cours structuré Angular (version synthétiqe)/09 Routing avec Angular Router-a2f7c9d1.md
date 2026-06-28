# Formation Angular — Routing avec Angular Router

**Public** : développeurs Angular (débutant → intermédiaire)

**Pré-requis** : TypeScript, bases Angular (components, templates, modules/standalone), RxJS notions.

**Objectifs pédagogiques**
- Comprendre le rôle d’Angular Router dans une SPA.
- Définir et organiser des routes (simples, avec paramètres, enfants).
- Utiliser `router-outlet`, `routerLink`, la navigation impérative.
- Mettre en place guards, resolvers, lazy loading, préchargement.
- Maîtriser la gestion des erreurs, redirections, route 404.
- Savoir déboguer et tester le routage.

**Durée suggérée** : 1 jour (7h) — adaptable 1/2 journée (3h30).

---

## Plan de la formation

1. **Introduction au routing SPA**
   - Concept SPA et navigation sans rechargement
   - Notions : URL, segment, état, historique
2. **Mise en place d’Angular Router**
   - Installation / génération
   - `provideRouter` (standalone) vs `RouterModule.forRoot` (NgModule)
   - `router-outlet`
3. **Définir des routes**
   - `path`, `component`, `redirectTo`, `pathMatch`
   - Route par défaut, redirections
   - Route 404 (wildcard)
4. **Navigation dans les templates**
   - `routerLink`, `routerLinkActive`, options
   - Navigation relative vs absolue
   - `queryParams` et `fragment`
5. **Navigation impérative**
   - `Router.navigate` / `navigateByUrl`
   - `ActivatedRoute` : `params`, `queryParams`, `data`, `snapshot`
6. **Paramètres de route et données**
   - Paramètres de chemin `:id`
   - Paramètres de requête
   - Données statiques `data`
7. **Routes enfants et layouts**
   - `children`
   - Plusieurs outlets et outlets nommés
   - Layouts (shell) et modules fonctionnels
8. **Guards : sécuriser et contrôler la navigation**
   - `CanActivate`, `CanDeactivate`, `CanMatch` (lazy), `CanActivateChild`
   - Cas d’usage : authentification, droits, formulaire non sauvegardé
9. **Resolvers : précharger les données**
   - Charger avant activation
   - Gestion des erreurs et fallback
10. **Lazy loading et performance**
   - `loadChildren` / `loadComponent`
   - Préchargement (preloading strategies)
   - Optimiser la taille du bundle
11. **Gestion des erreurs, titres et UX**
   - Title strategy
   - Scroll restoration
   - 404/500, redirections, pages perdues
12. **Debug, bonnes pratiques et tests**
   - Événements du Router
   - Structuration des routes
   - Tests unitaires/integ (RouterTestingHarness/RouterTestingModule)

---

# 1) Introduction au routing SPA

Une application **SPA** (Single Page Application) charge une fois l’application « shell » (HTML/CSS/JS), puis **met à jour la vue** en fonction de l’URL **sans rechargement complet**. Le routing permet :

- d’associer des **URL** à des **composants (vues)**,
- de gérer l’**historique** (back/forward),
- d’implémenter des comportements : **auth**, **lazy loading**, **préchargement**, **résolution de données**, **404**, etc.

### Termes clés
- **Route** : règle de correspondance entre un `path` et une cible (component, redirection, lazy).
- **Segment** : morceau d’URL (ex. `/products/42` → segments `products`, `42`).
- **Outlet** : emplacement dans le DOM où Angular affiche le composant lié à la route.

---

# 2) Mise en place d’Angular Router

## 2.1 Création de projet

```bash
ng new routing-demo --routing --style=scss
```

L’option `--routing` configure un fichier de routes.

## 2.2 Standalone (Angular moderne) : `provideRouter`

**main.ts**
```ts
import { bootstrapApplication } from '@angular/platform-browser';
import { provideRouter } from '@angular/router';
import { routes } from './app/app.routes';
import { AppComponent } from './app/app.component';

bootstrapApplication(AppComponent, {
  providers: [provideRouter(routes)]
});
```

**app.routes.ts**
```ts
import { Routes } from '@angular/router';
import { HomeComponent } from './home/home.component';

export const routes: Routes = [
  { path: '', component: HomeComponent }
];
```

## 2.3 NgModule (projets plus anciens) : `RouterModule.forRoot`

**app-routing.module.ts**
```ts
import { NgModule } from '@angular/core';
import { RouterModule, Routes } from '@angular/router';

const routes: Routes = [/* ... */];

@NgModule({
  imports: [RouterModule.forRoot(routes)],
  exports: [RouterModule]
})
export class AppRoutingModule {}
```

## 2.4 `router-outlet`

Dans le composant racine (ou layout), placez :

```html
<nav>
  <a routerLink="/">Accueil</a>
  <a routerLink="/products">Produits</a>
</nav>

<router-outlet></router-outlet>
```

Sans `router-outlet`, aucun composant routé ne s’affiche.

---

# 3) Définir des routes

## 3.1 Route simple

```ts
{ path: 'products', component: ProductsListComponent }
```

- `path` : segment d’URL
- `component` : composant affiché

## 3.2 Route par défaut

```ts
{ path: '', component: HomeComponent }
```

L’URL `/` match le path vide.

## 3.3 Redirection

```ts
{ path: 'home', redirectTo: '', pathMatch: 'full' }
```

- `redirectTo` : destination
- `pathMatch: 'full'` : indispensable pour les redirections depuis `''` ou des cas ambigus.

### `pathMatch` : `full` vs `prefix`
- `prefix` (par défaut) : une URL commence par le path (match partiel).
- `full` : l’URL doit correspondre exactement (recommandé pour `''` et redirections simples).

## 3.4 Route 404 (wildcard)

```ts
{ path: '**', component: NotFoundComponent }
```

À placer **en dernier**, car `**` match tout.

---

# 4) Navigation dans les templates

## 4.1 `routerLink`

```html
<a routerLink="/products">Produits</a>
<a [routerLink]="['/products', 42]">Produit 42</a>
```

- /products → liste
- /products/42 → détail

## 4.2 `routerLinkActive`

```html
<a routerLink="/products" routerLinkActive="active" [routerLinkActiveOptions]="{ exact: true }">
  Produits
</a>
```

- `exact: true` évite d’activer le lien sur les routes enfants.

## 4.3 Query params et fragment

```html
<a
  [routerLink]="['/products']"
  [queryParams]="{ page: 2, sort: 'price' }"
  fragment="top"
>
  Page 2
</a>
```

Génère : `/products?page=2&sort=price#top`

## 4.4 Navigation relative

Dans `/products/42`, un lien relatif :

```html
<a [routerLink]="['reviews']">Avis</a>
```

se combine avec la route courante → `/products/42/reviews`.

---

# 5) Navigation impérative

## 5.1 `Router.navigate`

```ts
import { Component } from '@angular/core';
import { Router } from '@angular/router';

@Component({ /* ... */ })
export class ProductsListComponent {
  constructor(private router: Router) {}

  goToDetails(id: number) {
    this.router.navigate(['/products', id]);
  }
}
```

## 5.2 `navigateByUrl`

```ts
this.router.navigateByUrl('/products/42?debug=true');
```

Utile si vous avez une URL déjà formée en chaîne.

## 5.3 Lire la route : `ActivatedRoute`

```ts
import { ActivatedRoute } from '@angular/router';

constructor(private route: ActivatedRoute) {}
```

- `route.snapshot` : lecture instantanée
- `route.params` / `route.queryParams` : streams observables

---

# 6) Paramètres de route et données

## 6.1 Paramètres de chemin

Routes :
```ts
{ path: 'products/:id', component: ProductDetailsComponent }
```

Lecture dans le composant :

```ts
import { map } from 'rxjs/operators';

id$ = this.route.paramMap.pipe(
  map(pm => Number(pm.get('id')))
);
```

### Bonnes pratiques
- Utiliser `paramMap` plutôt que `params` (API plus sûre)
- Convertir les types (les params sont des strings)

## 6.2 Query params

```ts
page$ = this.route.queryParamMap.pipe(
  map(qm => Number(qm.get('page') ?? 1))
);
```

## 6.3 Données statiques via `data`

```ts
{ path: 'products', component: ProductsListComponent, data: { title: 'Produits' } }
```

Puis :
```ts
title$ = this.route.data.pipe(map(d => d['title']));
```

---

# 7) Routes enfants et layouts

## 7.1 Layout (shell) + children

Exemple : une zone admin avec menu latéral.

```ts
{
  path: 'admin',
  component: AdminLayoutComponent,
  children: [
    { path: '', component: AdminHomeComponent },
    { path: 'users', component: AdminUsersComponent },
    { path: 'settings', component: AdminSettingsComponent },
  ]
}
```

**AdminLayoutComponent.html**
```html
<aside>
  <a routerLink="/admin">Accueil Admin</a>
  <a routerLink="/admin/users">Utilisateurs</a>
  <a routerLink="/admin/settings">Paramètres</a>
</aside>

<main>
  <router-outlet></router-outlet>
</main>
```

## 7.2 Outlets nommés (avancé)

Permet d’afficher plusieurs vues en parallèle.

```html
<router-outlet></router-outlet>
<router-outlet name="sidebar"></router-outlet>
```

Définition :
```ts
{ path: 'products', component: ProductsListComponent },
{ path: 'filters', component: FiltersComponent, outlet: 'sidebar' }
```

Navigation :
```ts
this.router.navigate([{ outlets: { primary: ['products'], sidebar: ['filters'] } }]);
```

---

# 8) Guards : sécuriser et contrôler la navigation

Les guards permettent de **bloquer**, **autoriser** ou **rediriger**.

> Depuis Angular 15+, on privilégie souvent les **functional guards**.

## 8.1 `CanActivate` (accès à une route)

**Exemple : authentification**

```ts
import { CanActivateFn, Router } from '@angular/router';
import { inject } from '@angular/core';

export const authGuard: CanActivateFn = () => {
  const router = inject(Router);
  const isLoggedIn = Boolean(localStorage.getItem('token'));

  return isLoggedIn ? true : router.createUrlTree(['/login']);
};
```

Usage :
```ts
{ path: 'admin', component: AdminLayoutComponent, canActivate: [authGuard] }
```

## 8.2 `CanMatch` (surtout pour lazy loading)

Empêche même le chargement du bundle lazy si non autorisé.

```ts
import { CanMatchFn, Router } from '@angular/router';
import { inject } from '@angular/core';

export const canMatchAdmin: CanMatchFn = () => {
  const router = inject(Router);
  const isAdmin = localStorage.getItem('role') === 'admin';
  return isAdmin ? true : router.createUrlTree(['/forbidden']);
};
```

## 8.3 `CanDeactivate` (quitter un composant)

Cas classique : formulaire non sauvegardé.

```ts
import { CanDeactivateFn } from '@angular/router';

export interface DirtyAware {
  isDirty(): boolean;
}

export const confirmLeaveGuard: CanDeactivateFn<DirtyAware> = (component) => {
  if (!component.isDirty()) return true;
  return confirm('Vous avez des modifications non sauvegardées. Quitter ?');
};
```

Route :
```ts
{ path: 'profile/edit', component: ProfileEditComponent, canDeactivate: [confirmLeaveGuard] }
```

Dans le composant :
```ts
isDirty() {
  return this.form.dirty;
}
```

---

# 9) Resolvers : précharger les données

Un **resolver** récupère les données **avant** d’activer la route, ce qui simplifie la vue (pas d’état “loading” dans le composant, ou réduit).

## 9.1 Exemple resolver

```ts
import { ResolveFn } from '@angular/router';
import { inject } from '@angular/core';
import { ProductsApi } from './products.api';

export const productResolver: ResolveFn<Product> = (route) => {
  const api = inject(ProductsApi);
  const id = Number(route.paramMap.get('id'));
  return api.getProduct(id);
};
```

Route :
```ts
{ path: 'products/:id', component: ProductDetailsComponent, resolve: { product: productResolver } }
```

Dans le composant :
```ts
product$ = this.route.data.pipe(map(d => d['product'] as Product));
```

## 9.2 Gestion d’erreurs (pattern)

- Intercepter l’erreur dans le resolver (ou via interceptor HTTP)
- Rediriger vers une page d’erreur ou afficher un fallback.

Exemple simple :
```ts
import { catchError, of } from 'rxjs';
import { Router } from '@angular/router';

export const productResolver: ResolveFn<Product | null> = (route) => {
  const api = inject(ProductsApi);
  const router = inject(Router);
  const id = Number(route.paramMap.get('id'));

  return api.getProduct(id).pipe(
    catchError(() => {
      router.navigate(['/not-found']);
      return of(null);
    })
  );
};
```

---

# 10) Lazy loading et performance

Le **lazy loading** charge une fonctionnalité **à la demande**, réduisant le bundle initial.

## 10.1 Lazy loading d’un ensemble de routes

```ts
{
  path: 'admin',
  canMatch: [canMatchAdmin],
  loadChildren: () => import('./admin/admin.routes').then(m => m.ADMIN_ROUTES)
}
```

**admin.routes.ts**
```ts
import { Routes } from '@angular/router';

export const ADMIN_ROUTES: Routes = [
  { path: '', loadComponent: () => import('./admin-home.component').then(c => c.AdminHomeComponent) },
  { path: 'users', loadComponent: () => import('./admin-users.component').then(c => c.AdminUsersComponent) },
];
```

## 10.2 Lazy loading d’un composant (standalone)

```ts
{ path: 'about', loadComponent: () => import('./about/about.component').then(c => c.AboutComponent) }
```

## 10.3 Préchargement

Précharger en arrière-plan les routes lazy après le rendu initial.

```ts
import { provideRouter, withPreloading, PreloadAllModules } from '@angular/router';

providers: [
  provideRouter(routes, withPreloading(PreloadAllModules))
]
```

> Alternative : stratégie custom basée sur la connexion, le rôle, ou `data: { preload: true }`.

---

# 11) Gestion des erreurs, titres et UX

## 11.1 Titres de page

Routes avec `title` :
```ts
{ path: '', component: HomeComponent, title: 'Accueil' },
{ path: 'products', component: ProductsListComponent, title: 'Produits' }
```

Angular met à jour `document.title`.

## 11.2 Scroll restoration

Améliore l’expérience (restaurer le scroll sur back/forward).

```ts
import { provideRouter, withInMemoryScrolling } from '@angular/router';

provideRouter(
  routes,
  withInMemoryScrolling({
    scrollPositionRestoration: 'enabled',
    anchorScrolling: 'enabled'
  })
);
```

## 11.3 404 et redirections structurées

Un schéma courant :

```ts
{ path: '', redirectTo: 'home', pathMatch: 'full' },
{ path: 'home', component: HomeComponent },
{ path: '**', component: NotFoundComponent }
```

---

# 12) Debug, bonnes pratiques et tests

## 12.1 Événements du Router

Permet de diagnostiquer navigation, guards, resolvers.

```ts
import { Router, NavigationEnd } from '@angular/router';
import { filter } from 'rxjs/operators';

constructor(router: Router) {
  router.events
    .pipe(filter(e => e instanceof NavigationEnd))
    .subscribe(e => console.log('NavigationEnd:', e));
}
```

Événements utiles : `NavigationStart`, `RoutesRecognized`, `GuardsCheckStart/End`, `ResolveStart/End`, `NavigationError`.

## 12.2 Bonnes pratiques d’architecture

- **Regrouper** les routes par feature (ex. `admin.routes.ts`).
- Favoriser le **lazy loading** des features lourdes.
- Éviter les dépendances circulaires entre features.
- Standardiser le pattern :
  - liste → détail (`/items`, `/items/:id`)
  - sous-onglets avec `children`
- Utiliser `CanMatch` pour éviter de télécharger des modules inutiles.

## 12.3 Tests (aperçu)

### 12.3.1 RouterTestingModule (NgModule)

```ts
import { TestBed } from '@angular/core/testing';
import { RouterTestingModule } from '@angular/router/testing';

TestBed.configureTestingModule({
  imports: [RouterTestingModule.withRoutes(routes)]
});
```

### 12.3.2 RouterTestingHarness (Angular récent)

Permet de naviguer et d’asserter facilement.

```ts
import { RouterTestingHarness } from '@angular/router/testing';

const harness = await RouterTestingHarness.create();
await harness.navigateByUrl('/products/42');
```

---

# TP fil rouge (exercices guidés)

## Contexte
Construire une mini SPA "Catalog" avec :
- Accueil
- Liste produits
- Détail produit
- Zone admin lazy
- Auth guard
- Resolver sur le détail
- 404

## Étapes
1. Créer les composants : `Home`, `ProductsList`, `ProductDetails`, `NotFound`, `Login`, `AdminLayout`.
2. Définir les routes :
   - `/` → redirect vers `/home`
   - `/products` → liste
   - `/products/:id` → détail avec resolver
   - `/admin` → lazy + guard
   - `**` → 404
3. Ajouter navigation via `routerLink` + menu.
4. Implémenter `authGuard` et une page `/login`.
5. Ajouter `withInMemoryScrolling` et `title`.
6. Bonus : query params `page`, `sort` et mise en évidence via `routerLinkActive`.

### Critères de validation
- Navigation sans rechargement.
- URL reflète l’état.
- Le détail charge ses données via resolver.
- Admin n’est pas chargé si non autorisé.
- 404 s’affiche sur route inconnue.

---

# Aide-mémoire (cheatsheet)

- Afficher une vue : `<router-outlet />`
- Lien : `routerLink="/path"`
- Active link : `routerLinkActive="active"`
- Paramètre : `{ path: 'items/:id', component: ... }`
- Lire param : `route.paramMap`
- Navigation code : `router.navigate([...])`
- Guard : `CanActivateFn`, redirection via `router.createUrlTree([...])`
- Resolver : `ResolveFn`
- Lazy : `loadChildren` / `loadComponent`
- 404 : `{ path: '**', component: NotFoundComponent }`

---

## Conclusion
Angular Router est le cœur de la navigation SPA : il mappe URL → composants, permet la structuration par features, la sécurisation par guards, l’optimisation via lazy loading et l’amélioration UX (titres, scroll). Une bonne architecture de routes rend l’application plus maintenable, testable et performante.
