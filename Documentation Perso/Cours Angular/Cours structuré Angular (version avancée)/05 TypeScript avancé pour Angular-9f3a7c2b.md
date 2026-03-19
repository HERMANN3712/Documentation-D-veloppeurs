# Formation – TypeScript avancé pour Angular

**Public** : développeurs Angular (intermédiaire → avancé)  
**Pré-requis** : bases TypeScript, RxJS, Angular CLI, composants/services, DI, modules/standalone, tests (souhaité).  
**Durée conseillée** : 2 jours (14 h) ou 4 demi-journées  
**Format** : cours + ateliers guidés + mini-projet fil rouge  

---

## Objectifs pédagogiques
À la fin de la formation, le participant sera capable de :

1. Modéliser des API/front en utilisant **interfaces**, **types**, **union types** et **discriminated unions**.
2. Exploiter les **types utilitaires** et les **mapped types** pour réduire la duplication.
3. Écrire des **génériques** robustes (fonctions, classes, services Angular, operators RxJS typés).
4. Appliquer le **type narrowing** et la création de **type guards** pour un code sûr.
5. Utiliser **readonly** et les patterns d’immutabilité (notamment pour l’état et les inputs).
6. Comprendre et employer les **decorators** TypeScript/Angular et créer ses propres décorateurs utilitaires.
7. Concevoir des **classes abstraites** et des contrats extensibles pour des composants/services réutilisables.
8. Construire des types avancés (mapping, inférence, template literal types) utiles dans un projet Angular.

---

## Plan de la formation

1. **Rappels TypeScript indispensables dans Angular** (strict mode, type vs interface, config)
2. **Interfaces & alias de types pour modéliser le domaine**
3. **Union types & discriminated unions** (UI state, API responses)
4. **Type narrowing & type guards** (runtime ↔ compile-time)
5. **Génériques avancés** (fonctions, classes, services, RxJS)
6. **Types utilitaires** (Partial, Pick, Omit, Record, ReturnType, etc.)
7. **Mapped types & patterns de mapping** (readonly, optional, transforms)
8. **Readonly, immutabilité et design Angular** (Inputs, state, signals)
9. **Decorators** : Angular + TypeScript (création et usage)
10. **Classes abstraites & architecture** (bases de composants, repos, adaptateurs)
11. **Atelier fil rouge** : refactor d’un mini-projet Angular avec TS avancé
12. **Bonnes pratiques, pièges, guidelines et checklists**

---

# 1) Rappels TypeScript indispensables dans Angular

## 1.1 Strictness : la base d’un Angular "sûr"
Angular moderne s’appuie fortement sur TypeScript. Pour profiter du typage, on active des options strictes.

**tsconfig.json (extrait recommandé)**
```json
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true
  }
}
```

### Impacts clés
- `strictNullChecks` : oblige à gérer `null | undefined` → moins de bugs d’UI.
- `noUncheckedIndexedAccess` : l’accès par index peut retourner `undefined` → sécurise les map/dicos.
- `exactOptionalPropertyTypes` : différencie `prop?: string` de `prop: string | undefined`.

## 1.2 `interface` vs `type`
- **`interface`** : idéal pour des "contrats" extensibles (open/merge, `extends`).
- **`type`** : idéal pour unions/intersections, mapped types, utilitaires complexes.

Exemple :
```ts
interface User {
  id: string;
  email: string;
}

type UserId = User['id'];

type ApiResult<T> =
  | { ok: true; data: T }
  | { ok: false; error: string };
```

---

# 2) Interfaces & alias de types для modéliser le domaine

## 2.1 Modéliser un DTO vs un modèle métier
**DTO** : reflète l’API (souvent imparfait, nullable, champs optionnels).  
**Modèle** : reflète vos règles (plus strict, invariants).

```ts
// DTO venant de l’API
export interface UserDto {
  id: string;
  email?: string | null;
  first_name?: string | null;
  last_name?: string | null;
}

// Modèle interne
export interface User {
  id: string;
  email: string;
  fullName: string;
}

export function toUser(dto: UserDto): User {
  return {
    id: dto.id,
    email: dto.email ?? '',
    fullName: `${dto.first_name ?? ''} ${dto.last_name ?? ''}`.trim(),
  };
}
```

## 2.2 Composition par intersections
```ts
type WithTimestamps = { createdAt: string; updatedAt: string };

type UserEntity = User & WithTimestamps;
```

## 2.3 Patterns Angular : Inputs/Outputs typés
```ts
export interface UserCardVm {
  id: string;
  displayName: string;
  isActive: boolean;
}

// Input typé + readonly (voir section immutabilité)
@Input({ required: true }) user!: Readonly<UserCardVm>;
```

### Atelier 1
- Transformer une réponse API (DTO) en VM (ViewModel) strict et non-nullable.
- Mettre en place des fonctions `toXxx` et valider au compilation-time.

