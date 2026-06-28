# Formation Angular — HTTP et consommation d’API

> **Objectif** : maîtriser l’utilisation de **HttpClient** dans Angular pour consommer des API REST (ou assimilées), comprendre la logique **Observable** (RxJS), gérer erreurs, headers, query params, authentification, intercepteurs, téléchargement/téléversement, bonnes pratiques de structuration.

---

## 1) Pré-requis et public cible

### Public
- Développeurs Angular (débutant à intermédiaire) souhaitant consommer des API.
- Formateurs/consultants voulant un support complet.

### Pré-requis
- Angular CLI, notions de composants/modules.
- TypeScript (interfaces, generics, async).
- Notions de Promises (utile pour comprendre la différence avec RxJS).

### Environnement
- Angular 15+ (les exemples restent valables dès Angular 7+).
- Node.js LTS.
- Un backend disponible (ex: JSONPlaceholder) ou API maison.

---

## 2) Plan global de la formation

1. **Rappels HTTP & API** (REST, verbes, statuts, CORS)
2. **HttpClient : installation et prise en main**
3. **Requêtes GET/POST/PUT/PATCH/DELETE**
4. **Typage des réponses (DTO, interfaces, generics)**
5. **Query params, headers, options de requête**
6. **RxJS : Observables & opérateurs essentiels**
7. **Gestion des erreurs et stratégies de retry**
8. **Intercepteurs HTTP** (auth, logging, cache, gestion globalisée)
9. **Patterns de structuration : services, facades, environment, base URL**
10. **Cas avancés : upload, download, progress, annulation**
11. **Tests : HttpClientTestingModule, HttpTestingController**
12. **Bonnes pratiques & checklist de production**

---

## 3) Rappels : HTTP, REST et consommation d’API

### 3.1 Les verbes HTTP les plus utilisés
- **GET** : récupérer une ressource (idempotent).
- **POST** : créer une ressource.
- **PUT** : remplacer intégralement une ressource.
- **PATCH** : modifier partiellement une ressource.
- **DELETE** : supprimer une ressource.

### 3.2 Codes de statut importants
- **2xx** : succès (200 OK, 201 Created, 204 No Content)
- **3xx** : redirection / cache (304 Not Modified)
- **4xx** : erreur client (400, 401, 403, 404, 409)
- **5xx** : erreur serveur (500, 502, 503)

### 3.3 JSON et DTO
Dans Angular, vous travaillez souvent avec des **DTO** (Data Transfer Objects). Le typage côté frontend sert à :
- documenter la forme des données,
- réduire les erreurs,
- mieux auto-compléter.

### 3.4 CORS (Cross-Origin Resource Sharing)
Si votre SPA Angular est servie sur `http://localhost:4200` et l’API sur `http://localhost:3000`, le navigateur applique des règles CORS.
- Solution : configurer le backend (headers `Access-Control-Allow-Origin`, etc.).
- Un **proxy Angular** peut aider en dev.

---

## 4) Mise en place de HttpClient

### 4.1 Import de `HttpClientModule`

#### Angular avec NgModules (classique)
```ts
// app.module.ts
import { NgModule } from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';
import { HttpClientModule } from '@angular/common/http';

import { AppComponent } from './app.component';

@NgModule({
  declarations: [AppComponent],
  imports: [BrowserModule, HttpClientModule],
  bootstrap: [AppComponent]
})
export class AppModule {}
```

#### Angular standalone
```ts
// main.ts
import { bootstrapApplication } from '@angular/platform-browser';
import { provideHttpClient } from '@angular/common/http';
import { AppComponent } from './app/app.component';

bootstrapApplication(AppComponent, {
  providers: [provideHttpClient()]
});
```

### 4.2 Injection de HttpClient
```ts
import { HttpClient } from '@angular/common/http';
import { Injectable } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class ApiService {
  constructor(private http: HttpClient) {}
}
```

---

## 5) Premier service API : GET simple

### 5.1 Exemple de modèle
```ts
export interface PostDto {
  userId: number;
  id: number;
  title: string;
  body: string;
}
```

### 5.2 GET : récupérer une liste
```ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';
import { PostDto } from './models/post.dto';

@Injectable({ providedIn: 'root' })
export class PostsApi {
  private readonly baseUrl = 'https://jsonplaceholder.typicode.com';

  constructor(private http: HttpClient) {}

  listPosts(): Observable<PostDto[]> {
    return this.http.get<PostDto[]>(`${this.baseUrl}/posts`);
  }
}
```

### 5.3 Consommer dans un composant

