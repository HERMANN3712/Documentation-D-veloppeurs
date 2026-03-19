# Formation Angular – Lazy loading et code splitting

**Public cible :** développeurs Angular (intermédiaire)

**Prérequis :** TypeScript, notions de routing Angular, architecture d’application (features), build Angular CLI.

**Durée recommandée :** 3h à 1 journée (selon profondeur des ateliers)

**Objectifs pédagogiques :**
- Comprendre le *lazy loading* et le *code splitting* et leur impact sur la performance.
- Mettre en place le lazy loading avec Angular moderne (standalone routes/components, `loadChildren`, `loadComponent`).
- Découper intelligemment une application en *features* et optimiser les bundles.
- Mesurer et vérifier les gains (budgets, source-map-analyzer, Web Vitals).
- Éviter les pièges (préchargement mal configuré, imports shared, guards/resolvers, SSR, erreurs de routing).

---

## Plan détaillé

1. **Introduction : performances, bundles et coût du JavaScript**
2. **Rappels : bundling, chunks, tree-shaking et notion de route**
3. **Lazy loading dans Angular : concepts et stratégies**
4. **Lazy loading avec le Router : `loadChildren` (features) et `loadComponent` (standalone)**
5. **Découpage en features : architecture recommandée**
6. **Préchargement (Preloading) : quand et comment**
7. **Code splitting au-delà du routing : imports dynamiques et micro-features**
8. **Mesure et validation : analyser les chunks et fixer des budgets**
9. **Erreurs fréquentes et anti-patterns**
10. **Atelier guidé : refactor d’une app vers standalone + lazy routes**
11. **Checklist de mise en production**

---

## 1) Introduction : performances, bundles et coût du JavaScript

### Pourquoi le lazy loading ?
Dans une SPA, l’application est souvent livrée sous forme de *bundles* (fichiers JavaScript). Si le bundle initial est volumineux :
- le téléchargement est plus long (réseau),
- l’exécution et le parsing JS sont plus coûteux (CPU),
- le *Time to Interactive (TTI)* et le *First Input Delay (FID/INP)* peuvent se dégrader.

Le **lazy loading** consiste à **charger une partie du code uniquement quand l’utilisateur en a besoin**. Avec Angular, cela se traduit généralement par :
- Chargement à la demande de **features** (ensembles de routes, composants, services) via le Router.
- Possibilité de charger un **composant** seul via `loadComponent`.

### Différence entre lazy loading et code splitting
- **Code splitting** : découpage du code en **chunks** séparés (plusieurs fichiers). C’est une capacité du bundler (Webpack / esbuild selon config Angular).
- **Lazy loading** : stratégie de chargement où certains chunks **ne sont pas téléchargés au démarrage**.

Angular moderne permet d’aligner architecture et performance grâce aux **routes standalone** et aux **features modulaires**.

---

## 2) Rappels : bundling, chunks, tree-shaking et notion de route

### Bundles et chunks
Lors du build, l’outil de bundling :
- regroupe et optimise le code,
- produit un ensemble de fichiers : `main.*.js`, `polyfills.*.js`, `styles.*.css`, et des **chunks** additionnels.

Un chunk peut être :
- **initial** (chargé au démarrage),
- **lazy** (chargé sur demande).

### Tree-shaking
Le *tree-shaking* élimine le code non utilisé. Il fonctionne mieux lorsque :
- les imports sont statiques et ESM-friendly,
- il n’y a pas d’effets de bord inutiles.

Le lazy loading complète le tree-shaking : même si une feature est utilisée (donc non “shakable”), elle peut être **retardée**.

---

## 3) Lazy loading dans Angular : concepts et stratégies

### Ce qu’on “lazy-load” typiquement
- Pages d’administration (accessibles à une minorité d’utilisateurs)
- Écrans rarement visités (settings, historique complet, etc.)
- Éditeurs lourds (Monaco, charts, WYSIWYG)
- Exports, assistants (wizards), vues avancées

