# Formation Angular — Architecture Clean Frontend

**Référence**: 38 — *Architecture Clean Frontend*  
**Public**: Développeurs Angular (débutant++ à confirmé)  
**Durée suggérée**: 1 à 2 jours (adaptable)  
**Prérequis**: TypeScript, Angular (components, services, DI), RxJS bases, HTTPClient, tests (Jasmine/Jest) recommandés  

---

## Objectifs pédagogiques

À l’issue de la formation, vous saurez :

- Expliquer les principes d’une *Clean Architecture* appliquée au frontend.
- Structurer un projet Angular en couches : **présentation**, **orchestration**, **logique métier**, **accès aux données**.
- Concevoir des **composants de présentation fins** et testables.
- Implémenter une **Facade** pour isoler la vue de la complexité des flux.
- Créer des **services métier** centrés sur le domaine (use-cases).
- Mettre en place des **adaptateurs API** et des **mappers** (DTO ↔ Domain).
- Définir des **modèles** (Domain Models / View Models) et des contrats clairs.
- Tester efficacement chaque couche.
- Faire évoluer le code (feature-driven / modular) sans dette excessive.

---

## Plan de la formation

1. **Introduction : pourquoi une architecture frontend “propre” ?**
2. **Principes clés : séparation des responsabilités & dépendances**
3. **Découpage en couches Angular (Clean Frontend)**
4. **Modèles : Domain Model, DTO, ViewModel**
5. **Présentation : composants “dumb” (présentation)**
6. **Orchestration : Facades (Smart layer)**
7. **Logique métier : services de cas d’usage (Use-Case services)**
8. **Accès aux données : API adapters, repositories, mappers**
9. **Gestion d’état : RxJS, Facade + store léger, signaux (option)**
10. **Erreurs, loading, caching, retry : patterns transverses**
11. **Structure de projet recommandée (feature-first + layers)**
12. **Tests : stratégie par couche (unit / integration / component)**
13. **Ateliers guidés : implémentation end-to-end**
14. **Checklist de revue & anti-patterns**

---

# 1) Introduction : pourquoi une architecture frontend “propre” ?

## Problème courant
Dans une application Angular qui grossit, on observe souvent :

- Composants trop gros (appels HTTP, règles métier, transformations, navigation…)
- Services “fourre-tout” (God services)
- Couplage fort au backend (DTO utilisés partout)
- Difficultés de test (logique dans la vue)
- Changements backend qui cassent l’UI

## Réponse : Clean Frontend
Une architecture propre vise à :

- **Séparer** la présentation de la logique
- **Rendre testable** chaque partie isolément
- **Limiter le couplage** à l’infrastructure (API)
- **Permettre l’évolution** (nouveaux endpoints, nouveaux écrans, refonte UI)

---

# 2) Principes clés

## 2.1 Séparation des responsabilités
On découpe le code selon **ce qu’il fait** :

- Présenter (afficher / input)
- Orchestrer (coller UI ↔ cas d’usage)
- Exécuter la logique métier (règles, invariants)
- Accéder aux données (HTTP, storage)

## 2.2 Inversion des dépendances
Le but : faire dépendre le code des **abstractions** et du **domaine**, pas de l’HTTP.

**Règle pratique** :
- Le *domaine* ne connaît pas Angular.
- Les cas d’usage ne connaissent pas les DTO.
- La vue ne connaît pas les endpoints.

---

# 3) Découpage en couches Angular (Clean Frontend)

On utilise 4 grandes couches (adaptées à Angular) :

1. **Presentation (UI)**
   - composants de présentation, pipes UI
   - inputs/outputs, templates
2. **Orchestration (Facade / Container)**
   - état UI (loading, error, data)
   - coordination des actions utilisateur
3. **Domain (Business / Use Cases)**
   - règles métier, services applicatifs
   - modèles de domaine (types)
4. **Data Access (Infrastructure)**
   - adapters HTTP, repositories
   - mapping DTO ↔ Domain

> Note : dans Angular, la couche “Orchestration” est souvent une **Facade injectable** consommée par un composant conteneur, ou directement par une route.

---

# 4) Modèles : Domain Model, DTO, ViewModel

## 4.1 DTO (Data Transfer Object)
Représentation de la donnée telle que fournie par l’API.