---

# 3) Union types & discriminated unions

## 3.1 Union types pour des valeurs contrôlées
Cas Angular : variantes d’affichage.
```ts
type Theme = 'light' | 'dark' | 'system';

function setTheme(theme: Theme) {
  // ...
}

setTheme('dark');
// setTheme('blue'); // ❌ compile-time
```

## 3.2 Discriminated unions : modéliser les états UI
Très utile pour :
- chargement
- erreur
- données prêtes

```ts
type LoadState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'error'; message: string }
  | { status: 'success'; data: T };

const state: LoadState<string[]> = { status: 'loading' };
```

### Utilisation dans un composant
```ts
state: LoadState<User[]> = { status: 'idle' };

loadUsers() {
  this.state = { status: 'loading' };
  this.userApi.getAll().subscribe({
    next: (users) => this.state = { status: 'success', data: users },
    error: (e) => this.state = { status: 'error', message: String(e) },
  });
}
```

### Template Angular
```html
<ng-container [ngSwitch]="state.status">
  <p *ngSwitchCase="'idle'">Prêt.</p>
  <p *ngSwitchCase="'loading'">Chargement...</p>
  <p *ngSwitchCase="'error'">Erreur: {{ state.message }}</p>

  <ul *ngSwitchCase="'success'">
    <li *ngFor="let u of state.data">{{ u.fullName }}</li>
  </ul>
</ng-container>
```

### Exhaustiveness check
```ts
function assertNever(x: never): never {
  throw new Error('Unexpected case: ' + JSON.stringify(x));
}

function render<T>(s: LoadState<T>): string {
  switch (s.status) {
    case 'idle': return 'Idle';
    case 'loading': return 'Loading';
    case 'error': return `Error: ${s.message}`;
    case 'success': return 'Ok';
    default: return assertNever(s);
  }
}
```

### Atelier 2
- Remplacer un `boolean isLoading + error + data` par un `LoadState<T>`.
- Ajouter un `assertNever` pour couvrir tous les cas.

---

# 4) Type narrowing & type guards (runtime ↔ compile-time)

## 4.1 Narrowing natif
```ts
function formatEmail(email: string | null) {
  if (!email) return '—';
  return email.toLowerCase();
}
```

## 4.2 "in" operator et narrowing sur unions d’objets
```ts
type ApiError = { error: string; code?: number };

type ApiOk<T> = { data: T };

type ApiResponse<T> = ApiOk<T> | ApiError;

function isOk<T>(r: ApiResponse<T>): r is ApiOk<T> {
  return 'data' in r;
}
```

## 4.3 Type guards réutilisables
```ts
export function isNonNullable<T>(v: T): v is NonNullable<T> {
  return v !== null && v !== undefined;
}

const values = [1, null, 2, undefined].filter(isNonNullable); // number[]
```

## 4.4 Validation : quand TS ne suffit pas
TypeScript ne valide pas au runtime : si vous parsez du JSON, il faut valider.

Approche : type guard + checks.
```ts
type UserJson = { id: unknown; email: unknown };

type User = { id: string; email: string };

function isUser(x: any): x is User {
  return x && typeof x.id === 'string' && typeof x.email === 'string';
}
```

### Atelier 3
- Écrire des guards `isUser`, `isUserDto`, et sécuriser un `map(dto => ...)`.

---

# 5) Génériques avancés (fonctions, classes, services, RxJS)

## 5.1 Fonctions génériques
```ts
function identity<T>(value: T): T {
  return value;
}

const a = identity(123); // number
const b = identity('abc'); // string
```

## 5.2 Contraintes (`extends`) et clé de propriété
Cas typique : accès sûr à une propriété.
```ts
function pluck<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const user = { id: 'u1', email: 'a@b.com' };
const id = pluck(user, 'id'); // string
```

## 5.3 Services Angular génériques (Repository pattern)
```ts
export interface Identifiable {
  id: string;
}

export abstract class CrudRepository<T extends Identifiable> {
  abstract getAll(): Observable<T[]>;
  abstract getById(id: T['id']): Observable<T>;
  abstract create(input: Omit<T, 'id'>): Observable<T>;
}
```

### Implémentation
```ts
@Injectable({ providedIn: 'root' })
export class UserRepository extends CrudRepository<User> {
  constructor(private http: HttpClient) { super(); }

  getAll() {
    return this.http.get<User[]>('/api/users');
  }

  getById(id: string) {
    return this.http.get<User>(`/api/users/${id}`);
  }

  create(input: Omit<User, 'id'>) {
    return this.http.post<User>('/api/users', input);
  }
}
```

## 5.4 RxJS : opérateurs typés + inference
Exemple : le `map` conserve le type.

