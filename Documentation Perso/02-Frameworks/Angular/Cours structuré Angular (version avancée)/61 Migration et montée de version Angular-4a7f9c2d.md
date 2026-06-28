# Formation — Migration et montée de version Angular

**Référence** : 61  
**Durée suggérée** : 1 à 2 jours (adaptable)  
**Public** : développeurs Angular (débutant+ à confirmé), lead dev, formateurs  
**Pré-requis** : TypeScript, CLI Angular, npm/yarn/pnpm, git, notions de tests (Jasmine/Karma ou Jest), CI (optionnel)  

---

## 1) Objectifs pédagogiques

À l’issue de la formation, le participant sera capable de :

- Comprendre le cycle de release Angular (majors/minors/patch), la politique de support et les impacts d’une montée de version.
- Lire et exploiter efficacement les **Release Notes**, **Breaking Changes** et **deprecations**.
- Exécuter une montée de version via **`ng update`** (ou alternative) en maîtrisant les **migrations automatiques**.
- Valider la compatibilité des **dépendances tierces** (Angular Material, RxJS, NgRx, libs UI, etc.).
- Définir une stratégie de **tests** et de **CI** pour sécuriser l’upgrade.
- Concevoir une approche progressive (upgrade incrémental, branches, feature flags quand utile) et établir un plan de rollback.

---

## 2) Format & pédagogie

- Alternance **apports structurés**, **démonstrations**, **TP**.
- Utilisation d’un projet Angular existant (réel ou fourni), et d’un dépôt git pour travailler par étapes.
- Checklists, scripts d’upgrade, et pratiques CI reproductibles.

---

## 3) Plan global de la formation

1. **Comprendre les versions Angular & les risques**
2. **Préparer l’upgrade (audit du projet)**
3. **Lire et exploiter les Release Notes / Breaking changes**
4. **Mettre à jour Angular avec `ng update`**
5. **Migrations automatiques : principes et contrôle**
6. **Gestion des dépendances tierces (Angular Material, RxJS, outils, libs)**
7. **Stratégie de tests (unitaires, intégration, e2e) & qualité**
8. **Stratégie de livraison : git, CI/CD, rollback, monitoring**
9. **Cas courants & dépannage (troubleshooting)**
10. **Synthèse : checklists et plan d’action**

---

# Module 1 — Comprendre les versions Angular & les risques

## 1.1 SemVer et réalité Angular
Angular suit globalement **SemVer** :

- **Major** (ex. 16 → 17) : changements potentiellement cassants, migrations.
- **Minor** (ex. 17.0 → 17.1) : nouvelles fonctionnalités, généralement non cassantes.
- **Patch** (ex. 17.1.0 → 17.1.2) : corrections.

⚠️ En pratique, une minor peut introduire :
- des warnings plus stricts,
- des comportements plus précis,
- des dépréciations.

## 1.2 Politique de support
- Les versions Angular ont une période de support (active + LTS selon périodes). 
- Impacts : sécurité, correctifs, compatibilité tooling.

## 1.3 Types de risques
- **API cassées** (Angular, RxJS, TypeScript)
- **Changements de tooling** (builders, CLI, Vite/webpack, esbuild, SSR)
- **Breaking changes dans libs** (Material, NgRx, PrimeNG, etc.)
- **Interop**: Node version, npm, browserslist.

### Livrable du module
- Clarifier la cible : passer en N+1 ? rattraper plusieurs majors ?
- Définir un **scope**, un **calendrier**, et un **budget de stabilisation**.

---

# Module 2 — Préparer l’upgrade (audit du projet)

## 2.1 État initial
Avant de toucher à quoi que ce soit :

1. **Créer une branche d’upgrade**
2. **Vérifier que la branche de base est verte** :
   - build ok
   - tests unitaires ok
   - lint ok
   - e2e ok (si existants)
3. **Tagger une version stable** (pour rollback rapide)

## 2.2 Identifier les versions de l’écosystème
Relever :
- Angular (core/cli/material)
- RxJS
- TypeScript
- Node
- Outils : ESLint, Prettier, Jest/Karma, Cypress/Playwright, Nx (si monorepo)

