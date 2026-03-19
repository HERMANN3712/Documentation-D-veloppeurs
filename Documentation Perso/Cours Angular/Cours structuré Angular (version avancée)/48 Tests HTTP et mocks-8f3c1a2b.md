# Formation Angular — Tests HTTP et mocks (HttpTestingController)

> Objectif : maîtriser les tests unitaires des services Angular qui consomment HTTP, **sans dépendre d’une API réelle**, en utilisant `HttpClientTestingModule` et `HttpTestingController`.

---

## 0) Pré-requis

- Connaissances de base en Angular (services, DI, modules)
- Connaissances de base en RxJS (Observable, `pipe`, opérateurs)
- Savoir lancer les tests Angular (Karma/Jasmine ou Jest)

**Environnement conseillé :**
- Angular ≥ 15 (valable aussi pour versions récentes)
- Tests via `TestBed` (standard Angular)

---

## 1) Pourquoi tester les services HTTP ?

Les services Angular basés sur `HttpClient` sont critiques : ils orchestrent l’accès aux API, la transformation de données et la gestion d’erreurs.

### Problèmes si on teste “en vrai” contre une API

- **Tests fragiles** : l’API peut être indisponible, lente, ou retourner des données variables.
- **Tests non déterministes** : dépendance au réseau, à l’état d’une base de données, etc.
- **Coût & complexité** : besoin d’un environnement de test complet (API, DB, mocks serveur).

### Ce que `HttpTestingController` apporte

`HttpTestingController` permet de :
- **intercepter** les requêtes HTTP émises par `HttpClient`
- **vérifier** que la requête est correcte (URL, méthode, headers, body, paramètres)
- **simuler** une réponse (`flush`) et tester le traitement côté service
- **simuler des erreurs** (HTTP status, erreurs réseau) et tester les comportements de fallback

---

## 2) Concepts clés

### 2.1 `HttpClientTestingModule`

Module Angular de test qui remplace le backend HTTP réel par une implémentation testable.

### 2.2 `HttpTestingController`

Contrôleur permettant :
- `expectOne(...)` : récupère une requête unique correspondant au critère.
- `match(...)` : récupère plusieurs requêtes.
- `verify()` : s’assure qu’aucune requête “en attente” n’a été oubliée.

### 2.3 Tester un flux Observable

Dans Angular, `HttpClient` renvoie des **Observables**. Le test doit :
- s’abonner (`subscribe`) et vérifier les valeurs reçues
- ou utiliser `firstValueFrom/lastValueFrom` pour simplifier

---

## 3) Mise en place : structure type d’un test HTTP

### 3.1 Exemple de service à tester

```ts
// user.service.ts
import { Injectable } from '@angular/core';
import { HttpClient, HttpErrorResponse } from '@angular/common/http';
import { Observable, catchError, map, throwError } from 'rxjs';

export interface UserDto {
  id: number;
  name: string;
  email: string;
}

@Injectable({ providedIn: 'root' })
export class UserService {
  private readonly baseUrl = '/api/users';

  constructor(private http: HttpClient) {}

  getUsers(): Observable<UserDto[]> {
    return this.http.get<UserDto[]>(this.baseUrl);
  }

  getUser(id: number): Observable<UserDto> {
    return this.http.get<UserDto>(`${this.baseUrl}/${id}`);
  }

  createUser(payload: { name: string; email: string }): Observable<UserDto> {
    return this.http.post<UserDto>(this.baseUrl, payload).pipe(
      map(user => ({ ...user, name: user.name.trim() })),
      catchError((err: HttpErrorResponse) => {
        // Exemple : transformer l’erreur en message métier
        if (err.status === 409) {
          return throwError(() => new Error('EMAIL_ALREADY_EXISTS'));
        }
        return throwError(() => new Error('UNKNOWN_ERROR'));
      })
    );
  }
}
```

### 3.2 Squelette de test avec `TestBed`

