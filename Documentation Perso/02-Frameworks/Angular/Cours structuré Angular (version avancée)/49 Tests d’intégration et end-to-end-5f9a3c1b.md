# Formation Angular — Tests d’intégration et End‑to‑End (E2E)

> **Objectif** : apprendre à concevoir, écrire et maintenir des **tests d’intégration** et des **tests E2E** pour valider des **parcours utilisateur complets** dans une application Angular. Ces tests complètent les tests unitaires en vérifiant l’**intégration réelle** entre composants, routes, formulaires et services, et sécurisent les **flux critiques**.

---

## Sommaire

1. [Prérequis et objectifs pédagogiques](#1-prérequis-et-objectifs-pédagogiques)
2. [Positionnement : unitaires vs intégration vs E2E](#2-positionnement--unitaires-vs-intégration-vs-e2e)
3. [Stratégie de test : pyramide, risques, couverture](#3-stratégie-de-test--pyramide-risques-couverture)
4. [Environnement et outillage](#4-environnement-et-outillage)
5. [Tests d’intégration Angular : principes et patterns](#5-tests-dintégration-angular--principes-et-patterns)
6. [Tester l’intégration avec le Router](#6-tester-lintégration-avec-le-router)
7. [Tester formulaires + services + UI](#7-tester-formulaires--services--ui)
8. [Gestion des dépendances : doubles, mocks, fakes et interceptors](#8-gestion-des-dépendances--doubles-mocks-fakes-et-interceptors)
9. [Asynchronisme : Observables, timers, animations](#9-asynchronisme--observables-timers-animations)
10. [E2E avec Playwright : mise en place](#10-e2e-avec-playwright--mise-en-place)
11. [Écrire des scénarios E2E robustes (parcours critique)](#11-écrire-des-scénarios-e2e-robustes-parcours-critique)
12. [Sélecteurs stables, accessibilité et tests](#12-sélecteurs-stables-accessibilité-et-tests)
13. [Gestion des données et environnements (mock serveur, seed, cleanup)](#13-gestion-des-données-et-environnements-mock-serveur-seed-cleanup)
14. [Fiabilité : flakiness, retries, timeouts, parallélisme](#14-fiabilité--flakiness-retries-timeouts-parallélisme)
15. [CI/CD : exécution, rapports, artefacts](#15-cicd--exécution-rapports-artefacts)
16. [Atelier fil rouge : conception d’une suite intégration + E2E](#16-atelier-fil-rouge--conception-dune-suite-intégration--e2e)
17. [Checklists et bonnes pratiques](#17-checklists-et-bonnes-pratiques)
18. [Annexes : templates, snippets et conventions](#18-annexes--templates-snippets-et-conventions)

---

## 1. Prérequis et objectifs pédagogiques

### Prérequis
- Connaissances de base Angular : composants, services, routing, forms.
- Notions de tests unitaires (Jest/Karma/Jasmine) : `describe/it/expect`.
- Connaissances TypeScript.

### Objectifs pédagogiques
À la fin, vous saurez :
- Identifier ce qui relève d’un **test unitaire**, **intégration**, **E2E**.
- Écrire des tests d’intégration Angular qui valident un **assemblage réel** (template + dépendances + interactions).
- Tester des parcours incluant **router**, **guards**, **resolvers**, **interceptors**.
- Mettre en place une stratégie E2E avec **Playwright** et écrire des scénarios robustes.
- Stabiliser les tests (sélecteurs, données, timeouts) et les intégrer à la **CI/CD**.

---

## 2. Positionnement : unitaires vs intégration vs E2E

### Définitions
- **Test unitaire** : teste une unité isolée (fonction, service, component class) avec le minimum de dépendances.
- **Test d’intégration** : teste plusieurs unités ensemble **dans Angular** (component + template + services + forms + routing partiel) en simulant certaines dépendances (HTTP par ex.).
- **Test E2E** : teste l’application du point de vue utilisateur, dans un navigateur, avec un maximum de couches réelles.

### Pourquoi l’intégration et l’E2E sont essentiels
- Ils détectent les problèmes que les unitaires ne voient pas :
  - Mauvais wiring (providers, DI, modules/standalone imports)
  - Bugs de template (bindings, directives, pipes)
  - Problèmes de routing (routes, guards, navigation)
  - Flows complets : formulaire → validation → appel service → navigation → affichage
- Ils sécurisent les flux critiques (authentification, paiement, onboarding, etc.).

---

## 3. Stratégie de test : pyramide, risques, couverture

### Pyramide des tests (adaptée Angular)
- **Beaucoup** de tests unitaires (rapides, précis)
- **Moins** de tests d’intégration (plus chers, plus robustes)
- **Quelques** tests E2E (les plus coûteux, valident les parcours critiques)

### Approche “risk‑based”
Prioriser les scénarios selon :
- Impact business (ex : achat)
- Fréquence d’usage (ex : login)
- Historique de bugs (zones instables)
- Complexité (routing + validations + back)

### Critères d’un bon scénario d’intégration/E2E
- Centré sur un **objectif utilisateur**
- Assertions sur des **résultats observables** (UI, navigation, messages)
- Données contrôlées (fixtures/seed)
- Temps d’exécution raisonnable

---

## 4. Environnement et outillage

### Angular : test d’intégration
- Outils possibles :
  - **Jest** (souvent préféré aujourd’hui : plus rapide)
  - Karma/Jasmine (historiquement)
  - `@angular/core/testing` + `TestBed`
  - `@angular/router/testing` (RouterTestingModule) ou tests Router modernes
  - `@angular/common/http/testing` : `HttpClientTestingModule` + `HttpTestingController`

### E2E : Playwright (recommandé)
- Alternatives : Cypress, WebdriverIO
- Pourquoi Playwright :
  - Multi‑navigateurs (Chromium, Firefox, WebKit)
  - Traces, vidéos, screenshots
  - API moderne, auto‑wait, parallélisme

### Convention de répertoires (proposition)
```
src/
  app/
    ...
  testing/
    test-ids.ts
    fixtures/
    fakes/

e2e/
  tests/
  fixtures/
  playwright.config.ts
```

---

## 5. Tests d’intégration Angular : principes et patterns

### Limite volontaire
Un test d’intégration Angular vise souvent :
- **composant + template + dépendances Angular**
- avec **backend simulé** (via `HttpTestingController` ou service fake)

### Créer un “System Under Test” (SUT)
- Importer ce qui est nécessaire : composants enfants réels ou stubs ?
- Configurer providers : services réels ou fakes ?
- Éviter d’importer toute l’app (sinon lenteur + fragilité)

### Exemple : composant de login (structure)
Imaginons :
- `LoginComponent` (formulaire)
- `AuthService` (login HTTP)
- navigation vers `/dashboard` au succès

#### Squelette de test (Jest ou Jasmine)
```ts
import { TestBed } from '@angular/core/testing';
import { ReactiveFormsModule } from '@angular/forms';
import { HttpClientTestingModule, HttpTestingController } from '@angular/common/http/testing';
import { Router } from '@angular/router';
import { RouterTestingModule } from '@angular/router/testing';

import { LoginComponent } from './login.component';

describe('LoginComponent (integration)', () => {
  let httpMock: HttpTestingController;
  let router: Router;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [
        ReactiveFormsModule,
        HttpClientTestingModule,
        RouterTestingModule.withRoutes([
          { path: 'dashboard', component: DummyComponent },
        ]),
        LoginComponent, // si standalone
      ],
      declarations: [DummyComponent],
    }).compileComponents();

    httpMock = TestBed.inject(HttpTestingController);
    router = TestBed.inject(Router);
  });

  it('logs in and navigates to dashboard', async () => {
    const fixture = TestBed.createComponent(LoginComponent);
    fixture.detectChanges();

    // ... remplir le formulaire, cliquer, mock HTTP, assert navigation

    httpMock.verify();
  });
});

@Component({ template: '' })
class DummyComponent {}
```

---

## 6. Tester l’intégration avec le Router

### Points à valider
- Navigation (URL, route active)
- Guards (auth/no-auth)
- Resolvers (données préchargées)
- Paramètres et query params

### Astuce : déclencher la navigation
Avec `RouterTestingModule`, vous pouvez appeler :
- `router.navigateByUrl('/dashboard')`
- `fixture.ngZone?.run(() => router.navigate(...))` selon contexte

### Exemple : navigation après soumission
```ts
jest.spyOn(router, 'navigateByUrl');

// action utilisateur → click

expect(router.navigateByUrl).toHaveBeenCalledWith('/dashboard');
```

### Tester guard (approche intégration)
- Utiliser un guard réel avec un fake `AuthStore`.
- Vérifier que la route est bloquée/redirigée.

---

## 7. Tester formulaires + services + UI

### Ce qu’on veut vérifier
- Validation UI (messages, boutons disabled)
- Mapping form → payload service
- Gestion d’erreur (message, reset, focus)

### Exemple : test d’un formulaire réactif
```ts
it('disables submit when form invalid and shows errors after touch', () => {
  const fixture = TestBed.createComponent(LoginComponent);
  fixture.detectChanges();

  const component = fixture.componentInstance;
  component.form.controls['email'].setValue('invalid');
  component.form.controls['password'].setValue('');
  component.form.markAllAsTouched();
  fixture.detectChanges();

  const button: HTMLButtonElement = fixture.nativeElement.querySelector('[data-testid="login-submit"]');
  expect(button.disabled).toBe(true);

  const error = fixture.nativeElement.querySelector('[data-testid="email-error"]');
  expect(error?.textContent).toContain('Email invalide');
});
```

### Conseils
- Utiliser des **data-testid** pour stabiliser les tests.
- Éviter les assertions sur structure DOM fragile (classes CSS, hiérarchie).

---

## 8. Gestion des dépendances : doubles, mocks, fakes et interceptors

### Types de doubles
- **Stub** : retourne une valeur fixe
- **Mock** : vérifie qu’il a été appelé (spy)
- **Fake** : implémentation simplifiée mais réaliste

### HTTP : `HttpClientTestingModule`
- Permet de capturer et répondre aux requêtes.

```ts
const req = httpMock.expectOne('/api/login');
expect(req.request.method).toBe('POST');
expect(req.request.body).toEqual({ email: 'a@b.com', password: 'secret' });
req.flush({ token: 'jwt' });
```

### Interceptors
- À tester en intégration selon criticité (ex: ajout header auth).
- Approche : inclure l’interceptor réel, et vérifier les requêtes sortantes.

---

## 9. Asynchronisme : Observables, timers, animations

### Problèmes fréquents
- Tests qui passent localement mais échouent en CI (timing)
- Observables non terminés

### Outils
- `fakeAsync` + `tick()` (Angular)
- `waitForAsync`
- `fixture.whenStable()`

### Règles pratiques
- Préférer **attendre un état observable** plutôt que dormir (pas de `setTimeout` arbitraire).
- Couper les animations en tests si nécessaire.

---

## 10. E2E avec Playwright : mise en place

### Installation (exemple)
```bash
npm i -D @playwright/test
npx playwright install
```

### Configuration (extrait)
Créer `playwright.config.ts` :
- baseURL (ex: `http://localhost:4200`)
- retries en CI
- traces sur échec

```ts
import { defineConfig } from '@playwright/test';

export default defineConfig({
  testDir: './e2e/tests',
  use: {
    baseURL: process.env['BASE_URL'] ?? 'http://localhost:4200',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
  },
  retries: process.env['CI'] ? 2 : 0,
  workers: process.env['CI'] ? 2 : undefined,
});
```

### Démarrage app + tests
- Option 1 : démarrer Angular via `npm run start` et lancer Playwright
- Option 2 : utiliser `webServer` dans config Playwright

---

## 11. Écrire des scénarios E2E robustes (parcours critique)

### Exemple : parcours login → dashboard
Hypothèse : présence de `data-testid`.

```ts
import { test, expect } from '@playwright/test';

test('Login success redirects to dashboard', async ({ page }) => {
  await page.goto('/login');

  await page.getByTestId('email').fill('user@example.com');
  await page.getByTestId('password').fill('secret');
  await page.getByTestId('login-submit').click();

  await expect(page).toHaveURL(/\/dashboard/);
  await expect(page.getByTestId('welcome')).toContainText('Bienvenue');
});
```

### Tester erreurs (mauvais mot de passe)
```ts
test('Login error shows message', async ({ page }) => {
  await page.goto('/login');

  await page.getByTestId('email').fill('user@example.com');
  await page.getByTestId('password').fill('wrong');
  await page.getByTestId('login-submit').click();

  await expect(page.getByTestId('toast-error')).toBeVisible();
  await expect(page.getByTestId('toast-error')).toContainText('Identifiants invalides');
});
```

### À valider dans un E2E
- URL finale
- Informations affichées
- États (loader, disabled)
- Accessibilité minimale (focus, erreurs)

---

## 12. Sélecteurs stables, accessibilité et tests

### Règle d’or
**Un test ne doit pas dépendre du CSS**.

### Approches recommandées (ordre)
1. `getByRole` (accessibilité) : plus proche utilisateur
2. `getByLabel` (form)
3. `getByTestId` (stable)

### Convention `data-testid`
- Format : `kebab-case`
- Préfixer par feature si besoin : `auth-login-submit`

Exemple :
```html
<input data-testid="email" type="email" />
<button data-testid="login-submit">Se connecter</button>
```

---

## 13. Gestion des données et environnements (mock serveur, seed, cleanup)

### Problème
Un E2E instable vient souvent de données non déterministes.

### Stratégies
- **Mock réseau** côté Playwright (ex: `page.route`) pour isoler le back.
- **Environnement de test** (API dédiée) avec base réinitialisée.
- Seed : création d’un utilisateur de test standard.

### Exemple : mock d’API login avec Playwright
```ts
test('Login with mocked API', async ({ page }) => {
  await page.route('**/api/login', async route => {
    await route.fulfill({
      status: 200,
      contentType: 'application/json',
      body: JSON.stringify({ token: 'fake-jwt' }),
    });
  });

  await page.goto('/login');
  // ...
});
```

### Pattern : cleanup
- Nettoyer ce que vous créez (compte, commande)
- Ou utiliser des données jetables

---

## 14. Fiabilité : flakiness, retries, timeouts, parallélisme

### Causes de flakiness
- Sélecteurs fragiles
- Attentes temporelles (`waitForTimeout`)
- Dépendances externes (API lente)
- Tests qui partagent des données

### Contremesures
- Attendre des états : `expect(locator).toBeVisible()`
- Activer traces/screenshots
- Isoler les données par worker
- Limiter les tests E2E aux parcours critiques

---

## 15. CI/CD : exécution, rapports, artefacts

### Objectifs CI
- Exécuter intégration + E2E sur PR
- Publier rapports
- Conserver artefacts sur échec (trace, vidéo)

### Exemple de scripts npm
```json
{
  "scripts": {
    "test:integration": "ng test --watch=false",
    "test:e2e": "playwright test",
    "test:all": "npm run test:integration && npm run test:e2e"
  }
}
```

### Bonnes pratiques
- Lancer E2E en parallèle si possible
- Cache des dépendances
- Rejouer les E2E en retry sur CI (mais corriger la cause)

---

## 16. Atelier fil rouge : conception d’une suite intégration + E2E

### Contexte
Feature « Authentification » :
- `/login` : formulaire
- `/dashboard` : page protégée
- `AuthInterceptor` ajoute `Authorization`

### Travail demandé
1. **Intégration** :
   - Test formulaire invalide (messages + submit disabled)
   - Test succès : POST `/api/login` → navigation `/dashboard`
   - Test erreur : affichage message
2. **E2E** :
   - Parcours login succès (mock ou env test)
   - Parcours accès direct `/dashboard` → redirection `/login` si non connecté

### Critères de réussite
- Tests stables, rapides
- Sélecteurs stables
- Assertions sur comportements utilisateur

---

## 17. Checklists et bonnes pratiques

### Checklist tests d’intégration
- [ ] Le test vérifie un comportement observable (UI/navigation)
- [ ] Les dépendances externes (HTTP) sont contrôlées
- [ ] Pas de sur‑mocking : garder le template réel
- [ ] Utiliser `data-testid` ou queries accessibilité
- [ ] Nettoyage : `httpMock.verify()`

### Checklist tests E2E
- [ ] Scénario = parcours critique
- [ ] Données déterministes (seed ou mocks)
- [ ] Sélecteurs robustes (role/label/testid)
- [ ] Pas de sleeps ; attentes sur états
- [ ] Artefacts sur échec (trace/vidéo)

---

## 18. Annexes : templates, snippets et conventions

### A. Template de test d’intégration
```ts
describe('Feature X (integration)', () => {
  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [],
      providers: [],
    }).compileComponents();
  });

  it('should ...', () => {
    const fixture = TestBed.createComponent(YourComponent);
    fixture.detectChanges();

    // arrange
    // act
    // assert
  });
});
```

### B. Convention `data-testid`
- Champs : `feature-field-name`
- Boutons : `feature-action`
- Messages : `feature-error-*`, `feature-success-*`

### C. Matrice “quoi tester où”
| Sujet | Unitaire | Intégration | E2E |
|---|---:|---:|---:|
| Validation pure (fonction) | ✅ | ➖ | ➖ |
| Composant + template + forms | ➖ | ✅ | ➖ |
| Routing + guard + resolver | ➖ | ✅ | ✅ (parcours) |
| Interceptor (header) | ✅/✅ | ✅ | ➖ |
| Parcours login complet | ➖ | ✅ (partiel) | ✅ |

---

## Conclusion
Les tests d’intégration et E2E sont indispensables pour élever la confiance produit : ils valident des parcours utilisateurs réels (routing, formulaires, services) et protègent les flux critiques. En combinant une bonne stratégie (pyramide + risque), des patterns solides (sélecteurs stables, données contrôlées, assertions comportementales) et une intégration CI, vous obtenez une suite de tests fiable, utile et maintenable.
