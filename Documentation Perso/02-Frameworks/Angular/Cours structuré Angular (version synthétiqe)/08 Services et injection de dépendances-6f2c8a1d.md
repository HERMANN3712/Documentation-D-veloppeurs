# 08 — Services et injection de dépendances (DI) Angular

## Objectifs pédagogiques
À la fin de ce module, vous serez capable de :

- Expliquer le rôle d’un **service Angular** et quand l’utiliser.
- Comprendre le fonctionnement de l’**injection de dépendances** (DI) dans Angular.
- Créer et fournir un service (provider) au bon niveau (**root**, module, composant).
- Injecter un service dans un composant, un autre service, un guard, un resolver.
- Gérer le cycle de vie et la portée (scope) d’une instance de service.
- Appliquer de bonnes pratiques : séparation des responsabilités, testabilité, typage, immutabilité.

---

## Prérequis
- TypeScript (classes, interfaces, génériques)
- Angular de base (composants, modules, templates)
- Notions HTTP/RxJS utiles (Observables), mais non obligatoires pour comprendre la DI.

---

## Plan de la formation

1. Introduction : pourquoi des services ?
2. Comprendre l’injection de dépendances (DI)
3. Créer un service avec le CLI
4. Fournir un service : `providedIn`, `providers` et hiérarchie des injecteurs
5. Injecter un service : constructeur, `inject()`, paramètres optionnels
6. Patterns courants avec services
   - State / partage de données
   - Facade
   - Service d’accès API
   - Utilitaires et configuration
7. Gestion du cycle de vie et des scopes
8. Anti‑patterns et bonnes pratiques
9. Tests unitaires de services et de composants injectant des services
10. Exercices guidés + corrigés

---

## 1) Introduction : pourquoi des services ?

Dans Angular, un **service** est une classe (souvent décorée avec `@Injectable`) dont la mission est de **centraliser une logique** ou de **partager des données** entre plusieurs éléments de l’application.

### Quand créer un service ?

- Logique métier réutilisée par plusieurs composants (ex. calculs, règles, orchestration).
- Accès à des ressources externes (HTTP, `localStorage`, WebSocket).
- Partage d’un état entre composants “frères” ou non liés directement.
- Encapsulation d’une dépendance (ex. librairie externe) pour faciliter le test et l’évolution.

### Ce que les services évitent

- Dupliquer du code dans plusieurs composants.
- Mettre trop de logique dans les composants (ils deviennent difficiles à maintenir).
- Coupler fortement la vue à la logique métier.

---

## 2) Comprendre l’injection de dépendances (DI)

Angular fournit un container de DI : un mécanisme capable de :

1. **Créer** une instance d’une dépendance (un service, par exemple).
2. **Conserver** cette instance selon une portée (scope).
3. **Injecter** cette instance là où elle est demandée.

### Concept clé

Au lieu d’écrire :

```ts
const service = new MyService();
```

Angular fait l’inversion de contrôle : le composant déclare ce dont il a besoin, et le framework fournit l’instance.

### Vocabulaire DI Angular

- **Token** : identifiant d’un provider (souvent une classe, mais peut être un `InjectionToken`).
- **Provider** : règle de création/résolution (ex. `useClass`, `useValue`, `useFactory`).
- **Injector** : moteur qui résout un token en instance.
- **Hiérarchie** : injecteurs imbriqués (root → module → composant → directives…).

### Exemple simple

```ts
@Injectable({ providedIn: 'root' })
export class LoggerService {
  log(message: string) {
    console.log('[LOG]', message);
  }
}

@Component({
  selector: 'app-demo',
  template: `<button (click)="run()">Run</button>`
})
export class DemoComponent {
  constructor(private logger: LoggerService) {}

  run() {
    this.logger.log('Action exécutée');
  }
}
```

Ici, `LoggerService` est fourni au niveau root ⇒ singleton applicatif (dans la plupart des cas).

---

## 3) Créer un service avec le CLI

Commande :

```bash
ng generate service core/logger
# ou
ng g s core/logger
```

Génère typiquement :

```ts
import { Injectable } from '@angular/core';

@Injectable({
  providedIn: 'root'
})
export class LoggerService {
  constructor() {}
}
```

### `@Injectable` : pourquoi ?

- Indique qu’Angular peut injecter des dépendances dans ce service.
- Permet de définir la stratégie de fourniture (`providedIn`).

---

## 4) Fournir un service : `providedIn`, `providers` et hiérarchie des injecteurs

### 4.1 `providedIn: 'root'`

```ts
@Injectable({ providedIn: 'root' })
export class UserService {}
```

