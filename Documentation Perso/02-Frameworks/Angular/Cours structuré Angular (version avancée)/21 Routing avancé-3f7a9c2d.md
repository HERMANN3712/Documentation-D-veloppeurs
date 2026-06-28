# Formation Angular – Routing avancé

**Référence** : 21  
**Durée conseillée** : 1 à 2 jours (adaptable)  
**Prérequis** : Angular (components, services, modules), TypeScript, RxJS de base, notions de DI  
**Objectifs pédagogiques** :
- Concevoir une stratégie de navigation modulaire et maintenable.
- Maîtriser les routes imbriquées, paramétrées, redirections et données de route.
- Utiliser **guards** et **resolvers** pour sécuriser et préparer la navigation.
- Mettre en place du **lazy loading** (et options associées) pour optimiser le chargement.
- Comprendre le **cycle de navigation** et diagnostiquer les problèmes de routing.

---

## Plan (vue d’ensemble)

1. **Rappels & architecture Router**
2. **Configuration avancée des routes**
3. **Routes imbriquées & layouts**
4. **Routes paramétrées, query params & fragments**
5. **Données de route (data) & titles**
6. **Redirections, routes de fallback & stratégie de matching**
7. **Guards avancés (auth, rôles, unsaved changes, redirections)**
8. **Resolvers (préchargement de données) & orchestration RxJS**
9. **Lazy loading moderne & architecture modulaire**
10. **Préchargement (PreloadingStrategy) & performance**
11. **Cycle de navigation, événements du Router & debug**
12. **Bonnes pratiques, patterns & hands-on final**

---

# 1) Rappels & architecture Router

## 1.1 Le rôle du Router
Le Router Angular fournit une navigation **déclarative** (via configuration de routes) et **impérative** (via API `router.navigate`). Il relie :
- une **URL**
- un **état de navigation**
- une **arborescence de composants** rendue via `router-outlet`

## 1.2 Concepts clés
- **Route configuration** : tableau de `Routes`.
- **Router outlet(s)** : points d’insertion de vues.
- **ActivatedRoute** : accès au contexte de la route courante (params, data, etc.).
- **RouterState / UrlTree** : représentation de l’état/URL.
- **Navigation** : suite d’étapes (matching, guards, resolvers, activation, rendering).

## 1.3 Standalone vs NgModule (contexte)
Angular supporte désormais largement les **standalone components**. Le routing s’intègre via :
- `provideRouter(routes, ...)` (standalone)
- ou `RouterModule.forRoot(routes)` (NgModule)

Dans le reste du cours, les exemples sont majoritairement en **standalone** (le plus moderne), mais les concepts sont identiques.

---

# 2) Configuration avancée des routes

## 2.1 Structure de base (standalone)
```ts
// app.routes.ts
import { Routes } from '@angular/router';

export const routes: Routes = [
  { path: '', pathMatch: 'full', redirectTo: 'home' },
  {
    path: 'home',
    loadComponent: () => import('./pages/home/home.page').then(m => m.HomePage),
    title: 'Accueil'
  },
  {
    path: '**',
    loadComponent: () => import('./pages/not-found/not-found.page').then(m => m.NotFoundPage),
    title: '404'
  }
];
```

```ts
// main.ts
import { bootstrapApplication } from '@angular/platform-browser';
import { provideRouter } from '@angular/router';
import { routes } from './app/app.routes';
import { AppComponent } from './app/app.component';

bootstrapApplication(AppComponent, {
  providers: [
    provideRouter(routes)
  ]
});
```

## 2.2 Options globales `provideRouter`
Exemples d’options importantes :
- `withEnabledBlockingInitialNavigation()` : bloque le bootstrap jusqu’à la navigation initiale (utile SSR).
- `withInMemoryScrolling({ anchorScrolling, scrollPositionRestoration })`
- `withPreloading(PreloadAllModules)` ou stratégie custom
- `withRouterConfig({ paramsInheritanceStrategy, onSameUrlNavigation })`