#### Option A — `async` pipe (recommandé)
```ts
import { Component } from '@angular/core';
import { Observable } from 'rxjs';
import { PostsApi } from './posts.api';
import { PostDto } from './models/post.dto';

@Component({
  selector: 'app-posts',
  template: `
    <h2>Posts</h2>
    <ul>
      <li *ngFor="let p of posts$ | async">
        <strong>{{ p.title }}</strong>
      </li>
    </ul>
  `
})
export class PostsComponent {
  posts$: Observable<PostDto[]> = this.api.listPosts();

  constructor(private api: PostsApi) {}
}
```

#### Option B — `subscribe()` dans le composant (à utiliser avec prudence)
```ts
posts: PostDto[] = [];

ngOnInit(): void {
  this.api.listPosts().subscribe(data => {
    this.posts = data;
  });
}
```

> **Bonne pratique** : privilégier `async` pipe pour limiter les fuites mémoire et simplifier la gestion du cycle de vie.

---

## 6) POST, PUT, PATCH, DELETE

### 6.1 POST : créer
```ts
createPost(payload: Pick<PostDto, 'title' | 'body' | 'userId'>): Observable<PostDto> {
  return this.http.post<PostDto>(`${this.baseUrl}/posts`, payload);
}
```

### 6.2 PUT : remplacer
```ts
updatePost(id: number, payload: PostDto): Observable<PostDto> {
  return this.http.put<PostDto>(`${this.baseUrl}/posts/${id}`, payload);
}
```

### 6.3 PATCH : modifier partiellement
```ts
patchPost(id: number, payload: Partial<PostDto>): Observable<PostDto> {
  return this.http.patch<PostDto>(`${this.baseUrl}/posts/${id}`, payload);
}
```

### 6.4 DELETE : supprimer
```ts
deletePost(id: number): Observable<void> {
  return this.http.delete<void>(`${this.baseUrl}/posts/${id}`);
}
```

---

## 7) Options de requête : params, headers, observe, responseType

### 7.1 Query params
```ts
import { HttpParams } from '@angular/common/http';

searchPosts(userId?: number): Observable<PostDto[]> {
  let params = new HttpParams();
  if (userId != null) params = params.set('userId', userId);

  return this.http.get<PostDto[]>(`${this.baseUrl}/posts`, { params });
}
```

### 7.2 Headers
```ts
import { HttpHeaders } from '@angular/common/http';

getWithHeaders(): Observable<PostDto[]> {
  const headers = new HttpHeaders({
    'X-App-Version': '1.0.0'
  });

  return this.http.get<PostDto[]>(`${this.baseUrl}/posts`, { headers });
}
```

### 7.3 Accéder à la réponse complète
Par défaut, `HttpClient` retourne le **body**. Pour récupérer status + headers :
```ts
import { HttpResponse } from '@angular/common/http';

getFullResponse(): Observable<HttpResponse<PostDto[]>> {
  return this.http.get<PostDto[]>(`${this.baseUrl}/posts`, {
    observe: 'response'
  });
}
```

### 7.4 `responseType` (texte, blob…)
```ts
downloadText(): Observable<string> {
  return this.http.get('https://example.com/readme.txt', {
    responseType: 'text'
  });
}
```

---

## 8) RxJS : Observables pour la consommation d’API

### 8.1 Pourquoi Observable ?
- **Lazy** : rien ne s’exécute tant qu’on ne s’abonne pas.
- **Annulable** : on peut se désabonner.
- **Composable** : opérateurs RxJS (`map`, `switchMap`, `catchError`, …).

### 8.2 Opérateurs indispensables

#### `map` : transformer les données
```ts
import { map } from 'rxjs/operators';

listTitles(): Observable<string[]> {
  return this.listPosts().pipe(
    map(posts => posts.map(p => p.title))
  );
}
```

#### `tap` : effets de bord
```ts
import { tap } from 'rxjs/operators';

listWithLog(): Observable<PostDto[]> {
  return this.listPosts().pipe(
    tap(posts => console.log('Fetched posts:', posts.length))
  );
}
```

#### `switchMap` : enchaîner dépendances (ex: route param -> API)
```ts
import { ActivatedRoute } from '@angular/router';
import { switchMap } from 'rxjs/operators';

post$ = this.route.paramMap.pipe(
  switchMap(params => this.api.getPost(Number(params.get('id'))))
);
```

#### `shareReplay` : partager un résultat (cache mémoire)
```ts
import { shareReplay } from 'rxjs/operators';

posts$ = this.api.listPosts().pipe(
  shareReplay({ bufferSize: 1, refCount: true })
);
```

