# Formation Angular — Resolvers et préchargement des données

**Référence** : 23

## Objectifs pédagogiques

À l’issue de cette formation, vous serez capable de :

- Expliquer le rôle des **Resolvers** dans Angular Router et quand les utiliser.
- Mettre en place un Resolver moderne (Angular >= 15) basé sur `ResolveFn` et de l’injection via `inject()`.
- Gérer **chargement**, **erreurs**, **annulation de navigation** et **redirections**.
- Comparer Resolver vs chargement dans le composant vs guards vs préchargement.
- Mettre en place un **préchargement** (preloading) des modules/lazy routes (stratégies Angular et stratégie custom).
- Concevoir une stratégie pragmatique pour éviter d’allonger inutilement la navigation.

---

## Pré-requis

- Maîtriser les bases d’Angular (components, services, DI, RxJS).
- Connaître les fondamentaux du router (`Routes`, `routerLink`, `ActivatedRoute`).

---

## Plan de la formation

1. Contexte : navigation et dépendance aux données
2. Résolveurs : principe, cycle de vie et effets UX
3. Mise en place : Resolver de base (API HTTP)
4. Exploiter les données résolues dans les composants
5. Gestion avancée : erreurs, redirections, timeouts, retries
6. Patterns d’architecture : cache, partage, SSR, performance
7. Resolver vs alternatives (chargement composant, stores, guards)
8. Préchargement Angular : `PreloadAllModules`, `NoPreloading`
9. Stratégie de préchargement personnalisée
10. Atelier guidé : liste & détail (critique) + préchargement (confort)
11. Checklist et bonnes pratiques

---

# 1) Contexte : navigation et dépendance aux données

Dans beaucoup d’applications Angular, une route correspond à une **page** dont le rendu dépend de données.

- Exemple : une page *Détail Produit* dépend du produit, des stocks, du prix, des avis…
- Sans données, la page est *peu utile* ou impose des états UI complexes : skeletons, placeholders, erreurs.

**Idée centrale des resolvers** :

> Charger les données *avant* d’activer la route, afin que le composant se crée avec des données déjà prêtes.

### Pourquoi c’est utile ?

- Évite un composant affiché « vide » puis mis à jour.
- Simplifie le code du composant (moins de logique de chargement/erreur).
- Centralise la logique de récupération dépendante de la route.

### Pourquoi l’utiliser « avec discernement » ?

- Parce que le resolver **bloque la navigation** jusqu’à la fin du chargement.
- Donc un resolver très lent ou trop ambitieux peut
  - rendre la navigation « lourde »,
  - augmenter les abandons,
  - donner l’impression que l’app est figée.

---

# 2) Résolveurs : principe, cycle de vie et effets UX

## 2.1 Définition

Un **resolver** est une fonction (ou classe) associée à une route, exécutée **avant l’activation** du composant de la route. Il retourne une valeur (synchrone) ou un `Observable` / `Promise`.

Angular attend la résolution de toutes les entrées `resolve` avant :

- de finaliser la navigation,
- d’instancier le composant route-cible,
- d’émettre `ActivatedRoute.data` sur la route activée.

## 2.2 Où se situe le resolver dans le pipeline du router ?

Ordre simplifié (selon configuration) :

1. Matching de route
2. Exécution des **guards** (CanMatch/CanActivate…)
3. Exécution des **resolvers**
4. Activation de la route et instanciation composant

**Conséquence** : un guard peut empêcher une navigation *avant même* de charger les données ; un resolver charge les données *avant de rendre la page*.

## 2.3 UX : naviguer sans « écran blanc »

Si vous utilisez des resolvers, prévoyez une UI de transition :

- barre de progression globale,
- overlay « loading »,
- skeleton via un composant parent,
- gestion des erreurs de navigation.

---

# 3) Mise en place : Resolver de base (API HTTP)

On va construire un exemple pédagogique :

- `/products` : liste
- `/products/:id` : détail produit (données critiques)

## 3.1 Service API

