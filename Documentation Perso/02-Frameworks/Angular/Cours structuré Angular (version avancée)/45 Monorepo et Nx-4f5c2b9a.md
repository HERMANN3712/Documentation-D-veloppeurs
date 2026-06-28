# Formation (45) — Monorepo et Nx (Angular)

> Public : développeurs Angular (débutant à intermédiaire sur Nx) • Durée conseillée : 1 à 2 jours • Format : théorie + ateliers guidés

## Objectifs pédagogiques
À l’issue de la formation, les participants seront capables de :

- Expliquer les bénéfices et limites d’un monorepo en contexte d’entreprise.
- Créer et structurer un monorepo Angular avec **Nx**.
- Mettre en place des **libs** (partagées, domain, UI, data-access, util) et gérer les **dépendances**.
- Exécuter des tâches ciblées (build, test, lint, e2e) et exploiter le **caching** Nx.
- Comprendre et mettre en œuvre **Affected**, la visualisation du graphe, et les règles de gouvernance.
- Intégrer Nx dans une chaîne CI/CD (cache local/remote, exécution incrémentale, qualité).

## Prérequis
- Connaissances de base en Angular (components, modules/standalone, services, routing).
- Node.js LTS, npm/pnpm/yarn.
- Git (branches, commits).

## Matériel requis
- Node.js LTS installé
- Git
- Un éditeur (VS Code recommandé)
- Accès réseau si vous souhaitez tester Nx Cloud (optionnel)

---

# Plan de la formation

1. **Monorepo : principes, cas d’usage entreprise et gouvernance**
2. **Découverte de Nx : concepts, architecture et workspace**
3. **Créer un monorepo Angular avec Nx**
4. **Structurer : apps vs libs, boundaries et architecture modulaire**
5. **Dépendances, project graph et règles d’accès (enforce-module-boundaries)**
6. **Tâches Nx : build/test/lint/e2e, exécution ciblée, caching**
7. **Affected & performance : builds incrémentaux, optimisation au quotidien**
8. **CI/CD : pipelines, cache distant, qualité et versioning**
9. **Atelier final : mise en place complète sur un cas “entreprise”**

---

# 1) Monorepo : principes, cas d’usage entreprise et gouvernance

## 1.1 Définition
Un **monorepo** est un dépôt Git unique qui contient **plusieurs projets** (applications, bibliothèques, services, outils) partageant souvent des dépendances et des conventions.

### Monorepo vs multirepo
- **Multirepo** : chaque projet (app, lib) est dans un dépôt différent.
- **Monorepo** : un seul dépôt, plusieurs projets, souvent plus simple à faire évoluer de manière cohérente.

## 1.2 Pourquoi en entreprise ?
Dans des environnements d’entreprise, un monorepo facilite :

- **La mutualisation du code** (UI kit, clients API, utilitaires, modèles de domaine).
- **La gouvernance des bibliothèques internes** (ownership, versioning, règles d’accès, dépréciations).
- **La cohérence des pratiques** (lint, formatage, tests, conventions de commit, outillage).
- **Le refactoring transversal** (modifier une API de lib et adapter toutes les apps dans le même PR).

## 1.3 Risques et limites
- **Coût initial** d’organisation et de formation.
- **Bruit** si pas de règles (tout le monde modifie tout).
- **CI lourde** si on rebuild/test tout à chaque commit.

➡️ D’où l’intérêt d’outils comme **Nx** qui améliorent :
- l’organisation (définition de projets),
- le caching,
- les dépendances entre projets,
- l’exécution ciblée des tâches.

## 1.4 Gouvernance (bonnes pratiques)
- Définir des **domaines** (ex: `customer`, `billing`, `shared`).
- Définir des **types de libs** (ui, data-access, feature, util).
- Mettre en place un **ownership** (CODEOWNERS, reviewers obligatoires).
- Documenter les conventions (README racine + docs d’architecture).

---

# 2) Découverte de Nx : concepts, architecture et workspace