### Le router comme déclencheur
Le déclencheur le plus naturel est la navigation :
- l’utilisateur va sur `/admin` → Angular télécharge le chunk admin.

### Standalone vs Modules
- **Standalone (recommandé aujourd’hui)** : routes et composants chargés via `loadComponent` / `loadChildren` vers un fichier de routes.
- **NgModules (legacy / existant)** : lazy loading via `loadChildren: () => import('...').then(m => m.FeatureModule)`.

Les deux sont valables selon contexte. L’objectif est de **réduire l’initial bundle**.

---

## 4) Lazy loading avec le Router

Nous allons couvrir deux scénarios principaux :
1. Lazy loading d’une *feature* (ensemble de routes)
2. Lazy loading d’un composant standalone

### 4.1 Lazy loading d’une feature via `loadChildren` (routes)

#### Exemple – structure de fichiers
```
src/app/
  app.routes.ts
  features/
    admin/
      admin.routes.ts
      pages/
        admin-home.page.ts
        users.page.ts
```

#### `app.routes.ts`
```ts
import { Routes } from '@angular/router';

export const routes: Routes = [
  {
    path: 'admin',
    loadChildren: () => import('./features/admin/admin.routes')
      .then(m => m.ADMIN_ROUTES),
  },
  {
    path: '',
    pathMatch: 'full',
    redirectTo: 'home',
  },
  {
    path: '**',
    loadComponent: () => import('./pages/not-found.page')
      .then(m => m.NotFoundPage),
  },
];
```

#### `admin.routes.ts`
```ts
import { Routes } from '@angular/router';

export const ADMIN_ROUTES: Routes = [
  {
    path: '',
    loadComponent: () => import('./pages/admin-home.page')
      .then(m => m.AdminHomePage),
  },
  {
    path: 'users',
    loadComponent: () => import('./pages/users.page')
      .then(m => m.UsersPage),
  },
];
```

**Ce que ça fait :**
- Le chunk `admin` n’est pas téléchargé au démarrage.
- Il est téléchargé à la première navigation vers `/admin`.

#### Bonnes pratiques
- Mettre une feature dans un dossier dédié (`features/admin`).
- Éviter d’importer depuis une feature lazy dans du code initial (sinon le bundler peut remonter des dépendances et casser l’isolation).

### 4.2 Lazy loading d’un composant via `loadComponent`

Utile si :
- vous avez une route simple,
- une page isolée,
- ou un écran expérimental.

```ts
export const routes: Routes = [
  {
    path: 'settings',
    loadComponent: () => import('./pages/settings.page')
      .then(m => m.SettingsPage),
  },
];
```

### 4.3 Guards et resolvers en lazy

**Point important :** guards/resolvers peuvent impacter le moment de chargement.

Exemple :
```ts
{
  path: 'admin',
  canMatch: [() => /* logique auth */ true],
  loadChildren: () => import('./features/admin/admin.routes')
    .then(m => m.ADMIN_ROUTES),
}
```

- `canMatch` est utile pour **éviter même le chargement** du chunk si la route n’est pas accessible.
- `canActivate` peut déclencher un chargement selon organisation des imports.

### 4.4 Erreurs courantes de configuration
- `pathMatch` mal positionné sur la route vide.
- `redirectTo` dans une feature qui crée une boucle.
- Importer un composant lazy dans un template initial (ex : via un import direct).

---

## 5) Découpage en features : architecture recommandée

### Principe
Une **feature lazy** doit être :
- **cohésive** : elle regroupe une fonctionnalité (admin, billing, onboarding)
- **autonome** : dépendances internes, routes internes
- **faiblement couplée** au reste

### Structure type
```
app/
  core/            # services singleton, interceptors, config globale
  shared/          # composants/pipes réutilisables, sans état
  features/
    catalog/
    checkout/
    admin/
```

### Core vs Shared (très important pour le lazy loading)
- `core` : importé **une seule fois** (souvent initial). Contient AuthService, Http interceptors, config.
- `shared` : UI réutilisable (boutons, pipes) que les features peuvent importer.