> Attention : `shareReplay` peut conserver en mémoire. À utiliser en connaissance de cause.

---

## 9) Gestion des erreurs HTTP

### 9.1 Erreurs côté Angular
`HttpClient` émet une erreur (branche `error`) quand le serveur répond en 4xx/5xx ou quand un problème réseau survient.

### 9.2 `catchError` : intercepter et transformer
```ts
import { catchError } from 'rxjs/operators';
import { throwError } from 'rxjs';
import { HttpErrorResponse } from '@angular/common/http';

listPostsSafe(): Observable<PostDto[]> {
  return this.http.get<PostDto[]>(`${this.baseUrl}/posts`).pipe(
    catchError((err: HttpErrorResponse) => {
      // Exemple : mapping en erreur métier
      const message = err.status === 0
        ? 'Réseau indisponible'
        : `Erreur API (${err.status})`;

      return throwError(() => new Error(message));
    })
  );
}
```

### 9.3 `retry` et `retryWhen` (stratégies simples)
```ts
import { retry } from 'rxjs/operators';

listWithRetry(): Observable<PostDto[]> {
  return this.http.get<PostDto[]>(`${this.baseUrl}/posts`).pipe(
    retry(2) // retente 2 fois en cas d'erreur
  );
}
```

> Bonne pratique : ne pas retry sur 4xx (erreur fonctionnelle). Préférer un retry sélectif via interceptors si nécessaire.

---

## 10) Intercepteurs HTTP (puissant et incontournable)

### 10.1 Cas d’usage
- Ajouter un **token JWT** à chaque requête.
- Logger les temps de réponse.
- Gérer un **spinner global**.
- Centraliser la gestion des erreurs.

### 10.2 Exemple : ajout d’un header Authorization

```ts
import { Injectable } from '@angular/core';
import {
  HttpInterceptor,
  HttpRequest,
  HttpHandler,
  HttpEvent
} from '@angular/common/http';
import { Observable } from 'rxjs';

@Injectable()
export class AuthInterceptor implements HttpInterceptor {
  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    const token = localStorage.getItem('token');

    if (!token) return next.handle(req);

    const authReq = req.clone({
      setHeaders: {
        Authorization: `Bearer ${token}`
      }
    });

    return next.handle(authReq);
  }
}
```

#### Enregistrement (NgModules)
```ts
import { HTTP_INTERCEPTORS } from '@angular/common/http';

providers: [
  { provide: HTTP_INTERCEPTORS, useClass: AuthInterceptor, multi: true }
]
```

#### Enregistrement (standalone)
```ts
import { provideHttpClient, withInterceptorsFromDi } from '@angular/common/http';

providers: [
  provideHttpClient(withInterceptorsFromDi())
]
```

---

## 11) Structurer proprement la couche API

### 11.1 Séparer responsabilités
- `*.api.ts` : appels HTTP bruts (DTO, endpoints)
- `*.service.ts` : logique métier / orchestration
- `*.facade.ts` ou store (NgRx) : état applicatif

### 11.2 Centraliser l’URL de base
Utiliser `environment` :
```ts
// environment.ts
export const environment = {
  apiUrl: 'https://api.example.com'
};
```

Puis :
```ts
import { environment } from '../environments/environment';

private readonly baseUrl = environment.apiUrl;
```

### 11.3 Normaliser les endpoints
Éviter les URLs en dur partout. Exemple :
```ts
const endpoints = {
  posts: () => `${environment.apiUrl}/posts`,
  post: (id: number) => `${environment.apiUrl}/posts/${id}`
};
```

---

## 12) Upload & Download (fichiers)

### 12.1 Upload `multipart/form-data`
```ts
uploadAvatar(file: File): Observable<any> {
  const form = new FormData();
  form.append('avatar', file);

  return this.http.post(`${this.baseUrl}/me/avatar`, form);
}
```

### 12.2 Suivi de progression
```ts
import { HttpEvent, HttpEventType } from '@angular/common/http';
import { filter, map } from 'rxjs/operators';

uploadWithProgress(file: File) {
  const form = new FormData();
  form.append('file', file);

  return this.http.post(`${this.baseUrl}/upload`, form, {
    reportProgress: true,
    observe: 'events'
  }).pipe(
    filter((event: HttpEvent<any>) => event.type === HttpEventType.UploadProgress),
    map(event => Math.round(100 * (event.loaded / (event.total ?? event.loaded))))
  );
}
```

### 12.3 Download de fichier (Blob)
```ts
downloadPdf(id: string): Observable<Blob> {
  return this.http.get(`${this.baseUrl}/reports/${id}`, {
    responseType: 'blob'
  });
}
```

