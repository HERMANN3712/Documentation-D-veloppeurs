# Formation Angular – Tests des services (47)

> Public : développeurs Angular (débutant à intermédiaire en tests)
>
> Pré-requis : bases TypeScript, Angular DI, RxJS, notions de Jasmine/Jest
>
> Objectif : savoir écrire des **tests unitaires** fiables pour les **services Angular** (logique métier, erreurs, transformations, dépendances externes) avec **TestBed**, **mocks/spies**, **HttpClientTestingModule** et des **patterns** réutilisables.

---

## Sommaire

1. [Pourquoi tester les services ?](#1-pourquoi-tester-les-services-)
2. [Outillage & configuration de test Angular](#2-outillage--configuration-de-test-angular)
3. [Principes clés : isolation, DI, mocks](#3-principes-clés--isolation-di-mocks)
4. [Tester la logique métier pure](#4-tester-la-logique-métier-pure)
5. [Tester les interactions avec des dépendances (spies/mocks)](#5-tester-les-interactions-avec-des-dépendances-spiesmocks)
6. [Tester les services RxJS : Observables, opérateurs, erreurs](#6-tester-les-services-rxjs--observables-opérateurs-erreurs)
7. [Tester les services HTTP : HttpClientTestingModule](#7-tester-les-services-http--httpclienttestingmodule)
8. [Gestion des erreurs : stratégies et assertions](#8-gestion-des-erreurs--stratégies-et-assertions)
9. [Transformations de données & mapping DTO → modèle](#9-transformations-de-données--mapping-dto--modèle)
10. [Cas pratiques complets (3 ateliers)](#10-cas-pratiques-complets-3-ateliers)
11. [Bonnes pratiques, anti-patterns et checklist](#11-bonnes-pratiques-anti-patterns-et-checklist)
12. [Annexes : snippets réutilisables](#12-annexes--snippets-réutilisables)

---

## 1. Pourquoi tester les services ?

Les **services** Angular concentrent souvent :

- la **logique métier** (calculs, règles)
- la **coordination** (orchestration de plusieurs appels)
- la **communication** (HTTP, stockage, etc.)
- la **transformation** des données (DTO ↔ modèle, mapping)
- la **gestion d’erreurs** (fallback, re-throw, retry, message utilisateur)

### Ce qu’un bon test de service doit valider

1. **La logique métier** (sorties attendues selon entrées)
2. **La gestion d’erreurs** (propagation, adaptation, fallback)
3. **Les transformations** (mapping correct, tri, filtrage)
4. **Les interactions** avec dépendances externes (HTTP, autres services)

### Unit tests vs integration tests

- **Test unitaire** : service isolé, dépendances mockées (objectif de cette formation).
- **Test d’intégration** (plus rare côté services) : ex. service + pipeline réel + module complet.

---

## 2. Outillage & configuration de test Angular

### Frameworks possibles

- **Jasmine + Karma** (setup historique Angular)
- **Jest** (de plus en plus courant)

Le contenu ci-dessous reste valable pour les deux. Les exemples utilisent majoritairement **Jasmine** (`describe/it/expect`, `spyOn`).

### Structure typique d’un fichier de test

`user.service.spec.ts` :

- `describe('UserService', ...)`
- `beforeEach(() => TestBed.configureTestingModule(...))`
- création/injection du service
- tests : `it('should ...', () => { ... })`

### TestBed : l’outil central

`TestBed` permet de créer un **mini-module Angular** pour les tests :

- `providers` : services à tester et dépendances
- `imports` : modules nécessaires (`HttpClientTestingModule`, etc.)

Exemple minimal :

```ts
import { TestBed } from '@angular/core/testing';
import { UserService } from './user.service';

describe('UserService', () => {
  let service: UserService;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [UserService]
    });
    service = TestBed.inject(UserService);
  });

  it('should be created', () => {
    expect(service).toBeTruthy();
  });
});
```

---

## 3. Principes clés : isolation, DI, mocks

### Pourquoi c’est « simple » à tester

Angular repose sur **l’injection de dépendances** :

- un service reçoit ses dépendances via le constructeur
- en test, on remplace facilement ces dépendances par des **mocks**

### Techniques de remplacement

1. **Spies** (Jasmine) : espionner une méthode réelle ou simulée.
2. **Stub** : implémentation minimale (retourne des valeurs fixes).
3. **Mock** : objet fake avec comportement contrôlé (retours, erreurs, assertions sur appels).

### Provider override : `useValue` et `useClass`

- `useValue` : fournir un objet mock
- `useClass` : fournir une classe fake

Exemple `useValue` :

```ts
const authServiceMock = {
  getToken: () => 'fake-token'
};

TestBed.configureTestingModule({
  providers: [
    UserService,
    { provide: AuthService, useValue: authServiceMock }
  ]
});
```

### Règles d’or

- Un test = un comportement.
- Le test doit être **déterministe** (pas de temps réel, pas de réseaux).
- Ne pas tester Angular : tester **votre** logique.

---

## 4. Tester la logique métier pure

Un bon service contient parfois des fonctions qui n’ont **pas besoin** d’Observable/HTTP.

### Exemple : service de tarification

```ts
export class PricingService {
  computeTotal(amount: number, vatRate: number, discount = 0): number {
    if (amount < 0) throw new Error('Amount must be positive');
    const withVat = amount * (1 + vatRate);
    return Math.max(0, withVat - discount);
  }
}
```

#### Tests

```ts
describe('PricingService', () => {
  let service: PricingService;

  beforeEach(() => {
    TestBed.configureTestingModule({ providers: [PricingService] });
    service = TestBed.inject(PricingService);
  });

  it('computeTotal should apply VAT', () => {
    expect(service.computeTotal(100, 0.2)).toBe(120);
  });

  it('computeTotal should apply discount but not go below 0', () => {
    expect(service.computeTotal(50, 0.2, 1000)).toBe(0);
  });

  it('computeTotal should throw when amount is negative', () => {
    expect(() => service.computeTotal(-1, 0.2)).toThrowError('Amount must be positive');
  });
});
```

Points clés :

- tests rapides
- assertions claires
- gestion d’exception synchrone

---

## 5. Tester les interactions avec des dépendances (spies/mocks)

### Exemple : service qui dépend d’un Logger

```ts
export class Logger {
  info(message: string) {}
  error(message: string, err?: unknown) {}
}

export class OrderService {
  constructor(private logger: Logger) {}

  validate(orderTotal: number): boolean {
    if (orderTotal <= 0) {
      this.logger.error('Invalid order total', { orderTotal });
      return false;
    }
    this.logger.info('Order validated');
    return true;
  }
}
```

#### Test avec spy

```ts
describe('OrderService', () => {
  let service: OrderService;
  let logger: Logger;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [
        OrderService,
        Logger
      ]
    });

    service = TestBed.inject(OrderService);
    logger = TestBed.inject(Logger);
  });

  it('should log error and return false when total <= 0', () => {
    spyOn(logger, 'error');

    const ok = service.validate(0);

    expect(ok).toBeFalse();
    expect(logger.error).toHaveBeenCalledWith('Invalid order total', { orderTotal: 0 });
  });

  it('should log info and return true when total > 0', () => {
    spyOn(logger, 'info');

    const ok = service.validate(10);

    expect(ok).toBeTrue();
    expect(logger.info).toHaveBeenCalled();
  });
});
```

Variantes :

- Fournir `Logger` via `useValue` et vérifier les appels.
- Créer une classe `FakeLogger` qui enregistre les messages.

---

## 6. Tester les services RxJS : Observables, opérateurs, erreurs

Les services Angular exposent très souvent des **Observables**.

### Rappels essentiels

- Un Observable ne fait rien tant qu’on ne **subscribe** pas.
- Les tests doivent vérifier :
  - valeur(s) émises
  - complétion
  - erreur émise
  - opérateurs appliqués (`map`, `filter`, `catchError`, `shareReplay`…)

### Stratégies de test

- Observables synchrones : `of(...)`, `throwError(...)` → test simple.
- Observables asynchrones : utiliser `fakeAsync/tick`, ou `done`, ou `firstValueFrom`.

#### Exemple : transformation `map`

```ts
import { Injectable } from '@angular/core';
import { of, Observable } from 'rxjs';
import { map } from 'rxjs/operators';

export interface UserDto { id: string; full_name: string; }
export interface User { id: string; fullName: string; }

@Injectable()
export class UserMapperService {
  toUser(dto: UserDto): User {
    return { id: dto.id, fullName: dto.full_name };
  }

  mapUsers(dtos: UserDto[]): Observable<User[]> {
    return of(dtos).pipe(map(list => list.map(dto => this.toUser(dto))));
  }
}
```

Test :

```ts
import { firstValueFrom } from 'rxjs';

describe('UserMapperService', () => {
  let service: UserMapperService;

  beforeEach(() => {
    TestBed.configureTestingModule({ providers: [UserMapperService] });
    service = TestBed.inject(UserMapperService);
  });

  it('mapUsers should map full_name to fullName', async () => {
    const dtos = [{ id: '1', full_name: 'Ada Lovelace' }];

    const users = await firstValueFrom(service.mapUsers(dtos));

    expect(users).toEqual([{ id: '1', fullName: 'Ada Lovelace' }]);
  });
});
```

#### Exemple : `catchError` et fallback

```ts
import { catchError } from 'rxjs/operators';
import { of, throwError } from 'rxjs';

export class FeatureFlagService {
  constructor(private remote: RemoteConfigService) {}

  getFlag$(name: string) {
    return this.remote.fetchFlag$(name).pipe(
      catchError(() => of(false))
    );
  }
}
```

Test :

```ts
import { firstValueFrom, of, throwError } from 'rxjs';

describe('FeatureFlagService', () => {
  let service: FeatureFlagService;
  let remote: jasmine.SpyObj<RemoteConfigService>;

  beforeEach(() => {
    remote = jasmine.createSpyObj<RemoteConfigService>('RemoteConfigService', ['fetchFlag$']);

    TestBed.configureTestingModule({
      providers: [
        FeatureFlagService,
        { provide: RemoteConfigService, useValue: remote }
      ]
    });

    service = TestBed.inject(FeatureFlagService);
  });

  it('should return true when remote emits true', async () => {
    remote.fetchFlag$.and.returnValue(of(true));
    await expectAsync(firstValueFrom(service.getFlag$('new-ui'))).toBeResolvedTo(true);
  });

  it('should return false when remote errors', async () => {
    remote.fetchFlag$.and.returnValue(throwError(() => new Error('network')));
    await expectAsync(firstValueFrom(service.getFlag$('new-ui'))).toBeResolvedTo(false);
  });
});
```

---

## 7. Tester les services HTTP : HttpClientTestingModule

Quand le service utilise `HttpClient`, privilégier :

- `HttpClientTestingModule`
- `HttpTestingController` pour intercepter les requêtes

### Exemple : service API

```ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable, map } from 'rxjs';

export interface ProductDto { id: number; name: string; price_cents: number; }
export interface Product { id: number; name: string; price: number; }

@Injectable()
export class ProductService {
  constructor(private http: HttpClient) {}

  list$(): Observable<Product[]> {
    return this.http.get<ProductDto[]>('/api/products').pipe(
      map(dtos => dtos.map(d => ({ id: d.id, name: d.name, price: d.price_cents / 100 })))
    );
  }

  get$(id: number): Observable<Product> {
    return this.http.get<ProductDto>(`/api/products/${id}`).pipe(
      map(d => ({ id: d.id, name: d.name, price: d.price_cents / 100 }))
    );
  }
}
```

### Test HTTP

```ts
import { TestBed } from '@angular/core/testing';
import { HttpClientTestingModule, HttpTestingController } from '@angular/common/http/testing';
import { firstValueFrom } from 'rxjs';

describe('ProductService (HTTP)', () => {
  let service: ProductService;
  let httpMock: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [HttpClientTestingModule],
      providers: [ProductService]
    });

    service = TestBed.inject(ProductService);
    httpMock = TestBed.inject(HttpTestingController);
  });

  afterEach(() => {
    // Vérifie qu’il n’y a pas de requêtes en attente.
    httpMock.verify();
  });

  it('list$ should call GET /api/products and map price', async () => {
    const promise = firstValueFrom(service.list$());

    const req = httpMock.expectOne('/api/products');
    expect(req.request.method).toBe('GET');

    req.flush([
      { id: 1, name: 'Keyboard', price_cents: 1999 }
    ]);

    await expectAsync(promise).toBeResolvedTo([
      { id: 1, name: 'Keyboard', price: 19.99 }
    ]);
  });

  it('get$ should call GET /api/products/2', async () => {
    const promise = firstValueFrom(service.get$(2));

    const req = httpMock.expectOne('/api/products/2');
    expect(req.request.method).toBe('GET');

    req.flush({ id: 2, name: 'Mouse', price_cents: 999 });

    await expectAsync(promise).toBeResolvedTo({
      id: 2,
      name: 'Mouse',
      price: 9.99
    });
  });
});
```

Ce que ce test valide :

- URL appelée
- verbe HTTP
- mapping DTO → modèle
- absence de requêtes non consommées (`verify`)

---

## 8. Gestion des erreurs : stratégies et assertions

### Trois stratégies courantes

1. **Propager** l’erreur : laisser l’Observable échouer.
2. **Transformer** l’erreur : ex. wrapper `DomainError`.
3. **Récupérer** : fallback (valeur par défaut) + log.

#### Propagation d’erreur (HTTP)

```ts
it('get$ should propagate HTTP error', async () => {
  const promise = firstValueFrom(service.get$(1));

  const req = httpMock.expectOne('/api/products/1');
  req.flush('Boom', { status: 500, statusText: 'Server Error' });

  await expectAsync(promise).toBeRejected();
});
```

> Avec Jest, on ferait souvent `await expect(firstValueFrom(...)).rejects.toBeTruthy()`.

#### Transformation d’erreur

Exemple service :

```ts
export class ApiError extends Error {
  constructor(message: string, public status?: number) {
    super(message);
  }
}

export class SecureProductService {
  constructor(private http: HttpClient) {}

  get$(id: number) {
    return this.http.get(`/api/products/${id}`).pipe(
      catchError((err) => {
        return throwError(() => new ApiError('Cannot fetch product', err.status));
      })
    );
  }
}
```

Test :

```ts
it('should throw ApiError with status', async () => {
  const promise = firstValueFrom(service.get$(1));

  const req = httpMock.expectOne('/api/products/1');
  req.flush('Nope', { status: 401, statusText: 'Unauthorized' });

  try {
    await promise;
    fail('Expected promise to reject');
  } catch (e: any) {
    expect(e instanceof ApiError).toBeTrue();
    expect(e.status).toBe(401);
    expect(e.message).toBe('Cannot fetch product');
  }
});
```

---

## 9. Transformations de données & mapping DTO → modèle

### Pourquoi c’est crucial

Les bugs fréquents :

- mauvais mapping de clés (`full_name` vs `fullName`)
- unités (cents → euros)
- dates (`string` → `Date`)
- valeurs optionnelles / `null`

### Pattern recommandé : extraire le mapping

- `mapDtoToX(dto)` pur (testable facilement)
- `service.method$()` orchestre (HTTP + mapping)

Exemple mapping date :

```ts
export interface EventDto { id: string; starts_at: string; }
export interface Event { id: string; startsAt: Date; }

export function mapEvent(dto: EventDto): Event {
  return { id: dto.id, startsAt: new Date(dto.starts_at) };
}
```

Test :

```ts
it('mapEvent should map starts_at to Date', () => {
  const e = mapEvent({ id: 'a', starts_at: '2025-01-02T10:00:00.000Z' });
  expect(e.startsAt instanceof Date).toBeTrue();
  expect(e.startsAt.toISOString()).toBe('2025-01-02T10:00:00.000Z');
});
```

---

## 10. Cas pratiques complets (3 ateliers)

### Atelier 1 — Service avec dépendance + règles métier

**Énoncé** : Un service `CheckoutService` calcule un total et dépend d’un `DiscountService`.

**Objectifs** :

- mock d’un service dépendant
- tests de règles métier (remise plafonnée)

Code :

```ts
export class DiscountService {
  getDiscount(amount: number): number { return 0; }
}

export class CheckoutService {
  constructor(private discount: DiscountService) {}

  computePayable(amount: number): number {
    if (amount <= 0) throw new Error('amount must be > 0');
    const d = this.discount.getDiscount(amount);
    const capped = Math.min(d, amount * 0.3);
    return amount - capped;
  }
}
```

Tests (à écrire) :

- retour normal avec remise
- remise plafonnée à 30%
- exception si `amount <= 0`

Correction :

```ts
describe('CheckoutService', () => {
  let service: CheckoutService;
  let discount: jasmine.SpyObj<DiscountService>;

  beforeEach(() => {
    discount = jasmine.createSpyObj('DiscountService', ['getDiscount']);

    TestBed.configureTestingModule({
      providers: [
        CheckoutService,
        { provide: DiscountService, useValue: discount }
      ]
    });

    service = TestBed.inject(CheckoutService);
  });

  it('should subtract discount', () => {
    discount.getDiscount.and.returnValue(10);
    expect(service.computePayable(100)).toBe(90);
  });

  it('should cap discount to 30%', () => {
    discount.getDiscount.and.returnValue(1000);
    expect(service.computePayable(100)).toBe(70);
  });

  it('should throw if amount <= 0', () => {
    expect(() => service.computePayable(0)).toThrowError('amount must be > 0');
  });
});
```

---

### Atelier 2 — Service HTTP + mapping + erreur transformée

**Énoncé** : `CustomerService` appelle `/api/customers/:id`, mappe DTO → modèle, et transforme les erreurs en `ApiError`.

**Objectifs** :

- `HttpTestingController`
- test de mapping
- test d’erreur 404 transformée

---

### Atelier 3 — Service RxJS avec cache (shareReplay)

**Énoncé** : `ConfigService` expose `config$` qui fetch une config distant et la cache.

**Objectifs** :

- vérifier qu’un seul appel HTTP est fait malgré plusieurs subscriptions
- comprendre `shareReplay(1)`

Exemple (simplifié) :

```ts
export class ConfigService {
  readonly config$ = this.http.get('/api/config').pipe(shareReplay(1));
  constructor(private http: HttpClient) {}
}
```

Test (idée) :

- subscribe 2 fois
- `expectOne('/api/config')` une seule fois
- `flush` puis vérifier les valeurs

---

## 11. Bonnes pratiques, anti-patterns et checklist

### Bonnes pratiques

- **Nommer** les tests avec un comportement : `should ... when ...`.
- Tester les **cas limites** : `null`, `[]`, valeurs négatives, erreurs HTTP.
- Isoler et tester le **mapping** séparément quand possible.
- Utiliser `httpMock.verify()` pour éviter les fuites.
- Préférer `firstValueFrom` pour obtenir une valeur unique.

### Anti-patterns

- Tester des implémentations internes au lieu du comportement.
- Utiliser des `setTimeout` en tests (fragile) au lieu de `fakeAsync`/RxJS.
- Trop de mocks complexes : viser le minimum.

### Checklist rapide

- [ ] Le service est instancié via `TestBed`.
- [ ] Dépendances remplacées (`useValue`, spies).
- [ ] Observable testé (valeur + erreur) si nécessaire.
- [ ] HTTP intercepté avec `HttpTestingController`.
- [ ] Mapping testé.
- [ ] `afterEach(httpMock.verify)` présent pour les tests HTTP.

---

## 12. Annexes : snippets réutilisables

### A. Créer un SpyObj typé (Jasmine)

```ts
const dep = jasmine.createSpyObj<MyDep>('MyDep', ['method1', 'method2']);
dep.method1.and.returnValue(of('x'));
```

### B. Attendre une valeur d’Observable

```ts
import { firstValueFrom } from 'rxjs';

const value = await firstValueFrom(service.some$());
expect(value).toEqual(...);
```

### C. Tester une erreur Observable (Promise rejetée)

```ts
await expectAsync(firstValueFrom(service.some$())).toBeRejected();
```

### D. Scaffold test HTTP

```ts
beforeEach(() => {
  TestBed.configureTestingModule({
    imports: [HttpClientTestingModule],
    providers: [MyService]
  });
  service = TestBed.inject(MyService);
  httpMock = TestBed.inject(HttpTestingController);
});

afterEach(() => httpMock.verify());
```

---

## Conclusion

Tester les services Angular est efficace grâce à :

- l’**injection de dépendances** (remplacement facile des dépendances)
- des **mocks/spies** pour vérifier les interactions
- des outils dédiés pour l’HTTP (`HttpTestingController`)
- des stratégies claires pour **erreurs** et **transformations**

Prochaine étape recommandée : appliquer les mêmes principes aux **composants** (tests shallow vs avec template), et aux **interceptors/guards/resolvers**.
