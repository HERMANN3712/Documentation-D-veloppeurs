# Formation — TypeScript pour Angular

**Public visé** : développeurs Angular (débutants à intermédiaires) souhaitant consolider ou structurer leur pratique de TypeScript dans le contexte Angular.

**Pré‑requis** :
- Bases JavaScript (ES6+), notions de programmation orientée objet.
- Connaissances Angular (composants, modules, services) recommandées.

**Durée suggérée** : 1 à 2 jours (adaptable)

**Objectifs pédagogiques** :
- Comprendre la valeur ajoutée de TypeScript dans un projet Angular.
- Maîtriser le typage (primitifs, unions, génériques) et l’inférence.
- Utiliser interfaces, types, classes, décorateurs et modules ES.
- Écrire du code Angular plus robuste (DTO, services, composants) avec un typage strict.
- Améliorer la maintenabilité (encapsulation, immutabilité, patterns idiomatiques).

**Modalités** : alternance d’explications, démonstrations et exercices corrigés.

---

## Plan détaillé

1. **Introduction : TypeScript dans l’écosystème Angular**
2. **Configuration & “mode strict”**
3. **Types fondamentaux et inférence**
4. **Fonctions typées (paramètres, retours, overload, this)**
5. **Objets : interfaces, type aliases, unions & intersections**
6. **Classes et POO utile pour Angular (encapsulation, readonly, abstract)**
7. **Génériques (Array, Observable, HttpClient, patterns)**
8. **Typage, null/undefined et sécurité : strictNullChecks**
9. **Modules ES, imports/exports et organisation du code**
10. **Décorateurs et métaprogrammation (ce qu’Angular en fait)**
11. **TypeScript appliqué à Angular : composants, templates, services, HTTP, forms**
12. **Bonnes pratiques, conventions, refactoring et anti‑patterns**
13. **Atelier final : mini‑feature Angular typée de bout en bout (DTO → service → UI)**

---

# 1) Introduction : TypeScript dans Angular

Angular utilise TypeScript comme langage principal.

### 1.1 Pourquoi TypeScript ?
TypeScript ajoute notamment :
- **Typage statique** : possibilité d’exprimer les types des variables, fonctions et objets.
- **Interfaces et types** : description de la “forme” des objets (contrats).
- **Classes et POO moderne** : encapsulation, héritage, abstractions.
- **Autocomplétion et refactoring** : les IDE (VS Code) exploitent le typage pour aider.
- **Robustesse** : réduction d’erreurs à l’exécution (undefined, mauvais champs, etc.).

### 1.2 TypeScript vs JavaScript
- TypeScript **compile** vers JavaScript.
- Le typage disparaît à l’exécution : c’est un outil **de développement** (build‑time).
- Angular s’appuie sur TypeScript pour :
  - Décrire des APIs (services, DTO),
  - Protéger les interactions avec le template,
  - Structurer les applications (modularité, testabilité).

---

# 2) Configuration & “mode strict”

Dans un projet Angular, la configuration TypeScript se fait via `tsconfig.json` (et parfois `tsconfig.app.json`, `tsconfig.spec.json`).

### 2.1 Options clés
- `target` : version JavaScript générée.
- `module` : système de modules (souvent `ES2020`/`ESNext`).
- `strict` : active un ensemble de contrôles de typage recommandés.

### 2.2 Options strict importantes
- `strictNullChecks` : `null`/`undefined` sont explicitement gérés.
- `noImplicitAny` : interdit les `any` implicites.
- `noImplicitReturns` : chemins de retour explicites.
- `strictPropertyInitialization` : propriétés de classe initialisées.

> Recommandation Angular moderne : activer `strict` (ou tendre vers `strict`).

### Exercice
- Identifier 3 erreurs détectées uniquement en mode `strict`.

---

# 3) Types fondamentaux et inférence

### 3.1 Types primitifs
```ts
let count: number = 0;
let title: string = 'Angular';
let active: boolean = true;
let n: null = null;
let u: undefined = undefined;
```

### 3.2 Tableaux
```ts
const ids: number[] = [1, 2, 3];
const names: Array<string> = ['A', 'B'];
```

### 3.3 Tuples
```ts
const point: [number, number] = [10, 20];
```

### 3.4 Enums (à utiliser avec parcimonie)
```ts
enum Role {
  Admin = 'ADMIN',
  User = 'USER'
}
```

