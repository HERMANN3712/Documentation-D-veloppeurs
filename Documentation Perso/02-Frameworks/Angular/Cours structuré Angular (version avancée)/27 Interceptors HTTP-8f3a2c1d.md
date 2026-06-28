# Formation Angular (27) — Interceptors HTTP

> **Public** : développeurs Angular (niveau intermédiaire)  
> **Durée suggérée** : 3h30 à 1 journée (avec atelier)  
> **Pré-requis** : Angular CLI, RxJS (Observable, pipe), HttpClient, DI (providers)

---

## 1. Objectifs pédagogiques

À l’issue de cette formation, vous saurez :

- Expliquer le rôle des **HTTP interceptors** et leurs bénéfices en architecture d’entreprise.
- Créer, enregistrer et chaîner plusieurs interceptors.
- Ajouter des en-têtes (ex. **Bearer token**) et manipuler les requêtes/réponses.
- Centraliser **journalisation**, **gestion d’erreurs**, **retries**, **caching** et **refresh token**.
- Identifier les pièges (boucles d’interception, ordre, immutabilité, SSR, zones sensibles).
- Tester un interceptor et diagnostiquer des problèmes (réseau, CORS, 401).

---

## 2. Contexte et intérêt en entreprise

Les applications Angular consomment des APIs via `HttpClient`. Dans un SI (micro-services, SSO, sécurité, observabilité), on retrouve des préoccupations transverses :

- **Authentification/autorisation** : ajouter/renouveler un token, gérer 401/403.
- **Traçabilité** : IDs de corrélation, logs, mesure de performance.
- **Robustesse** : retries avec backoff, gestion homogène des erreurs.
- **Conformité** : masquage de données, en-têtes obligatoires.
- **Transformation** : normaliser le format (dates, enveloppes), mapping.

Les **interceptors** sont le point d’extension standard d’Angular pour traiter ces aspects **en un seul endroit**.

---

## 3. Rappels : pipeline HTTP Angular

- `HttpClient` déclenche une requête.
- La requête traverse une **chaîne d’interceptors** (dans un ordre donné).
- Le dernier maillon est le **backend** (implémentation qui effectue effectivement l’appel réseau).
- La réponse revient en sens inverse (dernier interceptor → premier), permettant la transformation et la gestion d’erreur.

### 3.1 Immutabilité

`HttpRequest` et `HttpHeaders` sont **immuables**. Pour modifier une requête, on doit utiliser :

- `req.clone({ headers, setHeaders, params, url, body, ... })`

---

## 4. Base technique : créer un interceptor

### 4.1 API historique : interface `HttpInterceptor`

Dans beaucoup de projets, vous verrez :

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
export class ExampleInterceptor implements HttpInterceptor {
  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    // req est immuable => clone
    const cloned = req.clone({
      setHeaders: {
        'X-App': 'angular-demo'
      }
    });

    return next.handle(cloned);
  }
}
```

Enregistrement (approche module/historique) :

```ts
import { HTTP_INTERCEPTORS } from '@angular/common/http';

providers: [
  {
    provide: HTTP_INTERCEPTORS,
    useClass: ExampleInterceptor,
    multi: true,
  }
]
```

### 4.2 API moderne : interceptor fonctionnel (Angular récent)

Dans un projet standalone (ou moderne), on privilégie souvent :

```ts
import { HttpInterceptorFn } from '@angular/common/http';

export const exampleInterceptor: HttpInterceptorFn = (req, next) => {
  const cloned = req.clone({
    setHeaders: { 'X-App': 'angular-demo' }
  });
  return next(cloned);
};
```

Enregistrement :

```ts
import { provideHttpClient, withInterceptors } from '@angular/common/http';