## 2.1 Qu’est-ce que Nx ?
**Nx** est un ensemble d’outils (CLI, plugins, cache, graph) conçu pour gérer des monorepos (mais aussi des repos “single project”), notamment autour de JavaScript/TypeScript, Angular, React, Node.

## 2.2 Concepts clés
- **Project** : une application ou une bibliothèque détectée par Nx.
- **Target** : une tâche exécutable (ex: `build`, `test`, `lint`, `e2e`).
- **Executor** : implémentation de la tâche (Angular builder, Jest, ESLint...).
- **Project Graph** : graphe de dépendances entre projets.
- **Task Graph** : graphe réel des tâches à exécuter pour un target donné.
- **Cache** : Nx réutilise des résultats de tâches qui n’ont pas besoin d’être recalculées.
- **Affected** : exécute uniquement sur les projets impactés par des changements.

## 2.3 Structure d’un workspace Nx (vue d’ensemble)
- `apps/` : applications (front, back, outils)
- `libs/` : bibliothèques réutilisables
- `nx.json`, `project.json` : configuration Nx
- `tsconfig.base.json` : configuration TypeScript partagée

> Remarque : selon les versions, Nx peut aussi proposer une structure `packages/` (ou `apps`/`libs`).

---

# 3) Créer un monorepo Angular avec Nx

## 3.1 Initialisation
Deux approches courantes :

### Option A — Créer un nouveau workspace Nx
```bash
npx create-nx-workspace@latest my-org
cd my-org
```
Choisir :
- **Integrated Monorepo** (recommandé pour un layout apps/libs)
- Ajouter Angular plugin si demandé

### Option B — Ajouter Nx à un projet existant
```bash
npx nx@latest init
```

## 3.2 Ajouter une application Angular
Selon la version Nx, l’outil est via `@nx/angular`.

```bash
nx g @nx/angular:app my-app
```
Cela crée un projet `my-app` (dans `apps/my-app` par défaut) avec des targets :
- `build`, `serve`, `test`, `lint`, potentiellement `e2e`

## 3.3 Lancer l’application
```bash
nx serve my-app
```

## 3.4 Inspection rapide des targets
```bash
nx show project my-app
nx run my-app:build
```

---

# 4) Structurer : apps vs libs, boundaries et architecture modulaire

## 4.1 Apps vs libs : règle générale
- **App** : composition, routing, configuration, layering final.
- **Lib** : code réutilisable (UI, services, domaine, state management, util).

Objectif : **réduire le couplage** et améliorer la réutilisabilité.

## 4.2 Typologies de libs recommandées (style “enterprise”)
Une convention fréquente :

- `feature-*` : pages, flux métier (UI + orchestration)
- `ui-*` : composants de présentation (dumb components)
- `data-access-*` : accès aux données (API, state, facades)
- `util-*` : helpers, fonctions, types
- `domain-*` : modèles, règles métier partagées

Exemple de découpage pour un domaine `customer` :
- `libs/customer/feature-list`
- `libs/customer/feature-detail`
- `libs/customer/ui`
- `libs/customer/data-access`
- `libs/customer/domain`

## 4.3 Générer une lib Angular
```bash
nx g @nx/angular:lib customer-ui
nx g @nx/angular:lib customer-data-access
```

### Standalone components (recommandé si Angular récent)
Nx supporte les projets Angular modernes. Vous pouvez générer des composants standalone :
```bash
nx g @nx/angular:component customer-card --project=customer-ui --standalone
```

## 4.4 Importation via path aliases
Nx configure généralement des alias TypeScript (dans `tsconfig.base.json`).
Exemple :
```ts
import { CustomerCardComponent } from '@my-org/customer-ui';
```

## 4.5 Atelier 1 — Créer un domaine “customer”
1. Créer une app `crm`.
2. Créer les libs `customer-domain`, `customer-data-access`, `customer-ui`, `customer-feature-list`.
3. Afficher une liste mockée dans l’app via la lib `feature-list`.