### 3.5 Inference (inférence de type)
TypeScript infère souvent le type sans annotation :
```ts
const maxUsers = 10; // number
const label = 'Hello'; // string
```

### 3.6 `any`, `unknown`, `never`
- `any` : désactive le typage (à éviter).
- `unknown` : on doit **vérifier** avant d’utiliser.
- `never` : un cas impossible (exhaustivité).

```ts
let value: unknown;

if (typeof value === 'string') {
  console.log(value.toUpperCase());
}

function fail(msg: string): never {
  throw new Error(msg);
}
```

---

# 4) Fonctions typées

### 4.1 Paramètres et type de retour
```ts
function add(a: number, b: number): number {
  return a + b;
}
```

### 4.2 Paramètres optionnels et valeurs par défaut
```ts
function greet(name: string, prefix = 'Hello'): string {
  return `${prefix} ${name}`;
}

function format(label: string, uppercase?: boolean): string {
  return uppercase ? label.toUpperCase() : label;
}
```

### 4.3 Type de fonction (signature)
```ts
type Comparator<T> = (a: T, b: T) => number;

const byId: Comparator<{ id: number }> = (a, b) => a.id - b.id;
```

### 4.4 Surcharge (overload)
```ts
function toArray(value: string): string[];
function toArray(value: number): number[];
function toArray(value: string | number) {
  return [value];
}
```

### 4.5 Typage de `this` (cas avancé)
```ts
function handler(this: HTMLButtonElement, ev: MouseEvent) {
  this.disabled = true;
}
```

---

# 5) Objets : interfaces, types, unions & intersections

### 5.1 Interface vs type alias
- `interface` : idéal pour définir des **contrats** d’objet extensibles.
- `type` : plus flexible (unions, primitives, tuples, mapped types).

```ts
interface User {
  id: number;
  name: string;
  email?: string;
}

type UserId = number;
```

### 5.2 Unions
```ts
type LoadingState = 'idle' | 'loading' | 'success' | 'error';

function setState(state: LoadingState) {
  // ...
}
```

### 5.3 Narrowing (réduction de type)
```ts
function printId(id: string | number) {
  if (typeof id === 'string') {
    console.log(id.toUpperCase());
  } else {
    console.log(id.toFixed(0));
  }
}
```

### 5.4 Intersections
```ts
type WithTimestamps = { createdAt: string; updatedAt: string };

type UserWithMeta = User & WithTimestamps;
```

### 5.5 Types utilitaires (utility types)
- `Partial<T>` : rend les champs optionnels.
- `Pick<T, K>` : sélectionne des champs.
- `Omit<T, K>` : retire des champs.
- `Readonly<T>` : champs en lecture seule.

```ts
type UserUpdate = Partial<Pick<User, 'name' | 'email'>>;
```

### Exercice
- Modéliser un `Product`, un `CartItem`, et un `Cart` with `Readonly` + `Omit`.

---

# 6) Classes & POO utile pour Angular

Angular encourage souvent l’usage de **classes** pour les services, composants, modèles.

### 6.1 Propriétés, constructeur, access modifiers
```ts
class Person {
  constructor(
    public id: number,
    public name: string,
    private secret: string
  ) {}

  rename(newName: string) {
    this.name = newName;
  }
}
```

- `public` (par défaut), `private`, `protected`.
- `readonly` : immutabilité partielle.

```ts
class Session {
  readonly token: string;
  constructor(token: string) {
    this.token = token;
  }
}
```

### 6.2 Héritage et classes abstraites
```ts
abstract class BaseService {
  abstract getName(): string;
}

class UserService extends BaseService {
  getName() {
    return 'UserService';
  }
}
```

### 6.3 Quand préférer interfaces/types à des classes ?
- DTO/objets de données : souvent `interface`/`type` suffisent.
- Comportement/méthodes : classe pertinente.
- Angular DI : services = classes injectable.

---

# 7) Génériques (clé en Angular)

Les génériques permettent de rendre un code réutilisable en gardant le typage.

### 7.1 Exemple simple
```ts
function wrap<T>(value: T): { value: T } {
  return { value };
}
```

### 7.2 Génériques avec contraintes
```ts
function pluckId<T extends { id: number }>(items: T[]): number[] {
  return items.map(i => i.id);
}
```

