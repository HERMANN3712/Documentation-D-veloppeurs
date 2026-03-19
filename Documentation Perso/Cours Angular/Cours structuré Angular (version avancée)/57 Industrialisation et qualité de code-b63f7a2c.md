# Formation Angular — Industrialisation et qualité de code

- **Référence** : 57
- **Durée conseillée** : 2 jours (14h) — adaptable 1 à 3 jours
- **Public** : développeurs Angular, tech leads, formateurs
- **Pré-requis** : TypeScript, Angular CLI, Git, notions de tests
- **Format** : alternance théorie / démonstrations / ateliers

> Une application Angular professionnelle s’appuie sur *linting, formatage, conventions d’architecture, revues de code, tests automatiques, analyse statique, CI/CD et documentation technique*. La qualité ne repose pas seulement sur le code, mais aussi sur les **pratiques d’équipe**.

---

## 1. Objectifs pédagogiques

À l’issue de la formation, les participants seront capables de :

1. Mettre en place une **chaîne de qualité** cohérente (lint/format/test/build/analyse) pour un projet Angular.
2. Définir et appliquer des **conventions d’architecture** et de style (structure, modules, dépendances, nommage).
3. Organiser des **revues de code** efficaces (checklists, règles, qualité collective).
4. Écrire et automatiser des **tests** (unitaires, intégration, E2E) et gérer la couverture.
5. Utiliser l’**analyse statique** (ESLint, TypeScript, Sonar, règles custom) et traiter la dette.
6. Concevoir un pipeline **CI/CD** (GitHub Actions/GitLab CI) avec qualité bloquante.
7. Produire une **documentation technique** utile et maintenable.

---

## 2. Plan de la formation

1. **Industrialisation : enjeux et chaîne de valeur**
2. **Linting & Formatage : ESLint + Prettier**
3. **Conventions d’architecture Angular**
4. **Revues de code & pratiques d’équipe**
5. **Tests automatiques : unitaires, intégration, E2E**
6. **Analyse statique & sécurité : TypeScript, Sonar, dépendances**
7. **CI/CD : pipelines, qualité bloquante, releases**
8. **Documentation technique & gouvernance**
9. **Atelier fil rouge : industrialiser un projet Angular**

---

## 3. Déroulé détaillé (contenu complet)

### Module 1 — Industrialisation : enjeux et chaîne de valeur (1h)

#### 1.1 Définitions
- **Industrialisation** : rendre la production logicielle **répétable, automatisée, mesurable**.
- **Qualité** : conformité aux exigences + maintenabilité + robustesse + sécurité + performance.
- **Quality Gates** : seuils bloquants (tests OK, couverture minimale, lint sans erreurs, analyse statique). 

#### 1.2 Symptômes d’un projet non industrialisé
- Builds manuels, “ça marche sur ma machine”.
- Style incohérent, refactors risqués.
- Bugs de régression, dette technique non suivie.
- Code review au ressenti.
- Releases douloureuses.

#### 1.3 Chaîne de qualité cible (vision)
1. **Local** : hooks Git + scripts npm (lint/format/test).
2. **PR/MR** : CI exécute build + tests + analyse.
3. **Main** : pipeline complet, packaging, publication.
4. **Release** : versioning, changelog, déploiement, rollback.

#### 1.4 Indicateurs utiles
- Temps de build CI, taux de succès pipeline.
- Nombre d’issues lint/sonar, dette.
- Couverture tests utile (pas “au chiffre”).
- Lead time, fréquence de déploiement.

**Livrable attendu** : une checklist d’outils et de pratiques du projet.

---

### Module 2 — Linting & Formatage : ESLint + Prettier (2h)

#### 2.1 Pourquoi séparer lint et formatage
- **Prettier** : mise en forme automatique (opinionated).
- **ESLint** : règles de qualité (patterns, erreurs, bonnes pratiques).

#### 2.2 Configuration recommandée (Angular moderne)

**Dépendances (exemple)**
```bash
npm i -D eslint prettier eslint-config-prettier eslint-plugin-prettier \
  @typescript-eslint/parser @typescript-eslint/eslint-plugin \
  @angular-eslint/eslint-plugin @angular-eslint/eslint-plugin-template \
  @angular-eslint/template-parser
```

