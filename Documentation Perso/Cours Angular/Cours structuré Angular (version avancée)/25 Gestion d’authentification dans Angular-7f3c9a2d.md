# Formation Angular — Gestion d’authentification dans Angular

- **Niveau** : Intermédiaire → Avancé
- **Durée suggérée** : 1 à 2 jours (adaptable)
- **Pré-requis** : Angular (components/services/routing), RxJS (Observables, operators), TypeScript, notions HTTP/REST
- **Objectifs pédagogiques** :
  - Mettre en place un **flux d’authentification complet** (login/logout)
  - Gérer **stockage et cycle de vie** des jetons (**access/refresh**)
  - Implémenter une **session fluide** avec **refresh automatique**
  - Sécuriser la navigation via **guards** (auth, rôles)
  - Centraliser les appels API via **interceptors HTTP** (JWT, erreurs, retry)
  - Mettre en place la **séparation des rôles** (RBAC/claims) et la protection côté UI
  - Éviter les failles UX/sécurité courantes (boucles de refresh, stockage risqué, fuite de tokens)

---

## Plan de la formation

1. **Rappels et modèle de sécurité**
   - Authentification vs autorisation
   - JWT et sessions : access token, refresh token
   - Menaces fréquentes : XSS, CSRF, token leak, fixation de session
2. **Architecture d’un module d’auth Angular**
   - AuthService, TokenService, SessionService
   - Stockage: mémoire vs localStorage vs cookies HttpOnly
   - Événements applicatifs (state, signals, BehaviorSubject)
3. **Implémentation du login/logout**
   - Formulaire réactif, validation, UX
   - Gestion d’erreurs et messages
   - Redirection post-login (returnUrl)
4. **Gestion avancée des jetons (token storage)**
   - Modèle access/refresh
   - Expiration, horodatage, dérivation d’état
   - Synchronisation multi-onglets
5. **Intercepteur JWT**
   - Ajout automatique du header Authorization
   - Filtrage des endpoints (exclusions)
6. **Refresh automatique et gestion des 401**
   - Stratégies : proactive vs reactive
   - Queue des requêtes pendant refresh
   - Éviter les boucles et les courses critiques
7. **Guards et routing sécurisé**
   - canActivate / canMatch / canActivateChild
   - Protection des routes lazy
   - Gestion du returnUrl et pages publiques
8. **Rôles, permissions et séparation des accès (RBAC)**
   - Claims dans le token vs endpoint “me”
   - Directive/pipe de permission
   - Contrôle UI vs contrôle serveur
9. **Qualité, tests, observabilité**
   - Tests unitaires (services/interceptors/guards)
   - Tests d’intégration (HttpTestingController)
   - Logging, métriques, traçabilité
10. **Check-list sécurité et bonnes pratiques**
   - Recommandations concrètes
   - Anti-patterns

---

## 1) Rappels et modèle de sécurité

### 1.1 Authentification vs autorisation
- **Authentification** : prouver l’identité (login)
- **Autorisation** : vérifier les droits (rôles/permissions)

Dans une SPA Angular, on **authentifie côté front** (UX, routage) mais **l’autorité finale** reste le **backend**.

### 1.2 JWT : access token et refresh token
- **Access token** : courte durée (5–15 min), envoyé sur chaque appel API
- **Refresh token** : durée plus longue (jours/semaines), sert à régénérer un access token

> Recommandation : stocker le refresh token en **cookie HttpOnly/SameSite** si possible (réduit l’impact d’un XSS). Si ce n’est pas possible, redoubler de précautions (CSP, sanitization, etc.).

### 1.3 Menaces fréquentes
- **XSS** : vol de tokens si stockés en localStorage
- **CSRF** : risque si cookies utilisables cross-site (mitigé par SameSite + token CSRF)
- **Fuite dans logs** : ne pas logger les tokens
- **Boucles de refresh** : interceptor mal conçu
- **Mauvaise invalidation** : logout incomplet (côté serveur)

---

## 2) Architecture d’un module d’auth Angular