### 7.3 Génériques dans Angular
- `Observable<T>` : flux typés.
- `HttpClient.get<T>()` : réponse HTTP typée.

```ts
interface UserDto {
  id: number;
  name: string;
}

// Dans un service Angular
getUsers(): Observable<UserDto[]> {
  return this.http.get<UserDto[]>('/api/users');
}
```

---

# 8) Null/undefined et sécurité (strictNullChecks)

### 8.1 Pourquoi c’est important
Beaucoup de bugs viennent de valeurs absentes : `undefined`/`null`.

### 8.2 Union avec undefined
```ts
interface Profile {
  avatarUrl?: string; // string | undefined
}
```

### 8.3 Optional chaining et nullish coalescing
```ts
const url = profile.avatarUrl?.toLowerCase();
const displayUrl = profile.avatarUrl ?? '/assets/default.png';
```

### 8.4 Non-null assertion (à limiter)
```ts
const el = document.getElementById('app')!;
```

### 8.5 Patterns Angular
- Initialiser les champs.
- Utiliser des guards et des unions.
- Préférer des valeurs par défaut claires.

---

# 9) Modules ES : imports/exports

### 9.1 Export
```ts
// user.model.ts
export interface User {
  id: number;
  name: string;
}
```

### 9.2 Import
```ts
import { User } from './user.model';
```

### 9.3 Organisation recommandée
- `*.model.ts` ou `*.types.ts` pour types.
- `*.service.ts` pour services.
- Barrel files (`index.ts`) avec prudence (attention aux cycles).

---

# 10) Décorateurs et métaprogrammation (ce qu’Angular en fait)

TypeScript propose des **décorateurs** (selon configuration) que Angular utilise pour déclarer :
- `@Component`, `@Directive`, `@Pipe`, `@Injectable`, `@NgModule`.

Exemple :
```ts
import { Component } from '@angular/core';

@Component({
  selector: 'app-hello',
  template: `<h1>{{ title }}</h1>`
})
export class HelloComponent {
  title: string = 'Hello';
}
```

> Point clé : le décorateur décrit la **métadonnée**. Le typage reste géré par TypeScript.

---

# 11) TypeScript appliqué à Angular

## 11.1 Typage d’un composant
```ts
import { Component, Input } from '@angular/core';

type Status = 'draft' | 'published';

@Component({
  selector: 'app-post-badge',
  template: `
    <span [class]="status">{{ status }}</span>
  `
})
export class PostBadgeComponent {
  @Input({ required: true }) status!: Status;
}
```

Points pédagogiques :
- `@Input` typé.
- `required: true` (Angular moderne) renforce la contract.
- Union type pour éviter les strings libres.

## 11.2 Typage des événements
```ts
import { Component, Output, EventEmitter } from '@angular/core';

@Component({
  selector: 'app-search',
  template: `
    <input (input)="onInput($event)" />
  `
})
export class SearchComponent {
  @Output() queryChange = new EventEmitter<string>();

  onInput(ev: Event) {
    const value = (ev.target as HTMLInputElement).value;
    this.queryChange.emit(value);
  }
}
```

## 11.3 Services Angular et DTO
### Définir des DTO
```ts
export interface UserDto {
  id: number;
  name: string;
  email: string;
}

export interface CreateUserDto {
  name: string;
  email: string;
}
```

### Typage HTTP
```ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';
import { UserDto, CreateUserDto } from './user.dto';

@Injectable({ providedIn: 'root' })
export class UsersApi {
  constructor(private http: HttpClient) {}

  list(): Observable<UserDto[]> {
    return this.http.get<UserDto[]>('/api/users');
  }

  create(payload: CreateUserDto): Observable<UserDto> {
    return this.http.post<UserDto>('/api/users', payload);
  }
}
```

> Bonnes pratiques : différencier **DTO** (API) et **modèles UI** si nécessaire.

## 11.4 Typage RxJS (rappels utiles)
Les opérateurs conservent/transformment les types :
```ts
import { map } from 'rxjs/operators';

const names$ = this.usersApi.list().pipe(
  map(users => users.map(u => u.name))
);
```