Commandes utiles :

```bash
node -v
npm -v
ng version
npm ls @angular/core rxjs typescript
```

## 2.3 Nettoyage et mise en conformité
- Résoudre les warnings de build actuels.
- Supprimer code mort / packages inutilisés.
- Uniformiser scripts npm.

Checklist :
- `package-lock.json` / `pnpm-lock.yaml` cohérent
- dépendances en double / peer deps problématiques

---

# Module 3 — Lire et exploiter les Release Notes / Breaking Changes

## 3.1 Où chercher l’information
- Angular changelog / blog
- Guides officiels de migration
- Deprecations list
- Issues GitHub des libs tierces

## 3.2 Méthode de lecture efficace
Pour chaque version traversée (si on saute plusieurs majors) :

1. **Breaking changes**
2. **Deprecations**
3. **Required versions** (Node / TS / RxJS)
4. **Migrations disponibles** (schematics)
5. **Tooling changes**

## 3.3 Construire une matrice d’impact
Un tableau simple :

| Domaine | Changement | Impact | Action | Owner | Status |
|---|---|---:|---|---|---|
| Build | Nouveau builder | Moyen | Adapter scripts CI | DevOps | TODO |
| RxJS | opérateurs dépréciés | Haut | Refactor | Team | TODO |

---

# Module 4 — Mettre à jour Angular avec `ng update`

## 4.1 Principes
`ng update` :
- met à jour versions de packages Angular
- exécute des **migrations** (schematics)
- peut proposer des étapes supplémentaires

## 4.2 Stratégie recommandée : incrémentale
Plutôt que 12 → 17 d’un coup :
- monter **major par major** (ou au moins par paliers)
- valider, committer, tagger

## 4.3 Commandes types

### 4.3.1 Mettre à jour CLI et core

```bash
ng update @angular/core @angular/cli
```

### 4.3.2 Vérifier ce qui sera fait (dry run)

```bash
ng update @angular/core @angular/cli --dry-run
```

### 4.3.3 Forcer en cas de peer deps (avec prudence)

```bash
ng update @angular/core @angular/cli --force
```

⚠️ `--force` n’est pas une solution « propre ». À utiliser :
- si vous avez vérifié la compatibilité des libs tierces,
- et si vous prévoyez une phase de corrections juste après.

## 4.4 Ordre de mise à jour conseillé
1. Angular CLI + core
2. Angular Material/CDK (si utilisé)
3. RxJS/TypeScript (souvent aligné par Angular)
4. Outils de tests et lint

---

# Module 5 — Migrations automatiques : principes et contrôle

## 5.1 Qu’est-ce qu’une migration Angular ?
- Une migration est un script (schematic) qui modifie automatiquement :
  - code TS
  - templates
  - config JSON
  - polyfills / builders

## 5.2 Exécuter uniquement des migrations

```bash
ng update @angular/core --migrate-only
```

## 5.3 Contrôler les changements
- Toujours inspecter le diff git.
- Commits atomiques :
  1. migration
  2. corrections
  3. refactor optionnel

## 5.4 Stratégie anti-régressions
- Après chaque migration :
  - `ng build`
  - `ng test`
  - `ng lint` (ou `eslint`)

---

# Module 6 — Gestion des dépendances tierces

## 6.1 Problème classique : peerDependencies
Les libs Angular imposent souvent une version précise de `@angular/core`.

Approche :
1. Vérifier la version supportée (release notes)
2. Mettre à jour la lib
3. Corriger les breaking changes de la lib

## 6.2 Angular Material / CDK

```bash
ng update @angular/material
```

Attendre :
- migrations de composants
- changements de thèmes
- ajustements d’API

## 6.3 RxJS
Points d’attention :
- imports profonds (interdits ou déconseillés)
- dépréciations d’opérateurs
- changement de scheduler/interop

Bonnes pratiques :
- utiliser imports recommandés
- éviter patching

## 6.4 NgRx (si utilisé)
- match des versions NgRx ↔ Angular.
- exécuter migrations NgRx quand disponibles.

## 6.5 Outils de test et build
- Karma/Jasmine vs Jest
- Cypress vs Playwright
- changements de builders / config vite/webpack/esbuild

