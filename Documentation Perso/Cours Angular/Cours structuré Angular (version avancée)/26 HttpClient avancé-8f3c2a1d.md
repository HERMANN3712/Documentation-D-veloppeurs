# Formation Angular — HttpClient avancé

> **Public** : développeurs Angular intermédiaires
>
> **Pré‑requis** : TypeScript, RxJS (bases), services/injection de dépendances, notions de REST
>
> **Durée indicative** : 1 jour (7h) ou 2 demi‑journées
>
> **Objectifs pédagogiques**
>
> - Maîtriser `HttpClient` en mode **typé** (DTO), et la construction d’URLs via `HttpParams`.
> - Savoir gérer **headers**, **auth**, **content negotiation**.
> - Construire des **pipelines RxJS** robustes (mapping, cancellation, orchestration, pagination, parallélisme).
> - Centraliser et industrialiser les appels via **services dédiés**, **interceptors** et conventions.
> - Mettre en place **retries**, **timeouts**, **backoff**, et **mise en cache**.
> - Implémenter une **stratégie de gestion d’erreurs** cohérente (UI/UX + observabilité).

---

## Plan de la formation

1. **Rappels et architecture HttpClient**
   - Configuration (providers), `HttpClient`, `HttpRequest`, `HttpResponse`
   - JSON par défaut, typage et désérialisation
2. **Typage avancé & DTO (Data Transfer Objects)**
   - Génériques, modèles immutables, mappers
   - `observe` & `responseType` et leurs impacts sur le typage
3. **Paramètres, URLs, query strings et pagination**
   - `HttpParams`, encodage, tableaux, filtres complexes
4. **Headers & contenu : auth, lang, versioning, content types**
   - `HttpHeaders`, `set` vs `append`
   - Contrats API (Accept / Content-Type)
5. **Gestion des erreurs experte**
   - Modèle d’erreur, `HttpErrorResponse`, cas réseau vs serveur
   - Stratégies : retry sélectif, erreurs métiers, fallback
6. **Pipelines RxJS pour HTTP**
   - `switchMap`, `mergeMap`, `concatMap`, `exhaustMap`
   - Annulation, debouncing, orchestration
7. **Centralisation des appels : services, facades, interceptors**
   - Service API dédié, séparation des préoccupations
   - Interceptors : auth, corrélation, logs, erreurs, cache
8. **Retries, timeouts et résilience**
   - `retry`, `retryWhen`, backoff exponentiel, jitter
   - `timeout`, circuit breaker (approche)
9. **Mise en cache HTTP côté client**
   - Cache mémoire, invalidation, partage de requête, `shareReplay`
   - Cache via interceptor + clés
10. **Atelier fil rouge**
   - Implémenter un mini SDK de client API (produits)
   - Ajouter cache, retry, timeout, gestion d’erreurs et observabilité

---

# 1) Rappels : HttpClient en Angular

## 1.1. Où vit HttpClient ?

`HttpClient` est fourni par `HttpClientModule` (Angular <= 14) ou via `provideHttpClient()` (Angular >= 15+ standalone).

### Standalone (recommandé Angular moderne)

```ts
import { bootstrapApplication } from '@angular/platform-browser';
import { provideHttpClient, withInterceptors } from '@angular/common/http';

bootstrapApplication(AppComponent, {
  providers: [
    provideHttpClient(
      withInterceptors([]) // on ajoutera nos interceptors plus loin
    )
  ]
});
```

### Module classique

```ts
import { HttpClientModule } from '@angular/common/http';

@NgModule({
  imports: [HttpClientModule]
})
export class AppModule {}
```

## 1.2. Les types manipulés

- `HttpClient` : API haut niveau (`get/post/put/delete/...`).
- `HttpRequest<T>` : requête bas niveau (méthode, URL, headers, body…).
- `HttpResponse<T>` : réponse complète (status, headers, body).
- `HttpEvent<T>` : flux d’événements (progress upload/download, response, etc.).

---

# 2) Typage avancé, DTO et sérialisation

## 2.1. Typage basique : génériques

```ts
interface UserDto {
  id: string;
  email: string;
  roles: string[];
}

this.http.get<UserDto>('/api/users/me');
```

> Remarque : Angular ne « désérialise » pas en classe, il parse du JSON en objets JS.

## 2.2. DTO vs modèles de domaine