## 11.5 Forms : typage (approche moderne)
Avec les formulaires typés Angular :
```ts
import { FormControl, FormGroup, Validators } from '@angular/forms';

const form = new FormGroup({
  name: new FormControl<string>('', { nonNullable: true, validators: [Validators.required] }),
  email: new FormControl<string>('', { nonNullable: true, validators: [Validators.required] }),
});

form.value; // typé (avec possiblement undefined selon config)
form.controls.name.value; // string (nonNullable)
```

## 11.6 Template type-checking
Angular effectue un contrôle de type des templates.
Recommandations :
- éviter `any` dans les composants,
- typer les streams `Observable<T>`,
- gérer le `null` lors de l’async pipe.

```html
<div *ngIf="user$ | async as user">
  {{ user.name }}
</div>
```

---

# 12) Bonnes pratiques et anti‑patterns

### 12.1 Bonnes pratiques
- Activer `strict`.
- Favoriser `unknown` plutôt que `any`.
- Exprimer les états avec unions (`'loading' | 'success' | 'error'`).
- Créer des types dédiés : `UserId`, `Email` (type alias) lorsque pertinent.
- Utiliser `readonly` et l’immutabilité (surtout dans les state stores).
- Éviter la duplication : `Pick`, `Omit`, types utilitaires.

### 12.2 Anti‑patterns
- `any` partout “pour aller vite”.
- DTO non typés (réponses HTTP `any`).
- Objets “fourre-tout” avec propriétés optionnelles non justifiées.
- Abus de classes pour de simples structures de données.

### 12.3 Conseils de refactoring
- Partir des frontières : API/HTTP, inputs/outputs composants.
- Taper progressivement : remplacer `any` par `unknown` + narrowing.
- Introduire des unions d’état.

---

# 13) Atelier final — Feature typée de bout en bout

## Objectif
Implémenter une mini fonctionnalité `Users` :
- Liste,
- Ajout,
- Gestion d’état et typage strict.

## Étape 1 — Types/DTO
```ts
// users.types.ts
export type UserId = number;

export interface UserDto {
  id: UserId;
  name: string;
  email: string;
}

export type UsersState =
  | { status: 'idle'; users: [] }
  | { status: 'loading'; users: [] }
  | { status: 'success'; users: UserDto[] }
  | { status: 'error'; users: []; error: string };
```

## Étape 2 — Service API
```ts
@Injectable({ providedIn: 'root' })
export class UsersApi {
  constructor(private http: HttpClient) {}

  list(): Observable<UserDto[]> {
    return this.http.get<UserDto[]>('/api/users');
  }
}
```

## Étape 3 — Composant avec état typé
```ts
@Component({
  selector: 'app-users',
  template: `
    <ng-container [ngSwitch]="state.status">
      <p *ngSwitchCase="'idle'">Prêt.</p>
      <p *ngSwitchCase="'loading'">Chargement…</p>
      <ul *ngSwitchCase="'success'">
        <li *ngFor="let u of state.users">{{ u.name }} — {{ u.email }}</li>
      </ul>
      <p *ngSwitchCase="'error'">Erreur: {{ state.error }}</p>
    </ng-container>
  `
})
export class UsersComponent {
  state: UsersState = { status: 'idle', users: [] };

  constructor(private api: UsersApi) {}

  load(): void {
    this.state = { status: 'loading', users: [] };

    this.api.list().subscribe({
      next: (users) => (this.state = { status: 'success', users }),
      error: () => (this.state = { status: 'error', users: [], error: 'Impossible de charger' })
    });
  }
}
```

## Points de validation
- Aucune variable `any`.
- Les champs d’état existent selon la branche (`error` seulement quand `status === 'error'`).
- Les DTO sont explicitement typés.

---

# Annexes

## A) Cheatsheet TypeScript utile en Angular
- `type A = B | C` (union)
- `type A = B & C` (intersection)
- `Partial<T>`, `Pick<T, K>`, `Omit<T, K>`, `Readonly<T>`
- `Observable<T>`, `HttpClient.get<T>()`
- `?.` et `??`

## B) Suggestions d’évaluation
- Quiz : différence `any` vs `unknown`.
- Exercice : implémenter un `guard` pour narrowing.
- Mini‑projet : service + composant + form typés.

---

## Fin de formation

Ce cours a posé les bases TypeScript indispensables pour écrire des applications Angular maintenables : typage strict, contrats d’objets, génériques, et application directe aux composants/services/templates.