Exemple :
```ts
import { provideRouter, withInMemoryScrolling, withRouterConfig } from '@angular/router';

provideRouter(routes,
  withInMemoryScrolling({
    scrollPositionRestoration: 'enabled',
    anchorScrolling: 'enabled'
  }),
  withRouterConfig({
    onSameUrlNavigation: 'reload',
    paramsInheritanceStrategy: 'always'
  })
)
```

---

# 3) Routes imbriquées & layouts

## 3.1 Pourquoi imbriquer des routes ?
- Réutiliser des **layouts** (barre latérale, header, zones dédiées).
- Composition hiérarchique : `/admin/users/:id`.
- Responsabilité : un parent gère le contexte, les enfants gèrent les écrans.

## 3.2 Exemple de layout
```ts
export const routes: Routes = [
  {
    path: 'admin',
    loadComponent: () => import('./layouts/admin/admin.layout').then(m => m.AdminLayout),
    children: [
      {
        path: 'users',
        loadComponent: () => import('./pages/admin/users/users.page').then(m => m.UsersPage),
      },
      {
        path: 'users/:id',
        loadComponent: () => import('./pages/admin/user-detail/user-detail.page').then(m => m.UserDetailPage),
      },
      { path: '', pathMatch: 'full', redirectTo: 'users' }
    ]
  }
];
```

Dans `AdminLayout` :
```html
<!-- admin.layout.html -->
<header>Admin</header>
<nav>…</nav>
<main>
  <router-outlet></router-outlet>
</main>
```

## 3.3 `router-outlet` multiple / named outlets (avancé)
Cas d’usage : afficher un panneau latéral, une modale router-driven, etc.

Configuration :
```ts
{
  path: 'mail',
  component: MailLayoutComponent,
  children: [
    { path: 'inbox', component: InboxComponent },
    { path: 'message/:id', component: MessageComponent, outlet: 'panel' }
  ]
}
```
Navigation :
```ts
this.router.navigate([{ outlets: { primary: ['mail', 'inbox'], panel: ['mail', 'message', 42] } }]);
```
> À utiliser avec parcimonie : complexifie URLs et lisibilité.

---

# 4) Routes paramétrées, query params & fragments

## 4.1 Paramètres de route (path params)
Exemple : `/products/42`
```ts
{ path: 'products/:id', component: ProductDetailPage }
```

Accès aux params :
```ts
import { ActivatedRoute } from '@angular/router';
import { map, switchMap } from 'rxjs/operators';

constructor(private route: ActivatedRoute, private api: ProductsApi) {}

ngOnInit() {
  this.route.paramMap
    .pipe(
      map(pm => pm.get('id')),
      switchMap(id => this.api.getById(id!))
    )
    .subscribe(product => this.product = product);
}
```

### Bon réflexe : gérer les changements de params
Si on reste sur le même composant mais que `:id` change, `ngOnInit` ne se relance pas (selon navigation). L’abonnement à `paramMap` reste la bonne pratique.

## 4.2 Query params
Exemple : `/products?category=books&page=2`
```ts
this.route.queryParamMap.subscribe(qp => {
  const category = qp.get('category');
  const page = Number(qp.get('page') ?? '1');
});
```

Navigation avec query params :
```ts
this.router.navigate(['products'], {
  queryParams: { category: 'books', page: 2 },
  queryParamsHandling: 'merge' // ou 'preserve'
});
```

## 4.3 Fragments (ancres)
Exemple : `/docs#installation`
```ts
this.router.navigate(['docs'], { fragment: 'installation' });
```
Avec `withInMemoryScrolling({ anchorScrolling: 'enabled' })`, Angular gère le scroll automatique.

---

# 5) Données de route (data) & titles