bootstrapApplication(AppComponent, {
  providers: [
    provideHttpClient(withInterceptors([exampleInterceptor]))
  ]
});
```

> **Note** : les deux approches coexistent. L’essentiel est le concept de pipeline.

---

## 5. Ordre d’exécution et chaînage

### 5.1 Ordre côté requête

Si vous déclarez `[A, B, C]` :

- Requête : A → B → C → backend
- Réponse : backend → C → B → A

### 5.2 Recommandations d’architecture

- Mettre la **sécurité/auth** tôt (pour signer les requêtes).
- Mettre **logs** et **correlation-id** tôt.
- Mettre la **gestion d’erreur** plutôt tard si elle dépend d’un état (refresh), mais cela dépend des scénarios.

---

## 6. Cas d’usage 1 — Ajouter un token (Bearer)

### 6.1 Service d’accès au token

```ts
@Injectable({ providedIn: 'root' })
export class AuthTokenService {
  getAccessToken(): string | null {
    return localStorage.getItem('access_token');
  }
}
```

### 6.2 Interceptor d’authentification

```ts
@Injectable()
export class AuthInterceptor implements HttpInterceptor {
  constructor(private tokenService: AuthTokenService) {}

  intercept(req: HttpRequest<any>, next: HttpHandler) {
    const token = this.tokenService.getAccessToken();

    // Ne pas ajouter de token si absent ou si appel externe.
    const isApiCall = req.url.startsWith('/api');

    if (!token || !isApiCall) {
      return next.handle(req);
    }

    return next.handle(
      req.clone({
        setHeaders: {
          Authorization: `Bearer ${token}`
        }
      })
    );
  }
}
```

### 6.3 Points d’attention

- Exclure les endpoints **login/refresh** pour éviter les boucles.
- Gérer les appels vers des domaines externes (CDN, maps…).
- Attention à la sécurité : ne pas injecter le token sur un domaine tiers.

---

## 7. Cas d’usage 2 — Journalisation et métriques

Objectif : mesurer la durée et tracer les erreurs.

```ts
import { tap, finalize } from 'rxjs/operators';

@Injectable()
export class LoggingInterceptor implements HttpInterceptor {
  intercept(req: HttpRequest<any>, next: HttpHandler) {
    const started = performance.now();

    return next.handle(req).pipe(
      tap({
        next: () => {
          // On ne log pas forcément le body (sensible)
        },
        error: (err) => {
          console.error('[HTTP ERROR]', req.method, req.url, err.status, err.message);
        }
      }),
      finalize(() => {
        const elapsed = Math.round(performance.now() - started);
        console.debug('[HTTP]', req.method, req.url, `${elapsed}ms`);
      })
    );
  }
}
```

### Bonnes pratiques

- Utiliser un service d’observabilité (ex. OpenTelemetry, Datadog, Elastic APM).
- Nettoyer/redacter les données sensibles (RGPD).
- Ajouter un **Correlation ID** (voir section suivante).

---

## 8. Cas d’usage 3 — Correlation ID (traçabilité inter-systèmes)

But : injecter un identifiant pour corréler logs front ↔ back.

```ts
function uuidLike(): string {
  return Math.random().toString(16).slice(2) + '-' + Date.now().toString(16);
}

@Injectable()
export class CorrelationIdInterceptor implements HttpInterceptor {
  intercept(req: HttpRequest<any>, next: HttpHandler) {
    const correlationId = uuidLike();
    return next.handle(
      req.clone({ setHeaders: { 'X-Correlation-Id': correlationId } })
    );
  }
}
```

> En entreprise, le backend propage aussi cet ID sur les logs et réponses.

---

## 9. Cas d’usage 4 — Gestion uniforme des erreurs

### 9.1 Transformer les erreurs vers un modèle applicatif

Créer un type d’erreur homogène :

```ts
export interface AppHttpError {
  status: number;
  url?: string;
  message: string;
  details?: unknown;
}
```

Interceptor :

```ts
import { catchError, throwError } from 'rxjs';

@Injectable()
export class ErrorMappingInterceptor implements HttpInterceptor {
  intercept(req: HttpRequest<any>, next: HttpHandler) {
    return next.handle(req).pipe(
      catchError((err) => {
        const mapped: AppHttpError = {
          status: err.status ?? 0,
          url: req.url,
          message: this.mapMessage(err),
          details: err.error
        };

        return throwError(() => mapped);
      })
    );
  }