### 2.1 Découpage recommandé
- `AuthApiService` : appels HTTP vers endpoints auth (login, refresh, logout, me)
- `TokenService` : lecture/écriture des tokens + helpers (expiration)
- `SessionService` : état courant (user, roles, auth state) + synchronisation
- `AuthService` : façade (login/logout/isAuthenticated)
- `JwtInterceptor` : ajoute access token ; gère 401 + refresh
- `AuthGuard` / `RoleGuard` : sécurisation routing

### 2.2 Exemple d’arborescence
```txt
src/app/auth/
  auth.routes.ts
  auth.module.ts (si approche NgModule)
  services/
    auth-api.service.ts
    auth.service.ts
    token.service.ts
    session.service.ts
  guards/
    auth.guard.ts
    role.guard.ts
  interceptors/
    jwt.interceptor.ts
  models/
    auth.models.ts
```

### 2.3 Modèle de données minimal
```ts
export interface LoginRequest {
  email: string;
  password: string;
}

export interface TokenPair {
  accessToken: string;
  refreshToken?: string; // Si non géré en cookie HttpOnly
}

export interface UserProfile {
  id: string;
  email: string;
  displayName: string;
  roles: string[]; // ex: ['admin', 'trainer']
}
```

---

## 3) Implémentation du login/logout

### 3.1 Endpoints backend attendus (exemple)
- `POST /auth/login` → `{ accessToken, refreshToken? }`
- `POST /auth/refresh` → `{ accessToken }` (et éventuellement rotation refresh)
- `POST /auth/logout` → invalide refresh token côté serveur
- `GET /auth/me` → profil utilisateur + rôles

### 3.2 AuthApiService
```ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';

import { LoginRequest, TokenPair, UserProfile } from '../models/auth.models';

@Injectable({ providedIn: 'root' })
export class AuthApiService {
  private readonly baseUrl = '/api';

  constructor(private http: HttpClient) {}

  login(payload: LoginRequest): Observable<TokenPair> {
    return this.http.post<TokenPair>(`${this.baseUrl}/auth/login`, payload);
  }

  refresh(): Observable<{ accessToken: string }> {
    return this.http.post<{ accessToken: string }>(`${this.baseUrl}/auth/refresh`, {});
  }

  logout(): Observable<void> {
    return this.http.post<void>(`${this.baseUrl}/auth/logout`, {});
  }

  me(): Observable<UserProfile> {
    return this.http.get<UserProfile>(`${this.baseUrl}/auth/me`);
  }
}
```

### 3.3 TokenService (stockage)
#### Choix de stockage
- **Meilleure option** :
  - access token en **mémoire** (volatile)
  - refresh token en **cookie HttpOnly** (backend)
- **Option courante** (moins sûre) : access + refresh en `localStorage` / `sessionStorage`

> Dans cette formation, on présente une implémentation réaliste avec fallback, en gardant le principe : **minimiser ce qui est stocké**.

```ts
import { Injectable } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class TokenService {
  private accessToken: string | null = null;

  private readonly ACCESS_KEY = 'access_token';

  setAccessToken(token: string | null, persist = false) {
    this.accessToken = token;
    if (persist) {
      if (token) localStorage.setItem(this.ACCESS_KEY, token);
      else localStorage.removeItem(this.ACCESS_KEY);
    }
  }

  loadFromStorage() {
    const token = localStorage.getItem(this.ACCESS_KEY);
    this.accessToken = token;
  }

  getAccessToken(): string | null {
    return this.accessToken;
  }

  clear() {
    this.accessToken = null;
    localStorage.removeItem(this.ACCESS_KEY);
  }

  /**
   * Décodage simple du payload JWT pour lire exp.
   * (Alternative: lib jwt-decode)
   */
  getJwtPayload(token: string): any | null {
    try {
      const payload = token.split('.')[1];
      const json = atob(payload.replace(/-/g, '+').replace(/_/g, '/'));
      return JSON.parse(json);
    } catch {
      return null;
    }
  }

  isExpired(token: string, skewSeconds = 30): boolean {
    const payload = this.getJwtPayload(token);
    if (!payload?.exp) return true;
    const now = Math.floor(Date.now() / 1000);
    return payload.exp <= (now + skewSeconds);
  }
}
```