Livrable : une app qui dépend uniquement d’une ou plusieurs **feature libs**, pas directement de `data-access` ou `domain` (selon vos règles).

---

# 5) Dépendances, project graph et règles d’accès (enforce-module-boundaries)

## 5.1 Visualiser le graphe
```bash
nx graph
```
Le graph permet d’identifier :
- dépendances entre libs/apps,
- cycles potentiels,
- zones d’accumulation de couplage.

## 5.2 Tags et contraintes (gouvernance)
Nx permet de tagger les projets (ex: `scope:customer`, `type:ui`).

### Exemple de tags dans un `project.json`
```json
{
  "name": "customer-ui",
  "tags": ["scope:customer", "type:ui"]
}
```

## 5.3 Enforce module boundaries (ESLint)
Le plugin ESLint Nx permet d’interdire certaines importations (ex: une `ui` ne doit pas dépendre de `data-access`).

Exemple (à adapter) dans `.eslintrc.json` (ou config ESLint équivalente) :
```json
{
  "rules": {
    "@nx/enforce-module-boundaries": [
      "error",
      {
        "enforceBuildableLibDependency": true,
        "allow": [],
        "depConstraints": [
          {
            "sourceTag": "type:feature",
            "onlyDependOnLibsWithTags": ["type:ui", "type:data-access", "type:domain", "type:util"]
          },
          {
            "sourceTag": "type:ui",
            "onlyDependOnLibsWithTags": ["type:util", "type:domain"]
          },
          {
            "sourceTag": "type:data-access",
            "onlyDependOnLibsWithTags": ["type:domain", "type:util"]
          }
        ]
      }
    ]
  }
}
```

## 5.4 Atelier 2 — Mettre des contraintes
1. Tagger les libs : `type:ui`, `type:data-access`, `type:feature`, etc.
2. Ajouter les contraintes ESLint.
3. Provoquer une violation (ex: `ui` important `data-access`) et constater l’erreur.

---

# 6) Tâches Nx : build/test/lint/e2e, exécution ciblée, caching

## 6.1 Exécuter des targets
```bash
nx build my-app
nx test my-app
nx lint my-app
```

## 6.2 Exécuter pour plusieurs projets
Nx offre plusieurs mécanismes :

- `run-many` (explicite)
```bash
nx run-many -t test -p my-app customer-ui customer-data-access
```

- exécution par tag (selon config)
```bash
nx run-many -t lint --all
```

## 6.3 Comprendre le caching Nx
Nx calcule une “empreinte” de la tâche (inputs) :
- fichiers sources,
- config,
- dépendances,
- options.

Si rien n’a changé, Nx récupère le résultat depuis le cache :
- accélère les reruns locaux,
- utile en CI si cache partagé (Nx Cloud ou cache distant).

### Démo rapide
1. Lancer :
```bash
nx test customer-ui
```
2. Relancer immédiatement :
```bash
nx test customer-ui
```
➜ Nx devrait indiquer un résultat depuis le cache (selon configuration).

## 6.4 Inputs/outputs (bases)
Les outputs typiques :
- dist d’une lib/app,
- coverage,
- rapports.

Nx sait quoi mettre en cache selon les executors et config.

---

# 7) Affected & performance : builds incrémentaux, optimisation au quotidien

## 7.1 Pourquoi “Affected” ?
Dans un monorepo, tester/build **tout** à chaque commit est coûteux.
Le mode **Affected** exécute seulement les tâches sur les projets impactés par :
- les fichiers modifiés,
- et leurs dépendants.

## 7.2 Commandes Affected
```bash
nx affected -t lint
nx affected -t test
nx affected -t build
```

### Avec base/head (CI)
```bash
nx affected -t test --base=origin/main --head=HEAD
```

## 7.3 Atelier 3 — Mesurer l’impact
1. Lancer `nx graph`.
2. Modifier un fichier dans `customer-ui`.
3. Vérifier les projets affectés :
```bash
nx affected:graph --base=origin/main --head=HEAD
```
4. Exécuter seulement les tests affectés.

