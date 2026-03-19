# Formation Angular – Organisation par domaines fonctionnels (Feature Domains)

**Niveau :** Intermédiaire → Avancé  
**Public :** Développeurs Angular, lead devs, formateurs, équipes produit/tech  
**Durée indicative :** 1 journée (6–7h) ou 2 demi-journées  
**Pré-requis :** Angular CLI, modules/standalone, DI, routing, RxJS, TypeScript, tests unitaires  

---

## Objectifs pédagogiques

À l’issue de cette formation, vous saurez :

- Concevoir une **arborescence Angular scalable** basée sur des **domaines fonctionnels (features métiers)**.
- Distinguer et structurer les zones **core**, **shared** et **features** (auth, users, orders, admin, …).
- Organiser le **routing** (lazy loading, guards, resolvers) par domaine.
- Définir des frontières claires (API de domaine, encapsulation, dépendances autorisées).
- Favoriser la **maintenance**, la **collaboration en équipe** et l’**évolution indépendante** des domaines.
- Mettre en place des conventions, des règles (lint, eslint boundaries), et des pratiques de tests.

---

## Plan de formation

1. **Pourquoi organiser par domaines fonctionnels ?**
2. **Les grands blocs : core, shared, features**
3. **Exemple d’arborescence cible (auth, users, orders, admin)**
4. **Règles de dépendances et frontières entre domaines**
5. **Routing par feature : lazy loading, guards et shell**
6. **État et services : où placer quoi (data-access, ui, domain)**
7. **Conventions de nommage et API publique d’un domaine**
8. **Approche Angular moderne : standalone + lazy routes**
9. **Tests, qualité, et évolutivité (refactoring, extraction de libs)**
10. **Ateliers pratiques et checklists**

---

# 1) Pourquoi organiser par domaines fonctionnels ?

Une application Angular « avancée » grossit souvent sur plusieurs axes :

- **Fonctionnel** : ajout de nouvelles fonctionnalités (commande, paiement, support…)
- **Équipe** : plusieurs développeurs en parallèle
- **Technique** : multiples sources de données, caches, auth, i18n, design system

Une organisation par domaines fonctionnels (feature domains) consiste à regrouper le code **par “métier”**, plutôt que par type technique (components/services) à l’échelle de toute l’application.

### Bénéfices concrets

- **Maintenance** : on retrouve le code d’une feature au même endroit.
- **Cohérence** : limites claires, moins de “spaghetti imports”.
- **Scalabilité** : ajouter/retirer un domaine devient simple.
- **Travail en équipe** : chaque équipe/feature peut évoluer avec un minimum de conflits.
- **Lazy loading** : chargement à la demande par feature.

### Symptômes d’une mauvaise organisation (anti-patterns)

- Un `components/` global qui contient des dizaines de composants de tous les sujets.
- Une explosion des imports croisés entre features.
- Des services “fourre-tout” (`app.service.ts`) utilisés partout.
- Chaque modification casse autre chose car tout dépend de tout.

---

# 2) Les grands blocs : `core`, `shared`, `features`

## 2.1 `core` : l’infrastructure applicative

Le dossier `core` contient ce qui est **singleton** et transversal :

- Auth globale (interceptors, guards global, token storage)
- Configuration d’app (`environment`, injection tokens)
- Services “platform” : logger, error handler, monitoring, i18n setup
- Layout/shell global (si partagé par toutes les pages)

**Règle d’or :** `core` est utilisé par tout le monde, mais lui ne dépend pas des features.

## 2.2 `shared` : les briques réutilisables

Le dossier `shared` contient des éléments **réutilisables** sans logique métier spécifique :

- Composants UI génériques (button, modal, table, form controls…)
- Pipes, directives
- Helpers de présentation

**Attention :** `shared` ne doit pas contenir de code dépendant d’un domaine (pas de `OrdersService` dans `shared`).

## 2.3 `features` : les domaines métiers