### 3.4 SessionService (état utilisateur)
Deux approches modernes :
- RxJS via `BehaviorSubject`
- Angular Signals (si Angular v16+)

Ici, version RxJS simple et robuste :

```ts
import { Injectable } from '@angular/core';
import { BehaviorSubject, Observable } from 'rxjs';
import { UserProfile } from '../models/auth.models';

export interface AuthState {
  isAuthenticated: boolean;
  user: UserProfile | null;
}

@Injectable({ providedIn: 'root' })
export class SessionService {
  private readonly _state$ = new BehaviorSubject<AuthState>({
    isAuthenticated: false,
    user: null,
  });

  state$: Observable<AuthState> = this._state$.asObservable();

  get snapshot(): AuthState {
    return this._state$.value;
  }

  setAuthenticated(user: UserProfile | null) {
    this._state$.next({
      isAuthenticated: !!user,
      user,
    });
  }

  clear() {
    this._state$.next({ isAuthenticated: false, user: null });
  }
}
```

### 3.5 AuthService (façade)
```ts
import { Injectable } from '@angular/core';
import { finalize, map, switchMap, tap } from 'rxjs/operators';
import { Observable, of } from 'rxjs';

import { AuthApiService } from './auth-api.service';
import { TokenService } from './token.service';
import { SessionService } from './session.service';
import { LoginRequest } from '../models/auth.models';

@Injectable({ providedIn: 'root' })
export class AuthService {
  private loading = false;

  constructor(
    private api: AuthApiService,
    private tokens: TokenService,
    private session: SessionService
  ) {}

  initFromStorage(): Observable<boolean> {
    this.tokens.loadFromStorage();
    const token = this.tokens.getAccessToken();
    if (!token || this.tokens.isExpired(token)) {
      this.tokens.clear();
      this.session.clear();
      return of(false);
    }
    return this.api.me().pipe(
      tap(user => this.session.setAuthenticated(user)),
      map(() => true)
    );
  }

  login(payload: LoginRequest, persistToken = false): Observable<void> {
    this.loading = true;
    return this.api.login(payload).pipe(
      tap(res => {
        this.tokens.setAccessToken(res.accessToken, persistToken);
      }),
      switchMap(() => this.api.me()),
      tap(user => this.session.setAuthenticated(user)),
      map(() => void 0),
      finalize(() => (this.loading = false))
    );
  }

  logout(): Observable<void> {
    return this.api.logout().pipe(
      finalize(() => {
        this.tokens.clear();
        this.session.clear();
      })
    );
  }

  isAuthenticated(): boolean {
    const token = this.tokens.getAccessToken();
    return !!token && !this.tokens.isExpired(token) && this.session.snapshot.isAuthenticated;
  }
}
```

### 3.6 Composant de login (Reactive Forms)
```ts
import { Component } from '@angular/core';
import { FormBuilder, Validators } from '@angular/forms';
import { ActivatedRoute, Router } from '@angular/router';

import { AuthService } from '../auth/services/auth.service';

@Component({
  selector: 'app-login',
  template: `
  <h1>Connexion</h1>

  <form [formGroup]="form" (ngSubmit)="submit()">
    <label>
      Email
      <input formControlName="email" type="email" />
    </label>

    <label>
      Mot de passe
      <input formControlName="password" type="password" />
    </label>

    <label>
      <input type="checkbox" formControlName="remember" />
      Se souvenir de moi
    </label>

    <button type="submit" [disabled]="form.invalid || submitting">Se connecter</button>

    <p class="error" *ngIf="error">{{ error }}</p>
  </form>
  `,
})
export class LoginComponent {
  submitting = false;
  error: string | null = null;

  form = this.fb.group({
    email: ['', [Validators.required, Validators.email]],
    password: ['', [Validators.required, Validators.minLength(6)]],
    remember: [false],
  });

  constructor(
    private fb: FormBuilder,
    private auth: AuthService,
    private router: Router,
    private route: ActivatedRoute
  ) {}

  submit() {
    if (this.form.invalid) return;
    this.submitting = true;
    this.error = null;

    const { email, password, remember } = this.form.value;
    const returnUrl = this.route.snapshot.queryParamMap.get('returnUrl') || '/';

    this.auth.login({ email: email!, password: password! }, !!remember).subscribe({
      next: () => this.router.navigateByUrl(returnUrl),
      error: (err) => {
        this.error = err?.error?.message || 'Connexion impossible';
        this.submitting = false;
      },
      complete: () => (this.submitting = false),
    });
  }
}
```

