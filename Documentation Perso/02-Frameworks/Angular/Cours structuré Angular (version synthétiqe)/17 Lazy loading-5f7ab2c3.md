# Formation Angular – 17. Lazy Loading

## Objectifs pédagogiques
À l’issue de cette formation, le participant sera capable de :

- Expliquer le principe du **lazy loading** et ses impacts sur les performances.
- Mettre en place le **lazy loading de modules** avec le Router Angular.
- Structurer une application en **modules (ou routes standalone)** pour optimiser le chargement.
- Diagnostiquer et vérifier le découpage en **chunks** (bundles) et le chargement à la demande.
- Éviter les pièges courants : préchargement involontaire, routes mal configurées, dépendances partagées, etc.

## Prérequis
- Connaissances solides en Angular (composants, services, modules, routing).
- Compréhension basique de TypeScript et des builds (Angular CLI).

## Public cible
- Développeurs Angular.
- Formateurs / référents techniques.

## Durée recommandée
- **2h à 3h** (selon profondeur de la partie debug/build).

---

## Plan de la formation

1. [Contexte & motivations](#1-contexte--motivations)
2. [Rappels sur le Routing Angular](#2-rappels-sur-le-routing-angular)
3. [Concepts clés du lazy loading](#3-concepts-clés-du-lazy-loading)
4. [Mise en œuvre : Lazy loading d’un module](#4-mise-en-œuvre--lazy-loading-dun-module)
5. [Structuration d’application : modularisation par domaine](#5-structuration-dapplication--modularisation-par-domaine)
6. [Préchargement (Preloading) : quand et comment](#6-préchargement-preloading--quand-et-comment)
7. [Lazy loading avec des routes standalone (Angular moderne)](#7-lazy-loading-avec-des-routes-standalone-angular-moderne)
8. [Mesurer et vérifier le résultat (bundles, chunks, network)](#8-mesurer-et-vérifier-le-résultat-bundles-chunks-network)
9. [Pièges fréquents & bonnes pratiques](#9-pièges-fréquents--bonnes-pratiques)
10. [Atelier guidé (exercices)](#10-atelier-guidé-exercices)
11. [Résumé & checklist](#11-résumé--checklist)

---

## 1. Contexte & motivations

### 1.1. Problème : application monolithique
Dans une application Angular, si toutes les fonctionnalités sont emballées dans le **bundle initial**, l’utilisateur doit télécharger un volume important de JavaScript au premier affichage.

Conséquences :

- **Temps de chargement initial** plus long (TTFB/TTI/LCP dégradés).
- **Coût réseau** plus élevé sur mobile.
- Expérience utilisateur moins fluide, surtout sur des appareils modestes.

### 1.2. Solution : Lazy loading
Le **lazy loading** consiste à **charger un module (ou un ensemble de routes)** uniquement lorsque l’utilisateur navigue vers une section donnée.

En pratique :

- Le bundle initial est plus léger.
- Les fonctionnalités “secondaires” (admin, back-office, reporting, etc.) sont téléchargées **à la demande**.

---

## 2. Rappels sur le Routing Angular

### 2.1. Éléments essentiels
- `RouterModule.forRoot(routes)` : configuration globale.
- `RouterModule.forChild(routes)` : configuration dans un module de feature.
- `<router-outlet>` : point d’insertion de la vue.
- `Routes` : tableau d’objets route.

Exemple minimal :

```ts
import { Routes } from '@angular/router';

export const routes: Routes = [
  { path: '', component: HomeComponent },
  { path: 'about', component: AboutComponent },
];
```

### 2.2. Routage par feature
Une application bien structurée sépare souvent les routes en “features” :

- `Home` (public)
- `Auth`
- `Admin`
- `Settings`

Le lazy loading devient alors un choix naturel.

---

## 3. Concepts clés du lazy loading

### 3.1. Module eager vs lazy
- **Eager loaded** : chargé dès le démarrage (dans le bundle initial).
- **Lazy loaded** : chargé quand l’utilisateur active une route donnée.

### 3.2. “Chunk” (bundle fractionné)
Lors du build, Angular/webpack génère des fichiers supplémentaires (chunks) correspondant aux features lazy.

### 3.3. Limites à connaître
- Le lazy loading améliore surtout le **chargement initial**.
- Une feature lazy peut induire un **temps de chargement au moment de la navigation** (d’où l’intérêt éventuel du preloading).

---

## 4. Mise en œuvre : Lazy loading d’un module

### 4.1. Exemple de structure
Supposons une app avec :

- `AppModule`
- `ProductsModule` (à charger à la demande)

Arborescence :

```
src/app/
  app-routing.module.ts
  app.module.ts
  products/
    products.module.ts
    products-routing.module.ts
    pages/
      product-list.component.ts
      product-detail.component.ts
```

### 4.2. Déclarer une route lazy dans le routing principal
Dans `app-routing.module.ts` :

```ts
import { NgModule } from '@angular/core';
import { RouterModule, Routes } from '@angular/router';

const routes: Routes = [
  {
    path: 'products',
    loadChildren: () => import('./products/products.module')
      .then(m => m.ProductsModule)
  },
  { path: '', redirectTo: 'products', pathMatch: 'full' },
  { path: '**', redirectTo: 'products' }
];

@NgModule({
  imports: [RouterModule.forRoot(routes)],
  exports: [RouterModule]
})
export class AppRoutingModule {}
```

#### Ce que fait Angular ici
- L’import est **dynamique** : le code `products.module` n’est pas inclus dans le bundle initial.
- Lorsqu’un utilisateur navigue vers `/products`, Angular télécharge le chunk correspondant.

### 4.3. Routing interne du module lazy
`products-routing.module.ts` :

```ts
import { NgModule } from '@angular/core';
import { RouterModule, Routes } from '@angular/router';
import { ProductListComponent } from './pages/product-list.component';
import { ProductDetailComponent } from './pages/product-detail.component';

const routes: Routes = [
  { path: '', component: ProductListComponent },
  { path: ':id', component: ProductDetailComponent }
];

@NgModule({
  imports: [RouterModule.forChild(routes)],
  exports: [RouterModule]
})
export class ProductsRoutingModule {}
```

`products.module.ts` :

```ts
import { NgModule } from '@angular/core';
import { CommonModule } from '@angular/common';
import { ProductsRoutingModule } from './products-routing.module';
import { ProductListComponent } from './pages/product-list.component';
import { ProductDetailComponent } from './pages/product-detail.component';

@NgModule({
  declarations: [
    ProductListComponent,
    ProductDetailComponent
  ],
  imports: [
    CommonModule,
    ProductsRoutingModule
  ]
})
export class ProductsModule {}
```

### 4.4. Points de contrôle rapides
- Le module `ProductsModule` **ne doit pas** être importé dans `AppModule`.
- Le module lazy doit utiliser `RouterModule.forChild`.

---

## 5. Structuration d’application : modularisation par domaine

### 5.1. Bon découpage
Découper en features cohérentes :

- `AuthModule` (login/inscription)
- `AdminModule` (back-office)
- `BillingModule` (paiements)

### 5.2. SharedModule et CoreModule
Pour éviter les duplications et contrôler ce qui va dans le bundle initial :

- **CoreModule** : services singleton, interceptors, guards globaux, layout principal.
- **SharedModule** : composants/pipes/directives réutilisables (attention à ne pas y mettre des providers involontaires).

> Conseil : privilégier des imports “légers” dans les modules lazy pour limiter la taille de leurs chunks.

---

## 6. Préchargement (Preloading) : quand et comment

### 6.1. Pourquoi précharger ?
Le lazy loading peut créer une latence au premier accès d’une feature. Le **preloading** télécharge en arrière-plan des modules lazy après le chargement initial.

### 6.2. Stratégies intégrées
- `NoPreloading` (par défaut)
- `PreloadAllModules`

Activation :

```ts
import { NgModule } from '@angular/core';
import { RouterModule } from '@angular/router';
import { PreloadAllModules } from '@angular/router';
import { routes } from './app.routes';

@NgModule({
  imports: [
    RouterModule.forRoot(routes, { preloadingStrategy: PreloadAllModules })
  ],
  exports: [RouterModule]
})
export class AppRoutingModule {}
```

### 6.3. Préchargement sélectif (approche recommandée)
Idée : ne précharger que certaines routes (ex. admin pour un utilisateur admin).

On peut :
- créer une stratégie custom (`PreloadingStrategy`).
- prendre en compte un `data: { preload: true }` dans les routes.

---

## 7. Lazy loading avec des routes standalone (Angular moderne)

> Angular permet aujourd’hui de construire des apps sans NgModules (ou avec un mix). Le lazy loading reste possible via `loadComponent` et `loadChildren` avec routes.

### 7.1. Lazy load d’un ensemble de routes
Exemple `app.routes.ts` :

```ts
import { Routes } from '@angular/router';

export const routes: Routes = [
  {
    path: 'products',
    loadChildren: () => import('./products/products.routes')
      .then(r => r.PRODUCTS_ROUTES)
  }
];
```

`products.routes.ts` :

```ts
import { Routes } from '@angular/router';

export const PRODUCTS_ROUTES: Routes = [
  {
    path: '',
    loadComponent: () => import('./pages/product-list.component')
      .then(c => c.ProductListComponent)
  },
  {
    path: ':id',
    loadComponent: () => import('./pages/product-detail.component')
      .then(c => c.ProductDetailComponent)
  }
];
```

### 7.2. Quand choisir modules vs standalone ?
- Modules : utile si base existante, conventions d’équipe, encapsulation.
- Standalone : réduction de boilerplate, configuration plus directe.

---

## 8. Mesurer et vérifier le résultat (bundles, chunks, network)

### 8.1. Vérifier via le build
Commande :

```bash
ng build --configuration production
```

Dans `dist/`, on observe :
- `main.*.js` (bundle principal)
- fichiers “chunk” nommés ou numérotés (features lazy)

### 8.2. Analyser la taille des bundles
Options utiles :

```bash
ng build --stats-json
```

Puis analyse via un outil type `webpack-bundle-analyzer`.

### 8.3. Vérifier dans le navigateur
Dans Chrome DevTools :

- onglet **Network** : filtrer par “JS”, observer le téléchargement d’un chunk lorsque vous naviguez vers `/products`.
- onglet **Performance** ou **Lighthouse** : mesurer l’amélioration du chargement initial.

---

## 9. Pièges fréquents & bonnes pratiques

### 9.1. Import direct du module lazy
Erreur classique : importer `ProductsModule` dans `AppModule`.

Conséquence : la feature devient **eager** et perd l’intérêt du lazy loading.

### 9.2. SharedModule trop “gros”
Si `SharedModule` importe des libs lourdes (charts, editors) et qu’il est inclus partout, il peut :
- gonfler le bundle initial
- ou gonfler les chunks lazy inutilement

Bonne pratique : créer des `UiChartsModule`, `EditorModule`, etc. et les importer uniquement là où c’est nécessaire.

### 9.3. Services et providers
- Éviter de déclarer des `providers` dans `SharedModule`.
- Préférer `providedIn: 'root'` pour les singletons globaux.
- Pour un service “scopé feature” (un état spécifique), on peut le fournir au niveau du module lazy ou route-level provider (standalone).

### 9.4. Guards et lazy loading
Les guards (`canActivate`, `canMatch`) peuvent empêcher le chargement ou la navigation.

- `canMatch` est particulièrement intéressant pour éviter même la résolution de la route si l’accès n’est pas autorisé.

### 9.5. UX : état de chargement
Lors du chargement d’un module lazy, prévoir :
- un skeleton
- un loader discret
- une gestion d’erreurs de chargement réseau

---

## 10. Atelier guidé (exercices)

### Exercice 1 — Transformer une feature eager en lazy
1. Identifier une section (ex. `Admin`).
2. Créer `AdminModule` + `AdminRoutingModule`.
3. Déplacer les composants admin dedans.
4. Remplacer la route `component: AdminLayoutComponent` par un `loadChildren`.
5. Vérifier dans DevTools que le chunk admin n’est chargé qu’à la navigation.

### Exercice 2 — Mettre en place `PreloadAllModules`
1. Activer la stratégie de preloading.
2. Observer l’impact : le chunk se charge après le bootstrap (Network).
3. Discuter : bénéfices vs coût réseau.

### Exercice 3 — Préchargement sélectif
1. Ajouter `data: { preload: true }` sur une route.
2. Créer une stratégie de preloading custom.
3. Précharger uniquement cette route.

---

## 11. Résumé & checklist

### À retenir
- Le lazy loading **réduit** le bundle initial et améliore le **chargement initial**.
- Il se fait principalement via `loadChildren` (modules ou routes) et/ou `loadComponent`.
- L’efficacité se mesure : **build**, **network**, **Lighthouse**.

### Checklist de mise en place
- [ ] La feature est isolée (module ou routes dédiées).
- [ ] Elle n’est pas importée directement dans `AppModule`.
- [ ] Route principale configurée avec `loadChildren`.
- [ ] `forChild` côté feature (si modules).
- [ ] Vérification en build et en DevTools.
- [ ] Stratégie de preloading adaptée au contexte.

---

## Annexes (facultatif)

### Glossaire
- **Lazy loading** : chargement à la demande.
- **Chunk** : fragment de bundle généré au build.
- **Preloading** : chargement anticipé en arrière-plan après le chargement initial.

### Références
- Documentation Angular Router : https://angular.dev/guide/routing
- Guide sur le lazy loading : https://angular.dev/guide/routing/common-router-tasks#lazy-loading