Chaque domaine (feature) regroupe :

- Routes et pages
- Composants, forms
- Services, facades, data-access
- Modèles et règles métier

Exemples :

- `auth` : login, refresh token, register
- `users` : gestion profil, liste, détail
- `orders` : panier, paiement, historique commandes
- `admin` : gestion back-office, accès restreints

---

# 3) Arborescence cible (exemple)

Ci-dessous une arborescence **typique** qui fonctionne bien dans un mono-repo Angular (sans Nx) et s’adapte aux projets standalone.

```txt
src/
  app/
    core/
      config/
      interceptors/
      guards/
      services/
      core.providers.ts
    shared/
      ui/
      directives/
      pipes/
      utils/
      shared.module.ts (si vous êtes en approche NgModules)
    features/
      auth/
        api/
        data-access/
        ui/
        pages/
        auth.routes.ts
        auth.facade.ts
        auth.models.ts
      users/
        api/
        data-access/
        ui/
        pages/
        users.routes.ts
        users.facade.ts
        users.models.ts
      orders/
        api/
        data-access/
        ui/
        pages/
        orders.routes.ts
        orders.facade.ts
        orders.models.ts
      admin/
        api/
        data-access/
        ui/
        pages/
        admin.routes.ts
        admin.facade.ts
        admin.models.ts
    app.routes.ts
    app.component.ts
  main.ts
```

## Lecture rapide

- `pages/` : composants liés à un route (smart components)
- `ui/` : composants de présentation (dumb/presentational)
- `data-access/` : accès aux données (services http, store, caches)
- `api/` : couche d’intégration (clients d’API, DTO, mapping)
- `*.routes.ts` : définition des routes lazy du domaine
- `*.facade.ts` : façade “entrée unique” du domaine côté UI
- `*.models.ts` : modèles métier, types, interfaces

---

# 4) Frontières et règles de dépendances

L’objectif est d’éviter les dépendances circulaires et la pollution du code.

## 4.1 Règles simples (recommandées)

1. **Une feature ne dépend jamais d’une autre feature** (ou très rarement, via une API stable).
2. `shared` ne dépend d’aucune feature.
3. `core` ne dépend d’aucune feature.
4. Les pages d’un domaine n’importent pas directement des détails internes d’un autre domaine.

### Stratégies pour partager entre features

- Utiliser `shared` pour les composants UI génériques.
- Utiliser `core` pour les services transverses.
- Si deux features partagent un concept métier, créer :
  - soit une mini “lib” interne (dans `shared/domain` par exemple)
  - soit un **contrat** (types/interfaces) partagé et stable.

## 4.2 Encapsulation : API publique d’un domaine

Même dans un simple projet Angular, vous pouvez simuler une API publique.

### Exemple

- Tout ce qui est **public** dans `features/orders/` est importable via un point d’entrée (barrel).
- Tout ce qui est interne est gardé dans des sous-dossiers non exposés.

Exemple de fichier `features/orders/index.ts`:

```ts
export * from './orders.routes';
export * from './orders.models';
export * from './orders.facade';
```

**But :** les autres parties de l’app n’importent pas `./features/orders/data-access/private.service`.

---

# 5) Routing par feature : lazy loading, guards, shell

## 5.1 Routes racines

En Angular moderne (standalone), vous définissez les routes dans `app.routes.ts`.

```ts
// app/app.routes.ts
import { Routes } from '@angular/router';

export const APP_ROUTES: Routes = [
  {
    path: 'auth',
    loadChildren: () =>
      import('./features/auth/auth.routes').then(m => m.AUTH_ROUTES),
  },
  {
    path: 'users',
    loadChildren: () =>
      import('./features/users/users.routes').then(m => m.USERS_ROUTES),
  },
  {
    path: 'orders',
    loadChildren: () =>
      import('./features/orders/orders.routes').then(m => m.ORDERS_ROUTES),
  },
  {
    path: 'admin',
    loadChildren: () =>
      import('./features/admin/admin.routes').then(m => m.ADMIN_ROUTES),
  },
  { path: '', redirectTo: 'orders', pathMatch: 'full' },
  { path: '**', redirectTo: 'orders' },
];
```

