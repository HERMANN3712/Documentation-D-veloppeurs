# Formation Angular (Avancé) — Bonnes pratiques de conception

> **Référence** : bonnes pratiques Angular avancées — composants petits et cohérents, logique métier hors du template, services spécialisés, dépendances explicites, gestion d’état claire, noms cohérents, limitation du couplage entre couches.

---

## 1) Informations générales

### Objectifs pédagogiques
À l’issue de la formation, le participant sera capable de :

- Concevoir des **composants Angular petits, cohérents et réutilisables**.
- Déplacer la **logique métier hors des templates** et rendre les vues simples et testables.
- Structurer une application avec des **services spécialisés** et des responsabilités clairement définies.
- Rendre les **dépendances explicites**, maîtriser l’injection, et éviter les couplages implicites.
- Mettre en place une **gestion d’état claire** (locale vs globale, flux unidirectionnel, immutabilité).
- Appliquer des **conventions de nommage** cohérentes (fichiers/Classes/selectors) pour la maintenabilité.
- **Limiter le couplage** entre couches (UI / domaine / infra) et entre modules/features.

### Public et prérequis
- Développeurs Angular intermédiaires à avancés.
- Connaissance : composants, services, RxJS de base, routing.

### Durée suggérée
- 1 jour (7h) ou 2 demi-journées.

### Format
- Apports théoriques
- Démonstrations
- Exercices guidés
- Revue de code et check-lists

---

## 2) Plan détaillé

1. **Principes de conception** (cohésion, couplage, séparation des responsabilités)
2. **Composants petits et cohérents**
3. **Templates minces (smart vs dumb) & logique métier hors HTML**
4. **Services spécialisés & architecture par couches**
5. **Dépendances explicites (DI, tokens, interfaces, inversion)**
6. **Gestion d’état claire (local, partagé, global)**
7. **Nommage et conventions**
8. **Limiter le couplage entre couches** (UI / domain / data)
9. **Check-list “prête pour la production”**
10. **Atelier final : refactor d’un mini-feature**

---

## 3) Principes de conception (rappels utiles)

### 3.1 Cohésion vs couplage
- **Cohésion** : chaque unité (composant, service, module) a un rôle clair.
- **Couplage** : dépendances entre unités. On veut un couplage **faible** pour faciliter l’évolution.

**Bon objectif** : 
- *Haute cohésion* + *faible couplage*.

### 3.2 SRP (Single Responsibility Principle)
Un composant/service doit avoir **une raison principale** de changer.

### 3.3 “Make illegal states unrepresentable” (autant que possible)
- Modéliser les états UI/domaine (loading/success/error) via des types.

---

## 4) Composants petits et cohérents

### 4.1 Symptômes d’un composant “trop gros”
- Fichier > 300-400 lignes.
- Trop d’`@Input()` / `@Output()`.
- Mélange UI + orchestration + logique métier + appels HTTP.
- Multiples subscriptions dispersées.

### 4.2 Stratégie de découpage
- Séparer :
  - **Présentational components** (UI pure)
  - **Container components** (orchestration / data flow)
- Extraire :
  - un composant pour un widget récurrent
  - un composant pour une section de page
  - une directive pour un comportement transversal

### 4.3 Exemple : séparation container / présentational

#### Avant (composant monolithique)
```ts
@Component({
  selector: 'app-user-profile',
  templateUrl: './user-profile.component.html'
})
export class UserProfileComponent {
  user?: User;
  isLoading = false;

  constructor(private http: HttpClient) {}

  load(id: string) {
    this.isLoading = true;
    this.http.get<User>(`/api/users/${id}`).subscribe({
      next: u => this.user = u,
      error: () => alert('Erreur'),
      complete: () => this.isLoading = false
    });
  }
}
```