- Service disponible partout.
- Instance partagée (singleton) dans l’application.
- Tree‑shakable (peut être retiré du bundle si inutilisé).

### 4.2 `providers` au niveau composant

```ts
@Component({
  selector: 'app-feature',
  templateUrl: './feature.component.html',
  providers: [CartService]
})
export class FeatureComponent {
  constructor(public cart: CartService) {}
}
```

- **Nouvelle instance** de `CartService` pour ce composant **et ses enfants**.
- Utile pour isoler un état par section, par onglet, par instance de composant.

### 4.3 `providers` au niveau module (cas des modules)

Dans les projets avec modules (ou libs), on peut fournir dans un module :

```ts
@NgModule({
  providers: [SomeService]
})
export class FeatureModule {}
```

Note : avec les **standalone APIs** (Angular récent), on fournit souvent via `bootstrapApplication`/`providers` ou `providedIn`.

### 4.4 Hiérarchie des injecteurs

Règle : Angular cherche le provider dans l’injecteur le plus proche, puis remonte.

- Injecteur composant
- Injecteur parent
- …
- Injecteur root

Conséquence : si un token est fourni à plusieurs niveaux, la résolution dépend de la position dans l’arbre.

---

## 5) Injecter un service

### 5.1 Injection via le constructeur (classique)

```ts
constructor(private api: ApiService, private router: Router) {}
```

- Simple, lisible.
- Fonctionne très bien avec la plupart des cas.

### 5.2 API `inject()` (Angular moderne)

```ts
import { Component, inject } from '@angular/core';

@Component({
  selector: 'app-inject-demo',
  template: `...`
})
export class InjectDemoComponent {
  private api = inject(ApiService);

  load() {
    return this.api.getUsers();
  }
}
```

Avantages :
- Pratique dans les classes où le constructeur est moins souhaitable.
- Utilisable aussi dans des fonctions factory.

### 5.3 Dépendance optionnelle

```ts
import { Optional } from '@angular/core';

constructor(@Optional() private logger?: LoggerService) {}
```

Si aucun provider n’existe, `logger` vaut `undefined` au lieu de lever une erreur.

### 5.4 Self/SkipSelf/Host (contrôle de résolution)

- `@Self()` : uniquement l’injecteur local.
- `@SkipSelf()` : ignore l’injecteur local, cherche chez le parent.
- `@Host()` : limite la recherche à la frontière d’un composant hôte.

Cas d’usage : composants réutilisables, directives qui doivent s’ancrer à un parent spécifique.

---

## 6) Patterns courants avec services

### 6.1 Service de partage d’état (store simple)

Objectif : partager un état entre plusieurs composants.

```ts
@Injectable({ providedIn: 'root' })
export class ThemeService {
  private themeSubject = new BehaviorSubject<'light' | 'dark'>('light');
  theme$ = this.themeSubject.asObservable();

  setTheme(theme: 'light' | 'dark') {
    this.themeSubject.next(theme);
  }
}
```

Dans un composant :

```ts
@Component({
  selector: 'app-toolbar',
  template: `
    <button (click)="set('light')">Light</button>
    <button (click)="set('dark')">Dark</button>
  `
})
export class ToolbarComponent {
  constructor(private theme: ThemeService) {}
  set(t: 'light' | 'dark') { this.theme.setTheme(t); }
}
```

### 6.2 Façade (Facade Pattern)

Une façade expose une API métier simple, et cache des détails (HTTP, caching, mapping, erreurs…).

```ts
@Injectable({ providedIn: 'root' })
export class UsersFacade {
  constructor(private api: UsersApiService) {}

  getUsers() {
    return this.api.list();
  }
}
```

### 6.3 Service d’accès API (HTTP)

```ts
@Injectable({ providedIn: 'root' })
export class UsersApiService {
  constructor(private http: HttpClient) {}

  list() {
    return this.http.get<User[]>('/api/users');
  }
}
```

Bonnes pratiques :
- Définir des modèles (`User`) et typer les retours.
- Isoler les URLs (environments, constants).
- Centraliser la gestion d’erreur si nécessaire.

### 6.4 InjectionToken pour configuration

Quand la dépendance n’est pas une classe :

```ts
import { InjectionToken } from '@angular/core';

export interface ApiConfig {
  baseUrl: string;
}

export const API_CONFIG = new InjectionToken<ApiConfig>('API_CONFIG');
```

Fournir :

```ts
bootstrapApplication(AppComponent, {
  providers: [
    { provide: API_CONFIG, useValue: { baseUrl: 'https://example.com' } }
  ]
});
```

Consommer :

```ts
@Injectable({ providedIn: 'root' })
export class ApiService {
  constructor(@Inject(API_CONFIG) private config: ApiConfig) {}
}
```