  private mapMessage(err: any): string {
    if (err.status === 0) return 'Erreur réseau (CORS, offline, DNS…)';
    if (err.status === 401) return 'Non authentifié';
    if (err.status === 403) return 'Accès interdit';
    if (err.status >= 500) return 'Erreur serveur';
    return 'Erreur lors de la requête';
  }
}
```

### 9.2 Affichage centralisé (ex. toast)

L’interceptor ne doit pas forcément afficher des toasts (risque d’effets de bord). Souvent :

- Interceptor : **normalise**
- Une couche UI (facade/effets NgRx, service global) : **affiche**

---

## 10. Cas d’usage 5 — Refresh token automatique (401)

Problème : le token expire. On veut :

1. Intercepter les réponses 401
2. Lancer une requête `refresh`
3. Rejouer la requête initiale avec un nouveau token
4. Gérer la concurrence (plusieurs 401 simultanés)

### 10.1 Service d’auth (simplifié)

```ts
@Injectable({ providedIn: 'root' })
export class AuthService {
  private refreshing = false;

  refreshToken(): Observable<{ accessToken: string }> {
    // Exemple : POST /api/auth/refresh
    // IMPORTANT : cette requête doit être exclue de l'interceptor d'ajout de token si besoin
    return this.http.post<{ accessToken: string }>('/api/auth/refresh', {});
  }

  setAccessToken(token: string) {
    localStorage.setItem('access_token', token);
  }

  constructor(private http: HttpClient) {}
}
```

### 10.2 Interceptor de refresh (version robuste avec file d’attente)

Principe :

- Utiliser un `Subject` pour diffuser le nouveau token aux requêtes en attente.
- Éviter de lancer plusieurs refresh simultanés.

```ts
import { BehaviorSubject, catchError, filter, switchMap, take, throwError } from 'rxjs';

@Injectable()
export class RefreshTokenInterceptor implements HttpInterceptor {
  private refreshInProgress = false;
  private token$ = new BehaviorSubject<string | null>(null);

  constructor(
    private auth: AuthService,
    private tokenService: AuthTokenService
  ) {}

  intercept(req: HttpRequest<any>, next: HttpHandler) {
    return next.handle(req).pipe(
      catchError((err) => {
        const is401 = err.status === 401;
        const isRefreshCall = req.url.includes('/api/auth/refresh');

        if (!is401 || isRefreshCall) {
          return throwError(() => err);
        }

        if (!this.refreshInProgress) {
          this.refreshInProgress = true;
          this.token$.next(null);

          return this.auth.refreshToken().pipe(
            switchMap(({ accessToken }) => {
              this.auth.setAccessToken(accessToken);
              this.refreshInProgress = false;
              this.token$.next(accessToken);

              // Rejouer la requête initiale
              return next.handle(this.withToken(req, accessToken));
            }),
            catchError((refreshErr) => {
              this.refreshInProgress = false;
              // Optionnel : logout / redirect login
              return throwError(() => refreshErr);
            })
          );
        }

        // Un refresh est déjà en cours => attendre le nouveau token
        return this.token$.pipe(
          filter((t): t is string => t !== null),
          take(1),
          switchMap((token) => next.handle(this.withToken(req, token)))
        );
      })
    );
  }

  private withToken(req: HttpRequest<any>, token: string) {
    return req.clone({
      setHeaders: {
        Authorization: `Bearer ${token}`
      }
    });
  }
}
```

### 10.3 Points d’attention

- Éviter les boucles infinies : si refresh échoue, décider d’un **logout**.
- Concurrence : le mécanisme `BehaviorSubject` évite de multiplier les refresh.
- Séparer l’interceptor d’ajout de token et celui de refresh peut améliorer la lisibilité.

---

## 11. Cas d’usage 6 — Transformation des données

### 11.1 Normalisation d’un format « enveloppé »

Exemple : API renvoie `{ data: ..., meta: ... }`.

```ts
import { map } from 'rxjs/operators';