---

## 4) Gestion avancée des jetons

### 4.1 Expiration et “skew”
Toujours considérer un **décalage** (clock skew) : on traite le token comme expiré **30–60s avant** sa fin.

### 4.2 Synchronisation multi-onglets
Si token stocké en `localStorage`, un onglet peut se déconnecter et les autres doivent suivre.

```ts
// À ajouter dans SessionService ou AuthService (init)
window.addEventListener('storage', (e) => {
  if (e.key === 'access_token') {
    // Recharger l’état : token ajouté/supprimé dans un autre onglet
    // ex: tokens.loadFromStorage() + session refresh
  }
});
```

### 4.3 Rotation des refresh tokens
Si le backend effectue une **rotation** (refresh token change à chaque refresh), le front doit gérer ce retour (souvent porté par cookie HttpOnly côté serveur).

---

## 5) Intercepteur JWT (Authorization header)

### 5.1 Principe
- Intercepter toute requête HTTP
- Ajouter `Authorization: Bearer <accessToken>`
- Ne pas ajouter sur endpoints publics (login/refresh)

```ts
import { Injectable } from '@angular/core';
import {
  HttpEvent,
  HttpHandler,
  HttpInterceptor,
  HttpRequest,
} from '@angular/common/http';
import { Observable } from 'rxjs';

import { TokenService } from '../services/token.service';

@Injectable()
export class JwtInterceptor implements HttpInterceptor {
  private readonly excluded = ['/api/auth/login', '/api/auth/refresh'];

  constructor(private tokens: TokenService) {}

  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    if (this.excluded.some((u) => req.url.includes(u))) {
      return next.handle(req);
    }

    const token = this.tokens.getAccessToken();
    if (!token) return next.handle(req);

    const authReq = req.clone({
      setHeaders: {
        Authorization: `Bearer ${token}`,
      },
    });

    return next.handle(authReq);
  }
}
```

### 5.2 Configuration (Standalone ou providers)
```ts
import { HTTP_INTERCEPTORS } from '@angular/common/http';

providers: [
  { provide: HTTP_INTERCEPTORS, useClass: JwtInterceptor, multi: true },
]
```

---

## 6) Refresh automatique et gestion des 401

### 6.1 Deux stratégies
1. **Proactive** : refresh avant expiration (timer)
2. **Reactive** : refresh au premier **401**

En pratique : souvent **reactive** (simple) + option proactive selon UX.

### 6.2 Problème à résoudre : requêtes concurrentes
Si 5 requêtes reçoivent 401 simultanément, on ne veut **qu’un seul refresh**, puis rejouer les requêtes.

### 6.3 Interceptor avec refresh + queue
> Cette section apporte la partie “avancée”.