## 5.2 Routes dans un domaine

Exemple `features/orders/orders.routes.ts` :

```ts
import { Routes } from '@angular/router';
import { OrdersPageComponent } from './pages/orders-page.component';
import { OrderDetailsPageComponent } from './pages/order-details-page.component';

export const ORDERS_ROUTES: Routes = [
  { path: '', component: OrdersPageComponent },
  { path: ':id', component: OrderDetailsPageComponent },
];
```

## 5.3 Guards : où les placer ?

- Guard global d’auth → `core/guards/` (ex: `auth.guard.ts`)
- Guard spécifique admin (rôle) → peut être dans `admin/` si c’est strictement lié au domaine.

**Règle utile :** si un guard porte une logique globale (auth, permissions centralisées) → `core`. Si c’est purement “admin area” → `features/admin`.

---

# 6) État et services : `api`, `data-access`, `domain`, `ui`

L’organisation par domaine peut être renforcée avec un découpage interne.

## 6.1 `api/` (intégration)

Contient :

- Clients HTTP typés
- DTO (Data Transfer Objects)
- Mappers DTO → modèle

Exemple :

```ts
// features/orders/api/orders.api.ts
import { HttpClient } from '@angular/common/http';
import { inject, Injectable } from '@angular/core';
import { Observable } from 'rxjs';

export interface OrderDto { id: string; total: number; }

@Injectable({ providedIn: 'root' })
export class OrdersApi {
  private http = inject(HttpClient);

  list(): Observable<OrderDto[]> {
    return this.http.get<OrderDto[]>('/api/orders');
  }
}
```

## 6.2 `data-access/` (accès données + orchestration)

Contient :

- Stores (NgRx, ComponentStore, signals store)
- Facades d’accès aux données
- Cache, pagination, synchronisation

Exemple :

```ts
// features/orders/data-access/orders.repository.ts
import { inject, Injectable } from '@angular/core';
import { map, Observable } from 'rxjs';
import { OrdersApi, OrderDto } from '../api/orders.api';

export interface Order { id: string; total: number; }

function mapDto(dto: OrderDto): Order {
  return { id: dto.id, total: dto.total };
}

@Injectable({ providedIn: 'root' })
export class OrdersRepository {
  private api = inject(OrdersApi);

  list(): Observable<Order[]> {
    return this.api.list().pipe(map(list => list.map(mapDto)));
  }
}
```

## 6.3 `ui/` (présentation)

Composants réutilisables **dans** le domaine :

- `OrderCardComponent`, `OrdersTableComponent`
- Aucun appel direct HTTP
- Inputs/Outputs clairs

## 6.4 `pages/` (composition)

Les pages orchestrent :

- lecture des params de route
- appels à la façade du domaine
- composition de composants UI

---

# 7) Facade de domaine : point d’entrée stable

Une façade réduit la surface de dépendance entre la UI et la data.

```ts
// features/orders/orders.facade.ts
import { inject, Injectable } from '@angular/core';
import { OrdersRepository, Order } from './data-access/orders.repository';
import { Observable } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class OrdersFacade {
  private repo = inject(OrdersRepository);

  listOrders(): Observable<Order[]> {
    return this.repo.list();
  }
}
```

**Avantages :**

- La page dépend de `OrdersFacade`, pas d’un service HTTP.
- On peut changer l’implémentation (cache, store) sans casser la UI.

---

# 8) Angular moderne : standalone + providers par feature

## 8.1 Providers au niveau feature

Selon vos besoins, vous pouvez fournir certains services au niveau des routes :