@Injectable()
export class UnwrapDataInterceptor implements HttpInterceptor {
  intercept(req: HttpRequest<any>, next: HttpHandler) {
    return next.handle(req).pipe(
      map((event) => {
        // Ne transformer que les HttpResponse
        if (event.type !== 4) return event; // HttpEventType.Response = 4
        const body: any = event.body;
        if (body && typeof body === 'object' && 'data' in body) {
          return event.clone({ body: body.data });
        }
        return event;
      })
    );
  }
}
```

### 11.2 Conversion de dates ISO → Date (avec prudence)

Transformer automatiquement les dates peut être pratique mais risqué (performance, ambiguïtés). Souvent préférable au niveau des **DTO/adapters**.

---

## 12. Cas d’usage 7 — Retry avec backoff

Attention : ne pas retry sur toutes les erreurs (ex. 400). Prioriser :

- erreurs réseau (status 0)
- 502/503/504

```ts
import { timer, mergeMap, retryWhen } from 'rxjs';

@Injectable()
export class RetryInterceptor implements HttpInterceptor {
  intercept(req: HttpRequest<any>, next: HttpHandler) {
    return next.handle(req).pipe(
      retryWhen((errors) =>
        errors.pipe(
          mergeMap((err, i) => {
            const attempt = i + 1;
            const max = 3;
            const shouldRetry = err.status === 0 || [502, 503, 504].includes(err.status);

            if (!shouldRetry || attempt > max) {
              throw err;
            }

            const backoffMs = attempt * 300;
            return timer(backoffMs);
          })
        )
      )
    );
  }
}
```

---

## 13. Cas d’usage 8 — Cache GET simple (in-memory)

> À utiliser avec discernement : invalidation, mémoire, cohérence.

### 13.1 Stratégie

- Cacher seulement les `GET`.
- Clé = URL + query params.
- Invalidation via TTL.

```ts
import { HttpResponse } from '@angular/common/http';
import { of, tap } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class SimpleHttpCache {
  private cache = new Map<string, { expires: number; response: HttpResponse<any> }>();

  get(key: string): HttpResponse<any> | null {
    const entry = this.cache.get(key);
    if (!entry) return null;
    if (Date.now() > entry.expires) {
      this.cache.delete(key);
      return null;
    }
    return entry.response;
  }

  set(key: string, response: HttpResponse<any>, ttlMs: number) {
    this.cache.set(key, { expires: Date.now() + ttlMs, response });
  }
}

@Injectable()
export class CacheInterceptor implements HttpInterceptor {
  constructor(private cache: SimpleHttpCache) {}

  intercept(req: HttpRequest<any>, next: HttpHandler) {
    if (req.method !== 'GET') {
      return next.handle(req);
    }

    const key = req.urlWithParams;
    const cached = this.cache.get(key);
    if (cached) {
      return of(cached.clone());
    }

    return next.handle(req).pipe(
      tap((event) => {
        if (event instanceof HttpResponse) {
          this.cache.set(key, event, 10_000); // 10s
        }
      })
    );
  }
}
```

---

## 14. Configuration : enregistrer plusieurs interceptors

### 14.1 Avec `HTTP_INTERCEPTORS`

```ts
providers: [
  { provide: HTTP_INTERCEPTORS, useClass: CorrelationIdInterceptor, multi: true },
  { provide: HTTP_INTERCEPTORS, useClass: AuthInterceptor, multi: true },
  { provide: HTTP_INTERCEPTORS, useClass: RefreshTokenInterceptor, multi: true },
  { provide: HTTP_INTERCEPTORS, useClass: LoggingInterceptor, multi: true },
  { provide: HTTP_INTERCEPTORS, useClass: ErrorMappingInterceptor, multi: true },
]
```

### 14.2 Avec `withInterceptors`

```ts
provideHttpClient(
  withInterceptors([
    correlationIdInterceptor,
    authInterceptor,
    refreshTokenInterceptor,
    loggingInterceptor,
    errorMappingInterceptor,
  ])
)
```

> L’ordre compte : testez et documentez-le.

---

## 15. Tests d’un interceptor

### 15.1 Outils

- `HttpClientTestingModule` (approche module)
- `provideHttpClientTesting()` (approche moderne)
- `HttpTestingController`

### 15.2 Exemple : test de l’ajout d’Authorization

```ts
import { TestBed } from '@angular/core/testing';
import { HttpClient } from '@angular/common/http';
import { HttpClientTestingModule, HttpTestingController } from '@angular/common/http/testing';
import { HTTP_INTERCEPTORS } from '@angular/common/http';