```ts
import { Injectable } from '@angular/core';
import {
  HttpErrorResponse,
  HttpEvent,
  HttpHandler,
  HttpInterceptor,
  HttpRequest,
} from '@angular/common/http';
import { Observable, BehaviorSubject, throwError } from 'rxjs';
import { catchError, filter, switchMap, take } from 'rxjs/operators';

import { AuthApiService } from '../services/auth-api.service';
import { TokenService } from '../services/token.service';
import { SessionService } from '../services/session.service';

@Injectable()
export class AuthRefreshInterceptor implements HttpInterceptor {
  private refreshing = false;
  private refreshSubject = new BehaviorSubject<string | null>(null);

  private readonly excluded = ['/api/auth/login', '/api/auth/refresh'];

  constructor(
    private api: AuthApiService,
    private tokens: TokenService,
    private session: SessionService
  ) {}

  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    if (this.excluded.some((u) => req.url.includes(u))) {
      return next.handle(req);
    }

    return next.handle(this.withAuth(req)).pipe(
      catchError((err) => {
        if (err instanceof HttpErrorResponse && err.status === 401) {
          return this.handle401(req, next);
        }
        return throwError(() => err);
      })
    );
  }

  private withAuth(req: HttpRequest<any>): HttpRequest<any> {
    const token = this.tokens.getAccessToken();
    if (!token) return req;
    return req.clone({ setHeaders: { Authorization: `Bearer ${token}` } });
  }

  private handle401(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    // Si aucune session connue, on ne tente pas refresh indéfiniment
    if (!this.tokens.getAccessToken()) {
      this.session.clear();
      return throwError(() => new Error('Unauthenticated'));
    }

    if (!this.refreshing) {
      this.refreshing = true;
      this.refreshSubject.next(null);

      return this.api.refresh().pipe(
        switchMap(({ accessToken }) => {
          this.tokens.setAccessToken(accessToken, true);
          this.refreshing = false;
          this.refreshSubject.next(accessToken);
          return next.handle(this.withAuth(req));
        }),
        catchError((refreshErr) => {
          // Refresh impossible -> déconnexion
          this.refreshing = false;
          this.tokens.clear();
          this.session.clear();
          return throwError(() => refreshErr);
        })
      );
    }

    // Si refresh en cours, on attend qu’il se termine
    return this.refreshSubject.pipe(
      filter((t): t is string => t !== null),
      take(1),
      switchMap(() => next.handle(this.withAuth(req)))
    );
  }
}
```

#### Points d’attention
- Exclure `/auth/refresh` (sinon boucle)
- Au refresh fail → **logout** local et redirection
- Mettre `persist=true/false` selon stratégie de stockage

---

## 7) Guards et routing sécurisé

### 7.1 AuthGuard (canActivate / canMatch)
Avec Angular moderne, privilégier `canMatch` pour lazy routes.

```ts
import { inject } from '@angular/core';
import { CanMatchFn, Router, UrlTree } from '@angular/router';

import { AuthService } from '../services/auth.service';

export const authGuard: CanMatchFn = (route, segments): boolean | UrlTree => {
  const auth = inject(AuthService);
  const router = inject(Router);

  if (auth.isAuthenticated()) return true;

  const attemptedUrl = '/' + segments.map(s => s.path).join('/');
  return router.createUrlTree(['/login'], {
    queryParams: { returnUrl: attemptedUrl },
  });
};
```

### 7.2 RoleGuard
Approche simple : déclarer dans `data` les rôles requis.

```ts
import { inject } from '@angular/core';
import { CanActivateFn, Router } from '@angular/router';
import { SessionService } from '../services/session.service';

export const roleGuard: CanActivateFn = (route) => {
  const session = inject(SessionService);
  const router = inject(Router);

  const required: string[] = route.data['roles'] || [];
  const userRoles = session.snapshot.user?.roles ?? [];

  const ok = required.length === 0 || required.some(r => userRoles.includes(r));
  return ok ? true : router.createUrlTree(['/forbidden']);
};
```

### 7.3 Exemple de routes
```ts
export const routes = [
  { path: 'login', loadComponent: () => import('./login/login.component').then(m => m.LoginComponent) },
  {
    path: 'admin',
    canMatch: [authGuard],
    canActivate: [roleGuard],
    data: { roles: ['admin'] },
    loadChildren: () => import('./admin/admin.routes').then(r => r.ADMIN_ROUTES),
  },
];
```

---

## 8) Rôles, permissions et séparation des accès (RBAC)

### 8.1 Où obtenir les rôles ?
- **Dans le token** (claims `roles`) : rapide, mais attention à la synchronisation si les droits changent
- **Via `/me`** : plus fiable, permet profil complet

Pratique recommandée :
- Token = identité + expiration
- `/me` = profil et droits actuels