```ts
// user.service.spec.ts
import { TestBed } from '@angular/core/testing';
import { HttpClientTestingModule, HttpTestingController } from '@angular/common/http/testing';
import { UserService, UserDto } from './user.service';

describe('UserService (HTTP tests)', () => {
  let service: UserService;
  let httpMock: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [HttpClientTestingModule],
      providers: [UserService],
    });

    service = TestBed.inject(UserService);
    httpMock = TestBed.inject(HttpTestingController);
  });

  afterEach(() => {
    // Garantit qu’aucune requête ne reste sans réponse
    httpMock.verify();
  });

  it('should be created', () => {
    expect(service).toBeTruthy();
  });
});
```

**Points essentiels :**
- `HttpClientTestingModule` est dans `imports`
- `HttpTestingController` est injecté
- `httpMock.verify()` est appelé dans `afterEach`

---

## 4) Vérifier qu’un service envoie les bonnes requêtes

### 4.1 Tester une requête GET simple

```ts
it('getUsers() should call GET /api/users and return users', () => {
  const mockUsers: UserDto[] = [
    { id: 1, name: 'Alice', email: 'alice@mail.com' },
    { id: 2, name: 'Bob', email: 'bob@mail.com' },
  ];

  let received: UserDto[] | undefined;

  service.getUsers().subscribe(users => {
    received = users;
  });

  const req = httpMock.expectOne('/api/users');
  expect(req.request.method).toBe('GET');

  // Simule la réponse HTTP
  req.flush(mockUsers);

  expect(received).toEqual(mockUsers);
});
```

### 4.2 Tester l’URL dynamique (path param)

```ts
it('getUser(id) should call GET /api/users/:id', () => {
  const mockUser: UserDto = { id: 10, name: 'Chloé', email: 'chloe@mail.com' };

  let received: UserDto | undefined;

  service.getUser(10).subscribe(user => (received = user));

  const req = httpMock.expectOne('/api/users/10');
  expect(req.request.method).toBe('GET');

  req.flush(mockUser);

  expect(received).toEqual(mockUser);
});
```

---

## 5) Tester un POST : body, headers, transformations

### 5.1 Vérifier le body envoyé

```ts
it('createUser() should POST payload to /api/users', () => {
  const payload = { name: '  Alice  ', email: 'alice@mail.com' };
  const apiResponse: UserDto = { id: 1, name: '  Alice  ', email: 'alice@mail.com' };

  let received: UserDto | undefined;

  service.createUser(payload).subscribe(user => (received = user));

  const req = httpMock.expectOne('/api/users');
  expect(req.request.method).toBe('POST');
  expect(req.request.body).toEqual(payload);

  req.flush(apiResponse);

  // Vérifie la transformation map(name.trim())
  expect(received).toEqual({ id: 1, name: 'Alice', email: 'alice@mail.com' });
});
```

### 5.2 Vérifier headers / params (si utilisés)

Si votre service ajoute des headers / params, vous pouvez les tester :

```ts
// Exemple si votre service utilisait :
// this.http.get(url, { params: new HttpParams().set('q', search) })

it('should send query param q', () => {
  // ... appel service ...
  const req = httpMock.expectOne(r => r.url === '/api/users' && r.params.get('q') === 'alice');
  expect(req.request.method).toBe('GET');
  req.flush([]);
});
```

---

## 6) Simuler des erreurs : statuts HTTP et erreurs réseau

### 6.1 Simuler une erreur HTTP (ex: 409)

```ts
it('createUser() should map 409 to EMAIL_ALREADY_EXISTS error', (done) => {
  const payload = { name: 'Alice', email: 'alice@mail.com' };

  service.createUser(payload).subscribe({
    next: () => done.fail('Expected an error'),
    error: (err: Error) => {
      expect(err.message).toBe('EMAIL_ALREADY_EXISTS');
      done();
    },
  });

  const req = httpMock.expectOne('/api/users');
  expect(req.request.method).toBe('POST');

  req.flush(
    { message: 'Conflict' },
    { status: 409, statusText: 'Conflict' }
  );
});
```

### 6.2 Simuler une erreur “réseau” (status 0)

