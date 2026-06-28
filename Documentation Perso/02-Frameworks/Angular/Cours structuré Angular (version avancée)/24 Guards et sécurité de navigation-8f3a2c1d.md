# Formation Angular — Guards et sécurité de navigation

> **Objectif** : maîtriser les *Route Guards* Angular (CanActivate, CanDeactivate, CanMatch, CanActivateChild) pour **contrôler l’accès aux routes**, **sécuriser l’application**, **éviter les pertes de données** et **adapter le parcours utilisateur** selon les droits et le contexte.

---

## 1) Pré-requis et contexte

### Pré-requis
- Angular (Router) : notions de base sur le routage (routes, lazy-loading, paramètres)
- TypeScript : classes, interfaces, fonctions, RxJS (Observable)
- Notions d’authentification (JWT, sessions) appréciées

### Contexte et enjeux
Dans une application Angular, la navigation est un point critique :
- **Sécurité** : empêcher l’accès à des vues ou données à des utilisateurs non autorisés.
- **Expérience utilisateur** : rediriger vers une page de login, gérer des parcours conditionnels.
- **Qualité** : éviter la perte de données (formulaire non sauvegardé).
- **Performance** : empêcher de charger des modules inutiles via un filtrage avant le *lazy-load*.

> Les guards s’exécutent lors des transitions de navigation et peuvent **autoriser** ou **bloquer** la navigation, ou **rediriger**.

---

## 2) Vue d’ensemble des Guards Angular (router)

### 2.1 Les types principaux
- **CanActivate** : autorise/empêche l’accès à une route.
- **CanDeactivate** : autorise/empêche la sortie d’un composant (utile pour formulaires).
- **CanMatch** : décide si une route *match* une URL (très utile pour *lazy loading* et variantes de routes).
- **CanActivateChild** : sécurise les routes enfants d’une route.

### 2.2 Valeurs de retour possibles
Un guard peut retourner :
- `boolean` : `true` => navigation autorisée, `false` => navigation annulée.
- `UrlTree` : navigation redirigée vers une autre URL.
- `Observable<boolean | UrlTree>` ou `Promise<boolean | UrlTree>`.

> **Bon réflexe** sécurité : préférer renvoyer un `UrlTree` (redirection contrôlée) plutôt que `false` (navigation simplement annulée).

---

## 3) Architecture recommandée : AuthN, AuthZ et Guards

### 3.1 Distinguer Authentification et Autorisation
- **Authentification (AuthN)** : “Qui es-tu ?” (connecté ou non, token valide).
- **Autorisation (AuthZ)** : “As-tu le droit ?” (rôles, permissions, règles métier).

### 3.2 Services typiques
- `AuthService` : login/logout, token, état de session
- `PermissionsService` : rôles/permissions, règles
- `UserContextService` : contexte (organisation, feature flags, tenant)

### 3.3 Données de route (route data)
Angular Router permet d’associer des métadonnées aux routes :
```ts
{
  path: 'admin',
  component: AdminPage,
  data: { roles: ['ADMIN'], feature: 'backoffice' }
}
```
Ces données permettent aux guards de rester **génériques** et **réutilisables**.

---

## 4) CanActivate : contrôler l’accès à une route

### 4.1 Cas d’usage
- Route accessible uniquement si utilisateur connecté
- Route accessible selon rôle/permission
- Route accessible selon contexte (ex: organisation sélectionnée)

### 4.2 Exemple d’un guard fonctionnel (Angular moderne)
> Angular supporte les *functional guards* (recommandé pour des guards simples).

```ts
import { inject } from '@angular/core';
import { CanActivateFn, Router, ActivatedRouteSnapshot, RouterStateSnapshot } from '@angular/router';
import { AuthService } from './auth.service';

export const authGuard: CanActivateFn = (
  route: ActivatedRouteSnapshot,
  state: RouterStateSnapshot
) => {
  const auth = inject(AuthService);
  const router = inject(Router);

  if (auth.isAuthenticated()) {
    return true;
  }

  // On redirige vers /login en conservant l’URL de retour
  return router.createUrlTree(['/login'], {
    queryParams: { returnUrl: state.url }
  });
};
```