Bonnes pratiques :

- **DTO** : reflète la structure API.
- **Modèle** : reflète les besoins du front (noms, types, invariants).
- **Mapper** : conversion DTO → modèle.

### Exemple

```ts
// DTO de l'API
export interface ProductDto {
  id: string;
  label: string;
  price_cents: number;
  created_at: string; // ISO
}

// Modèle front
export interface Product {
  id: string;
  label: string;
  price: number;      // euros
  createdAt: Date;
}

export function mapProductDto(dto: ProductDto): Product {
  return {
    id: dto.id,
    label: dto.label,
    price: dto.price_cents / 100,
    createdAt: new Date(dto.created_at)
  };
}
```

### Appel typé + mapping

```ts
this.http.get<ProductDto[]>('/api/products').pipe(
  map(dtos => dtos.map(mapProductDto))
);
```

## 2.3. `observe` et `responseType` : impacts

### Lire le body uniquement (défaut)

```ts
this.http.get<ProductDto[]>('/api/products');
```

### Lire la réponse complète (status + headers)

```ts
this.http.get<ProductDto[]>('/api/products', { observe: 'response' });
```

Type renvoyé : `Observable<HttpResponse<ProductDto[]>>`.

### Télécharger un fichier (blob)

```ts
this.http.get('/api/export', {
  responseType: 'blob'
});
```

> Attention au typage : `responseType: 'blob'` impose `Observable<Blob>`.

### Suivre les événements (progress)

```ts
this.http.post('/api/upload', file, {
  reportProgress: true,
  observe: 'events'
});
```

Type : `Observable<HttpEvent<any>>`.

---

# 3) Paramètres, URLs et pagination

## 3.1. `HttpParams` : immutabilité

`HttpParams` est immutable : chaque `set/append` retourne une nouvelle instance.

```ts
let params = new HttpParams();
params = params.set('page', 1);
params = params.set('pageSize', 20);

this.http.get('/api/products', { params });
```

## 3.2. Tableaux et filtres

### Tableau simple (multi-params)

```ts
let params = new HttpParams();
['angular', 'rxjs'].forEach(tag => {
  params = params.append('tags', tag);
});
```

Génère : `?tags=angular&tags=rxjs`.

### Encodage personnalisé

Pour certains backends, on veut contrôler l’encodage (espaces, `+`, etc.).

```ts
import { HttpParams, HttpUrlEncodingCodec } from '@angular/common/http';

class CustomCodec extends HttpUrlEncodingCodec {
  override encodeValue(v: string): string {
    return super.encodeValue(v).replace(/%20/g, '+');
  }
}

const params = new HttpParams({ encoder: new CustomCodec() })
  .set('q', 'hello world');
```

## 3.3. Pagination & tri (convention)

Proposer une convention de paramètres :

- `page`, `pageSize`
- `sort=field,asc|desc`
- `filter[field]=value` ou `q=`

Exemple :

```ts
interface ProductQuery {
  page: number;
  pageSize: number;
  q?: string;
  sort?: string;
}

function toHttpParams(q: ProductQuery): HttpParams {
  let p = new HttpParams()
    .set('page', q.page)
    .set('pageSize', q.pageSize);
  if (q.q) p = p.set('q', q.q);
  if (q.sort) p = p.set('sort', q.sort);
  return p;
}
```

---

# 4) Headers & contenu

## 4.1. `HttpHeaders` : `set` vs `append`

- `set` remplace la valeur.
- `append` ajoute une autre occurrence du même header.

```ts
let headers = new HttpHeaders();
headers = headers.set('Accept', 'application/json');
headers = headers.set('X-App-Version', '1.2.3');
```

## 4.2. Auth : bearer token (sans dupliquer partout)

Ne pas mettre l’auth dans chaque appel : utiliser un **interceptor**.

---

# 5) Gestion des erreurs experte

## 5.1. Comprendre `HttpErrorResponse`

Cas fréquents :

- **Erreur réseau / CORS / offline** : `status = 0`.
- **Erreur serveur** : `status` 4xx/5xx.
- **Erreur métier** : API renvoie un objet d’erreur structuré.

Exemple de format d’erreur côté API :

```json
{
  "code": "VALIDATION_ERROR",
  "message": "Invalid payload",
  "details": {
    "email": "Already used"
  }
}
```