## 6.6 Stratégie “compatibility gate”
Avant de corriger du code applicatif :
- obtenir un **build qui passe**
- puis un **test minimal**

---

# Module 7 — Stratégie de tests & qualité

## 7.1 Pourquoi les tests sont la clé d’une upgrade
Une migration est « réussie » quand :
- l’application compile,
- les comportements métier sont inchangés,
- les parcours critiques fonctionnent.

## 7.2 Pyramide de tests recommandée
- **Unitaires** : services, pipes, composants (isolés)
- **Intégration** : composants avec dépendances, routing
- **E2E** : parcours critiques (login, checkout, recherche...)

## 7.3 Approche progressive
1. Stabiliser build + unit tests
2. Corriger lint/format
3. Lancer e2e ensuite

## 7.4 Metriques utiles
- taux de réussite CI
- temps de build/test
- nombre de warnings

## 7.5 Exemples de “smoke tests”
- build prod
- démarrage local
- rendu page home
- appels API mockés

---

# Module 8 — Stratégie de livraison : git, CI/CD, rollback

## 8.1 Branching
- branche dédiée `upgrade/angular-XX`
- PR fréquentes, petites

## 8.2 CI : pipeline minimal
Étapes :
1. install (lockfile)
2. lint
3. test
4. build
5. e2e (optionnel nightly)

## 8.3 Rollback
- tag de la dernière version stable
- stratégie de revert PR
- feature flags (rarement nécessaire pour une upgrade seule)

## 8.4 Post-release
- monitoring (erreurs runtime)
- logs / RUM
- plan de hotfix

---

# Module 9 — Cas courants & dépannage (troubleshooting)

## 9.1 Erreurs de peer deps
Symptômes : `ERESOLVE`, incompatibilité.

Actions :
- lire le message : quel package bloque
- vérifier version compatible
- mettre à jour package concerné
- recours à `--force` uniquement si nécessaire

## 9.2 TypeScript trop récent / trop ancien
- Angular impose une plage de versions TS.
- Alignement via `ng update` + release notes.

## 9.3 Problèmes de build
- builders changés
- config `angular.json` modifiée par migration
- assets/styles mal résolus

## 9.4 Tests cassés
- changements de zone.js, async utilities
- snapshots (Jest)
- timing e2e

## 9.5 SSR / Universal
- vérifier compatibilité avec la version Angular.
- valider rendu serveur, hydration si activée.

---

# Module 10 — Synthèse : checklists et plan d’action

## 10.1 Checklist “avant upgrade”
- [ ] Branche dédiée
- [ ] CI verte sur la version actuelle
- [ ] Tag version stable
- [ ] Audit des dépendances et versions
- [ ] Lecture release notes des versions à traverser

## 10.2 Checklist “pendant upgrade”
- [ ] `ng update` par paliers
- [ ] migrations exécutées et diff relu
- [ ] commits atomiques
- [ ] build + tests après chaque palier
- [ ] upgrade libs tierces (Material, NgRx, etc.)

## 10.3 Checklist “après upgrade”
- [ ] suppression de drapeaux temporaires
- [ ] nettoyage dépendances
- [ ] mise à jour docs (README, prérequis Node)
- [ ] monitoring post-release

## 10.4 Plan d’action type (exemple)
1. J0 : audit + matrice d’impact
2. J1 : upgrade Angular N→N+1 + corrections
3. J2 : upgrade libs tierces + corrections
4. J3 : stabilisation test + e2e
5. J4 : release contrôlée + monitoring

---

## Annexes

### A) Commandes utiles

```bash
# Vérifier versions
g version

# Upgrade dry-run
ng update @angular/core @angular/cli --dry-run

# Upgrade réel
ng update @angular/core @angular/cli

# Material
ng update @angular/material

# Nettoyage
rm -rf node_modules package-lock.json
npm ci
```

### B) Conseils de formateur (animation)
- Partir d’un projet avec quelques dépendances et tests.
- Forcer un cas “peer deps conflict” pour entraîner les stagiaires.
- Faire un départ « CI rouge », puis guider vers la résolution.

---

**Fin de cours**