```ts
// product.api.ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';

export interface Product {
  id: string;
  name: string;
  price: number;
}

@Injectable({ providedIn: 'root' })
export class ProductApi {
  constructor(private http: HttpClient) {}

  getProduct(id: string): Observable<Product> {
    return this.http.get<Product>(`/api/products/${id}`);
  }
}
```

## 3.2 Resolver moderne (fonction) — `ResolveFn`

Angular (>= 15) encourage l’approche fonctionnelle :

```ts
// product.resolver.ts
import { ResolveFn, ActivatedRouteSnapshot, RouterStateSnapshot, Router } from '@angular/router';
import { inject } from '@angular/core';
import { Product, ProductApi } from './product.api';
import { catchError, throwError } from 'rxjs';

export const productResolver: ResolveFn<Product> = (
  route: ActivatedRouteSnapshot,
  state: RouterStateSnapshot
) => {
  const api = inject(ProductApi);
  const router = inject(Router);

  const id = route.paramMap.get('id');
  if (!id) {
    // Paramètre absent => on peut rediriger
    router.navigateByUrl('/products');
    // ou lever une erreur
    return throwError(() => new Error('Missing product id'));
  }

  return api.getProduct(id).pipe(
    catchError((err) => {
      // Exemple de stratégie : redirection vers une page d’erreur
      router.navigate(['/error'], { queryParams: { from: state.url } });
      return throwError(() => err);
    })
  );
};
```

Points clés :

- `inject()` permet d’injecter des services dans une fonction.
- Le resolver **retourne un Observable** ; Angular attend la complétion.
- La gestion d’erreur doit être pensée : redirection, message global, log…

## 3.3 Configuration de route avec `resolve`

```ts
// app.routes.ts
import { Routes } from '@angular/router';
import { productResolver } from './product.resolver';

export const routes: Routes = [
  {
    path: 'products',
    loadComponent: () => import('./products/products.page').then(m => m.ProductsPage)
  },
  {
    path: 'products/:id',
    loadComponent: () => import('./product-detail/product-detail.page').then(m => m.ProductDetailPage),
    resolve: {
      product: productResolver
    }
  }
];
```

Ici, la clé `product` devient un champ dans `ActivatedRoute.data`.

---

# 4) Exploiter les données résolues dans les composants

## 4.1 Lecture via `ActivatedRoute.data`

```ts
// product-detail.page.ts
import { Component, inject } from '@angular/core';
import { ActivatedRoute } from '@angular/router';
import { map } from 'rxjs';
import { Product } from '../product.api';

@Component({
  standalone: true,
  selector: 'app-product-detail',
  template: `
    <section *ngIf="product$ | async as product">
      <h1>{{ product.name }}</h1>
      <p>Prix : {{ product.price | number:'1.2-2' }} €</p>
    </section>
  `
})
export class ProductDetailPage {
  private route = inject(ActivatedRoute);

  product$ = this.route.data.pipe(map(d => d['product'] as Product));
}
```

Avantages :

- le composant ne gère pas l’appel API (ou beaucoup moins),
- vous standardisez l’approvisionnement en données.

## 4.2 Typage : sécuriser `data`

Astuce : créer une interface pour `data` :

```ts
interface ProductDetailRouteData {
  product: Product;
}

product$ = this.route.data.pipe(map(d => (d as ProductDetailRouteData).product));
```

---

# 5) Gestion avancée : erreurs, redirections, timeouts, retries

## 5.1 Ne pas bloquer indéfiniment : `timeout`

Si votre backend peut être lent, vous pouvez limiter :

```ts
import { timeout, catchError, throwError } from 'rxjs';

return api.getProduct(id).pipe(
  timeout({ first: 5000 }),
  catchError(err => {
    router.navigate(['/error'], { queryParams: { reason: 'timeout' } });
    return throwError(() => err);
  })
);
```

## 5.2 Réessayer intelligemment : `retry`

```ts
import { retry, catchError } from 'rxjs';

return api.getProduct(id).pipe(
  retry({ count: 2, delay: 300 }),
  catchError(err => {
    router.navigate(['/error']);
    return throwError(() => err);
  })
);
```

## 5.3 Stratégies d’erreur possibles