```ts
it('createUser() should map network error to UNKNOWN_ERROR', (done) => {
  const payload = { name: 'Alice', email: 'alice@mail.com' };

  service.createUser(payload).subscribe({
    next: () => done.fail('Expected an error'),
    error: (err: Error) => {
      expect(err.message).toBe('UNKNOWN_ERROR');
      done();
    },
  });

  const req = httpMock.expectOne('/api/users');

  const mockError = new ProgressEvent('error');
  req.error(mockError, { status: 0, statusText: 'Unknown Error' });
});
```

---

## 7) Tester plusieurs requêtes et scénarios

### 7.1 Quand un appel déclenche plusieurs requêtes

Si une méthode effectue plusieurs appels HTTP (ou si elle est appelée plusieurs fois), utilisez `match` :

```ts
it('should allow matching multiple requests', () => {
  service.getUsers().subscribe();
  service.getUsers().subscribe();

  const requests = httpMock.match('/api/users');
  expect(requests.length).toBe(2);
  requests.forEach(r => {
    expect(r.request.method).toBe('GET');
    r.flush([]);
  });
});
```

### 7.2 Vérifier qu’aucun appel “inattendu” n’est parti

Le `verify()` en `afterEach` fournit une garantie forte :
- si vous oubliez un `flush()`
- ou si votre service lance une requête non prévue

Le test échoue, ce qui **durcit** votre suite de tests.

---

## 8) Bonnes pratiques

### 8.1 Toujours appeler `httpMock.verify()`

- Dans `afterEach` systématiquement.
- Permet de détecter les requêtes non consommées.

### 8.2 Tester le contrat HTTP, pas l’implémentation interne

- Vérifier : **URL**, **méthode**, **body**, **params**, **headers**, **réponse traitée**, **erreurs**.
- Éviter de tester des détails non pertinents (variables locales, etc.).

### 8.3 Garder les tests lisibles

- Un test = un scénario clair.
- Préférer des DTO simples.
- Nommer explicitement : `should call GET /... and return ...`.

### 8.4 Préférer des fonctions prédicats pour `expectOne`

Plus robuste si l’URL contient des paramètres :

```ts
const req = httpMock.expectOne(r => r.method === 'GET' && r.url === '/api/users');
```

---

## 9) Atelier pratique (guidé)

### Objectif
Écrire une batterie de tests couvrant :
1. GET liste
2. GET détail
3. POST succès + transformation
4. POST erreur 409
5. erreur réseau

### Étapes
1. Créer le service (ou reprendre celui fourni)
2. Configurer `HttpClientTestingModule` et `HttpTestingController`
3. Implémenter les tests unitaires
4. Ajouter `verify()`
5. Refactor : rendre les tests plus lisibles (helpers, constantes)

---

## 10) Quiz de consolidation

1. Quel module faut-il importer pour tester `HttpClient` sans réseau ?
2. À quoi sert `httpMock.verify()` ?
3. Différence entre `expectOne` et `match` ?
4. Comment simuler une erreur HTTP 500 avec `HttpTestingController` ?
5. Comment simuler une erreur réseau (status 0) ?

---

## 11) Résumé

- `HttpTestingController` permet de **mock** les requêtes HTTP et de valider le **contrat** entre votre service et l’API.
- Vous pouvez tester les scénarios **succès** et **erreur** de manière déterministe.
- Le trio gagnant :
  1. `HttpClientTestingModule`
  2. `HttpTestingController.expectOne()/match()`
  3. `httpMock.verify()`

---

## Annexes — Cheatsheet

### Pattern minimal

```ts
service.someCall().subscribe(result => {
  expect(result).toEqual(expected);
});

const req = httpMock.expectOne('/api/endpoint');
expect(req.request.method).toBe('GET');
req.flush(expected);
```

### Erreur HTTP

```ts
req.flush({ message: 'Server error' }, { status: 500, statusText: 'Server Error' });
```

### Erreur réseau

```ts
req.error(new ProgressEvent('error'), { status: 0, statusText: 'Unknown Error' });
```