describe('AuthInterceptor', () => {
  let http: HttpClient;
  let ctrl: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [HttpClientTestingModule],
      providers: [
        {
          provide: HTTP_INTERCEPTORS,
          useClass: AuthInterceptor,
          multi: true,
        },
        {
          provide: AuthTokenService,
          useValue: { getAccessToken: () => 'abc' },
        },
      ],
    });

    http = TestBed.inject(HttpClient);
    ctrl = TestBed.inject(HttpTestingController);
  });

  afterEach(() => ctrl.verify());

  it('ajoute le header Authorization pour /api', () => {
    http.get('/api/users').subscribe();

    const req = ctrl.expectOne('/api/users');
    expect(req.request.headers.get('Authorization')).toBe('Bearer abc');

    req.flush([]);
  });
});
```

---

## 16. Pièges fréquents et bonnes pratiques

### 16.1 Boucles infinies

- Un interceptor qui déclenche un appel HTTP peut se ré-intercepter.
- Exclure explicitement les URLs de login/refresh.

### 16.2 Trop de logique UI

- Éviter d’afficher des popups/toasts directement dans l’interceptor.
- Préférer un mécanisme d’événements (store/subject) ou une couche dédiée.

### 16.3 Duplications et ordre

- Plusieurs interceptors qui modifient les headers peuvent se contredire.
- Documenter l’ordre et le rôle de chacun.

### 16.4 Sécurité

- Ne jamais loguer des tokens.
- Ne pas attacher de token à un domaine tiers.

### 16.5 Performance

- Les transformations lourdes sur chaque réponse coûtent cher.
- Le cache peut exploser la mémoire si mal géré.

---

## 17. Atelier pratique (guidé)

### Objectif

Mettre en place une chaîne d’interceptors typique entreprise :

1. `CorrelationIdInterceptor`
2. `AuthInterceptor` (Bearer)
3. `RefreshTokenInterceptor` (401)
4. `ErrorMappingInterceptor`
5. `LoggingInterceptor`

### Étapes

1. Créer un projet Angular (ou utiliser un existant) et une API mock.
2. Ajouter un endpoint protégé qui renvoie 401 si token expiré.
3. Implémenter l’ajout du token.
4. Implémenter le refresh + relecture de la requête.
5. Ajouter logs et mapping d’erreurs.

### Critères de validation

- Toute requête `/api/*` inclut un `Authorization: Bearer ...`.
- Sur 401, le front fait un seul appel refresh, puis rejoue les requêtes en attente.
- Les erreurs remontent sous un format homogène.
- Logs affichent la durée et le status (sans données sensibles).

---

## 18. Synthèse

- Les **interceptors** permettent un traitement **transversal** : token, logs, erreurs, refresh, transformations.
- Ils sont fondamentaux pour standardiser et industrialiser un front Angular en contexte entreprise.
- Les points clés : **immutabilité**, **ordre**, **exclusions**, **concurrence** (refresh), **tests**.

---

## 19. Annexes

### 19.1 Check-list « production-ready »

- [ ] Exclusion endpoints `auth/*` (login/refresh) selon votre design
- [ ] Correlation ID généré et propagé
- [ ] Redaction/masquage des données sensibles dans les logs
- [ ] Mapping d’erreurs cohérent (status 0, 401, 403, 5xx)
- [ ] Gestion du refresh token avec concurrence
- [ ] Tests unitaires (headers, refresh, retry)
- [ ] Documentation de l’ordre des interceptors

### 19.2 Exemple d’ordre recommandé (indicatif)

1. Correlation ID
2. Auth (attach token)
3. Retry (selon cas)
4. Refresh on 401 (ou avant mapping)
5. Error mapping
6. Logging (ou logging en tout début selon stratégie)

> Ajustez selon vos contraintes (observabilité, sécurité, backend).