## 5.2. Stratégie : normaliser l’erreur

Créer un type d’erreur front :

```ts
export type AppErrorKind = 'Network' | 'Unauthorized' | 'Forbidden' | 'NotFound' | 'Validation' | 'Server' | 'Unknown';

export interface AppError {
  kind: AppErrorKind;
  status?: number;
  message: string;
  details?: unknown;
  correlationId?: string;
}
```

Mapper depuis `HttpErrorResponse` :

```ts
import { HttpErrorResponse } from '@angular/common/http';

export function toAppError(err: unknown): AppError {
  if (!(err instanceof HttpErrorResponse)) {
    return { kind: 'Unknown', message: 'Unexpected error', details: err };
  }

  if (err.status === 0) {
    return { kind: 'Network', status: 0, message: 'Network error / offline', details: err.error };
  }

  if (err.status === 401) return { kind: 'Unauthorized', status: 401, message: 'Unauthorized' };
  if (err.status === 403) return { kind: 'Forbidden', status: 403, message: 'Forbidden' };
  if (err.status === 404) return { kind: 'NotFound', status: 404, message: 'Not found' };

  if (err.status === 400) {
    return {
      kind: 'Validation',
      status: 400,
      message: err.error?.message ?? 'Validation error',
      details: err.error?.details ?? err.error
    };
  }

  if (err.status >= 500) {
    return { kind: 'Server', status: err.status, message: 'Server error', details: err.error };
  }

  return { kind: 'Unknown', status: err.status, message: err.message, details: err.error };
}
```

## 5.3. Propager ou gérer ?

- **Service API** : renvoie un `Observable` et normalise (erreur **technique**).
- **UI / Facade** : décide affichage, retry manuel, redirection login…

Exemple :

```ts
this.api.getProducts().pipe(
  catchError(err => throwError(() => toAppError(err)))
);
```

---

# 6) Pipelines RxJS appliqués à HTTP

## 6.1. Choisir le bon opérateur : `switchMap/mergeMap/concatMap/exhaustMap`

### Recherche live : `switchMap`

Annule la requête précédente lorsqu’une nouvelle valeur arrive.

```ts
searchControl.valueChanges.pipe(
  debounceTime(250),
  distinctUntilChanged(),
  switchMap(q => this.api.searchProducts(q))
);
```

### Charger plusieurs ressources en parallèle : `mergeMap` ou `forkJoin`

```ts
forkJoin({
  me: this.api.getMe(),
  settings: this.api.getSettings()
});
```

### Traitement en file : `concatMap`

```ts
from(ids).pipe(
  concatMap(id => this.api.deleteProduct(id))
);
```

### Empêcher double clic : `exhaustMap`

```ts
saveClicks$.pipe(
  exhaustMap(() => this.api.save(form.value))
);
```

## 6.2. Annulation & composant : patterns

### Pattern `takeUntilDestroyed` (Angular >= 16)

```ts
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';

this.api.getProducts().pipe(
  takeUntilDestroyed()
).subscribe();
```

### Pattern async pipe (préféré)

Éviter les `subscribe()` manuels au niveau composant quand possible.

```ts
products$ = this.api.getProducts();
```

---

# 7) Centralisation : services dédiés & interceptors

## 7.1. Service API dédié (mini-SDK)

Objectif :

- une seule source de vérité pour les URLs
- typage DTO
- mapping DTO → modèles
- options communes (headers, params)

### Exemple `ProductsApi`

```ts
import { Injectable } from '@angular/core';
import { HttpClient, HttpParams } from '@angular/common/http';
import { map, Observable } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class ProductsApi {
  private readonly baseUrl = '/api/products';

  constructor(private readonly http: HttpClient) {}

  list(query: { page: number; pageSize: number; q?: string }): Observable<Product[]> {
    let params = new HttpParams()
      .set('page', query.page)
      .set('pageSize', query.pageSize);
    if (query.q) params = params.set('q', query.q);

    return this.http.get<ProductDto[]>(this.baseUrl, { params }).pipe(
      map(dtos => dtos.map(mapProductDto))
    );
  }

  getById(id: string): Observable<Product> {
    return this.http.get<ProductDto>(`${this.baseUrl}/${id}`).pipe(
      map(mapProductDto)
    );
  }

  create(payload: { label: string; price: number }): Observable<Product> {
    const dtoPayload = {
      label: payload.label,
      price_cents: Math.round(payload.price * 100)
    };

    return this.http.post<ProductDto>(this.baseUrl, dtoPayload).pipe(
      map(mapProductDto)
    );
  }
}
```