- Souvent couplée au backend
- Peut changer fréquemment
- Exemple: `UserDto` avec des champs API (`first_name`) etc.

## 4.2 Domain Model
Représentation métier **stable**.

- Vocabulaire du domaine
- Invariants mieux exprimés
- Exemple: `User` avec `firstName`, `roles`, `isActive`

## 4.3 ViewModel (VM)
Modèle orienté affichage.

- Peut contenir des champs formatés (`fullName`, `statusLabel`)
- Prépare la vue à être simple

### Règle simple
- **DTO**: frontière externe (API)
- **Domain**: cœur
- **VM**: frontière UI

---

# 5) Présentation : composants fins (présentation)

## 5.1 Objectif
Un composant de présentation :

- Reçoit des `@Input()`
- Émet des événements via `@Output()`
- Ne contient pas d’appel HTTP
- Contient le minimum de logique (formatage simple)

## 5.2 Exemple

```ts
// user-card.component.ts
import { ChangeDetectionStrategy, Component, EventEmitter, Input, Output } from '@angular/core';

export interface UserCardVm {
  id: string;
  fullName: string;
  statusLabel: 'Active' | 'Inactive';
}

@Component({
  selector: 'app-user-card',
  templateUrl: './user-card.component.html',
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class UserCardComponent {
  @Input({ required: true }) user!: UserCardVm;
  @Output() toggle = new EventEmitter<string>();

  onToggle(): void {
    this.toggle.emit(this.user.id);
  }
}
```

```html
<!-- user-card.component.html -->
<article class="card">
  <h3>{{ user.fullName }}</h3>
  <p>Status: {{ user.statusLabel }}</p>
  <button type="button" (click)="onToggle()">Toggle status</button>
</article>
```

---

# 6) Orchestration : Facades

## 6.1 Rôle de la Facade
Une facade fournit à la vue :

- Des **observables** / signaux (state)
- Des **actions** (methods) pour déclencher des use-cases
- La gestion de `loading` / `error`

La vue ne sait pas si la donnée vient :
- d’un cache
- d’un store
- d’un appel HTTP

## 6.2 Patron de state minimal

```ts
export type LoadState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'error'; message: string }
  | { status: 'success'; data: T };
```

## 6.3 Implémentation Facade (RxJS)

```ts
import { Injectable } from '@angular/core';
import { BehaviorSubject, Observable, catchError, map, of, switchMap, tap } from 'rxjs';
import { UsersService } from '../domain/users.service';
import { User } from '../domain/user.model';

interface UsersVm {
  users: { id: string; fullName: string; statusLabel: 'Active' | 'Inactive' }[];
}

@Injectable()
export class UsersFacade {
  private readonly reload$ = new BehaviorSubject<void>(undefined);
  private readonly stateSubject = new BehaviorSubject<LoadState<UsersVm>>({ status: 'idle' });

  readonly state$: Observable<LoadState<UsersVm>> = this.stateSubject.asObservable();

  constructor(private readonly usersService: UsersService) {
    this.reload$
      .pipe(
        tap(() => this.stateSubject.next({ status: 'loading' })),
        switchMap(() =>
          this.usersService.getUsers().pipe(
            map((users) => ({ users: users.map(toUserCardVm) })),
            map((vm) => ({ status: 'success', data: vm } as const)),
            catchError((e) => of({ status: 'error', message: toMessage(e) } as const))
          )
        )
      )
      .subscribe((s) => this.stateSubject.next(s));
  }

  reload(): void {
    this.reload$.next();
  }

  toggleUserStatus(userId: string): void {
    // action orchestrée : appelle un use-case puis reload si besoin
    this.usersService.toggleStatus(userId).subscribe({
      next: () => this.reload(),
      error: (e) => this.stateSubject.next({ status: 'error', message: toMessage(e) }),
    });
  }
}

function toUserCardVm(u: User) {
  return {
    id: u.id,
    fullName: `${u.firstName} ${u.lastName}`,
    statusLabel: u.isActive ? 'Active' : 'Inactive',
  } as const;
}

function toMessage(e: unknown): string {
  return e instanceof Error ? e.message : 'Unknown error';
}
```

### Points-clés
- La transformation Domain → VM se fait **hors du template**.
- Une vue consomme un **state simple**.
- Facade = point d’entrée unique côté UI.