### 6.5 `useClass`, `useExisting`, `useFactory`

#### `useClass` (implémentation alternative)

```ts
{ provide: LoggerService, useClass: AdvancedLoggerService }
```

#### `useExisting` (alias)

```ts
{ provide: 'LEGACY_LOGGER', useExisting: LoggerService }
```

#### `useFactory` (construction dynamique)

```ts
export function loggerFactory(isProd: boolean) {
  return isProd ? new ProdLogger() : new DevLogger();
}

{ provide: LoggerService, useFactory: () => loggerFactory(environment.production) }
```

---

## 7) Cycle de vie, portée et implications

### Singleton vs instance par composant

- `providedIn: 'root'` : une instance partagée (souvent recommandé pour services stateless ou single source of truth).
- `providers` d’un composant : une instance **par instance de composant**.

### Impacts

- **Mémoire** : plusieurs instances peuvent être coûteuses.
- **État** : instance par composant = état isolé (parfois souhaité).
- **Tests** : un bon découplage rend le mocking facile.

### Destruction

- Les services root vivent généralement jusqu’à la fin de l’application.
- Les services fournis par composant sont détruits avec le composant (en pratique quand il est retiré de l’arbre).

---

## 8) Anti‑patterns et bonnes pratiques

### Anti‑patterns

- Services “fourre‑tout” : un service énorme qui fait tout.
- Mettre l’état global partout sans stratégie (risque d’effets de bords).
- Injecter des composants dans des services (couplage à la vue).
- Logique métier dans le template ou dans le composant par facilité.

### Bonnes pratiques

- Un service = une responsabilité claire.
- Favoriser des APIs explicites (méthodes nommées, types précis).
- Éviter les `any`.
- Préférer les Observables (ou signals selon stratégie) pour exposer de l’état.
- Documenter les side‑effects (écriture storage, navigation, etc.).

---

## 9) Tests unitaires

### 9.1 Tester un service

```ts
describe('ThemeService', () => {
  let service: ThemeService;

  beforeEach(() => {
    TestBed.configureTestingModule({});
    service = TestBed.inject(ThemeService);
  });

  it('should set theme', (done) => {
    service.theme$.subscribe(t => {
      expect(t).toBe('dark');
      done();
    });

    service.setTheme('dark');
  });
});
```

### 9.2 Tester un composant avec mock service

```ts
class ThemeServiceMock {
  theme$ = of<'light' | 'dark'>('light');
  setTheme = jasmine.createSpy('setTheme');
}

TestBed.configureTestingModule({
  declarations: [ToolbarComponent],
  providers: [{ provide: ThemeService, useClass: ThemeServiceMock }]
});
```

---

## 10) Exercices (guidés)

### Exercice 1 — Créer un service de panier

**Objectif :** centraliser la logique du panier.

- Créer `CartService`
- Exposer :
  - `items$` (Observable)
  - `add(item)`
  - `remove(id)`

**Indice :** utiliser un `BehaviorSubject`.

#### Correction (exemple)

```ts
export interface CartItem {
  id: string;
  label: string;
  price: number;
}

@Injectable({ providedIn: 'root' })
export class CartService {
  private itemsSubject = new BehaviorSubject<CartItem[]>([]);
  items$ = this.itemsSubject.asObservable();

  add(item: CartItem) {
    const items = this.itemsSubject.value;
    this.itemsSubject.next([...items, item]);
  }

  remove(id: string) {
    const items = this.itemsSubject.value;
    this.itemsSubject.next(items.filter(i => i.id !== id));
  }
}
```

### Exercice 2 — Fournir un service au niveau composant

**Objectif :** isoler un panier par onglet.

- Ajouter `providers: [CartService]` sur un composant `TabComponent`.
- Vérifier que chaque onglet a son propre état.

### Exercice 3 — InjectionToken de configuration

**Objectif :** configurer `baseUrl` d’une API via DI.

- Créer `API_CONFIG` et `ApiConfig`.
- Injecter la config dans `UsersApiService`.

---

## Résumé

- Les **services** centralisent la logique et facilitent le partage de données.
- La **DI** d’Angular fournit les services via des **providers** et une **hiérarchie d’injecteurs**.
- `providedIn: 'root'` est la stratégie la plus courante (singleton + tree‑shaking).
- Le scope (root vs composant) influence l’**état**, la **mémoire** et la **testabilité**.

---

## Annexes : mini‑glossaire

- **Singleton** : instance unique partagée.
- **Provider** : définition de comment construire/résoudre une dépendance.
- **Token** : clé d’accès à une dépendance.
- **Injector** : container de dépendances.