**Scripts package.json**
```json
{
  "scripts": {
    "lint": "ng lint",
    "format": "prettier . --write",
    "format:check": "prettier . --check",
    "lint:fix": "ng lint --fix"
  }
}
```

#### 2.3 Règles essentielles (exemples)
- TypeScript
  - `@typescript-eslint/no-floating-promises` (promises non gérées)
  - `@typescript-eslint/consistent-type-imports`
  - `@typescript-eslint/no-explicit-any` (à adapter)
- Angular
  - sélecteurs cohérents (prefix + style)
  - templates : accessibilité et bonnes pratiques
- Qualité
  - `complexity`, `max-lines`, `max-params` (avec pragmatisme)

#### 2.4 Conventions de commit et hooks Git
- Objectif : empêcher l’entrée de code non conforme.

**Husky + lint-staged (exemple)**
```bash
npm i -D husky lint-staged
npx husky init
```

**lint-staged (exemple)**
```json
{
  "lint-staged": {
    "*.{ts,html,scss,md}": [
      "prettier --write",
      "eslint --fix"
    ]
  }
}
```

**Bonnes pratiques**
- Limiter aux fichiers modifiés.
- Garder des règles qui **aident** (pas “police”).

#### 2.5 Atelier (30–45 min)
- Ajouter Prettier + ESLint sur un projet.
- Corriger les erreurs, activer `--fix`, valider en commit.

---

### Module 3 — Conventions d’architecture Angular (2h)

#### 3.1 Objectifs
- Prévisibilité : *où mettre quoi*.
- Encapsulation et limites : réduire les dépendances.
- Faciliter le refactor et l’onboarding.

#### 3.2 Structure de projet recommandée (exemple)
```text
src/app/
  core/            # singletons, services transverses, interceptors, guards
  shared/          # composants/pipes/directives réutilisables (sans état métier)
  features/        # domaines (bounded contexts)
    orders/
      pages/
      components/
      services/
      state/
      models/
  app.routes.ts
```

#### 3.3 Règles d’import et boundaries
- `core` ne dépend de rien (ou presque).
- `shared` ne dépend pas des `features`.
- Chaque `feature` peut dépendre de `shared` et `core`.

**Outils**
- ESLint rules + `eslint-plugin-boundaries` (optionnel)
- Nx (si monorepo) : `enforce-module-boundaries`

#### 3.4 Patterns Angular à standardiser
- **Standalone components** vs NgModules (choisir une stratégie).
- **Injection** : préférer `providedIn: 'root'` + services par feature.
- **Routing** : lazy-loading par feature.
- **State management** : signaux/RxJS/NgRx, conventions de dossiers.
- **Smart/Dumb components** (container/presentational).

#### 3.5 Dette technique d’architecture
- Éviter : “shared fourre-tout”.
- Scinder par domaine.
- Appliquer la règle : *une feature = une responsabilité.*

#### 3.6 Atelier
- Refactor de structure : déplacer composants, créer `core/shared/features`.
- Ajouter des règles d’import qui empêchent les dépendances interdites.

---

### Module 4 — Revues de code & pratiques d’équipe (2h)

#### 4.1 La review comme outil de qualité collective
- Détection d’erreurs.
- Alignement sur les conventions.
- Partage de connaissances.

#### 4.2 Standard de Pull Request
- Taille limitée (idéal < 300 lignes diff).
- Titre clair + description.
- Liens vers issue/ticket.
- Checklist (tests, doc, accessibilité, perf).

**Template PR (exemple)**
```md
## Pourquoi ?

## Quoi ?

## Comment tester ?

## Checklist
- [ ] Lint/format OK
- [ ] Tests unitaires ajoutés/ajustés
- [ ] Pas de dette introduite (ou justifiée)
- [ ] Doc/README mis à jour si nécessaire
```