---

## 13) Annulation et gestion du cycle de vie

### 13.1 Annuler via `takeUntilDestroyed` (Angular récent)
```ts
import { Component, DestroyRef, inject } from '@angular/core';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';

@Component({
  selector: 'app-demo',
  template: `...`
})
export class DemoComponent {
  private destroyRef = inject(DestroyRef);

  ngOnInit() {
    this.api.listPosts()
      .pipe(takeUntilDestroyed(this.destroyRef))
      .subscribe();
  }
}
```

> Si vous utilisez `async` pipe, Angular gère le subscribe/unsubscribe automatiquement.

---

## 14) Tests unitaires de la couche HTTP

### 14.1 Configuration
```ts
import { TestBed } from '@angular/core/testing';
import { HttpClientTestingModule, HttpTestingController } from '@angular/common/http/testing';

describe('PostsApi', () => {
  let api: PostsApi;
  let httpMock: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [HttpClientTestingModule],
      providers: [PostsApi]
    });

    api = TestBed.inject(PostsApi);
    httpMock = TestBed.inject(HttpTestingController);
  });

  afterEach(() => {
    httpMock.verify();
  });

  it('should list posts', () => {
    const mock = [{ userId: 1, id: 1, title: 'A', body: 'B' }];

    api.listPosts().subscribe(posts => {
      expect(posts.length).toBe(1);
      expect(posts[0].title).toBe('A');
    });

    const req = httpMock.expectOne('https://jsonplaceholder.typicode.com/posts');
    expect(req.request.method).toBe('GET');
    req.flush(mock);
  });
});
```

### 14.2 Tester les erreurs
```ts
it('should handle error', () => {
  api.listPosts().subscribe({
    next: () => fail('should error'),
    error: (err) => expect(err).toBeTruthy()
  });

  const req = httpMock.expectOne('https://jsonplaceholder.typicode.com/posts');
  req.flush('Boom', { status: 500, statusText: 'Server Error' });
});
```

---

## 15) Atelier pratique (proposé)

### Objectif
Construire une mini-feature “Posts” avec :
- liste + détail,
- recherche par `userId`,
- création,
- gestion d’erreur,
- interceptor d’auth fictif.

### Étapes
1. Créer `PostDto`.
2. Implémenter `PostsApi` (GET/POST/DELETE).
3. Créer composants `PostsListComponent` + `PostDetailComponent`.
4. Utiliser `async` pipe + `switchMap` sur param route.
5. Ajouter `catchError` et afficher un message utilisateur.
6. Ajouter `AuthInterceptor` (token stocké en localStorage).
7. Ajouter tests sur `PostsApi`.

---

## 16) Bonnes pratiques & checklist production

### API & sécurité
- Ne jamais stocker de secrets dans le frontend.
- Préférer short-lived tokens, refresh tokens géré côté backend.
- Valider côté backend ; le frontend ne sécurise pas.

### Code
- Centraliser `apiUrl` (environment)
- Nommage clair : `*.api.ts` pour endpoints, `*.service.ts` pour orchestration
- Typage systématique des retours `get<T>()`

### RxJS
- Éviter les `subscribe()` imbriqués → utiliser `switchMap/mergeMap/concatMap`.
- Privilégier `async` pipe.
- Gérer les erreurs au bon niveau (UI vs global via interceptor).

### UX
- Charger/erreur/empty states.
- Timeout éventuel (via opérateur RxJS `timeout`).

---

## 17) Résumé

- Angular fournit **HttpClient** pour exécuter des requêtes HTTP.
- Les appels retournent des **Observables** (RxJS) : composition, annulation, patterns réactifs.
- La consommation d’API robuste passe par : **typage**, **gestion d’erreurs**, **interceptors**, bonne **architecture** et **tests**.

---

## Annexes

### A) Proxy Angular (dev) — optionnel
`proxy.conf.json`
```json
{
  "/api": {
    "target": "http://localhost:3000",
    "secure": false,
    "changeOrigin": true
  }
}
```
Puis :
```bash
ng serve --proxy-config proxy.conf.json
```

### B) Snippets utiles

#### Timeout
```ts
import { timeout } from 'rxjs/operators';

this.http.get(url).pipe(timeout(8000));
```

#### Pagination (params)
```ts
getPaged(page: number, pageSize: number) {
  const params = new HttpParams()
    .set('page', page)
    .set('pageSize', pageSize);

  return this.http.get<any>(`${this.baseUrl}/items`, { params });
}
```