### 4.3 Variante asynchrone (ex: token refresh)
```ts
import { of } from 'rxjs';
import { map, catchError } from 'rxjs/operators';

export const authGuardAsync: CanActivateFn = (route, state) => {
  const auth = inject(AuthService);
  const router = inject(Router);

  return auth.ensureSession$().pipe(
    map(ok => ok ? true : router.createUrlTree(['/login'], { queryParams: { returnUrl: state.url } })),
    catchError(() => of(router.createUrlTree(['/login'], { queryParams: { returnUrl: state.url } })))
  );
};
```

### 4.4 Déclaration dans les routes
```ts
import { Routes } from '@angular/router';
import { authGuard } from './auth.guard';

export const routes: Routes = [
  {
    path: 'dashboard',
    canActivate: [authGuard],
    loadComponent: () => import('./dashboard/dashboard.page').then(m => m.DashboardPage)
  }
];
```

---

## 5) Autorisation (rôles/permissions) via CanActivate

### 5.1 Approche avec `data.roles`
```ts
import { inject } from '@angular/core';
import { CanActivateFn, Router } from '@angular/router';
import { AuthService } from './auth.service';

export const roleGuard: CanActivateFn = (route) => {
  const auth = inject(AuthService);
  const router = inject(Router);

  const requiredRoles: string[] = route.data['roles'] ?? [];
  if (requiredRoles.length === 0) return true;

  const userRoles = auth.userRoles();
  const allowed = requiredRoles.some(r => userRoles.includes(r));

  return allowed ? true : router.createUrlTree(['/forbidden']);
};
```

### 5.2 Exemple de routes
```ts
{
  path: 'admin',
  canActivate: [authGuard, roleGuard],
  data: { roles: ['ADMIN'] },
  loadComponent: () => import('./admin/admin.page').then(m => m.AdminPage)
},
{
  path: 'forbidden',
  loadComponent: () => import('./errors/forbidden.page').then(m => m.ForbiddenPage)
}
```

> **Conseil** : séparer `authGuard` (connecté) et `roleGuard` (droits) rend l’architecture plus claire et testable.

---

## 6) CanDeactivate : empêcher la perte de données

### 6.1 Cas d’usage
- Formulaire en édition : empêcher de quitter si modifications non sauvegardées.
- Wizard multi-étapes : confirmation avant sortie.

### 6.2 Modèle : composant “déactivable”
Créer un contrat (interface) que le composant implémente.

```ts
export interface CanComponentDeactivate {
  canDeactivate: () => boolean | Promise<boolean> | import('rxjs').Observable<boolean>;
}
```

### 6.3 Guard de désactivation
```ts
import { CanDeactivateFn } from '@angular/router';
import { CanComponentDeactivate } from './can-component-deactivate';

export const pendingChangesGuard: CanDeactivateFn<CanComponentDeactivate> = (component) => {
  return component.canDeactivate();
};
```

### 6.4 Exemple : composant avec formulaire
```ts
import { Component } from '@angular/core';
import { FormBuilder, Validators } from '@angular/forms';
import { CanComponentDeactivate } from '../guards/can-component-deactivate';

@Component({
  selector: 'app-profile-edit',
  template: `
    <h1>Profil</h1>
    <form [formGroup]="form">
      <label>Nom <input formControlName="name" /></label>
      <button type="button" (click)="save()" [disabled]="form.invalid">Sauvegarder</button>
    </form>
  `
})
export class ProfileEditPage implements CanComponentDeactivate {
  form = this.fb.group({
    name: ['', Validators.required]
  });

  private saved = false;

  constructor(private fb: FormBuilder) {}

  save() {
    // ... appel API
    this.saved = true;
    this.form.markAsPristine();
  }

  canDeactivate(): boolean {
    const hasUnsavedChanges = this.form.dirty && !this.saved;
    return hasUnsavedChanges ? confirm('Vous avez des modifications non sauvegardées. Quitter ?') : true;
  }
}
```