## 7.4 Conseils performance
- Découper en libs cohérentes (éviter des “mega libs”).
- Éviter les dépendances circulaires.
- Garder des frontières strictes (règles Nx ESLint).
- Activer un cache distant en CI.

---

# 8) CI/CD : pipelines, cache distant, qualité et versioning

## 8.1 Pipeline type (concept)
Étapes fréquentes :
- Install
- Lint (affected)
- Test (affected)
- Build (affected)
- Artifacts

## 8.2 Exemple de commandes CI
```bash
nx affected -t lint --base=origin/main --head=HEAD
nx affected -t test --base=origin/main --head=HEAD
nx affected -t build --base=origin/main --head=HEAD
```

## 8.3 Cache distant (optionnel)
Pour partager les résultats entre développeurs/CI, on utilise :
- Nx Cloud (SaaS) ou
- solutions d’auto-hébergement selon la politique interne.

> L’idée : éviter de recalculer des tâches déjà calculées ailleurs.

## 8.4 Qualité et standardisation
- ESLint + règles Nx boundaries
- Tests (unitaires, intégration)
- Conventions (Prettier, commitlint)
- Générateurs Nx internes (schematics) pour imposer une architecture

## 8.5 Versioning des libs internes (stratégies)
- **Versioning unifié** (tous les packages versionnés ensemble)
- **Versioning indépendant** (selon lib)
- Publication (si nécessaire) vers un registry interne

> Dans beaucoup d’entreprises, les libs restent dans le monorepo sans publication externe, mais d’autres publient des libs partagées.

---

# 9) Atelier final — Cas “entreprise” : CRM + UI Kit + Data Access

## 9.1 Énoncé
Construire un monorepo Angular où :
- une app `crm` consomme des features,
- un UI kit est partagé (`shared-ui`),
- l’accès aux données est centralisé (`shared-data-access`),
- des règles empêchent `ui` d’importer `data-access`.

## 9.2 Étapes guidées
1. Créer le workspace Nx.
2. Générer l’app `crm`.
3. Générer libs :
   - `shared-ui`
   - `shared-util`
   - `customer-domain`
   - `customer-data-access`
   - `customer-feature-list`
4. Implémenter :
   - `customer-domain` : interface `Customer`.
   - `customer-data-access` : service `CustomerApi` (mock ou HttpClient).
   - `customer-feature-list` : composant page qui utilise `CustomerApi`.
   - `shared-ui` : composant `Card` générique.
5. Brancher la page dans `crm` (routing + affichage).
6. Ajouter tags et règles `enforce-module-boundaries`.
7. Vérifier `nx graph`.
8. Tester `nx affected -t test` après un changement.

## 9.3 Critères de réussite
- Le routage de l’app ne dépend que des libs `feature-*`.
- Les composants UI n’importent pas directement `data-access`.
- `nx affected` n’exécute que les projets impactés.
- Les tâches répétées sont servies par le cache.

---

# Annexes

## A) Commandes utiles (cheat sheet)
```bash
# Infos sur un projet
nx show project <name>

# Lister les projets
nx show projects

# Graphe
nx graph

# Exécuter une target
nx run <project>:<target>

# Plusieurs projets
nx run-many -t <target> -p <p1> <p2>

# Affected
nx affected -t <target> --base=origin/main --head=HEAD
```

## B) Recommandations d’architecture (résumé)
- Apps = composition, libs = réutilisable.
- Découper par **domaines** et **types**.
- Mettre des **règles** (tags + enforce-module-boundaries).
- Utiliser **Affected + cache** pour accélérer CI et dev.

## C) Questions de validation (quiz)
1. Quelle différence entre project graph et task graph ?
2. Pourquoi un monorepo aide-t-il la gouvernance en entreprise ?
3. Que fait Nx “Affected” et sur quoi se base-t-il ?
4. Comment empêcher une lib `ui` d’importer une lib `data-access` ?
5. Quel est l’intérêt d’un cache distant en CI ?