- **Bloquante** : la navigation échoue et l’utilisateur reste sur la page précédente.
- **Redirection** : vers `/error` ou `/products`.
- **Fallback** : retourner des données « par défaut » (souvent déconseillé si la page dépend fortement de données critiques).

À retenir :

> Une page « critique » justifie un resolver ; une page « confort » préfère souvent du chargement côté composant avec skeletons.

---

# 6) Patterns d’architecture : cache, partage, SSR, performance

## 6.1 Éviter les appels répétés : cache côté service

Quand l’utilisateur navigue avant/arrière, la route peut se réactiver.

Pattern courant : mémoriser les entités déjà chargées :

```ts
// product.store-ish.service.ts
import { Injectable } from '@angular/core';
import { Observable, shareReplay } from 'rxjs';
import { Product, ProductApi } from './product.api';

@Injectable({ providedIn: 'root' })
export class ProductRepository {
  private cache = new Map<string, Observable<Product>>();

  constructor(private api: ProductApi) {}

  getProduct(id: string): Observable<Product> {
    if (!this.cache.has(id)) {
      this.cache.set(id, this.api.getProduct(id).pipe(shareReplay(1)));
    }
    return this.cache.get(id)!;
  }
}
```

Puis dans le resolver, utiliser `ProductRepository`.

## 6.2 Navigations successives : attention aux surcharges

- Si votre resolver charge 4 APIs et attend tout, vous multipliez les risques de latence.
- Préférez :
  - une API serveur agrégée,
  - ou un « minimum vital » en resolver + le reste en lazy (dans composant).

## 6.3 SSR / Hydration

En SSR, les resolvers peuvent aider à fournir des données au rendu serveur. Selon votre stack, combinez :

- TransferState,
- caches,
- et une API idempotente.

---

# 7) Resolver vs alternatives

## 7.1 Charger dans le composant (approche classique)

**Pour** :

- navigation rapide,
- affichage immédiat avec skeleton,
- plus flexible pour des pages non critiques.

**Contre** :

- composants plus complexes (chargement/erreurs),
- clignotement possible,
- risque de duplication de logique.

## 7.2 Guard vs Resolver

- **Guard** : décide si on a le droit d’aller sur la route.
- **Resolver** : fournit des données nécessaires à la route.

Ne pas surcharger un guard avec du chargement (même si techniquement possible) : séparez les responsabilités.

## 7.3 Store global (NgRx, Signals store, etc.)

Alternative : déclencher un chargement sur navigation, et laisser le composant lire dans le store.

- très utile à grande échelle,
- mais un resolver reste pertinent si vous voulez une **garantie de disponibilité** avant écran.

---

# 8) Préchargement Angular : `PreloadAllModules`, `NoPreloading`

À ne pas confondre :

- **Resolver** : précharge des *données* avant activation d’une route.
- **PreloadingStrategy** : précharge des *bundles* (modules lazy) après le chargement initial, en arrière-plan.

## 8.1 Activer le préchargement

Avec le router moderne :

```ts
import { provideRouter, withPreloading, PreloadAllModules } from '@angular/router';

bootstrapApplication(AppComponent, {
  providers: [
    provideRouter(routes, withPreloading(PreloadAllModules))
  ]
});
```

- `PreloadAllModules` : précharge toutes les routes lazy dès que possible.
- `NoPreloading` (par défaut) : ne précharge rien.

Quand l’utiliser ?

- SPAs dont les utilisateurs naviguent beaucoup.
- Connexions rapides.

Quand l’éviter ?

- mobiles/lenteurs réseau, restrictions data.
- apps où l’utilisateur ne visitera qu’une petite partie.

---

# 9) Stratégie de préchargement personnalisée

Objectif : précharger uniquement certaines routes, via `data.preload = true`.

## 9.1 Marquer les routes

```ts
export const routes: Routes = [
  {
    path: 'admin',
    data: { preload: true },
    loadChildren: () => import('./admin/admin.routes').then(m => m.ADMIN_ROUTES)
  },
  {
    path: 'help',
    data: { preload: false },
    loadChildren: () => import('./help/help.routes').then(m => m.HELP_ROUTES)
  }
];
```