### 8.2 Directive de permission (UI)
> But : masquer/afficher des blocs, mais **sans remplacer** la sécurité serveur.

```ts
import { Directive, Input, TemplateRef, ViewContainerRef } from '@angular/core';
import { SessionService } from '../services/session.service';

@Directive({ selector: '[appHasRole]' })
export class HasRoleDirective {
  @Input('appHasRole') role!: string;

  constructor(
    private tpl: TemplateRef<any>,
    private vcr: ViewContainerRef,
    private session: SessionService
  ) {
    this.session.state$.subscribe(s => {
      this.vcr.clear();
      const roles = s.user?.roles ?? [];
      if (roles.includes(this.role)) {
        this.vcr.createEmbeddedView(this.tpl);
      }
    });
  }
}
```

Utilisation :
```html
<button *appHasRole="'admin'">Supprimer</button>
```

### 8.3 Matrice permissions (option avancée)
Pour des applications complexes, préférer des **permissions** (ex: `invoice:read`, `invoice:write`) plutôt que des rôles.

---

## 9) Qualité, tests, observabilité

### 9.1 Tests unitaires (interceptor)
- Attendre qu’un 401 déclenche un refresh
- Vérifier que 2 requêtes concurrentes n’appellent qu’un refresh

Pistes avec `HttpClientTestingModule` + `HttpTestingController`.

### 9.2 Tests guard
- Utiliser `RouterTestingModule`
- Vérifier retour `UrlTree` vers login en non-auth

### 9.3 Observabilité
- Journaliser : `login success/fail`, `refresh start/success/fail`
- Ne jamais logger un token

---

## 10) Check-list sécurité et bonnes pratiques

### 10.1 Bonnes pratiques
- Access token **court** + refresh token sécurisé
- **Exclure** login/refresh des interceptors JWT/refresh
- Implémenter **queue** lors du refresh
- Nettoyer l’état (tokens + session) au logout
- Minimiser la persistance : “remember me” explicite
- Protéger l’app d’un XSS (sanitization, CSP, dépendances)

### 10.2 Anti-patterns à éviter
- Stocker refresh token dans localStorage sans protections
- Refresh proactif trop fréquent (spam réseau)
- Boucle : refresh → 401 → refresh
- Cacher des routes UI sans protection back

---

## TP (Travaux pratiques) — Fil rouge

### TP1 — Mettre en place login + récupération profil
- Créer `AuthApiService`, `TokenService`, `SessionService`
- Login -> stock access token -> `/me` -> afficher user

### TP2 — Guards de navigation
- Implémenter `authGuard` + `returnUrl`
- Créer page `forbidden`

### TP3 — Interceptor JWT
- Ajouter header Authorization automatiquement
- Exclure login/refresh

### TP4 — Refresh automatique robuste
- Interceptor 401 -> refresh unique -> replay
- Cas d’erreur refresh -> logout

### TP5 — RBAC
- Ajouter `roleGuard`
- Ajouter directive `*appHasRole`

---

## Annexes

### A) Endpoints et statuts HTTP recommandés
- `401 Unauthorized` : token manquant/invalide/expiré
- `403 Forbidden` : authentifié mais droit insuffisant

### B) Stratégie cookie HttpOnly (résumé)
- Backend met `Set-Cookie: refreshToken=...; HttpOnly; Secure; SameSite=Strict`
- Front n’accède pas au contenu du refresh token
- `POST /auth/refresh` enverra automatiquement le cookie

### C) Exemple d’ergonomie
- Tant que `initFromStorage()` n’a pas fini, afficher un **splash/loading**
- Éviter un “flash” de contenu protégé

---

## Conclusion
Une implémentation avancée de l’authentification dans Angular repose sur :
- une **architecture claire** (services dédiés)
- des **interceptors** robustes (JWT + refresh)
- des **guards** adaptés au routing moderne
- une **gestion correcte des rôles/permissions**
- une attention constante à l’UX (returnUrl, refresh transparent) et à la sécurité (XSS/CSRF, stockage).
