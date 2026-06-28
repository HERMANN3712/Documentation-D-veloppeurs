# Formation Angular — Tests (Jasmine & Karma)

- **Niveau** : intermédiaire
- **Durée suggérée** : 1 jour (7h) ou 2 demi-journées
- **Pré-requis** : TypeScript, bases Angular (components, services, DI), RxJS (notions).
- **Objectif global** : savoir écrire, exécuter et maintenir des tests unitaires Angular avec **Jasmine** et **Karma**, pour vérifier le bon fonctionnement des **composants** et **services**.

---

## Plan de la formation

1. [Pourquoi tester ? Principes et stratégie](#1-pourquoi-tester--principes-et-stratégie)
2. [L’écosystème Angular de test : Jasmine, Karma, TestBed](#2-lécosystème-angular-de-test--jasmine-karma-testbed)
3. [Mettre en place un projet et exécuter les tests](#3-mettre-en-place-un-projet-et-exécuter-les-tests)
4. [Les fondamentaux Jasmine : specs, matchers, hooks](#4-les-fondamentaux-jasmine--specs-matchers-hooks)
5. [Tester un service Angular (DI, HttpClient, Observables)](#5-tester-un-service-angular-di-httpclient-observables)
6. [Tester un composant Angular](#6-tester-un-composant-angular)
7. [Tester le DOM et les interactions utilisateur](#7-tester-le-dom-et-les-interactions-utilisateur)
8. [Dépendances : stubs, spies, mocks et doubles de test](#8-dépendances--stubs-spies-mocks-et-doubles-de-test)
9. [Formulaires : Reactive Forms & Template-driven](#9-formulaires--reactive-forms--template-driven)
10. [Routage : RouterTestingModule, navigation et guards](#10-routage--routertestingmodule-navigation-et-guards)
11. [Asynchronisme : fakeAsync, tick, async/await](#11-asynchronisme--fakeasync-tick-asyncawait)
12. [Bonnes pratiques : lisibilité, DRY, couverture, CI](#12-bonnes-pratiques--lisibilité-dry-couverture-ci)
13. [Atelier final : stratégie et implémentation sur un mini-cas](#13-atelier-final--stratégie-et-implémentation-sur-un-mini-cas)

---

## 1. Pourquoi tester ? Principes et stratégie

### 1.1. Pourquoi écrire des tests unitaires ?
Les tests unitaires servent à :

- **Valider** le comportement attendu d’une unité de code (fonction, méthode, service, composant).
- **Détecter les régressions** lors des refactors et évolutions.
- **Documenter** le comportement du code (les tests font office de spécification exécutable).
- **Accélérer** le développement à long terme (confiance + feedback immédiat).

### 1.2. Types de tests dans une application Angular
- **Unit tests** : testent une unité isolée (service/composant) avec dépendances simulées.
- **Integration tests** : testent plusieurs unités ensemble (ex : composant + service + template).
- **E2E (end-to-end)** : testent l’application dans un navigateur (ex : Cypress, Playwright).

> Cette formation se concentre sur les **tests unitaires** (et un peu d’intégration) avec **Jasmine** et **Karma**.

### 1.3. Qu’est-ce qu’un “bon” test ?
- **Fiable** (pas flaky)
- **Rapide**
- **Lisible** (Arrange–Act–Assert)
- **Indépendant** (pas d’ordre imposé)
- **Pertinent** (teste un comportement, pas l’implémentation)

---

## 2. L’écosystème Angular de test : Jasmine, Karma, TestBed

### 2.1. Jasmine
**Jasmine** est un framework de test BDD :
- `describe` : regroupe des tests
- `it` : un scénario de test
- `expect` + matchers : assertions
- `beforeEach/afterEach` : hooks
- `spyOn` : espionner des fonctions / méthodes

### 2.2. Karma
**Karma** exécute les tests dans un navigateur (ChromeHeadless en général) et remonte les résultats.

### 2.3. Angular TestBed
`TestBed` est l’outil Angular qui permet de :
- configurer un module de test (`declarations`, `imports`, `providers`…)
- instancier composants/services avec l’injection de dépendances
- compiler les templates

Concepts clés :
- `TestBed.configureTestingModule(...)`
- `TestBed.inject(...)`
- `TestBed.createComponent(...)`
- `fixture.detectChanges()`

---

## 3. Mettre en place un projet et exécuter les tests

### 3.1. Commandes Angular CLI
Dans un projet Angular standard :

```bash
ng test
```

Options utiles :

```bash
ng test --watch=false
ng test --browsers=ChromeHeadless
```

### 3.2. Structure typique des tests
- `*.spec.ts` : fichiers de tests
- `test.ts` : bootstrap des tests

### 3.3. Où tester quoi ?
- **Services** : comportement métier, appels HTTP, logique pure.
- **Composants** : interactions UI, binding, intégration légère template + logique.

---

## 4. Les fondamentaux Jasmine : specs, matchers, hooks

### 4.1. Structure AAA (Arrange–Act–Assert)
```ts
describe('calc', () => {
  it('should add two numbers', () => {
    // Arrange
    const a = 2;
    const b = 3;

    // Act
    const result = a + b;

    // Assert
    expect(result).toBe(5);
  });
});
```

### 4.2. Matchers courants
- `toBe` (égalité stricte)
- `toEqual` (égalité structurelle)
- `toContain`
- `toBeTruthy` / `toBeFalsey`
- `toThrow`

### 4.3. Hooks
```ts
describe('suite', () => {
  beforeEach(() => {
    // run before each test
  });

  afterEach(() => {
    // run after each test
  });

  it('...', () => {});
});
```

### 4.4. Spies (espions)
```ts
const obj = {
  doWork: (x: number) => x * 2,
};

spyOn(obj, 'doWork').and.returnValue(10);
expect(obj.doWork(2)).toBe(10);
expect(obj.doWork).toHaveBeenCalledWith(2);
```

---

## 5. Tester un service Angular (DI, HttpClient, Observables)

### 5.1. Exemple de service à tester
```ts
// user.service.ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable, map } from 'rxjs';

export interface UserDto {
  id: number;
  name: string;
}

@Injectable({ providedIn: 'root' })
export class UserService {
  constructor(private http: HttpClient) {}

  getUsers(): Observable<UserDto[]> {
    return this.http.get<UserDto[]>('/api/users');
  }

  getUserNames(): Observable<string[]> {
    return this.getUsers().pipe(map(users => users.map(u => u.name)));
  }
}
```

### 5.2. Test de service sans HTTP (logique pure)
Si possible, isoler la logique pure dans des fonctions :

```ts
export function extractNames(users: UserDto[]): string[] {
  return users.map(u => u.name);
}
```

Test :

```ts
import { extractNames } from './user.service';

describe('extractNames', () => {
  it('should extract user names', () => {
    const users = [{ id: 1, name: 'Ada' }, { id: 2, name: 'Bob' }];
    expect(extractNames(users)).toEqual(['Ada', 'Bob']);
  });
});
```

### 5.3. Tester un service avec HttpClientTestingModule
On utilise :
- `HttpClientTestingModule`
- `HttpTestingController`

```ts
// user.service.spec.ts
import { TestBed } from '@angular/core/testing';
import { HttpClientTestingModule, HttpTestingController } from '@angular/common/http/testing';
import { UserService } from './user.service';

describe('UserService', () => {
  let service: UserService;
  let httpMock: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [HttpClientTestingModule],
    });

    service = TestBed.inject(UserService);
    httpMock = TestBed.inject(HttpTestingController);
  });

  afterEach(() => {
    httpMock.verify(); // vérifie qu’aucune requête n’est restée en attente
  });

  it('should GET /api/users', () => {
    const mockUsers = [{ id: 1, name: 'Ada' }];

    service.getUsers().subscribe(users => {
      expect(users).toEqual(mockUsers);
    });

    const req = httpMock.expectOne('/api/users');
    expect(req.request.method).toBe('GET');

    req.flush(mockUsers);
  });

  it('should map user names with getUserNames()', () => {
    service.getUserNames().subscribe(names => {
      expect(names).toEqual(['Ada', 'Bob']);
    });

    const req = httpMock.expectOne('/api/users');
    req.flush([
      { id: 1, name: 'Ada' },
      { id: 2, name: 'Bob' },
    ]);
  });
});
```

### 5.4. Tester des erreurs HTTP
```ts
it('should handle server error', () => {
  service.getUsers().subscribe({
    next: () => fail('expected an error'),
    error: (err) => {
      expect(err.status).toBe(500);
    },
  });

  const req = httpMock.expectOne('/api/users');
  req.flush('boom', { status: 500, statusText: 'Server Error' });
});
```

---

## 6. Tester un composant Angular

### 6.1. Exemple de composant
```ts
// counter.component.ts
import { Component, Input } from '@angular/core';

@Component({
  selector: 'app-counter',
  template: `
    <button (click)="decrement()">-</button>
    <span class="value">{{ value }}</span>
    <button (click)="increment()">+</button>
  `,
})
export class CounterComponent {
  @Input() value = 0;

  increment() { this.value++; }
  decrement() { this.value--; }
}
```

### 6.2. Setup de test avec TestBed
```ts
// counter.component.spec.ts
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { CounterComponent } from './counter.component';

describe('CounterComponent', () => {
  let fixture: ComponentFixture<CounterComponent>;
  let component: CounterComponent;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      declarations: [CounterComponent],
    }).compileComponents();

    fixture = TestBed.createComponent(CounterComponent);
    component = fixture.componentInstance;
  });

  it('should create', () => {
    expect(component).toBeTruthy();
  });

  it('should increment value', () => {
    component.value = 1;
    component.increment();
    expect(component.value).toBe(2);
  });
});
```

### 6.3. Tester le rendu du template (DOM)
Pour que le template se mette à jour : `fixture.detectChanges()`.

```ts
it('should render the value', () => {
  component.value = 5;
  fixture.detectChanges();

  const el: HTMLElement = fixture.nativeElement;
  const span = el.querySelector('.value');

  expect(span?.textContent?.trim()).toBe('5');
});
```

---

## 7. Tester le DOM et les interactions utilisateur

### 7.1. Déclencher un événement clic
```ts
it('should react to + button click (integration)', () => {
  component.value = 0;
  fixture.detectChanges();

  const el: HTMLElement = fixture.nativeElement;
  const plusButton = el.querySelectorAll('button')[1] as HTMLButtonElement;

  plusButton.click();
  fixture.detectChanges();

  expect(el.querySelector('.value')?.textContent?.trim()).toBe('1');
});
```

### 7.2. DebugElement pour un accès Angular-friendly
```ts
import { By } from '@angular/platform-browser';

it('should click using DebugElement', () => {
  fixture.detectChanges();

  const plusBtn = fixture.debugElement.queryAll(By.css('button'))[1];
  plusBtn.triggerEventHandler('click');
  fixture.detectChanges();

  expect(component.value).toBe(1);
});
```

---

## 8. Dépendances : stubs, spies, mocks et doubles de test

### 8.1. Quand simuler une dépendance ?
- dépendance lente (HTTP)
- dépendance non déterministe (Date, random)
- dépendance hors scope (Router, storage)

### 8.2. Exemple : composant dépendant d’un service
```ts
// greeting.service.ts
import { Injectable } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class GreetingService {
  getGreeting(name: string) {
    return `Hello ${name}`;
  }
}

// hello.component.ts
import { Component } from '@angular/core';
import { GreetingService } from './greeting.service';

@Component({
  selector: 'app-hello',
  template: `<p class="msg">{{ message }}</p>`,
})
export class HelloComponent {
  message = '';

  constructor(private greeting: GreetingService) {}

  ngOnInit() {
    this.message = this.greeting.getGreeting('Angular');
  }
}
```

### 8.3. Fournir un stub
```ts
import { TestBed } from '@angular/core/testing';
import { HelloComponent } from './hello.component';
import { GreetingService } from './greeting.service';

class GreetingServiceStub {
  getGreeting(name: string) {
    return `Stubbed ${name}`;
  }
}

describe('HelloComponent', () => {
  beforeEach(async () => {
    await TestBed.configureTestingModule({
      declarations: [HelloComponent],
      providers: [{ provide: GreetingService, useClass: GreetingServiceStub }],
    }).compileComponents();
  });

  it('should display stubbed greeting', () => {
    const fixture = TestBed.createComponent(HelloComponent);
    fixture.detectChanges();

    const el: HTMLElement = fixture.nativeElement;
    expect(el.querySelector('.msg')?.textContent?.trim()).toBe('Stubbed Angular');
  });
});
```

### 8.4. Fournir un spy (objet Jasmine)
```ts
it('should call service with correct parameter', async () => {
  const spy = jasmine.createSpyObj<GreetingService>('GreetingService', ['getGreeting']);
  spy.getGreeting.and.returnValue('Hello mocked');

  await TestBed.configureTestingModule({
    declarations: [HelloComponent],
    providers: [{ provide: GreetingService, useValue: spy }],
  }).compileComponents();

  const fixture = TestBed.createComponent(HelloComponent);
  fixture.detectChanges();

  expect(spy.getGreeting).toHaveBeenCalledOnceWith('Angular');
});
```

---

## 9. Formulaires : Reactive Forms & Template-driven

### 9.1. Tester un Reactive Form
```ts
// login.component.ts
import { Component } from '@angular/core';
import { FormControl, FormGroup, Validators } from '@angular/forms';

@Component({
  selector: 'app-login',
  template: `
    <form [formGroup]="form">
      <input formControlName="email" />
      <input formControlName="password" type="password" />
    </form>
  `,
})
export class LoginComponent {
  form = new FormGroup({
    email: new FormControl('', [Validators.required, Validators.email]),
    password: new FormControl('', [Validators.required, Validators.minLength(8)]),
  });
}
```

Test :

```ts
import { ReactiveFormsModule } from '@angular/forms';

it('should validate email and password', async () => {
  await TestBed.configureTestingModule({
    declarations: [LoginComponent],
    imports: [ReactiveFormsModule],
  }).compileComponents();

  const fixture = TestBed.createComponent(LoginComponent);
  const component = fixture.componentInstance;

  component.form.setValue({ email: 'bad', password: '123' });
  expect(component.form.valid).toBeFalse();

  component.form.setValue({ email: 'a@b.com', password: '12345678' });
  expect(component.form.valid).toBeTrue();
});
```

### 9.2. Tester le binding template
```ts
it('should update input value in DOM', async () => {
  await TestBed.configureTestingModule({
    declarations: [LoginComponent],
    imports: [ReactiveFormsModule],
  }).compileComponents();

  const fixture = TestBed.createComponent(LoginComponent);
  fixture.detectChanges();

  const el: HTMLElement = fixture.nativeElement;
  const emailInput = el.querySelector('input[formcontrolname="email"]') as HTMLInputElement;

  emailInput.value = 'x@y.com';
  emailInput.dispatchEvent(new Event('input'));

  expect(fixture.componentInstance.form.value.email).toBe('x@y.com');
});
```

---

## 10. Routage : RouterTestingModule, navigation et guards

### 10.1. RouterTestingModule
Pour tester des composants impliquant la navigation, utiliser `RouterTestingModule`.

Exemple (test de navigation via `Router` spy) :

```ts
import { Router } from '@angular/router';
import { RouterTestingModule } from '@angular/router/testing';

it('should navigate to /home', async () => {
  const router = jasmine.createSpyObj<Router>('Router', ['navigateByUrl']);

  await TestBed.configureTestingModule({
    imports: [RouterTestingModule],
    providers: [{ provide: Router, useValue: router }],
  }).compileComponents();

  // ... dans un vrai composant, déclencher l’action puis :
  router.navigateByUrl('/home');
  expect(router.navigateByUrl).toHaveBeenCalledWith('/home');
});
```

### 10.2. Guards (idée générale)
- Tester la logique du guard en isolation
- Fournir des mocks de dépendances (auth service, router)

---

## 11. Asynchronisme : fakeAsync, tick, async/await

### 11.1. Cas fréquents
- Observables (HTTP, timers)
- Promises (APIs, libs)
- `setTimeout`, `debounceTime`

### 11.2. async/await (Promesses)
```ts
it('should resolve async operation', async () => {
  const result = await Promise.resolve(42);
  expect(result).toBe(42);
});
```

### 11.3. fakeAsync + tick
Utile pour contrôler le temps virtuel.

```ts
import { fakeAsync, tick } from '@angular/core/testing';

it('should run timer-based code', fakeAsync(() => {
  let value = 0;
  setTimeout(() => (value = 10), 1000);

  tick(999);
  expect(value).toBe(0);

  tick(1);
  expect(value).toBe(10);
}));
```

### 11.4. Zones et change detection
Après un événement async, il faut souvent :
- `fixture.detectChanges()`
- ou `fixture.whenStable()` (selon le cas)

---

## 12. Bonnes pratiques : lisibilité, DRY, couverture, CI

### 12.1. Arrange–Act–Assert & naming
- Nommer les tests comme des comportements :
  - `it('should disable submit when form is invalid')`
  - `it('should call UserService.getUsers on init')`

### 12.2. Éviter de tester l’implémentation
Préférer :
- tester le rendu (DOM) ou l’effet observable
- tester les interactions (appel de service)

### 12.3. Isoler vs intégrer
- **Service** : tests très isolés (mocks)
- **Composant** : un peu plus d’intégration (template + events)

### 12.4. Couverture de test
La couverture est un **indicateur**, pas une fin.
- viser les chemins critiques
- ajouter des tests sur les bugs rencontrés (tests de non-régression)

### 12.5. CI
En CI, exécuter :

```bash
ng test --watch=false --browsers=ChromeHeadless
```

---

## 13. Atelier final : stratégie et implémentation sur un mini-cas

### 13.1. Énoncé
Créer une mini-fonctionnalité “Liste d’utilisateurs” :
- un `UserService` qui récupère des utilisateurs
- un `UserListComponent` qui affiche la liste + un compteur
- gestion d’erreur (afficher un message)

### 13.2. Checklist des tests attendus
**Service**
- [ ] `getUsers()` effectue un `GET /api/users`
- [ ] en cas d’erreur 500, l’observable émet une erreur

**Composant**
- [ ] affiche N éléments `li` pour N users
- [ ] affiche un message d’erreur si le service échoue
- [ ] vérifie que `getUsers()` est appelé au `ngOnInit`

### 13.3. Exemple de composant et tests (solution type)

Composant :

```ts
import { Component } from '@angular/core';
import { UserService, UserDto } from './user.service';

@Component({
  selector: 'app-user-list',
  template: `
    <p *ngIf="error" class="error">{{ error }}</p>
    <ul>
      <li *ngFor="let u of users">{{ u.name }}</li>
    </ul>
    <p class="count">Count: {{ users.length }}</p>
  `,
})
export class UserListComponent {
  users: UserDto[] = [];
  error = '';

  constructor(private userService: UserService) {}

  ngOnInit() {
    this.userService.getUsers().subscribe({
      next: (users) => (this.users = users),
      error: () => (this.error = 'Unable to load users'),
    });
  }
}
```

Test avec spy de service :

```ts
import { TestBed } from '@angular/core/testing';
import { of, throwError } from 'rxjs';
import { UserListComponent } from './user-list.component';
import { UserService } from './user.service';

describe('UserListComponent', () => {
  it('should call getUsers on init and render list', async () => {
    const userService = jasmine.createSpyObj<UserService>('UserService', ['getUsers']);
    userService.getUsers.and.returnValue(of([
      { id: 1, name: 'Ada' },
      { id: 2, name: 'Bob' },
    ]));

    await TestBed.configureTestingModule({
      declarations: [UserListComponent],
      providers: [{ provide: UserService, useValue: userService }],
    }).compileComponents();

    const fixture = TestBed.createComponent(UserListComponent);
    fixture.detectChanges();

    expect(userService.getUsers).toHaveBeenCalledTimes(1);

    const el: HTMLElement = fixture.nativeElement;
    expect(el.querySelectorAll('li').length).toBe(2);
    expect(el.querySelector('.count')?.textContent).toContain('2');
  });

  it('should render error message when service fails', async () => {
    const userService = jasmine.createSpyObj<UserService>('UserService', ['getUsers']);
    userService.getUsers.and.returnValue(throwError(() => new Error('fail')));

    await TestBed.configureTestingModule({
      declarations: [UserListComponent],
      providers: [{ provide: UserService, useValue: userService }],
    }).compileComponents();

    const fixture = TestBed.createComponent(UserListComponent);
    fixture.detectChanges();

    const el: HTMLElement = fixture.nativeElement;
    expect(el.querySelector('.error')?.textContent?.trim()).toBe('Unable to load users');
  });
});
```

---

## Annexe A — Anti-patterns fréquents
- tests trop couplés au HTML (fragiles) : tester le comportement, pas la mise en page.
- trop de mocks : on perd la valeur du test.
- pas de `httpMock.verify()` : requêtes pendantes.
- oublier `fixture.detectChanges()` : DOM non mis à jour.

## Annexe B — Raccourcis de diagnostic
- Exécuter un seul fichier test : (selon config) via pattern / grep
- Utiliser `fdescribe` / `fit` temporairement pour isoler (à retirer avant merge)

---

### Résultats attendus en fin de formation
À la fin, vous savez :
- écrire des specs Jasmine lisibles
- configurer des modules de test avec `TestBed`
- tester services (y compris HTTP) et composants (DOM + interactions)
- gérer l’asynchronisme (fakeAsync/tick)
- appliquer des bonnes pratiques pour des tests maintenables