## 5.1 Le champ `data`
Permet d’associer des métadonnées :
- droits/roles attendus
- configuration UI (breadcrumb, icône)
- feature flags

Exemple :
```ts
{
  path: 'admin/users',
  component: UsersPage,
  data: {
    breadcrumb: 'Utilisateurs',
    requiredRoles: ['ADMIN']
  }
}
```

Récupération :
```ts
this.route.data.subscribe(d => {
  this.breadcrumb = d['breadcrumb'];
});
```

## 5.2 `title`
Angular supporte `title` dans la route, pouvant être une chaîne ou une fonction (selon version), et peut s’intégrer avec `TitleStrategy` pour un comportement global.

Exemple simple :
```ts
{ path: 'home', component: HomePage, title: 'Accueil' }
```

---

# 6) Redirections, routes de fallback & stratégie de matching

## 6.1 `redirectTo` et `pathMatch`
- `pathMatch: 'full'` : correspond à l’URL complète (souvent pour `''`).
- `pathMatch: 'prefix'` : par défaut, correspond au préfixe.

Exemple classique :
```ts
{ path: '', pathMatch: 'full', redirectTo: 'home' }
```

## 6.2 Wildcard `**`
Toujours en dernier :
```ts
{ path: '**', component: NotFoundPage }
```

## 6.3 Redirection conditionnelle (via guard)
Pour rediriger selon un état (auth/feature flag), utilisez plutôt un **guard** qui retourne un `UrlTree`.

---

# 7) Guards avancés

## 7.1 Panorama
- `CanActivate` : autoriser l’accès à une route.
- `CanActivateChild` : autoriser l’accès aux enfants.
- `CanDeactivate` : empêcher de quitter si modifications non sauvegardées.
- `CanMatch` : contrôler le **matching** (souvent pour lazy routes / A/B tests).

> Les guards peuvent retourner : `boolean | UrlTree | Observable<boolean|UrlTree> | Promise<...>`.

## 7.2 Auth guard (redirection vers login)
```ts
import { CanActivateFn, Router } from '@angular/router';
import { inject } from '@angular/core';

export const authGuard: CanActivateFn = () => {
  const router = inject(Router);
  const isLoggedIn = /* lire un AuthService */ false;

  return isLoggedIn ? true : router.createUrlTree(['/login'], {
    queryParams: { redirect: router.url }
  });
};
```

Usage :
```ts
{ path: 'admin', canActivate: [authGuard], loadComponent: ... }
```

## 7.3 Guard de rôles basé sur `data`
```ts
import { CanActivateFn, ActivatedRouteSnapshot, Router } from '@angular/router';
import { inject } from '@angular/core';

export const rolesGuard: CanActivateFn = (route: ActivatedRouteSnapshot) => {
  const router = inject(Router);
  const requiredRoles = route.data['requiredRoles'] as string[] | undefined;
  const userRoles = /* AuthService */ ['USER'];

  if (!requiredRoles || requiredRoles.every(r => userRoles.includes(r))) return true;
  return router.createUrlTree(['/forbidden']);
};
```

## 7.4 `CanDeactivate` (unsaved changes)
Contrat : le composant expose `canDeactivate()`.

```ts
export interface CanComponentDeactivate {
  canDeactivate: () => boolean | Promise<boolean>;
}
```

Guard :
```ts
import { CanDeactivateFn } from '@angular/router';

export const unsavedChangesGuard: CanDeactivateFn<CanComponentDeactivate> = (cmp) => {
  return cmp.canDeactivate();
};
```

Composant :
```ts
export class EditProfilePage implements CanComponentDeactivate {
  dirty = true;

  canDeactivate() {
    return !this.dirty || confirm('Vous avez des modifications non sauvegardées. Quitter ?');
  }
}
```

## 7.5 `CanMatch` (contrôle de matching)
Très utile pour :
- activer/désactiver une feature
- router vers une version v2/v1