---

# 7) Logique métier : services de cas d’usage (Use-Case services)

## 7.1 Principe
Un service métier :

- exprime des opérations métier (use-cases)
- dépend d’abstractions (repositories)
- expose des types Domain

## 7.2 Exemple

```ts
import { Injectable } from '@angular/core';
import { Observable, map, switchMap } from 'rxjs';
import { UsersRepository } from './users.repository';
import { User } from './user.model';

@Injectable({ providedIn: 'root' })
export class UsersService {
  constructor(private readonly repo: UsersRepository) {}

  getUsers(): Observable<User[]> {
    return this.repo.getAll();
  }

  toggleStatus(userId: string): Observable<User> {
    return this.repo.getById(userId).pipe(
      switchMap((user) => {
        const next = { ...user, isActive: !user.isActive };
        return this.repo.save(next);
      })
    );
  }
}
```

> Ici, `UsersService` ne connaît pas HTTP, seulement un `UsersRepository`.

---

# 8) Accès aux données : API adapters, repositories, mappers

## 8.1 Repository (contrat)

```ts
import { Observable } from 'rxjs';
import { User } from './user.model';

export abstract class UsersRepository {
  abstract getAll(): Observable<User[]>;
  abstract getById(id: string): Observable<User>;
  abstract save(user: User): Observable<User>;
}
```

## 8.2 Adapter HTTP (implémentation)

```ts
import { HttpClient } from '@angular/common/http';
import { Injectable } from '@angular/core';
import { Observable, map } from 'rxjs';
import { UsersRepository } from '../domain/users.repository';
import { User } from '../domain/user.model';

interface UserDto {
  id: string;
  first_name: string;
  last_name: string;
  active: boolean;
}

@Injectable()
export class UsersApiAdapter implements UsersRepository {
  constructor(private readonly http: HttpClient) {}

  getAll(): Observable<User[]> {
    return this.http
      .get<UserDto[]>('/api/users')
      .pipe(map((dtos) => dtos.map(fromDto)));
  }

  getById(id: string): Observable<User> {
    return this.http
      .get<UserDto>(`/api/users/${id}`)
      .pipe(map(fromDto));
  }

  save(user: User): Observable<User> {
    return this.http
      .put<UserDto>(`/api/users/${user.id}`, toDto(user))
      .pipe(map(fromDto));
  }
}

function fromDto(dto: UserDto): User {
  return {
    id: dto.id,
    firstName: dto.first_name,
    lastName: dto.last_name,
    isActive: dto.active,
  };
}

function toDto(user: User): UserDto {
  return {
    id: user.id,
    first_name: user.firstName,
    last_name: user.lastName,
    active: user.isActive,
  };
}
```

## 8.3 Wiring (providers)

Dans `users.feature.module.ts` (ou route providers) :

```ts
import { NgModule } from '@angular/core';
import { UsersRepository } from './domain/users.repository';
import { UsersApiAdapter } from './data/users-api.adapter';
import { UsersFacade } from './ui/users.facade';

@NgModule({
  providers: [
    UsersFacade,
    { provide: UsersRepository, useClass: UsersApiAdapter },
  ],
})
export class UsersFeatureModule {}
```

---

# 9) Gestion d’état : RxJS, Facade + store léger, signaux (option)

## 9.1 Pourquoi un état ?
La vue doit représenter :

- chargement
- erreur
- données
- actions en cours (optimistic update, disable button)

La Facade est un endroit naturel.

## 9.2 Store léger (BehaviorSubject)
- Suffisant pour beaucoup de pages
- Facile à tester
- Limite l’over-engineering

## 9.3 Avec Signals (Angular >= 16) — option
Alternative : utiliser `signal`, `computed`, `effect` au lieu de `BehaviorSubject`.

Règle : garder des **interfaces** claires et ne pas exposer l’implémentation aux composants.

---

# 10) Erreurs, loading, caching, retry : patterns transverses

## 10.1 Pattern “state machine” simple
Déjà vu via `LoadState<T>`.

## 10.2 Caching côté facade
Attention : le cache appartient souvent à l’orchestration (ou data layer si partagé).

Exemple de caching simple :
- garder la dernière valeur en mémoire
- recharger explicitement

## 10.3 Retry & exponential backoff (data layer)
À réserver aux erreurs transitoires (réseau).