```ts
this.http.get<UserDto[]>('/api/users').pipe(
  map(dtos => dtos.map(toUser)), // User[]
  map(users => users.filter(u => u.email.length > 0))
);
```

### Atelier 4
- Écrire un `CrudRepository<T>` générique et l’utiliser pour 2 ressources (User, Product).

---

# 6) Types utilitaires (briques essentielles)

## 6.1 Les plus courants
- `Partial<T>` : toutes propriétés optionnelles.
- `Required<T>` : toutes propriétés requises.
- `Pick<T, K>` : sous-ensemble de propriétés.
- `Omit<T, K>` : exclusion de propriétés.
- `Record<K, T>` : dictionnaire typé.
- `Readonly<T>` : propriétés immuables.
- `NonNullable<T>` : exclut `null | undefined`.

Exemples :
```ts
type UserPatch = Partial<Pick<User, 'email' | 'fullName'>>;

type UserCreate = Omit<User, 'id'>;

type UsersById = Record<User['id'], User>;
```

## 6.2 Utilitaires d’inférence
- `ReturnType<F>`
- `Parameters<F>`
- `Awaited<T>`

```ts
function buildUserVm(u: User) {
  return { id: u.id, label: u.fullName };
}

type UserVm = ReturnType<typeof buildUserVm>;
```

### Atelier 5
- Définir `CreateDto`, `UpdateDto`, `PatchDto` avec `Omit/Partial/Pick` à partir d’un type source.

---

# 7) Mapped types & patterns de mapping

## 7.1 Mapped type simple
```ts
type Nullable<T> = { [K in keyof T]: T[K] | null };

type UserNullable = Nullable<User>;
```

## 7.2 `as` dans mapped types (remapping de clés)
Exemple : générer des getters.
```ts
type Getterify<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K]
};

type UserGetters = Getterify<User>;
// getId(): string, getEmail(): string, getFullName(): string
```

## 7.3 Deep readonly (attention aux limites)
```ts
type Primitive = string | number | boolean | bigint | symbol | null | undefined;

type DeepReadonly<T> =
  T extends Primitive ? T :
  T extends (...args: any[]) => any ? T :
  { readonly [K in keyof T]: DeepReadonly<T[K]> };

type FrozenUser = DeepReadonly<User>;
```

### Cas Angular
- Éviter de muter accidentellement des `@Input()`.
- Aider au pattern "immutabilité" (state reducers).

### Atelier 6
- Créer `DeepPartial<T>` et `DeepReadonly<T>` et l’appliquer à un state complexe.

---

# 8) `readonly`, immutabilité et design Angular

## 8.1 `readonly` sur propriétés
```ts
class Session {
  constructor(public readonly userId: string) {}
}
```

## 8.2 `ReadonlyArray<T>` et usage dans les composants
```ts
users: ReadonlyArray<User> = [];

// Toujours faire une copie pour modifier
addUser(u: User) {
  this.users = [...this.users, u];
}
```

## 8.3 Inputs immuables
```ts
@Component({
  selector: 'app-user-list',
  template: `...`
})
export class UserListComponent {
  @Input({ required: true })
  users!: ReadonlyArray<Readonly<User>>;
}
```

## 8.4 Immutabilité et refactor facilité
- Un composant pur est plus simple à tester.
- Moins d’effets de bord.
- Les signatures (types) guident les changements.

---

# 9) Decorators : Angular + TypeScript (création et usage)

## 9.1 Decorators dans Angular
- `@Component`, `@Directive`, `@Injectable`, `@Pipe`
- `@Input`, `@Output`, `@HostListener`, `@HostBinding`

Rappel : ce sont des **métadonnées** interprétées par Angular.

## 9.2 Decorator TypeScript : principe
Un décorateur est une fonction appliquée à : classe, propriété, méthode, paramètre.

## 9.3 Créer un property decorator utilitaire
Exemple pédagogique : logger les changements d’une propriété (attention : patterns à utiliser avec parcimonie).

```ts
export function LogProperty(prefix = '') {
  return function (target: any, propertyKey: string) {
    let value: any;

    Object.defineProperty(target, propertyKey, {
      get() { return value; },
      set(newValue) {
        console.log(prefix + propertyKey, { newValue });
        value = newValue;
      },
      enumerable: true,
      configurable: true,
    });
  };
}

class Demo {
  @LogProperty('[Demo] ')
  name = 'initial';
}
```

### Mise en garde
- Les decorators custom peuvent surprendre (difficiles à tracer/debugger).
- Ils peuvent interagir avec les mécanismes Angular (change detection, AOT).
- Préférer des solutions explicites (fonctions utilitaires, interceptors, wrappers) quand possible.

### Atelier 7
- Créer un décorateur `@Debounce(ms)` pour une méthode (ex: handler) et l’utiliser en composant.

---