```ts
import { CanMatchFn, Route, UrlSegment, Router } from '@angular/router';
import { inject } from '@angular/core';

export const featureFlagMatch: CanMatchFn = (route: Route, segments: UrlSegment[]) => {
  const enabled = /* FeatureFlags */ true;
  return enabled;
};
```

---

# 8) Resolvers (préchargement de données) & orchestration RxJS

## 8.1 Pourquoi un resolver ?
Un **resolver** charge les données **avant** l’activation de la route. Bénéfices :
- évite les écrans vides / loaders bricolés
- garantit la présence de données pour le composant
- centralise la gestion d’erreurs et redirections

## 8.2 Exemple de resolver
```ts
import { ResolveFn, ActivatedRouteSnapshot, Router } from '@angular/router';
import { inject } from '@angular/core';
import { catchError, of } from 'rxjs';

export const productResolver: ResolveFn<any> = (route: ActivatedRouteSnapshot) => {
  const api = inject(ProductsApi);
  const router = inject(Router);

  const id = route.paramMap.get('id')!;

  return api.getById(id).pipe(
    catchError(() => of(router.createUrlTree(['/products'])))
  );
};
```

Route :
```ts
{
  path: 'products/:id',
  component: ProductDetailPage,
  resolve: { product: productResolver }
}
```

Composant :
```ts
this.route.data.subscribe(d => {
  // d['product'] est soit le produit, soit un UrlTree si vous avez choisi ce pattern
  this.product = d['product'];
});
```

### Pattern recommandé
- Faire retourner au resolver *uniquement* des données.
- Gérer redirection/erreur via un guard ou via `catchError` qui renvoie une valeur fallback.

## 8.3 Résolvers multiples et dépendances
```ts
resolve: {
  product: productResolver,
  reviews: reviewsResolver
}
```
Angular attend que tout soit résolu avant activation. Attention à la latence cumulée.

---

# 9) Lazy loading moderne & architecture modulaire

## 9.1 Pourquoi le lazy loading ?
- Réduire le bundle initial.
- Charger une feature quand l’utilisateur en a besoin.
- Découpler l’architecture (features autonomes).

## 9.2 `loadChildren` (lazy routes)
```ts
{
  path: 'admin',
  loadChildren: () => import('./features/admin/admin.routes').then(m => m.ADMIN_ROUTES)
}
```

Dans `admin.routes.ts` :
```ts
import { Routes } from '@angular/router';

export const ADMIN_ROUTES: Routes = [
  {
    path: '',
    loadComponent: () => import('./admin.layout').then(m => m.AdminLayout),
    children: [
      {
        path: 'users',
        loadComponent: () => import('./pages/users.page').then(m => m.UsersPage)
      }
    ]
  }
];
```

## 9.3 `loadComponent` vs `loadChildren`
- `loadComponent` : lazy d’un composant standalone.
- `loadChildren` : lazy d’un **ensemble de routes** (feature).

Bonnes pratiques :
- `loadChildren` pour features complètes.
- `loadComponent` pour petites pages indépendantes.

---

# 10) Préchargement (PreloadingStrategy) & performance

## 10.1 Préchargement : principe
Après le chargement initial, Angular peut **précharger** les modules/features lazy en arrière-plan.

## 10.2 `PreloadAllModules`
Simple mais parfois trop agressif.

## 10.3 Stratégie de préchargement custom
Précharger uniquement si `data.preload === true`.

```ts
import { PreloadingStrategy, Route } from '@angular/router';
import { Observable, of } from 'rxjs';

export class SelectivePreloadStrategy implements PreloadingStrategy {
  preload(route: Route, load: () => Observable<any>): Observable<any> {
    return route.data?.['preload'] ? load() : of(null);
  }
}
```

Injection :
```ts
import { withPreloading } from '@angular/router';

provideRouter(routes,
  withPreloading(SelectivePreloadStrategy)
);
```