```ts
// features/users/users.routes.ts
import { Routes } from '@angular/router';
import { UsersPageComponent } from './pages/users-page.component';
import { UsersFacade } from './users.facade';

export const USERS_ROUTES: Routes = [
  {
    path: '',
    component: UsersPageComponent,
    providers: [UsersFacade],
  },
];
```

**Quand c’est utile ?**

- si vous voulez une instance de façade/scoped store par navigation
- si le service n’a pas vocation à être singleton global

## 8.2 Lazy loading et performance

L’organisation par domaines rend naturel :

- lazy load de routes
- preloading stratégique (ex: `admin` jamais preload)
- séparation du code et réduction du bundle initial

---

# 9) Tests, qualité, et évolutivité

## 9.1 Tests par domaine

- Tests unitaires de `data-access` / mappers / facades
- Tests de composants `ui` (inputs/outputs)
- Tests d’intégration sur `pages`

**Conseil :** structurez les tests au plus près du code.

```txt
features/orders/
  data-access/
    orders.repository.spec.ts
  pages/
    orders-page.component.spec.ts
```

## 9.2 Règles ESLint et boundaries

Pour faire respecter l’architecture :

- interdire aux features de s’importer directement entre elles
- interdire à `shared` d’importer une feature

Selon votre outillage (Nx aide beaucoup), vous pouvez :

- utiliser des règles de zones (`eslint-plugin-boundaries`)
- établir des conventions d’import

## 9.3 Évoluer vers une architecture “libraries”

Quand l’app grandit, vous pouvez extraire les domaines en libs. Exemples :

- `@app/auth` → lib auth
- `@app/orders` → lib orders

C’est une progression naturelle depuis l’organisation par domaines.

---

# 10) Ateliers pratiques

## Atelier 1 – Refactor d’un projet “par type” vers “par domaines”

**Situation :**

```txt
app/
  components/
  services/
  models/
```

**Objectif :**

- créer `features/auth`, `features/users`, `features/orders`
- déplacer composants/services au bon endroit
- adapter imports

**Livrable :** arborescence + compilation OK + navigation OK.

## Atelier 2 – Mise en place du routing lazy par domaine

- créer `*.routes.ts` par feature
- migrer `app-routing` en `app.routes.ts`
- vérifier que le code est lazy chargé

## Atelier 3 – Définir une façade et supprimer les accès directs HTTP dans les pages

- créer `OrdersFacade`
- faire dépendre la page uniquement de la façade
- mocker la façade en test

---

# Checklists et bonnes pratiques

## Checklist “domaine propre”

- [ ] Un dossier `features/<domain>` contient tout ce qui relève du domaine
- [ ] Les routes du domaine sont isolées (`<domain>.routes.ts`)
- [ ] Les pages n’appellent pas directement l’API HTTP
- [ ] Une façade existe pour limiter les dépendances
- [ ] Aucun import direct depuis une autre feature (sauf API publique)

## Checklist “shared & core”

- [ ] `shared` ne contient pas de logique métier
- [ ] `core` contient les singletons transverses (auth, interceptors, config)
- [ ] `core` ne dépend d’aucune feature

---

# Conclusion

L’organisation par **domaines fonctionnels** (auth, users, orders, admin) associée à des dossiers **core** et **shared** est l’un des meilleurs compromis pour une application Angular avancée :

- elle **structure** le code de manière intuitive par métier,
- elle **facilite le travail en équipe** (ownership par domaine),
- elle rend le **lazy loading** et l’évolution plus simples,
- elle limite la dette technique grâce à des frontières claires.

---

## Annexes

### Annex A – Variante “simple” (moins de sous-dossiers)

```txt
features/orders/
  components/
  services/
  pages/
  orders.routes.ts
```

À privilégier si :

- l’équipe est petite
- le domaine est limité

### Annex B – Variante “très scalable” (type Nx)

```txt
features/orders/
  domain/
  data-access/
  feature-shell/
  ui/
```

À privilégier si :

- plusieurs équipes
- beaucoup de règles métier
- plusieurs canaux (web, mobile web)