# 10) Classes abstraites & architecture

## 10.1 Pourquoi des classes abstraites ?
- Partager du comportement
- Imposer un contrat
- Réduire duplication tout en gardant une structure claire

## 10.2 Base component abstrait
Exemple : composant qui gère un `LoadState<T>`.

```ts
export abstract class LoadableComponent<T> {
  state: LoadState<T> = { status: 'idle' };

  protected setLoading() { this.state = { status: 'loading' }; }
  protected setError(message: string) { this.state = { status: 'error', message }; }
  protected setData(data: T) { this.state = { status: 'success', data }; }
}

@Component({
  selector: 'app-users',
  templateUrl: './users.component.html'
})
export class UsersComponent extends LoadableComponent<User[]> {
  constructor(private repo: UserRepository) { super(); }

  ngOnInit() {
    this.setLoading();
    this.repo.getAll().subscribe({
      next: (users) => this.setData(users),
      error: (e) => this.setError(String(e)),
    });
  }
}
```

## 10.3 Abstractions HTTP : adaptateurs et normalisation
- `UserDto` → `User`
- Gestion d’erreurs typées
- `ApiResult<T>` uniformisé

```ts
type ApiResult<T> =
  | { ok: true; data: T }
  | { ok: false; error: { message: string; status?: number } };

abstract class ApiClient {
  protected toResult<T>(obs: Observable<T>): Observable<ApiResult<T>> {
    return obs.pipe(
      map(data => ({ ok: true, data } as const)),
      catchError((e: any) => of({
        ok: false,
        error: { message: String(e?.message ?? e), status: e?.status }
      } as const))
    );
  }
}
```

### Atelier 8
- Factoriser la gestion `loading/error/success` via une classe abstraite.
- Adapter 2 composants existants.

---

# 11) Atelier fil rouge (mini-projet)

## Contexte
Vous disposez d’une mini-app Angular :
- liste d’utilisateurs
- détail utilisateur
- édition
- API mock (JSON)

Le code initial contient :
- `any` et types faibles
- gestion d’état via `isLoading`, `error`, `data` séparés
- DTO non transformés

## Étapes
1. Activer/renforcer les options strictes TS.
2. Introduire `UserDto`, `User`, `UserVm`.
3. Mettre en place `LoadState<T>` pour les vues.
4. Ajouter des type guards sur parsing/validation.
5. Créer un `CrudRepository<T>` générique + implémentations.
6. Employer `Partial/Pick/Omit` pour `CreateUser`, `UpdateUser`.
7. Utiliser `DeepReadonly` sur inputs et state.
8. (Option) Ajouter un décorateur `@Debounce` sur des handlers de recherche.

## Critères de réussite
- Zéro `any` dans les zones refactorées
- Templates compatibles avec états discriminés (exhaustivité)
- Services fortement typés, signatures stables
- Refactor facile (changement d’un champ → propagation guidée)

---

# 12) Bonnes pratiques, pièges, guidelines et checklists

## 12.1 Quand préférer `interface`
- Contrats publics de librairie
- Modèles évolutifs (`extends`)

## 12.2 Quand préférer `type`
- Unions, intersections, mapped types, helper types

## 12.3 Éviter
- `any` (préférer `unknown` + narrowing)
- Sur-typage (types trop abstraits difficiles à lire)
- Décorateurs custom partout

## 12.4 Checklist refactor TypeScript avancé (Angular)
- [ ] `strict` activé et corrections principales faites
- [ ] DTO séparés des modèles & mapping centralisé
- [ ] États UI modélisés en discriminated union
- [ ] Guards utilitaires (`isNonNullable`, `isXxx`)
- [ ] Génériques pour réduire duplication (repo, helpers)
- [ ] Utilitaires (`Partial`, `Omit`, `Pick`, `Record`) employés de façon lisible
- [ ] `readonly` sur inputs/state importants
- [ ] `assertNever` sur switch d’union pour exhaustivité

---

## Annexes

### Annex A – Snippets utiles

**`assertNever`**
```ts
export function assertNever(x: never): never {
  throw new Error('Unexpected object: ' + x);
}
```

**`isNonNullable`**
```ts
export const isNonNullable = <T>(v: T): v is NonNullable<T> => v != null;
```

**`DeepPartial`**
```ts
type DeepPartial<T> = {
  [K in keyof T]?: T[K] extends object ? DeepPartial<T[K]> : T[K];
};
```

### Annex B – Suggestions d’évaluation
- QCM (strictNullChecks, narrowing, unions)
- Exercice : convertir `boolean flags` → `LoadState<T>`
- Exercice : repo générique + DTO→Model mapping

---

## Fin de formation
Livrables conseillés :
- Support Markdown (ce document)
- Code du mini-projet refactoré (branche Git)
- Corrections des ateliers
