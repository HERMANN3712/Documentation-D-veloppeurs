# Formation — Architecture globale d’une application Angular (avancée)

- **Public** : Développeurs Angular (intermédiaire → avancé), formateurs, tech leads
- **Prérequis** : TypeScript, RxJS (bases), Angular CLI, notions de DI, routing
- **Durée indicative** : 1 jour (7h) ou 2 demi‑journées
- **Objectifs pédagogiques**
  - Structurer une application Angular **par domaine fonctionnel** (feature‑first) plutôt que par type technique.
  - Mettre en place une séparation claire des responsabilités : **presentation vs container**, services métier, accès aux données, guards, interceptors, modèles, modules/features autonomes.
  - Concevoir une architecture évolutive, testable, maintenable et cohérente.

---

## Plan de la formation

1. [Pourquoi une architecture globale ?](#1-pourquoi-une-architecture-globale-)
2. [Découpage par domaine (feature-first) vs découpage par type](#2-découpage-par-domaine-feature-first-vs-découpage-par-type)
3. [Couches et responsabilités : la “carte” de l’application](#3-couches-et-responsabilités--la-carte-de-lapplication)
4. [Composants : présentation vs conteneurs](#4-composants--présentation-vs-conteneurs)
5. [Services : métier vs accès aux données](#5-services--métier-vs-accès-aux-données)
6. [Modèles, DTO, mapping et validation](#6-modèles-dto-mapping-et-validation)
7. [Routing, modules/features autonomes et lazy loading](#7-routing-modulesfeatures-autonomes-et-lazy-loading)
8. [Guards : sécurité, navigation et prérequis métier](#8-guards--sécurité-navigation-et-prérequis-métier)
9. [Interceptors : cross‑cutting concerns HTTP](#9-interceptors--crosscutting-concerns-http)
10. [Gestion d’état : quand, où, comment ?](#10-gestion-détat--quand-où-comment-)
11. [Convention de nommage, structure de dossiers et “barrels”](#11-convention-de-nommage-structure-de-dossiers-et-barrels)
12. [Tests & maintenabilité : stratégies et points de contrôle](#12-tests--maintenabilité--stratégies-et-points-de-contrôle)
13. [Atelier guidé : refactoriser une app “type-first” en “feature-first”](#13-atelier-guidé--refactoriser-une-app-type-first-en-feature-first)

---

## 1. Pourquoi une architecture globale ?

Dans une application Angular simple, un découpage minimal peut suffire. Mais dès que l’on monte en complexité (plusieurs équipes, fonctionnalités nombreuses, API multiples, contraintes de sécurité), l’absence d’architecture explicite provoque :

- **Couplage fort** entre UI et logique métier (composants “God components”).
- Services “fourre‑tout” difficiles à tester.
- Dossiers par type (`components/`, `services/`, `models/`) qui deviennent gigantesques.
- Duplications (mêmes DTO, même mapping, même gestion d’erreur, etc.).
- Temps d’onboarding élevé et évolution risquée.

**But** : rendre l’application lisible et évolutive en établissant des responsabilités nettes et un découpage cohérent.

### Principes directeurs

- **Single Responsibility Principle** : chaque élément a une responsabilité principale.
- **Feature-first** : on organise le code par **domaine fonctionnel**.
- **Dépendances orientées vers l’intérieur** : l’UI dépend du métier, le métier dépend d’abstractions, l’accès aux données est “branché” derrière des services/facades.
- **Cross-cutting concerns** (auth, logs, erreurs, i18n) centralisés via interceptors, services dédiés, etc.

---

## 2. Découpage par domaine (feature-first) vs découpage par type

### Découpage “type-first” (à éviter à grande échelle)

```text
src/app/
  components/
  services/
  models/
  guards/
  interceptors/
```

**Problèmes** :
- Un même domaine (ex: `orders`) est dispersé dans 5 dossiers.
- Refactor d’une fonctionnalité = modifications dans plusieurs endroits.
- Difficile de supprimer/extraire une feature.

### Découpage “feature-first” (recommandé)

```text
src/app/
  core/
  shared/
  features/
    orders/
    auth/
    admin/
```

**Avantages** :
- Tout ce qui concerne `orders` est au même endroit.
- Meilleure modularité, découplage et évolutivité.
- Favorise le lazy loading, le split par équipe, la suppression d’une feature.

> Règle simple : **ce qui change ensemble doit être regroupé ensemble**.

---

## 3. Couches et responsabilités : la “carte” de l’application

On vise une architecture en couches, sans rigidité excessive.

### Les grandes zones

- **Core** : infrastructure globale *singleton* (auth, interceptors, config, logger, error handling, layout shell…)
- **Shared** : éléments réutilisables, stateless autant que possible (UI components, pipes, directives, utils)
- **Features** : domaines fonctionnels (ex: `orders`, `customers`, `billing`)

### Dans une feature

On sépare généralement :

- **UI / components** : présentation (dumb components)
- **Containers / pages** : orchestration, routing, composition d’écrans
- **Domain / services** : logique métier, cas d’usage
- **Data access** : appels HTTP / persistance / mapping DTO
- **Models** : types métier et structures associées

Schéma mental :

```text
[UI Components]  <-- inputs/outputs -->  [Containers/Pages]
                                           |
                                           v
                                   [Domain Services]
                                           |
                                           v
                                   [Data Access Services]
                                           |
                                           v
                                         [HTTP]
```

---

## 4. Composants : présentation vs conteneurs

### Objectif

- **Composants de présentation** : affichage + interactions UI simples.
- **Composants conteneurs** (ou pages) : récupèrent les données, orchestrent flux RxJS, navigation, appels aux services.

### 4.1 Composant de présentation (dumb)

Caractéristiques :
- Reçoit ses données via `@Input()`
- Émet des événements via `@Output()`
- Pas d’appel direct à HTTP / services métier (ou très limité)
- Facile à tester et réutiliser

Exemple : `order-list.component.ts`

```ts
import { Component, EventEmitter, Input, Output } from '@angular/core';

export interface OrderListItemVm {
  id: string;
  label: string;
  total: number;
  status: 'draft' | 'confirmed' | 'shipped';
}

@Component({
  selector: 'app-order-list',
  template: `
    <section>
      <h2>Commandes</h2>
      <ul>
        <li *ngFor="let o of orders">
          <button type="button" (click)="select.emit(o.id)">
            {{ o.label }} — {{ o.total | currency:'EUR' }} — {{ o.status }}
          </button>
        </li>
      </ul>
    </section>
  `
})
export class OrderListComponent {
  @Input({ required: true }) orders: OrderListItemVm[] = [];
  @Output() select = new EventEmitter<string>();
}
```

### 4.2 Composant conteneur (smart)

Caractéristiques :
- Compose des composants de présentation
- Gère le routeur, les observables, l’état d’écran (loading/error)
- Appelle des services métier

Exemple : `orders-page.component.ts`

```ts
import { Component, inject } from '@angular/core';
import { Router } from '@angular/router';
import { OrdersFacade } from '../domain/orders.facade';

@Component({
  selector: 'app-orders-page',
  template: `
    <app-order-list
      [orders]="vm$ | async"
      (select)="onSelect($event)">
    </app-order-list>
  `
})
export class OrdersPageComponent {
  private readonly router = inject(Router);
  private readonly facade = inject(OrdersFacade);

  vm$ = this.facade.orderListVm$;

  onSelect(id: string) {
    this.router.navigate(['./', id]);
  }
}
```

### Bonnes pratiques

- Un composant de présentation peut être mis dans `shared/` s’il est réellement générique.
- Les conteneurs restent dans la feature car ils portent du contexte.
- Éviter les `subscribe()` dans les composants ; privilégier `async` pipe + orchestration via services/facades.

---

## 5. Services : métier vs accès aux données

### 5.1 Services métier (Domain / Use-cases)

Ils encapsulent :
- règles métier,
- enchaînements (workflow),
- validation métier,
- orchestration de plusieurs sources (API, cache, state).

Ils ne devraient pas connaître les détails HTTP (URLs, headers), ni les DTO bruts.

Exemple : façade orientée UI (souvent appelée `Facade`)

```ts
import { Injectable, inject } from '@angular/core';
import { map, shareReplay } from 'rxjs/operators';
import { OrdersRepository } from '../data-access/orders.repository';

@Injectable({ providedIn: 'root' })
export class OrdersFacade {
  private readonly repo = inject(OrdersRepository);

  // Exemple : VM prête pour l’UI
  orderListVm$ = this.repo.getOrders().pipe(
    map(orders => orders.map(o => ({
      id: o.id,
      label: o.reference,
      total: o.totalAmount,
      status: o.status,
    }))),
    shareReplay({ bufferSize: 1, refCount: true })
  );
}
```

> Le terme *facade* est utile pour clarifier que ce service est un **point d’entrée** unique vers une fonctionnalité côté UI.

### 5.2 Services d’accès aux données (Data access)

Responsabilités :
- appels HTTP,
- mapping DTO <-> modèles,
- traitement des erreurs techniques,
- éventuellement cache (en cohérence avec la stratégie de l’app).

Exemple : `orders.repository.ts`

```ts
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { map } from 'rxjs/operators';

export interface OrderDto {
  id: string;
  reference: string;
  total_amount: number;
  status: string;
}

export interface Order {
  id: string;
  reference: string;
  totalAmount: number;
  status: 'draft' | 'confirmed' | 'shipped';
}

@Injectable({ providedIn: 'root' })
export class OrdersRepository {
  private readonly http = inject(HttpClient);

  getOrders() {
    return this.http.get<OrderDto[]>('/api/orders').pipe(
      map(dtos => dtos.map(dto => ({
        id: dto.id,
        reference: dto.reference,
        totalAmount: dto.total_amount,
        status: dto.status as Order['status'],
      })))
    );
  }
}
```

### Règles de dépendances recommandées

- UI/Containers → Facade/Domain
- Facade/Domain → Repository/Data access
- Repository → HttpClient

Éviter :
- UI → Repository direct (court-circuit du domaine)
- Domain → HttpClient direct

---

## 6. Modèles, DTO, mapping et validation

### Pourquoi séparer modèles et DTO ?

- Les DTO reflètent le contrat API (naming, optionalité, champs techniques).
- Les modèles reflètent le domaine de l’app (types stricts, invariants).

### Patterns de mapping

- Mapping simple inline (ok si petit)
- Mapper dédié (`orders.mapper.ts`) si volumineux, réutilisable, ou si transformations complexes

Exemple : `orders.mapper.ts`

```ts
import { Order, OrderDto } from './orders.types';

export const toOrder = (dto: OrderDto): Order => ({
  id: dto.id,
  reference: dto.reference,
  totalAmount: dto.total_amount,
  status: dto.status as Order['status'],
});
```

### Validation

- **UI validation** : Reactive Forms (validators)
- **Validation métier** : à placer dans le domaine (service / use-case)
- Ne pas confondre validation de formulaire et règles métier (ex: “un panier ne peut pas être validé si stock insuffisant”).

---

## 7. Routing, modules/features autonomes et lazy loading

### Objectif

- Des features **autonomes** (contenant routes, pages, composants, services locaux).
- Lazy loading pour réduire le bundle initial.

### Structure typique

```text
src/app/
  core/
    core.providers.ts
    interceptors/
  shared/
    ui/
    pipes/
  features/
    orders/
      pages/
      components/
      domain/
      data-access/
      orders.routes.ts
```

### Routing lazy-loaded (Angular moderne)

`app.routes.ts`

```ts
import { Routes } from '@angular/router';

export const appRoutes: Routes = [
  {
    path: 'orders',
    loadChildren: () => import('./features/orders/orders.routes')
      .then(m => m.ordersRoutes)
  },
  { path: '', pathMatch: 'full', redirectTo: 'orders' },
];
```

`features/orders/orders.routes.ts`

```ts
import { Routes } from '@angular/router';
import { OrdersPageComponent } from './pages/orders-page.component';

export const ordersRoutes: Routes = [
  { path: '', component: OrdersPageComponent },
  // { path: ':id', component: OrderDetailsPageComponent }
];
```

> Remarque : selon votre version et stratégie, vous pouvez utiliser modules classiques (`NgModule`) ou `standalone` components. Les principes d’architecture restent les mêmes.

---

## 8. Guards : sécurité, navigation et prérequis métier

### Rôles typiques des guards

- **AuthGuard** : accès conditionné à l’authentification
- **Role/Permission guard** : accès par rôle/droit
- **Prerequisite guard** : prérequis métier (ex: profil complété)

Exemple : `auth.guard.ts`

```ts
import { CanActivateFn, Router } from '@angular/router';
import { inject } from '@angular/core';
import { AuthService } from '../core/auth/auth.service';

export const authGuard: CanActivateFn = () => {
  const auth = inject(AuthService);
  const router = inject(Router);

  if (auth.isAuthenticated()) return true;
  return router.parseUrl('/login');
};
```

### Bonnes pratiques

- Les guards doivent rester **simples** et rapides.
- Mettre les règles de permission complexes dans un **service d’autorisation** (ex: `AuthorizationService`).
- Ne pas dupliquer la sécurité côté client : l’API doit rester l’autorité.

---

## 9. Interceptors : cross-cutting concerns HTTP

### Cas d’usage courants

- Ajout du token d’authentification
- Gestion centralisée des erreurs HTTP
- Retry/backoff pour erreurs transient
- Tracing / correlation id
- Normalisation de réponse (avec prudence)

Exemple : `auth.interceptor.ts`

```ts
import { HttpInterceptorFn } from '@angular/common/http';
import { inject } from '@angular/core';
import { AuthService } from './auth.service';

export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const auth = inject(AuthService);
  const token = auth.getToken();

  if (!token) return next(req);

  return next(req.clone({
    setHeaders: { Authorization: `Bearer ${token}` }
  }));
};
```

Exemple : `http-error.interceptor.ts`

```ts
import { HttpInterceptorFn } from '@angular/common/http';
import { catchError, throwError } from 'rxjs';

export const httpErrorInterceptor: HttpInterceptorFn = (req, next) =>
  next(req).pipe(
    catchError(err => {
      // centraliser logging / mapping technique
      return throwError(() => err);
    })
  );
```

### Où enregistrer les interceptors ?

Dans le `core` (configuration globale) :

```ts
import { provideHttpClient, withInterceptors } from '@angular/common/http';
import { authInterceptor } from './auth/auth.interceptor';
import { httpErrorInterceptor } from './errors/http-error.interceptor';

export const CORE_PROVIDERS = [
  provideHttpClient(withInterceptors([authInterceptor, httpErrorInterceptor]))
];
```

---

## 10. Gestion d’état : quand, où, comment ?

La gestion d’état n’est pas un prérequis pour une bonne architecture, mais elle doit rester cohérente.

### Niveaux d’état

- **État local de composant** : UI state (toggle, filtre local, champ de formulaire)
- **État de feature** : données partagées par plusieurs pages de même domaine
- **État global** : utilisateur courant, préférences, permissions, thème

### Stratégies

- **RxJS + Services/Facades** (souvent suffisant)
- **Signal-based state** (si l’app et l’équipe sont alignées)
- **Store (NgRx, Akita, NGXS, Elf, etc.)** si :
  - beaucoup d’actions, d’écrans, de collaboration multi-équipes,
  - besoin d’outils (devtools), time-travel, patterns stricts,
  - données complexes et fortement partagées.

> Conseil : commencer simple (facade + RxJS), introduire un store quand la complexité le justifie.

---

## 11. Convention de nommage, structure de dossiers et “barrels”

### Exemple de structure feature-first

```text
features/orders/
  pages/
    orders-page.component.ts
    order-details-page.component.ts
  components/
    order-list/
      order-list.component.ts
  domain/
    orders.facade.ts
    orders.service.ts
  data-access/
    orders.repository.ts
    orders.mapper.ts
  models/
    order.model.ts
  orders.routes.ts
```

### Conventions

- `*.page.component.ts` pour les pages/containers
- `*.component.ts` pour presentation components
- `*.facade.ts` pour point d’entrée UI vers le domaine
- `*.repository.ts` pour data access
- `*.dto.ts` si vous séparez DTO explicitement
- `*.model.ts` pour modèles métier

### Barrels (`index.ts`) — avec prudence

- Avantage : imports plus courts (`from './data-access'`)
- Inconvénient : cycles de dépendances, coûts de build parfois, ambiguïté

Recommandation :
- Barrels **limités** aux frontières (ex: `shared/ui/index.ts`), éviter les barrels omniprésents.

---

## 12. Tests & maintenabilité : stratégies et points de contrôle

### Tests unitaires

- **Presentation components** : test d’affichage et d’émission d’événements.
- **Domain/facade** : test de mapping VM, orchestration RxJS (marbles si besoin).
- **Mappers** : tests purs faciles.

### Tests d’intégration

- Repository : tests avec `HttpTestingController`.
- Guards/Interceptors : tests ciblés (vérifier headers, redirections, etc.).

### Points de contrôle (checklist)

- Les composants UI n’appellent pas directement `HttpClient`.
- Aucun service ne devient “God service”.
- Les modèles domaine ne dépendent pas des DTO.
- Les features sont lazy-loadées si pertinentes.
- Les cross-cutting concerns sont centralisés (interceptors, core services).

---

## 13. Atelier guidé : refactoriser une app “type-first” en “feature-first”

### Situation initiale

```text
app/
  components/
    orders-list.component.ts
    order-details.component.ts
  services/
    orders.service.ts
  models/
    order.ts
```

**Problèmes constatés** :
- `orders.service.ts` mélange HTTP + rules + mapping
- composants font des `subscribe()`
- difficile d’isoler la fonctionnalité `orders`

### Étapes de refactor

1. **Créer la feature `orders/`**
   - déplacer pages, composants, modèles
2. **Introduire un repository**
   - déplacer les appels HTTP + mapping DTO
3. **Créer une facade**
   - exposer `vm$`, `commands` (méthodes) vers l’UI
4. **Découper les composants**
   - transformer une page en conteneur
   - transformer liste/détail en composants de présentation
5. **Ajouter routing lazy**
   - `loadChildren` vers `orders.routes.ts`
6. **Ajouter guards/interceptors si besoin**
   - auth + gestion erreurs

### Livrables attendus

- Une structure par domaine.
- Un flux de données lisible : Page → Facade → Repository.
- Une base solide pour faire évoluer le domaine (ajout de filtres, pagination, cache, etc.).

---

## Annexes

### A. Exemple “core/shared/features”

```text
src/app/
  core/
    auth/
      auth.service.ts
      auth.interceptor.ts
      auth.guard.ts
    errors/
      http-error.interceptor.ts
    core.providers.ts
  shared/
    ui/
      button/
      card/
    pipes/
    directives/
  features/
    orders/
      components/
      pages/
      domain/
      data-access/
      models/
      orders.routes.ts
  app.routes.ts
  app.config.ts
```

### B. Rappels : séparation des responsabilités

- **Presentation component** : UI pure, test simple, réutilisable
- **Container/page** : orchestration écran, routing, composition
- **Domain/facade/service métier** : règles + use-cases
- **Repository/data-access** : HTTP, persistance, mapping DTO
- **Guards** : accès, prérequis
- **Interceptors** : cross-cutting HTTP
- **Models** : types du domaine, invariants, helpers

---

## Conclusion

Une application Angular avancée gagne en robustesse lorsque :

1. Elle est organisée **par domaine fonctionnel**.
2. Les responsabilités sont explicites (UI vs orchestration vs métier vs data access).
3. Les préoccupations transverses (auth, erreurs, headers) sont centralisées.
4. Les features deviennent modulaires, testables et facilement évolutives.

Ce cadre vous permet d’industrialiser le développement : onboarding rapide, refactorings sûrs, et croissance maîtrisée du codebase.