### 6.5 Route
```ts
{
  path: 'profile/edit',
  canActivate: [authGuard],
  canDeactivate: [pendingChangesGuard],
  loadComponent: () => import('./profile/profile-edit.page').then(m => m.ProfileEditPage)
}
```

> **Attention** : `CanDeactivate` ne protège pas contre la fermeture d’onglet par le navigateur. Pour cela, utiliser aussi `beforeunload` côté fenêtre (avec parcimonie).

---

## 7) CanMatch : filtrer le matching d’une route (et optimiser le lazy-load)

### 7.1 Pourquoi CanMatch ?
`CanMatch` décide si une route est **éligible** pour matcher une URL.
- Très utile avec **lazy-loading** : éviter de charger un module si l’utilisateur n’a pas accès.
- Permet de choisir une route alternative selon contexte/feature flags.

### 7.2 Exemple : accès à un module lazy-load selon permission
```ts
import { inject } from '@angular/core';
import { CanMatchFn, Router, Route, UrlSegment } from '@angular/router';
import { PermissionsService } from './permissions.service';

export const reportsCanMatch: CanMatchFn = (route: Route, segments: UrlSegment[]) => {
  const permissions = inject(PermissionsService);
  const router = inject(Router);

  const allowed = permissions.has('REPORTS_ACCESS');
  return allowed ? true : router.createUrlTree(['/forbidden']);
};
```

### 7.3 Route lazy-load
```ts
{
  path: 'reports',
  canMatch: [reportsCanMatch],
  loadChildren: () => import('./reports/reports.routes').then(m => m.REPORTS_ROUTES)
}
```

### 7.4 Cas avancé : feature flag / A-B routing
On peut faire varier le composant accessible :
- `/checkout` vers un nouveau flow si `featureFlag = true`
- sinon vers l’ancien flow

Ici, `CanMatch` peut sélectionner la route “A” et empêcher la route “B” de matcher.

---

## 8) CanActivateChild : sécuriser une arborescence de routes

### 8.1 Pourquoi ?
Quand une section a plusieurs routes enfants (admin, settings, etc.), `CanActivateChild` évite de répéter `canActivate` partout.

### 8.2 Exemple
```ts
import { inject } from '@angular/core';
import { CanActivateChildFn, Router } from '@angular/router';
import { AuthService } from './auth.service';

export const authChildGuard: CanActivateChildFn = (childRoute, state) => {
  const auth = inject(AuthService);
  const router = inject(Router);

  return auth.isAuthenticated()
    ? true
    : router.createUrlTree(['/login'], { queryParams: { returnUrl: state.url } });
};
```

### 8.3 Route parent
```ts
{
  path: 'settings',
  canActivateChild: [authChildGuard],
  children: [
    {
      path: 'account',
      loadComponent: () => import('./settings/account.page').then(m => m.AccountPage)
    },
    {
      path: 'security',
      loadComponent: () => import('./settings/security.page').then(m => m.SecurityPage)
    }
  ]
}
```

---

## 9) Chaînage, ordre d’exécution et comportements

### 9.1 Enchaîner plusieurs guards
Sur une route, on peut cumuler :
- Authentification
- Rôles
- Conditions métier

Angular évalue les guards ; si l’un retourne :
- `false` : navigation annulée
- `UrlTree` : redirection

### 9.2 Ordre (conceptuel)
Lors d’une navigation vers une route :
1. Matching de route (incluant `CanMatch`)
2. Exécution des guards d’activation (`CanActivate`, `CanActivateChild`)
3. Activation des composants
4. À la sortie d’une route : `CanDeactivate` du composant courant

> En pratique, `CanDeactivate` intervient au moment où Angular veut quitter la route actuelle.

### 9.3 Conseils pratiques
- **Toujours gérer les cas async** (token expiré, appel API en erreur).
- **Renvoi de `UrlTree`** pour guider l’utilisateur (login/forbidden).
- **Garder les guards idempotents** et sans effets de bord.

---

## 10) Stratégies de redirection et UX

### 10.1 Conserver l’URL de retour
Comme vu dans `authGuard`, conserver `returnUrl` :
- après login, redirection vers la page initialement demandée.