#### 4.3 Checklist de review Angular (extraits)
- **Lisibilité** : nommage, fonctions courtes, logique claire.
- **Templates** : `trackBy`, `async` pipe, accessibilité.
- **RxJS** : pas de subscriptions oubliées, gestion erreurs, `takeUntilDestroyed`.
- **Performance** : `OnPush` quand pertinent, éviter recomputations.
- **Sécurité** : sanitizer, pas d’innerHTML non maîtrisé.

#### 4.4 Gestion des désaccords
- Prioriser : conventions du projet > préférences personnelles.
- Documenter une décision (ADR) quand nécessaire.
- Timebox des débats.

#### 4.5 Atelier
- Simulation de review : détecter problèmes (architecture, tests, risques).

---

### Module 5 — Tests automatiques : unitaires, intégration, E2E (3h)

#### 5.1 Pyramide de tests
- Beaucoup d’unitaires (rapides).
- Intégration ciblée (composants + services).
- E2E (plus lents, parcours critiques).

#### 5.2 Tests Angular : stratégie outillée
- **Unitaires** : Jest (souvent plus rapide) ou Karma (selon contexte).
- **Component testing** : Testing Library + Jest (recommandé).
- **E2E** : Playwright (moderne) ou Cypress.

#### 5.3 Principes
- Tester le comportement, pas l’implémentation.
- Stable : éviter les timings arbitraires.
- Un test = un objectif clair.

#### 5.4 Exemples (pseudo-code orienté Angular)

**Service + HttpClientTesting**
```ts
it('charge la liste', () => {
  const http = TestBed.inject(HttpTestingController);
  service.getAll().subscribe(res => expect(res).toEqual([{ id: 1 }]));

  const req = http.expectOne('/api/items');
  expect(req.request.method).toBe('GET');
  req.flush([{ id: 1 }]);
});
```

**Component test (Testing Library)**
```ts
const { getByText } = await render(MyComponent, {
  providers: [{ provide: MyApi, useValue: mockApi }]
});
expect(getByText('Bonjour')).toBeTruthy();
```

**Playwright (E2E)**
```ts
test('connexion', async ({ page }) => {
  await page.goto('/login');
  await page.getByLabel('Email').fill('a@b.com');
  await page.getByLabel('Mot de passe').fill('secret');
  await page.getByRole('button', { name: 'Se connecter' }).click();
  await expect(page).toHaveURL('/home');
});
```

#### 5.5 Couverture et qualité des tests
- Mesurer, mais éviter le “gaming”.
- Couvrir : logique métier, erreurs, cas limites.
- Exiger tests sur code critique.

#### 5.6 Atelier
- Écrire 3 tests : service + composant + E2E parcours.
- Ajouter exécution en CI.

---

### Module 6 — Analyse statique & sécurité (2h)

#### 6.1 Analyse TypeScript
- `strict: true` (objectif).
- `noImplicitAny`, `strictNullChecks`.
- Types utilitaires, narrowing.

#### 6.2 Sonar et Quality Gate
- Détection : code smells, bugs, duplications.
- Concepts : dette, maintainability rating.
- Gérer les faux positifs : baseline + règles adaptées.

#### 6.3 Dépendances & vulnérabilités
- `npm audit` / GitHub Dependabot / Snyk.
- Politique de mise à jour : mineures fréquentes, majeures planifiées.

#### 6.4 Accessibilité et qualité UI
- Lint template (Angular ESLint template).
- Tests a11y possibles (axe-core) sur parcours clés.

#### 6.5 Atelier
- Activer `strict` graduellement.
- Ajouter audit de dépendances dans le pipeline.

---

### Module 7 — CI/CD : pipelines, qualité bloquante, releases (2h)

#### 7.1 Objectifs CI/CD
- Reproductibilité.
- Feedback rapide.
- Déploiement fiable.

#### 7.2 Jobs typiques
1. Install + cache
2. Lint + format check
3. Tests unitaires + couverture
4. Build
5. E2E (optionnel, nightly)
6. Analyse statique (Sonar)
7. Packaging / publication
8. Déploiement (staging → prod)

#### 7.3 Exemple GitHub Actions (simplifié)
```yaml
name: ci
on: [pull_request, push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'
      - run: npm ci
      - run: npm run format:check
      - run: npm run lint
      - run: npm test -- --watch=false
      - run: npm run build -- --configuration production
```