## 9.2 Implémenter `PreloadingStrategy`

```ts
import { Injectable } from '@angular/core';
import { PreloadingStrategy, Route } from '@angular/router';
import { Observable, of } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class SelectivePreloadingStrategy implements PreloadingStrategy {
  preload(route: Route, load: () => Observable<any>): Observable<any> {
    const shouldPreload = route.data?.['preload'] === true;
    return shouldPreload ? load() : of(null);
  }
}
```

## 9.3 Brancher la stratégie

```ts
import { provideRouter, withPreloading } from '@angular/router';
import { SelectivePreloadingStrategy } from './selective-preloading.strategy';

bootstrapApplication(AppComponent, {
  providers: [
    provideRouter(routes, withPreloading(SelectivePreloadingStrategy))
  ]
});
```

---

# 10) Atelier guidé : route critique + préchargement confort

## Scénario

- La page **détail** produit dépend fortement des données : elle doit s’afficher « prête ».
- La zone **admin** est rarement visitée, mais quand l’utilisateur clique, on veut que ce soit fluide.

### Étapes

1. Créer `ProductApi` et afficher un détail via `ActivatedRoute.data`.
2. Ajouter `productResolver` à la route `/products/:id`.
3. Ajouter une page `/error` simple.
4. Rajouter `timeout` + redirection en cas d’échec.
5. Mettre en place `SelectivePreloadingStrategy`.
6. Marquer la route admin en `data.preload = true`.

## Critères de réussite

- Navigation vers `/products/:id` : le composant n’apparaît qu’avec la donnée.
- Erreur API : redirection vers `/error`.
- Route admin : bundle téléchargé après le chargement initial (observé via DevTools Network).

---

# 11) Checklist et bonnes pratiques

## Resolvers — recommandations

- Utilisez un resolver quand :
  - la page est **inutilisable** sans données,
  - vous voulez éviter un rendu partiel,
  - vous avez besoin de **contrats** de données pour la page.

- Évitez de :
  - charger trop de choses « au cas où »,
  - synchroniser plusieurs appels lents séparés côté client,
  - masquer les erreurs (mieux : redirection + message explicite).

- Mettez en place :
  - un indicateur global de navigation/chargement,
  - une stratégie de timeout et gestion d’erreur,
  - du cache (si pertinent) pour éviter les appels répétés.

## Préchargement — recommandations

- `PreloadAllModules` : simple mais potentiellement coûteux.
- Préchargement sélectif : généralement la meilleure approche.
- Basez le préchargement sur :
  - fréquence d’usage,
  - coûts réseau,
  - segmentation par profil (admin vs user).

---

## Annexes

### A) Resolver « class-based » (legacy)

Angular supporte aussi les classes :

```ts
import { Injectable } from '@angular/core';
import { Resolve, ActivatedRouteSnapshot, Router } from '@angular/router';
import { Observable } from 'rxjs';
import { Product, ProductApi } from './product.api';

@Injectable({ providedIn: 'root' })
export class ProductResolver implements Resolve<Product> {
  constructor(private api: ProductApi, private router: Router) {}

  resolve(route: ActivatedRouteSnapshot): Observable<Product> {
    const id = route.paramMap.get('id')!;
    return this.api.getProduct(id);
  }
}
```

### B) Debug : observer les évènements du router

```ts
import { Router, NavigationStart, ResolveStart, ResolveEnd, NavigationEnd } from '@angular/router';
import { filter } from 'rxjs';

router.events.pipe(
  filter(e => e instanceof NavigationStart || e instanceof ResolveStart || e instanceof ResolveEnd || e instanceof NavigationEnd)
).subscribe(console.log);
```

---

## Message clé

Les **Resolvers** chargent des données **avant** l’activation d’une route : parfait pour les pages à dépendance critique, mais à utiliser avec discernement car ils peuvent rendre la navigation plus lente. Le **préchargement** (`PreloadingStrategy`) optimise la perception de performance en chargeant des bundles lazy en arrière-plan : combinez les deux pour une UX rapide et fiable.