Route :
```ts
{ path: 'admin', data: { preload: true }, loadChildren: ... }
```

---

# 11) Cycle de navigation, événements du Router & debug

## 11.1 Étapes mental model
1. **Parsing URL**
2. **Matching** (route config)
3. Exécution des **guards** (canMatch/canActivate/…)
4. Exécution des **resolvers**
5. **Activation** (création composants, `router-outlet`)
6. Mise à jour URL/historique

## 11.2 Events du Router
Écoute des événements :
```ts
import { Router, NavigationStart, NavigationEnd, NavigationCancel, NavigationError } from '@angular/router';
import { filter } from 'rxjs/operators';

constructor(router: Router) {
  router.events
    .pipe(filter(e => e instanceof NavigationStart || e instanceof NavigationEnd || e instanceof NavigationError || e instanceof NavigationCancel))
    .subscribe(e => console.log(e));
}
```

## 11.3 Debug des routes
- Vérifier l’ordre des routes (spécifiques avant wildcard).
- Vérifier `pathMatch`.
- Vérifier la hiérarchie `children`.
- Utiliser `router.serializeUrl(router.createUrlTree(...))` pour inspecter.

---

# 12) Bonnes pratiques, patterns & atelier final

## 12.1 Patterns recommandés
- **Routes orientées features** : 1 dossier = 1 feature, fichiers `*.routes.ts`.
- **Layouts** pour mutualiser l’UI.
- **data** pour config (breadcrumb, roles, flags) plutôt que duplications.
- **Guards stateless** (functions) + services injectés.
- **Resolvers** pour données nécessaires à l’écran, pas pour tout.
- Favoriser la **composition RxJS** (switchMap, combineLatest) et gérer erreurs.

## 12.2 Anti-patterns
- Beaucoup de logique métier dans les guards/resolvers.
- Routes trop profondes / outlets nommés partout.
- Redirections en chaîne difficiles à comprendre.
- Mélanger composants page et composants UI (séparer `pages/` et `components/`).

## 12.3 Atelier final (fil rouge)
### Objectif
Construire une mini-application avec :
- Layout public + layout admin
- Auth guard + roles guard
- Résolver sur fiche produit
- Lazy loading de la feature admin
- Préchargement sélectif
- Gestion des scroll/anchors

### Conseils de réalisation
1. Créer `routes` racines avec redirections et 404.
2. Implémenter `AuthService` (mock), `authGuard` et `rolesGuard`.
3. Créer la feature `admin` en lazy avec `loadChildren`.
4. Ajouter `productResolver` sur `/products/:id`.
5. Ajouter la stratégie de préchargement.
6. Brancher un logger d’événements Router pour constater l’ordre d’exécution.

---

## Annexe A – Exemples de structure de projet

```
app/
  core/
    auth/
      auth.service.ts
      guards/
        auth.guard.ts
        roles.guard.ts
    routing/
      selective-preload.strategy.ts
  features/
    admin/
      admin.routes.ts
      admin.layout.ts
      pages/
        users.page.ts
        user-detail.page.ts
    products/
      products.routes.ts
      pages/
        product-detail.page.ts
      resolvers/
        product.resolver.ts
  pages/
    home/
    login/
    not-found/
  app.routes.ts
  app.component.ts
```

## Annexe B – Checklist de revue routing
- [ ] Les routes sont-elles groupées par feature ?
- [ ] `**` est-il bien en dernier ?
- [ ] Les redirections utilisent-elles `pathMatch: 'full'` quand nécessaire ?
- [ ] Les guards retournent-ils `UrlTree` pour rediriger (plutôt que `navigate`) ?
- [ ] Les resolvers gèrent-ils erreurs/timeout si besoin ?
- [ ] Lazy loading sur les features lourdes ?
- [ ] Préchargement contrôlé (data.preload) ?
- [ ] Scroll restoration et anchors configurés ?

---

**Fin de formation – Routing avancé Angular**