## 7.2. Interceptor d’authentification

### Source de token

```ts
@Injectable({ providedIn: 'root' })
export class AuthTokenService {
  getToken(): string | null {
    return localStorage.getItem('access_token');
  }
}
```

### Interceptor (Angular moderne fonctionnel)

```ts
import { HttpInterceptorFn } from '@angular/common/http';
import { inject } from '@angular/core';

export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const token = inject(AuthTokenService).getToken();
  if (!token) return next(req);

  return next(req.clone({
    setHeaders: {
      Authorization: `Bearer ${token}`
    }
  }));
};
```

## 7.3. Correlation ID, logs, tracing

Ajouter un identifiant de corrélation pour suivre une requête bout à bout.

```ts
export const correlationInterceptor: HttpInterceptorFn = (req, next) => {
  const correlationId = crypto.randomUUID();

  const cloned = req.clone({
    setHeaders: { 'X-Correlation-Id': correlationId }
  });

  return next(cloned);
};
```

> Côté server, propager ce header dans les logs.

---

# 8) Retries, backoff, timeout : résilience

## 8.1. Retry simple

```ts
this.http.get('/api/products').pipe(
  retry(2)
);
```

Limite : retry **pour toutes** les erreurs, sans discrimination.

## 8.2. Retry sélectif + backoff exponentiel (recommandé)

Créer une stratégie :

- retry seulement sur : status 0, 502, 503, 504
- backoff exponentiel : 200ms, 400ms, 800ms…
- jitter : petite randomisation pour éviter « thundering herd »

```ts
import { HttpErrorResponse } from '@angular/common/http';
import { timer } from 'rxjs';
import { retry } from 'rxjs/operators';

function isRetryable(err: unknown): boolean {
  return err instanceof HttpErrorResponse &&
    (err.status === 0 || [502, 503, 504].includes(err.status));
}

export function retryBackoff(maxRetries = 3, initialMs = 200) {
  return retry({
    count: maxRetries,
    delay: (error, retryCount) => {
      if (!isRetryable(error)) throw error;

      const exp = initialMs * Math.pow(2, retryCount - 1);
      const jitter = Math.floor(Math.random() * 100);
      return timer(exp + jitter);
    }
  });
}
```

Usage :

```ts
this.http.get('/api/products').pipe(
  retryBackoff(3, 200)
);
```

## 8.3. Timeout

Utiliser `timeout` pour éviter les requêtes qui pendent.

```ts
import { timeout } from 'rxjs/operators';

this.http.get('/api/products').pipe(
  timeout({ first: 5000 })
);
```

Gestion : attraper l’erreur de timeout via `catchError` (elle sera une `TimeoutError`).

```ts
import { TimeoutError } from 'rxjs';

catchError(err => {
  if (err instanceof TimeoutError) {
    return throwError(() => ({ kind: 'Network', message: 'Request timeout' } satisfies AppError));
  }
  return throwError(() => toAppError(err));
})
```

## 8.4. Circuit breaker (approche)

Angular/RxJS n’a pas un « circuit breaker » prêt à l’emploi, mais vous pouvez :

- compter les erreurs récentes
- ouvrir le circuit pendant X secondes
- renvoyer un fallback (cache) ou une erreur rapide

Recommandation : implémenter côté service (facade) ou via un opérateur custom si besoin.

---

# 9) Mise en cache côté client

## 9.1. Cache au niveau service : `shareReplay`

Cas d’usage : éviter plusieurs appels identiques pour un même écran.

```ts
private productsOnce$?: Observable<Product[]>;

getProductsOnce(): Observable<Product[]> {
  if (!this.productsOnce$) {
    this.productsOnce$ = this.api.list({ page: 1, pageSize: 50 }).pipe(
      shareReplay({ bufferSize: 1, refCount: true })
    );
  }
  return this.productsOnce$;
}
```

Invalidation :

- réinitialiser `productsOnce$ = undefined` après `create/update/delete`
- ou utiliser un `Subject<void>` en tant que signal d’invalidation

## 9.2. Cache générique en mémoire avec TTL

### Implémentation