### 10.2 Page Forbidden vs Not Found
- **403 Forbidden** : utilisateur authentifié mais manque de droits.
- **404 Not Found** : route inexistante.

### 10.3 Exemple : after login
```ts
// pseudo-code
login().subscribe(() => {
  const returnUrl = this.route.snapshot.queryParamMap.get('returnUrl') ?? '/';
  this.router.navigateByUrl(returnUrl);
});
```

---

## 11) Sécurité : limites côté Front et bonnes pratiques

### 11.1 Ce que les guards ne font pas
Les guards ne remplacent pas la sécurité serveur.
- Un utilisateur malveillant peut appeler l’API directement.
- Donc **l’API doit vérifier** authentification et autorisation.

### 11.2 Ce que les guards apportent
- Réduction de surface UX (masquer des chemins non autorisés)
- Cohérence de navigation
- Optimisation (éviter de charger des modules inutiles)

### 11.3 Bonnes pratiques
- Centraliser la logique : `AuthService`, `PermissionService`
- Ne jamais stocker de secrets dans le frontend
- Utiliser `HttpInterceptor` pour ajouter token + gérer 401/403
- Coupler guards et *resolvers* (si besoin) avec prudence pour éviter latences

---

## 12) Atelier pratique (guidé)

### Objectif
Mettre en place une mini-application avec :
- `/login`
- `/dashboard` (auth requis)
- `/admin` (auth + rôle ADMIN)
- `/profile/edit` (CanDeactivate)
- `/reports` (lazy-load + CanMatch)

### Étapes
1. Créer `AuthService` avec `isAuthenticated()`, `userRoles()`
2. Implémenter `authGuard` + `roleGuard`
3. Implémenter `pendingChangesGuard` et un composant formulaire
4. Créer `reportsCanMatch` et lazy-load `reports.routes`
5. Ajouter pages `ForbiddenPage` et `LoginPage`

### Critères de validation
- Un utilisateur non connecté est redirigé vers `/login?returnUrl=...`
- Un utilisateur connecté sans rôle ADMIN arrive sur `/forbidden`
- Sortie de `/profile/edit` demande confirmation si formulaire dirty
- `/reports` ne charge pas le module si permission absente

---

## 13) Tests (recommandations)

### 13.1 Niveaux de tests
- **Unit tests** : tester la logique de guard (mocks services, router)
- **E2E** : vérifier les parcours (Cypress/Playwright)

### 13.2 Exemple de test unitaire (idée)
- Mock `AuthService.isAuthenticated()`
- Vérifier que le guard renvoie `true` ou un `UrlTree` selon cas

> Selon votre setup (Jasmine/Jest), adaptez l’injection/mocking.

---

## 14) Récapitulatif

- **CanActivate** : contrôle l’entrée sur une route (auth, droits).
- **CanDeactivate** : contrôle la sortie d’un composant (anti-perte de données).
- **CanMatch** : contrôle le matching (et le lazy-loading), très utile pour droits/feature flags.
- **CanActivateChild** : sécurise une arborescence de routes.

### Checklist de production
- [ ] Guards renvoient `UrlTree` pour redirections
- [ ] Gestion token expiré / erreurs async
- [ ] API sécurisée indépendamment du front
- [ ] Pages dédiées (login/forbidden)
- [ ] Couverture de tests sur scénarios critiques

---

## Annexes — Squelette minimal de services

### AuthService (exemple simplifié)
```ts
import { Injectable } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class AuthService {
  private _roles: string[] = ['USER'];

  isAuthenticated(): boolean {
    return !!localStorage.getItem('token');
  }

  userRoles(): string[] {
    return this._roles;
  }

  // Exemple : assure une session (refresh token, etc.)
  ensureSession$() {
    // Remplacer par vraie logique
    return import('rxjs').then(rx => rx.of(this.isAuthenticated()));
  }
}
```

### PermissionsService (exemple)
```ts
import { Injectable } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class PermissionsService {
  private permissions = new Set<string>(['REPORTS_ACCESS']);

  has(permission: string): boolean {
    return this.permissions.has(permission);
  }
}
```