Attention :
- si `shared` devient énorme, il gonfle les bundles (et est souvent initial). Il peut être utile de scinder `shared` en sous-parties (ex: `shared/ui`, `shared/forms`, `shared/charts`).

---

## 6) Préchargement (Preloading) : quand et comment

Le preloading télécharge certains chunks **après** le chargement initial (quand le navigateur est plus “disponible”).

### Stratégies intégrées
- `NoPreloading` : rien n’est préchargé.
- `PreloadAllModules` : toutes les routes lazy sont préchargées.

Avec standalone API :
```ts
import { provideRouter, withPreloading, PreloadAllModules } from '@angular/router';

bootstrapApplication(AppComponent, {
  providers: [
    provideRouter(routes, withPreloading(PreloadAllModules)),
  ],
});
```

### Stratégie sur mesure (preload conditionnel)
Objectif : précharger seulement certaines features (ex : “catalog”), pas “admin”.

1) Marquer la route :
```ts
{
  path: 'catalog',
  data: { preload: true },
  loadChildren: () => import('./features/catalog/catalog.routes')
    .then(m => m.CATALOG_ROUTES),
}
```

2) Implémenter une stratégie :
```ts
import { PreloadingStrategy, Route } from '@angular/router';
import { Observable, of } from 'rxjs';

export class SelectivePreloadingStrategy implements PreloadingStrategy {
  preload(route: Route, load: () => Observable<unknown>): Observable<unknown> {
    return route.data?.['preload'] ? load() : of(null);
  }
}
```

3) Enregistrer :
```ts
import { withPreloading } from '@angular/router';

provideRouter(routes, withPreloading(SelectivePreloadingStrategy))
```

**Bon usage :**
- Précharger les features très probables (ex : parcours principal)
- Laisser en lazy pur les features rares (admin)

---

## 7) Code splitting au-delà du routing

Le routing est la source principale de lazy loading, mais on peut aussi découper via **imports dynamiques**.

### 7.1 Import dynamique d’une librairie lourde
Exemple : charger `chart.js` uniquement quand on affiche un dashboard.

```ts
async loadCharts() {
  const { Chart } = await import('chart.js/auto');
  // utiliser Chart
}
```

Avantages :
- évite de mettre la lib dans le bundle initial.

### 7.2 Feature toggles et expérimentation
Charger une feature expérimentale uniquement si un flag est actif :
```ts
if (this.flags.isEnabled('new-editor')) {
  const { NewEditorComponent } = await import('./new-editor/new-editor.component');
  // créer dynamiquement / router vers une route dédiée
}
```

### 7.3 Attention aux “barrels” (`index.ts`)
Un `index.ts` qui réexporte trop peut :
- rendre les dépendances moins visibles,
- provoquer l’inclusion de code non désiré.

Recommandation : importer les symboles depuis leurs fichiers ciblés, surtout entre initial et lazy.

---

## 8) Mesure et validation : analyser les chunks et fixer des budgets

### 8.1 Build en production et stats
Commandes utiles :
```bash
ng build -c production
```

Selon version/config, vous pouvez générer des stats Webpack (si applicable) ou utiliser des analyseurs compatibles.

### 8.2 Vérifier le découpage
Ce que vous cherchez :
- un `main` plus petit,
- des chunks `lazy` par feature.

### 8.3 Budgets (angular.json)
Les budgets déclenchent des warnings/erreurs si un bundle dépasse un seuil.

Exemple :
```json
{
  "budgets": [
    {
      "type": "initial",
      "maximumWarning": "300kb",
      "maximumError": "500kb"
    },
    {
      "type": "anyComponentStyle",
      "maximumWarning": "6kb",
      "maximumError": "10kb"
    }
  ]
}
```

### 8.4 Indicateurs UX
- LCP (Largest Contentful Paint)
- INP (Interaction to Next Paint)
- CLS (Cumulative Layout Shift)