```ts
interface CacheEntry<T> {
  value$: Observable<T>;
  expiresAt: number;
}

@Injectable({ providedIn: 'root' })
export class HttpMemoryCache {
  private readonly store = new Map<string, CacheEntry<unknown>>();

  getOrSet<T>(key: string, ttlMs: number, factory: () => Observable<T>): Observable<T> {
    const now = Date.now();
    const existing = this.store.get(key) as CacheEntry<T> | undefined;

    if (existing && existing.expiresAt > now) {
      return existing.value$;
    }

    const value$ = factory().pipe(
      shareReplay({ bufferSize: 1, refCount: false })
    );

    this.store.set(key, { value$, expiresAt: now + ttlMs });
    return value$;
  }

  invalidate(prefix?: string) {
    if (!prefix) return this.store.clear();
    for (const k of this.store.keys()) {
      if (k.startsWith(prefix)) this.store.delete(k);
    }
  }
}
```

### Usage

```ts
listCached(query: ProductQuery): Observable<Product[]> {
  const key = `products:list:${JSON.stringify(query)}`;
  return this.cache.getOrSet(key, 30_000, () => this.list(query));
}
```

## 9.3. Cache via interceptor (approche)

Un interceptor de cache peut :

- ne mettre en cache que les `GET`
- utiliser une clé (URL + params + headers pertinents)
- gérer TTL et invalidation (au moins sur mutating verbs)

> En pratique : cache fin au niveau service (métier) + cache technique (interceptor) selon besoins.

---

# 10) Atelier fil rouge : mini client API résilient

## 10.1. Spécifications

- Endpoint :
  - `GET /api/products?page=&pageSize=&q=`
  - `GET /api/products/:id`
  - `POST /api/products`
- Besoins :
  - auth par bearer token
  - correlation id
  - retry backoff sur erreurs réseau/503
  - timeout 5s
  - cache GET list (TTL 30s)
  - erreurs normalisées en `AppError`

## 10.2. Assemblage : providers

```ts
import { provideHttpClient, withInterceptors } from '@angular/common/http';

providers: [
  provideHttpClient(
    withInterceptors([
      correlationInterceptor,
      authInterceptor
    ])
  )
]
```

## 10.3. Pipeline type pour un appel

Dans le service API :

```ts
list(query: ProductQuery): Observable<Product[]> {
  return this.http.get<ProductDto[]>(this.baseUrl, { params: toHttpParams(query) }).pipe(
    timeout({ first: 5000 }),
    retryBackoff(3, 200),
    map(dtos => dtos.map(mapProductDto)),
    catchError(err => throwError(() => toAppError(err)))
  );
}
```

## 10.4. Check-list de validation

- [ ] Les requêtes portent `Authorization` si token.
- [ ] `X-Correlation-Id` présent.
- [ ] En cas de 503/offline, on observe plusieurs tentatives.
- [ ] En cas de lenteur, timeout en 5s.
- [ ] Deux abonnements simultanés à `listCached()` ne déclenchent qu’un seul call.
- [ ] Les erreurs sont converties en `AppError`.

---

# Annexes

## A) Recommandations de style et conventions

- Un service API par ressource (`ProductsApi`, `UsersApi`).
- DTO explicites et séparés des modèles UI.
- Ne pas faire de `subscribe` dans les services (laisser l’appelant décider).
- Utiliser `async` pipe en composant dès que possible.
- Centraliser auth/log/corrélation via interceptors.
- Documenter les conventions de query params.

## B) Points d’attention

- `shareReplay` peut provoquer des effets de « cache éternel » si mal configuré.
- Un retry mal ciblé peut aggraver une panne (flood). Toujours filtrer.
- Timeout trop agressif dégrade UX sur mobile.
- CORS et status 0 : distinguer offline, CORS, DNS.

## C) Exercices supplémentaires

1. Ajouter un interceptor de logs qui mesure le temps des requêtes (durée + status).
2. Ajouter un cache invalidé automatiquement après `POST/PUT/DELETE` sur un prefix.
3. Ajouter un mode « stale-while-revalidate » : afficher le cache puis rafraîchir en arrière-plan.

---

## Fin de formation

Le participant est capable de construire des clients HTTP Angular typés, résilients et maintenables, en combinant `HttpClient`, interceptors et RxJS (retries, timeout, cache), avec une stratégie d’erreurs homogène.