---

# 11) Structure de projet recommandée

## 11.1 Approche “feature-first + layers”

```
src/app/
  features/
    users/
      data/
        users-api.adapter.ts
        users.dto.ts
        users.mappers.ts
      domain/
        user.model.ts
        users.repository.ts
        users.service.ts
      ui/
        users-page.component.ts
        users.facade.ts
        components/
          user-card/
            user-card.component.ts
            user-card.component.html
      users.routes.ts
```

### Pourquoi
- Les features sont autonomes.
- Chaque feature contient ses couches.
- Plus simple à extraire, refactorer, tester.

---

# 12) Tests : stratégie par couche

## 12.1 Tests de la couche Domain
- Tester `UsersService` avec un faux repository.

```ts
class FakeUsersRepo extends UsersRepository {
  private users: User[] = [
    { id: '1', firstName: 'Ada', lastName: 'Lovelace', isActive: true },
  ];

  getAll() { return of(this.users); }
  getById(id: string) { return of(this.users.find(u => u.id === id)!); }
  save(user: User) {
    this.users = this.users.map(u => u.id === user.id ? user : u);
    return of(user);
  }
}
```

## 12.2 Tests de la couche Data
- Tester `fromDto/toDto`.
- Tester l’adapter HTTP avec `HttpTestingController`.

## 12.3 Tests de Facade
- Vérifier les transitions `idle → loading → success|error`.
- Vérifier que `toggleUserStatus()` déclenche un reload.

## 12.4 Tests UI
- Les composants de présentation : tests simples (inputs/outputs).
- Les pages : tests d’intégration légers possibles.

---

# 13) Ateliers guidés (end-to-end)

## Atelier 1 — Refactor d’un composant “fat”
**Énoncé** : un composant page contient HTTP + mapping + logique.  
**Objectif** : extraire couche data, puis domain et facade.

Étapes :
1. Identifier DTO vs Domain.
2. Créer mapper DTO → Domain.
3. Créer `UsersRepository` + `UsersApiAdapter`.
4. Créer `UsersService` (use-cases).
5. Créer `UsersFacade`.
6. Simplifier la page : consommation du state, triggers sur actions.

## Atelier 2 — Ajouter un use-case
Ajouter “Désactiver tous les utilisateurs inactifs depuis 30 jours” (exemple).  
Objectif : montrer où placer la logique et comment tester.

## Atelier 3 — Gestion des erreurs
- Afficher un message d’erreur internationalisable
- Ajouter un bouton “Réessayer” qui appelle `facade.reload()`

---

# 14) Checklist de revue & anti-patterns

## 14.1 Checklist
- [ ] La vue ne fait pas d’HTTP.
- [ ] Les DTO ne sortent pas de la couche data.
- [ ] Les règles métier sont dans domain/use-cases.
- [ ] Les transformations Domain→VM ne sont pas dans le template.
- [ ] Les dépendances vont vers l’intérieur (domain), pas l’inverse.
- [ ] Chaque couche a ses tests.

## 14.2 Anti-patterns fréquents
- Coupler l’UI à l’API (`UserDto` utilisé dans les components).
- Mettre la logique métier dans la facade (la facade orchestre, elle ne “décide” pas).
- Mélanger state UI et state domain sans distinction.
- “Service unique” qui fait tout.

---

## Annexes

### A) Modèle domain exemple

```ts
export interface User {
  id: string;
  firstName: string;
  lastName: string;
  isActive: boolean;
}
```

### B) Recommandations pratiques
- **ChangeDetectionStrategy.OnPush** pour les composants de présentation.
- Préférer des **observables** ou signaux en façade, mais exposer une interface stable.
- Utiliser des mappers explicites (fonctions pures) : meilleure lisibilité et testabilité.
- Documenter les frontières : `data/` est la seule couche autorisée à connaître l’API.

---

## Conclusion
Une *Architecture Clean Frontend* dans Angular se traduit par :

- **Composants UI** simples et centrés sur l’affichage
- **Facades** pour orchestrer et exposer un état UI consommable
- **Services métier** (use-cases) testables et centrés sur le domaine
- **Adaptateurs API** et **mappers** isolant la forme des données externes

Cette séparation rend l’application plus maintenable, testable et robuste face au changement.