Le lazy loading aide surtout sur :
- temps de chargement initial,
- réactivité au démarrage.

---

## 9) Erreurs fréquentes et anti-patterns

### 9.1 Mettre trop dans le bundle initial
- `shared` géant importé partout.
- Librairies lourdes (éditeur, charting) importées globalement.

**Correctif :** import dynamique + modules UI plus petits.

### 9.2 Couplage entre initial et lazy
Si le code initial importe un symbole d’une feature lazy, le bundler peut l’inclure dans l’initial chunk.

**Règle :**
- Le code initial (core/app) ne doit pas importer directement une feature lazy.

### 9.3 Précharger tout sans réfléchir
`PreloadAllModules` peut annuler une partie du gain réseau, surtout sur mobile.

**Réflexe :** utiliser une stratégie sélective.

### 9.4 Résolveurs trop lourds
Un resolver qui déclenche des appels HTTP ou parse beaucoup de données peut rendre la navigation vers un chunk lazy “lente”.

**Solution :**
- minimiser le resolver,
- afficher un skeleton,
- charger le reste après le rendu.

### 9.5 SSR / Hydration (si applicable)
- En SSR, le lazy loading se comporte différemment (le serveur peut rendre la route demandée).
- Sur le client, les chunks doivent quand même être disponibles et correctement préchargés selon cas.

---

## 10) Atelier guidé : refactor d’une app vers lazy routes (standalone)

### Objectif de l’atelier
Transformer une application qui a toutes ses pages dans le bundle initial en une app :
- avec 2 features lazy (`admin`, `reports`)
- route `settings` lazy via `loadComponent`
- préchargement sélectif de `reports`

### Étapes

#### Étape A — Identifier les pages candidates
- `AdminPage`, `UserManagementPage` → feature `admin`
- `ReportsPage`, `ReportDetailsPage` → feature `reports`
- `SettingsPage` → `loadComponent`

#### Étape B — Créer les fichiers de routes de features
Créer :
- `features/admin/admin.routes.ts`
- `features/reports/reports.routes.ts`

#### Étape C — Câbler `app.routes.ts`
- Remplacer les imports directs de pages par `loadChildren` / `loadComponent`.

#### Étape D — Mettre en place la stratégie de preloading
- `reports` préchargé
- `admin` non préchargé

#### Étape E — Vérifier
- Build prod
- Vérifier la présence de chunks additionnels
- Naviguer : constater le téléchargement réseau au moment de la navigation

### Critères de réussite
- `main.*.js` diminue.
- `admin` et `reports` sont des chunks distincts.
- `reports` se précharge après le premier rendu.

---

## 11) Checklist de mise en production

- [ ] Les routes principales sont en initial, les secondaires en lazy.
- [ ] Les features rares (admin) ne sont pas préchargées.
- [ ] Les librairies lourdes sont importées dynamiquement.
- [ ] Les budgets `initial` sont définis et respectés.
- [ ] Les pages lazy ont un *loading state* (spinner/skeleton) pour la navigation.
- [ ] Les guards utilisent `canMatch` si l’accès doit empêcher le chargement.
- [ ] Aucun import direct du lazy dans l’initial.

---

## Annexes

### A) Exemple minimal standalone bootstrap
```ts
import { bootstrapApplication } from '@angular/platform-browser';
import { provideRouter } from '@angular/router';
import { AppComponent } from './app/app.component';
import { routes } from './app/app.routes';

bootstrapApplication(AppComponent, {
  providers: [provideRouter(routes)],
});
```

### B) Glossaire
- **Initial bundle** : code chargé au démarrage.
- **Lazy chunk** : fichier JS chargé ultérieurement.
- **Preloading** : téléchargement anticipé de chunks lazy après le démarrage.
- **Standalone** : composants/routes sans NgModule.

---

## Fin de formation

Livrables recommandés :
- un projet exemple avec 2 features lazy + 1 route `loadComponent`
- un rapport de comparaison de taille des bundles avant/après
