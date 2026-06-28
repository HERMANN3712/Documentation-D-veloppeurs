# Projet fil rouge Angular avancé (formation complète)

**Public visé** : développeurs Angular confirmés (niveau intermédiaire à avancé)

**Objectif général** : concevoir, développer, tester et livrer une application métier modulaire Angular en appliquant les bonnes pratiques modernes (standalone, lazy loading, RxJS, state, sécurité, performance, tests et CI).

**Projet fil rouge** : **BizBoard** – application de gestion de demandes (tickets) et de validation interne (workflows) avec authentification, rôles, dashboard, formulaires complexes, consommation API REST, store (local/global), composants réutilisables, gestion d’erreurs, i18n et tests automatisés.

---

## Table des matières

1. [Pré-requis & setup](#1-pré-requis--setup)
2. [Architecture & cadrage du fil rouge](#2-architecture--cadrage-du-fil-rouge)
3. [Initialisation du projet Angular (standalone) & qualité](#3-initialisation-du-projet-angular-standalone--qualité)
4. [Navigation, layout, lazy loading & guards](#4-navigation-layout-lazy-loading--guards)
5. [Authentification, JWT, rôles & sécurité](#5-authentification-jwt-rôles--sécurité)
6. [API REST, HttpClient, interception & erreurs](#6-api-rest-httpclient-interception--erreurs)
7. [State management : patterns, store global, store local](#7-state-management--patterns-store-global-store-local)
8. [Dashboard : charts, agrégations, performance](#8-dashboard--charts-agrégations-performance)
9. [Formulaires complexes : RxForms, CVA, validations, wizard](#9-formulaires-complexes--rxforms-cva-validations-wizard)
10. [Composants réutilisables & design system](#10-composants-réutilisables--design-system)
11. [Gestion d’erreurs, logs, observabilité](#11-gestion-derreurs-logs-observabilité)
12. [Tests automatisés : unit, integration, e2e](#12-tests-automatisés--unit-integration-e2e)
13. [Accessibilité, i18n, UX avancée](#13-accessibilité-i18n-ux-avancée)
14. [Optimisations : performance, SSR/Prerender, build](#14-optimisations--performance-ssrprerender-build)
15. [Packaging, CI/CD, versioning, livraison](#15-packaging-cicd-versioning-livraison)
16. [Annexes : conventions, checklists, exercices](#16-annexes--conventions-checklists-exercices)

---

## 1. Pré-requis & setup

### Objectifs
- Valider le socle technique Angular avancé.
- Mettre en place un environnement de dev et une base de projet robuste.

### Pré-requis
- Angular (routing, DI, RxJS, forms, modules/standalone)
- Typescript avancé (types, generics, utility types)
- REST / HTTP / JWT
- Git (branches, PR)

### Outils
- Node LTS, npm/pnpm
- Angular CLI
- VS Code (Angular Language Service)
- Chrome DevTools
- (Option) Docker pour lancer une API de démo

### Livrables
- Repo Git initialisé
- Checklist qualité de code et conventions

---

## 2. Architecture & cadrage du fil rouge

### Domaine fonctionnel (BizBoard)
**Entités principales** :
- **User** : id, email, name, roles (ADMIN, MANAGER, EMPLOYEE)
- **Ticket** : id, title, description, category, priority, status, assignee, requester, createdAt, updatedAt
- **Comment** : id, ticketId, authorId, message, createdAt
- **Dashboard** : indicateurs agrégés (tickets ouverts, délais moyens, SLA)

### Use cases (par rôle)
- **EMPLOYEE** : créer une demande, suivre l’avancement, commenter.
- **MANAGER** : assigner, valider/refuser, voir dashboard équipe.
- **ADMIN** : gérer utilisateurs, catégories, paramètres.

### Architecture cible Angular
- **Approche standalone** (components + routes)
- **Features en lazy loading** : auth, dashboard, tickets, admin
- **Core** : services singleton (api, auth, interceptors), environnement, utilitaires
- **Shared** : UI réutilisable (table, modal, toast), pipes, directives
- **State** :
  - store global (ex. NgRx ou Signals Store)
  - stores locaux (component store / signals) pour pages complexes

> Le choix du store est adaptable. Le plan propose une implémentation de référence avec **NgRx** (mature, outillage complet). Une alternative **Signals Store** est mentionnée.

### Critères d’acceptation (Definition of Done)
- Routing lazy-loaded + guards
- Auth JWT + refresh (option)
- Interceptors (auth + error)
- UI cohérente + composants partagés
- Formulaire wizard avec validations et champs dynamiques
- Store : cache + optimistic update
- Tests unitaires et e2e de scénarios clés

---

## 3. Initialisation du projet Angular (standalone) & qualité

### Objectifs
- Créer une base stable : lint, format, tests, structure.

### 3.1 Création du projet
```bash
ng new bizboard --standalone --routing --style=scss
cd bizboard
```

### 3.2 Structure recommandée
```
src/
  app/
    core/           # singletons, interceptors, guards
    shared/         # components/pipes/directives réutilisables
    features/
      auth/
      dashboard/
      tickets/
      admin/
    app.routes.ts
    app.config.ts
```

### 3.3 Code quality
- ESLint + Prettier
- Husky + lint-staged (option)
- Conventional commits

### 3.4 Environnements
`environment.ts` / `environment.prod.ts`
- apiBaseUrl
- feature flags (ex. enableMock)

### Exercices
- Ajouter une règle ESLint personnalisée (interdiction de `any`)
- Mettre une CI minimale (lint + test)

---

## 4. Navigation, layout, lazy loading & guards

### Objectifs
- Mettre un layout applicatif (shell), routes lazy, guards/auth.

### 4.1 Layout & shell
- `AppShellComponent` : header, sidenav, router-outlet
- `AuthLayoutComponent` : pages login/register

### 4.2 Routing standalone avec lazy loading
Exemple `app.routes.ts` :
```ts
import { Routes } from '@angular/router';
import { authGuard } from './core/guards/auth.guard';

export const routes: Routes = [
  {
    path: 'auth',
    loadChildren: () => import('./features/auth/auth.routes').then(m => m.AUTH_ROUTES),
  },
  {
    path: '',
    canActivate: [authGuard],
    loadComponent: () => import('./core/layout/app-shell.component').then(c => c.AppShellComponent),
    children: [
      {
        path: 'dashboard',
        loadChildren: () => import('./features/dashboard/dashboard.routes').then(m => m.DASHBOARD_ROUTES),
      },
      {
        path: 'tickets',
        loadChildren: () => import('./features/tickets/tickets.routes').then(m => m.TICKETS_ROUTES),
      },
      {
        path: 'admin',
        loadChildren: () => import('./features/admin/admin.routes').then(m => m.ADMIN_ROUTES),
      },
      { path: '', pathMatch: 'full', redirectTo: 'dashboard' },
    ],
  },
  { path: '**', redirectTo: '' },
];
```

### 4.3 Guards & resolvers
- `authGuard` : redirige vers `/auth/login` si non authentifié
- `roleGuard` : contrôle par rôle
- Resolver (option) : précharger des données (ex. catégories)

### Exercices
- Ajouter une route `tickets/:id` + guard vérifiant l’accès

---

## 5. Authentification, JWT, rôles & sécurité

### Objectifs
- Implémenter un flow JWT propre, stockage sécurisé, gestion session.

### 5.1 Modèle et service d’auth
- `AuthService` : login, logout, refresh (option)
- Stockage :
  - access token en mémoire (idéal)
  - refresh token en cookie httpOnly (si backend le permet)
  - sinon fallback : sessionStorage (risques XSS à expliquer)

### 5.2 Interceptor d’auth
- Ajoute `Authorization: Bearer <token>` aux requêtes
- Exclut les endpoints public (login)

### 5.3 Rôles
- Décodage JWT (claims)
- `hasRole(['ADMIN'])`

### 5.4 Bonnes pratiques
- Ne jamais mettre d’info sensible dans le localStorage
- CSRF : pertinent si cookies
- Content Security Policy (CSP)

### Exercices
- Ajouter un `roleGuard` et protéger `/admin`

---

## 6. API REST, HttpClient, interception & erreurs

### Objectifs
- Standardiser les appels API, les erreurs, la pagination et le cache.

### 6.1 Service API générique
Créer un wrapper `ApiClient` :
- baseUrl
- méthodes typées : get/post/put/patch/delete
- gestion des query params

### 6.2 Interceptor d’erreur
- Normaliser les erreurs backend → `AppError`
- Afficher des notifications (toast)
- Rediriger sur 401 → logout

### 6.3 Patterns RxJS
- `switchMap` pour dépendances de requêtes
- `shareReplay` pour cache local
- `catchError` + `throwError`

### 6.4 Pagination & tri (Tickets)
- endpoint : `GET /tickets?page=1&pageSize=20&sort=-createdAt`
- UI : table paginée

### Exercices
- Implémenter une stratégie de retry sur erreurs réseau (`retry` + backoff)

---

## 7. State management : patterns, store global, store local

### Objectifs
- Choisir et implémenter un store global, et des stores locaux.

### 7.1 Quand un store ?
- Global : session utilisateur, préférences, cache global
- Feature store : tickets, admin
- Local : formulaire wizard, filtres temporaires

### 7.2 Implémentation de référence : NgRx
- `actions` (loadTickets, loadTicketsSuccess, failure)
- `reducer` + `selectors`
- `effects` : orchestration API
- DevTools

### 7.3 Cas d’usage : Tickets
- Liste des tickets :
  - load initial
  - filtres (status, priority)
  - optimistic update (changer status)

### 7.4 Alternative : Signals Store
- store basé sur signals
- computed + effect
- plus simple pour projets sans gros besoin d’outillage

### Exercices
- Mettre en place un cache par filtre (clé : JSON.stringify(filters))

---

## 8. Dashboard : charts, agrégations, performance

### Objectifs
- Construire un dashboard modulaire et performant.

### 8.1 Widgets
- KPI cards : ouverte/fermée/en retard
- Chart par statut (bar/pie)
- Chart SLA (ligne)

### 8.2 Données
- endpoint agrégé : `GET /dashboard/metrics?range=30d`
- fallback : agrégation côté client (si nécessaire)

### 8.3 Performance
- `OnPush` / signals
- `trackBy` sur listes
- éviter recalculs (memo via selectors)

### Exercices
- Créer une grille responsive de widgets (CSS grid)

---

## 9. Formulaires complexes : RxForms, CVA, validations, wizard

### Objectifs
- Réaliser un formulaire métier avancé : dynamique, validé, maintenable.

### 9.1 Formulaire ticket (wizard)
Étapes :
1. **Contexte** : catégorie, priorité
2. **Détails** : titre, description, pièces jointes
3. **Assignation** (manager) : assignee, date cible
4. **Récap** + submit

### 9.2 Reactive Forms avancés
- `FormGroup` fortement typé
- `FormArray` (commentaires/contacts)
- validations sync/async (unicité titre, ex.)

Exemple validation async :
```ts
function uniqueTitleValidator(api: TicketsApi) {
  return (control: AbstractControl) => api.checkTitle(control.value).pipe(
    map(res => (res.isUnique ? null : { notUnique: true })),
    catchError(() => of(null))
  );
}
```

### 9.3 ControlValueAccessor (CVA)
Créer un composant réutilisable :
- `PrioritySelectComponent` (CVA)
- `UserAutocompleteComponent` (CVA)

### 9.4 UX de formulaire
- erreurs contextualisées
- désactivation submit
- autosave brouillon (store local)

### Exercices
- Ajouter un champ dynamique : si catégorie = “IT”, afficher “Asset tag” requis

---

## 10. Composants réutilisables & design system

### Objectifs
- Construire une base UI cohérente et réutilisable.

### 10.1 Shared UI
- `DataTableComponent` (inputs: columns, data, loading)
- `ConfirmDialogComponent`
- `ToastService` + `ToastContainerComponent`

### 10.2 Directives & pipes
- `hasRole` directive structurelle
- `dateAgo` pipe

### 10.3 Stratégies d’API de composants
- inputs immutables
- outputs explicites
- `ControlValueAccessor` pour champs

### Exercices
- Ajouter le pattern “headless component” pour une table

---

## 11. Gestion d’erreurs, logs, observabilité

### Objectifs
- Avoir une stratégie complète client (UX + debug + monitoring).

### 11.1 Global ErrorHandler
- Capturer erreurs non gérées
- Envoyer vers un service (Sentry, Elastic APM) (option)

### 11.2 Logging
- Un `LoggerService` (levels, environment)
- Masquer données sensibles

### 11.3 Erreurs utilisateur
- messages clairs
- codes d’erreurs mappés

### Exercices
- Ajouter une page “incident” affichant un code de trace

---

## 12. Tests automatisés : unit, integration, e2e

### Objectifs
- Mettre une pyramide de tests réaliste.

### 12.1 Tests unitaires (Jest ou Karma)
Cibles :
- reducers/selectors
- services (mock Http)
- pipes et composants UI

### 12.2 Tests d’intégration Angular
- tester un écran avec routing + store mock
- `HttpClientTestingModule` / `provideHttpClientTesting`

### 12.3 E2E (Cypress/Playwright)
Scénarios :
1. login
2. création ticket (wizard)
3. changement statut
4. contrôle accès admin

### 12.4 Stratégie de mock API
- MSW (Mock Service Worker) (option)
- environnements

### Exercices
- Écrire un test e2e sur la création d’un ticket

---

## 13. Accessibilité, i18n, UX avancée

### Objectifs
- Appliquer a11y et internationalisation.

### 13.1 Accessibilité
- navigation clavier
- aria-label
- contrastes
- focus management (modals)

### 13.2 i18n
- Angular i18n ou Transloco
- structure des clés

### 13.3 UX
- skeleton loaders
- empty states
- debounced search

---

## 14. Optimisations : performance, SSR/Prerender, build

### Objectifs
- Optimiser la charge et l’exécution.

### 14.1 Lazy loading & preloading
- stratégie `Quicklink` ou custom preloading

### 14.2 Bundle analysis
- `ng build --stats-json`
- analyser les dépendances lourdes

### 14.3 SSR/Prerender (option)
- Angular SSR pour SEO (moins pertinent pour app interne)
- prerender pour pages publiques

---

## 15. Packaging, CI/CD, versioning, livraison

### Objectifs
- Industrialiser : build, tests, déploiement, versioning.

### 15.1 CI pipeline
- lint
- tests unit
- e2e (option)
- build prod

### 15.2 Déploiement
- hébergement (Nginx, Firebase Hosting, Azure Static Web Apps)
- variables d’environnement (runtime config)

### 15.3 Versioning & changelog
- semver
- conventional-changelog

### Livrable final
- application BizBoard fonctionnelle
- rapport de tests
- documentation README (setup, scripts, architecture)

---

## 16. Annexes : conventions, checklists, exercices

### A. Conventions de code
- `feature/*` pour branches
- `core` pour singletons
- services suffix `*Service`

### B. Checklist sécurité
- éviter stockage token long terme
- sanitization des inputs
- CSP

### C. Checklist performance
- OnPush
- trackBy
- lazy-loading + code splitting

### D. Exercices capstone (évaluation)
1. Ajouter une fonctionnalité “tags” sur tickets (CRUD) et filtrage.
2. Ajouter un export CSV de la liste.
3. Ajouter une page “Profil” (préférences utilisateur) avec store global.
4. Ajouter un test e2e de non-régression.

---

## Planning suggéré (3 jours)

### Jour 1
- Setup, architecture, routing, auth, interceptors

### Jour 2
- Store (NgRx), tickets list + détails, dashboard

### Jour 3
- Form wizard + CVA, tests, CI, optimisations

---

## Ressources
- Angular Docs : https://angular.dev
- RxJS : https://rxjs.dev
- NgRx : https://ngrx.io
- Testing Library : https://testing-library.com/docs/angular-testing-library/intro