#### Après (container + service + présentational)
```ts
// user-profile.container.ts
@Component({
  selector: 'app-user-profile-container',
  template: `
    <app-user-profile
      [user]="user$ | async"
      [loading]="loading$ | async">
    </app-user-profile>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class UserProfileContainer {
  readonly user$ = this.facade.user$;
  readonly loading$ = this.facade.loading$;

  constructor(private facade: UserProfileFacade) {}
}
```

```ts
// user-profile.component.ts (présentation)
@Component({
  selector: 'app-user-profile',
  templateUrl: './user-profile.component.html',
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class UserProfileComponent {
  @Input({ required: true }) user: User | null = null;
  @Input() loading = false;
}
```

```ts
// user-profile.facade.ts
@Injectable()
export class UserProfileFacade {
  private readonly state = new BehaviorSubject<{loading: boolean; user: User | null}>({
    loading: false,
    user: null
  });

  readonly user$ = this.state.pipe(map(s => s.user), distinctUntilChanged());
  readonly loading$ = this.state.pipe(map(s => s.loading), distinctUntilChanged());

  constructor(private api: UsersApi) {}

  load(userId: string) {
    this.patch({ loading: true });
    this.api.getUser(userId).pipe(finalize(() => this.patch({ loading: false })))
      .subscribe(user => this.patch({ user }));
  }

  private patch(p: Partial<{loading: boolean; user: User | null}>) {
    this.state.next({ ...this.state.value, ...p });
  }
}
```

### 4.4 Bonnes pratiques associées
- **ChangeDetectionStrategy.OnPush** par défaut pour les composants de présentation.
- Éviter les effets de bord dans les getters utilisés par le template.
- Préférer des **entrées immutables** (nouvel objet plutôt que mutation) pour OnPush.

---

## 5) Logique métier hors du template

### 5.1 Règle : le template doit rester déclaratif
Le template devrait :
- afficher des données
- déclencher des événements UI
- composer des composants

Il ne devrait pas :
- contenir des calculs lourds
- enchaîner des ternaires complexes
- appeler des fonctions non pures à chaque cycle de détection

### 5.2 Éviter l’“HTML programming”

#### Anti-pattern
```html
<div *ngIf="(user$ | async) as user">
  {{ user.firstName + ' ' + user.lastName }}
  <span *ngIf="user.roles.includes('admin') && user.isActive && (now | date)">
    ADMIN
  </span>
</div>
```

#### Mieux : construire un ViewModel
```ts
// user-profile.vm.ts
export type UserProfileVm = {
  fullName: string;
  isAdminBadgeVisible: boolean;
};

function toVm(user: User): UserProfileVm {
  return {
    fullName: `${user.firstName} ${user.lastName}`,
    isAdminBadgeVisible: user.roles.includes('admin') && user.isActive
  };
}
```

```ts
vm$ = this.user$.pipe(map(u => (u ? toVm(u) : null)));
```

```html
<div *ngIf="(vm$ | async) as vm">
  {{ vm.fullName }}
  <span *ngIf="vm.isAdminBadgeVisible">ADMIN</span>
</div>
```

### 5.3 Pipes : quand les utiliser
- **OK** pour transformation simple, pure, réutilisable.
- **À éviter** : pipes impures ou coûteuses.

### 5.4 Gestion des subscriptions
- Préférer l’`async` pipe.
- Sinon : `takeUntilDestroyed()` (Angular >= 16) ou `DestroyRef`.

```ts
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';

this.api.data$
  .pipe(takeUntilDestroyed(this.destroyRef))
  .subscribe();
```

---

## 6) Services spécialisés

### 6.1 Responsabilités typiques
- **Api service** : appels HTTP, mapping DTO ↔ modèle.
- **Facade / Store** : orchestration, état, commandes.
- **Domain service** : règles métier (validation, calcul, policies).
- **Util service** : fonctions techniques réutilisables (format, parsing), idéalement pures.

### 6.2 Exemple : Api service + mapping
```ts
export type UserDto = { id: string; first_name: string; last_name: string; };
export type User = { id: string; firstName: string; lastName: string; };

@Injectable({ providedIn: 'root' })
export class UsersApi {
  constructor(private http: HttpClient) {}

  getUser(id: string) {
    return this.http.get<UserDto>(`/api/users/${id}`).pipe(
      map(dto => ({
        id: dto.id,
        firstName: dto.first_name,
        lastName: dto.last_name
      }))
    );
  }
}
```

### 6.3 Anti-patterns services
- “God service” : un service qui fait tout.
- Services qui manipulent directement le DOM (préférer `Renderer2`, directives, ou service d’abstraction).

---

## 7) Dépendances explicites (DI propre)

### 7.1 Pourquoi rendre les dépendances explicites ?
- Testabilité (mocks)
- Remplaçabilité (implémentations multiples)
- Lecture du code (contrats)

### 7.2 Utiliser des abstractions via injection tokens
Utile quand on veut dépendre d’un **contrat** plutôt que d’une classe concrète.

```ts
export interface Clock {
  now(): Date;
}

export const CLOCK = new InjectionToken<Clock>('CLOCK');

export class SystemClock implements Clock {
  now() { return new Date(); }
}

@NgModule({
  providers: [{ provide: CLOCK, useClass: SystemClock }]
})
export class CoreModule {}

@Injectable()
export class AuditService {
  constructor(@Inject(CLOCK) private clock: Clock) {}
}
```

### 7.3 Éviter les singletons accidentels
- Attention aux services `providedIn: 'root'` contenant de l’état mutable.
- Si état par feature/composant : fournir au niveau du composant/feature module.

### 7.4 Inputs/Outputs = dépendances explicites côté composants
- Préférer passer les données en `@Input()` plutôt que de “tirer” depuis des services globaux.

---

## 8) Gestion d’état claire

### 8.1 État : local vs partagé vs global
- **Local** : état de formulaire, toggle, UI locale → dans le composant.
- **Partagé (feature)** : liste + filtres + sélection → facade/service fourni dans la feature.
- **Global** : session, permissions, préférences → store global (NgRx, Akita, Signals store, etc.).

### 8.2 Modéliser les états de chargement
Recommandation : un modèle de type *request state*.

```ts
type Loadable<T> =
  | { state: 'idle' }
  | { state: 'loading' }
  | { state: 'loaded'; data: T }
  | { state: 'error'; error: unknown };
```

### 8.3 Flux unidirectionnel
- Actions/Events -> Mise à jour état -> Projection VM -> Vue.
- Interdire les modifications directes de données venant du store.

### 8.4 Immutabilité
- Facilite OnPush, le debug, et la prédictibilité.

### 8.5 Exemple simple de “mini store”
```ts
@Injectable()
export class ProductsFacade {
  private readonly state$ = new BehaviorSubject<Loadable<Product[]>>({ state: 'idle' });

  readonly vm$ = this.state$.pipe(
    map(s => ({
      loading: s.state === 'loading',
      products: s.state === 'loaded' ? s.data : [],
      error: s.state === 'error' ? s.error : null
    }))
  );

  constructor(private api: ProductsApi) {}

  load() {
    this.state$.next({ state: 'loading' });
    this.api.list().subscribe({
      next: data => this.state$.next({ state: 'loaded', data }),
      error: error => this.state$.next({ state: 'error', error })
    });
  }
}
```

---

## 9) Nommage et conventions

### 9.1 Objectifs
- Lire la structure sans ouvrir les fichiers.
- Standardiser pour réduire la charge cognitive.

### 9.2 Conventions recommandées (exemples)
- Composants : `user-profile.component.ts`, selector `app-user-profile`.
- Containers : `user-profile.container.ts` ou `user-profile-page.component.ts`.
- Services API : `users.api.ts` ou `users-api.service.ts`.
- Facades : `user-profile.facade.ts`.
- Types : `user.model.ts`, `user.dto.ts`.

### 9.3 Noms orientés intention
- Éviter `data`, `info`, `obj`, `service2`.
- Préférer `selectedUserId`, `isSaving`, `loadUser()`, `saveChanges()`.

---

## 10) Limitation du couplage entre couches

### 10.1 Couches typiques
- **UI (presentation)** : composants, directives, pipes.
- **Application/Orchestration** : facades, stores.
- **Domain** : règles métier, policies, types.
- **Infrastructure/Data** : HTTP, storage, adapter.

### 10.2 Règles pratiques
- Le composant ne doit pas connaître les DTO API.
- Le domaine ne dépend pas d’Angular.
- Les règles métier ne vivent pas dans un `component.ts`.

### 10.3 Adapter pattern (DTO ↔ domaine)
- Centraliser les conversions.

```ts
export function userFromDto(dto: UserDto): User {
  return { id: dto.id, firstName: dto.first_name, lastName: dto.last_name };
}
```

### 10.4 Éviter les imports transversaux
- Une feature ne doit pas dépendre d’une autre feature via import direct.
- Passer par un **core**, des **libs partagées**, ou un contrat.

---

## 11) Check-list revue de code (pratique)

### Composants
- [ ] < 200-300 lignes et responsabilité claire
- [ ] `OnPush` utilisé par défaut (sauf besoin)
- [ ] Pas d’appels métier dans le template
- [ ] Pas de subscriptions non gérées

### Services
- [ ] Services spécialisés (API/domain/facade)
- [ ] Pas d’état global mutable non maîtrisé
- [ ] Mappings DTO centralisés

### État
- [ ] État modélisé (loading/error/empty)
- [ ] Flux unidirectionnel
- [ ] Immutabilité respectée

### Couplage
- [ ] UI ne dépend pas de DTO
- [ ] Dépendances explicites (DI, tokens, inputs)
- [ ] Faible couplage inter-features

---

## 12) Atelier final (guidé)

### Objectif
Refactorer un mini-feature “Users” initialement monolithique en :
- 1 composant container
- 1 composant de présentation
- 1 service API
- 1 facade/store local
- 1 mapping DTO → modèle

### Étapes
1. Identifier la logique métier dans le component.
2. Extraire `UsersApi`.
3. Introduire une `UsersFacade` avec un état `Loadable`.
4. Construire un `Vm` consommé par la vue.
5. Passer le composant de présentation en `OnPush`.
6. Ajouter une check-list de validation.

### Critères de réussite
- Templates lisibles
- Composants testables
- Services cohérents
- Flux data clair

---

## 13) Annexes

### A) Arborescence type (feature-first)
```txt
src/app/
  core/
  shared/
  features/
    users/
      data-access/
        users.api.ts
        users.mapper.ts
      ui/
        user-list/
          user-list.component.ts
          user-list.component.html
      feature-users-page/
        users-page.component.ts
      users.facade.ts
      users.routes.ts
```

### B) Rappels performance
- `trackBy` sur `*ngFor`
- Éviter les fonctions dans le template
- `OnPush` + immutabilité

---

## 14) Conclusion
Une conception Angular maintenable repose sur :
- des composants simples,
- une séparation nette UI / orchestration / domaine / data,
- des dépendances explicites,
- une gestion d’état structurée,
- des conventions cohérentes,
- et un couplage maîtrisé.