#### 7.4 Stratégies de branches & release
- Trunk-based (recommandation) vs GitFlow.
- Feature flags pour livrer sans activer.
- Versioning : SemVer + changelog.

#### 7.5 Secrets, environnements, déploiement
- Variables chiffrées dans CI.
- Environnements : dev/staging/prod.
- Stratégie de rollback.

#### 7.6 Atelier
- Construire une pipeline qui bloque si : lint KO, tests KO, format KO.

---

### Module 8 — Documentation technique & gouvernance (1h30)

#### 8.1 Pourquoi documenter
- Réduire la dépendance aux individus.
- Accélérer onboarding, faciliter maintenance.

#### 8.2 Artefacts utiles
- **README** : lancer le projet, scripts, conventions.
- **CONTRIBUTING** : règles de PR, commit, review.
- **ADR** (Architecture Decision Records) : décisions, contexte.
- **Docs d’architecture** : schémas simples, boundaries.
- **Runbook** : exploitation, incidents.

#### 8.3 Documentation “juste suffisante”
- Favoriser le proche du code.
- Mettre à jour via PR.
- Automatiser : génération de changelog, doc coverage.

#### 8.4 Atelier
- Rédiger 1 ADR (ex : “Jest vs Karma”, “Playwright”).

---

## 4. Atelier fil rouge (2h à 4h)

### Objectif
Prendre une application Angular (existante ou starter) et mettre en place une **chaîne d’industrialisation** cohérente.

### Étapes
1. Ajouter ESLint/Prettier + scripts.
2. Appliquer conventions d’architecture (structure features/core/shared).
3. Mettre en place template PR + checklist review.
4. Ajouter tests unitaires + un E2E critique.
5. Ajouter analyse statique (strict TS + audit deps + option Sonar).
6. Mettre en CI (quality gate) + artefacts (build).

### Critères de réussite
- `npm run lint`, `npm run format:check`, `npm test`, `npm run build` OK en local et en CI.
- PR type avec checklist.
- Documentation minimale (README + 1 ADR).

---

## 5. Annexes (références rapides)

### A. Checklist “Definition of Done” (exemple)
- [ ] Code formaté (Prettier) et lint sans erreurs.
- [ ] Tests ajoutés/ajustés, couverture pertinente.
- [ ] Pas de dette non justifiée.
- [ ] Documentation mise à jour si nécessaire.
- [ ] PR petite, review effectuée.

### B. Bonnes pratiques Angular ciblées qualité
- Activer `changeDetection: ChangeDetectionStrategy.OnPush` sur composants de présentation.
- Utiliser `trackBy` sur listes.
- Gérer désabonnements (`takeUntilDestroyed`).
- Centraliser la configuration HTTP (interceptors).
- Éviter logique métier dans template.

### C. Suggestions d’outils
- Lint/format : ESLint, Prettier
- Tests : Jest, Testing Library, Playwright
- CI : GitHub Actions, GitLab CI
- Analyse : SonarQube/SonarCloud, npm audit, Dependabot
- Doc : MkDocs/Docusaurus (optionnel), ADR en Markdown

---

## 6. Proposition de timing (2 jours)

### Jour 1
- M1 Industrialisation (1h)
- M2 Lint/Format (2h)
- M3 Architecture (2h)
- M4 Code review (2h)

### Jour 2
- M5 Tests (3h)
- M6 Analyse statique & sécurité (2h)
- M7 CI/CD (2h)
- M8 Documentation + gouvernance (1h30)
- Synthèse + plan d’action équipe (0h30)

---

## 7. Synthèse et plan d’action équipe

1. **Aligner** conventions (format/lint/architecture) et les automatiser.
2. Définir une **DoD** et une **checklist de review**.
3. Introduire des **quality gates** réalistes.
4. Monter le niveau de tests sur chemins critiques.
5. Mettre la **CI** au centre : tout passe par PR, feedback rapide.
6. Documenter les décisions (ADR) et le process (CONTRIBUTING).
