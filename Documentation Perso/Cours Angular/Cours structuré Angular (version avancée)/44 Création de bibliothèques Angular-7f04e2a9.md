# Formation Angular — Création de bibliothèques Angular

> **Objectif général** : savoir concevoir, développer, packager, versionner, documenter et publier des **bibliothèques Angular** (components/directives/pipes/services) réutilisables dans plusieurs applications, avec une attention particulière à l’**API publique**, aux **dépendances**, au **versioning** et à la **documentation**.

---

## Informations pratiques

- **Public** : développeurs Angular (intermédiaire → avancé), formateurs, équipes front.
- **Pré-requis** :
  - Connaissance d’Angular (modules, composants, services, DI, RxJS de base)
  - Connaissance de npm et git
  - Node.js + Angular CLI
- **Durée conseillée** : 1 à 2 jours (adaptable)
- **Modalité** : théorie + démonstrations + ateliers guidés

---

## Compétences visées

À la fin de la formation, vous saurez :

1. Créer une lib avec **Angular CLI** et **ng-packagr**.
2. Concevoir une **API publique stable** via `public-api.ts`.
3. Structurer une lib (modules, standalone, entry points, assets).
4. Gérer correctement les **dépendances** (`dependencies`, `peerDependencies`, `optionalDependencies`).
5. Mettre en place un **versioning** (SemVer), changelog et politiques de breaking changes.
6. Tester (unit, intégration) et assurer la qualité (lint, build, CI).
7. Documenter (README, API docs, exemples) et publier (registry privé/public).

---

# Plan de formation

1. **Introduction aux bibliothèques Angular**
2. **Création d’une librairie avec Angular CLI**
3. **Conception de l’API publique (public-api) & bonnes pratiques**
4. **Organisation interne : modules, standalone, entry points**
5. **Gestion des dépendances & packaging**
6. **Théming, styles, assets, i18n (selon besoin)**
7. **Tests, QA, CI/CD**
8. **Versioning, release, compatibilité Angular**
9. **Documentation & adoption : DX pour les consommateurs**
10. **Publication & consommation (npm, registries privés, monorepos)**

Chaque chapitre contient : objectifs, notions clés, checklists, et ateliers.

---

# 1) Introduction aux bibliothèques Angular

## 1.1 Pourquoi créer une librairie ?

- **Réutilisation** : composants UI, directives, pipes, services, intercepteurs HTTP.
- **Standardisation** : design system, patterns, règles de validation.
- **Productivité** : éviter la duplication de code entre plusieurs apps.
- **Qualité** : tests centralisés, CI, release contrôlée.

## 1.2 Librairie vs application

- Une **application** Angular a un `main.ts` et bootstrappe un root component.
- Une **librairie** Angular expose du code **consommable** : aucun bootstrap.
- Une lib doit être **compilée** et **packagée** pour être importable.

## 1.3 Concepts à maîtriser

- **Public API** : ce qui est importable depuis `@scope/lib`.
- **Encapsulation** : tout ce qui n’est pas dans l’API publique est interne.
- **SemVer** : versioning et stabilité.
- **Dépendances** : éviter de « capturer » Angular dans `dependencies`.

## 1.4 Typologies de bibliothèques

- **UI Components** (design system)
- **Core Services** (auth, config, telemetry)
- **Utilities** (pipes, directives, helpers)
- **Feature libraries** (domain-specific)

### Checklist — quand créer une lib ?

- [ ] Deux apps ou plus partagent le même code
- [ ] Besoin d’une API claire et stable
- [ ] On veut versionner et publier indépendamment
- [ ] On veut des tests et documentation centralisés

---

# 2) Création d’une librairie avec Angular CLI

## 2.1 Initialisation d’un workspace

### Option A — workspace avec application + lib

```bash
ng new demo-workspace --create-application=true
cd demo-workspace
ng generate library ui-kit
```

### Option B — workspace « libraries only »

```bash
ng new demo-libs --create-application=false
cd demo-libs
ng generate library ui-kit
```

> Recommandation : avoir au moins **une app de démonstration** pour tester la lib rapidement.

## 2.2 Structure générée

Angular crée typiquement :

- `projects/ui-kit/` : code source de la lib
- `projects/ui-kit/src/public-api.ts` : API publique
- `projects/ui-kit/ng-package.json` : config de packaging
- `dist/ui-kit/` : sortie build (après compilation)

## 2.3 Build de la librairie

```bash
ng build ui-kit
```

Résultat : `dist/ui-kit` contient un package consommable.

## 2.4 Utiliser la lib dans une app du workspace

```bash
ng serve
```

Dans l’app, import possible :

```ts
import { UiKitModule } from 'ui-kit';
```

> Dans un workspace, Angular configure les chemins TypeScript pour permettre l’import avant publication.

---

# 3) Conception de l’API publique & bonnes pratiques

## 3.1 Le fichier `public-api.ts`

Rôle : **contrôler précisément** ce que les consommateurs peuvent importer.

Exemple :

```ts
/*
 * Public API Surface of ui-kit
 */

export * from './lib/button/button.component';
export * from './lib/button/button.module';
export * from './lib/ui-kit.module';
```

### Règles importantes

1. Ne pas exposer tout `./lib/*` automatiquement.
2. Exporter explicitement : modules, components, services, types.
3. Protéger la liberté de refactor interne : les chemins internes ne doivent pas être utilisés par les apps.

> Objectif : empêcher les imports du type `ui-kit/lib/internal/foo` (anti-pattern).

## 3.2 API Design : principes

- **Cohérence** : naming, suffixes (`Component`, `Directive`), conventions.
- **Stabilité** : éviter de casser l’API sans major version.
- **Simplicité** : une consommation intuitive.
- **Extensibilité** : prévoir les cas d’usage (inputs/outputs, injection tokens).

## 3.3 Encapsulation via patterns Angular

### Pattern : configuration via `forRoot()` (modules)

```ts
export interface UiKitConfig {
  defaultLocale?: string;
}

export const UI_KIT_CONFIG = new InjectionToken<UiKitConfig>('UI_KIT_CONFIG');

@NgModule({
  // declarations/exports
})
export class UiKitModule {
  static forRoot(config: UiKitConfig): ModuleWithProviders<UiKitModule> {
    return {
      ngModule: UiKitModule,
      providers: [{ provide: UI_KIT_CONFIG, useValue: config }],
    };
  }
}
```

### Pattern : standalone + providers

Pour du Angular récent, on peut préférer des **standalone components** et une fonction `provideX()` :

```ts
export function provideUiKit(config: UiKitConfig) {
  return [
    { provide: UI_KIT_CONFIG, useValue: config },
  ];
}
```

## 3.4 Politique de dépréciation

Avant de supprimer un élément public :

- Le marquer `@deprecated` (JSDoc)
- Documenter la migration
- Garder au moins une version mineure

Exemple :

```ts
/** @deprecated Use NewButtonComponent instead. */
export class OldButtonComponent {}
```

---

# 4) Organisation interne : modules, standalone, entry points

## 4.1 Structuration recommandée

Exemple simple :

```
projects/ui-kit/src/lib/
  button/
    button.component.ts
    button.component.html
    button.component.scss
    button.module.ts
  form/
  layout/
  ui-kit.module.ts
```

Idées clés :

- Regrouper par feature/composant
- Limiter les dépendances croisées internes
- Centraliser les exports dans `public-api.ts`

## 4.2 Module “barrel” vs exports ciblés

- **Barrel module** (`UiKitModule`) : simple pour les consommateurs
- **Exports ciblés** : permet tree-shaking plus fin dans certains cas

Recommandation : proposer les deux si la lib est large :

- `UiKitModule` (usage rapide)
- Exports par composant (usage granulaire)

## 4.3 Secondary entry points (entry points secondaires)

Utile pour :

- Séparer `@scope/ui-kit/forms`, `@scope/ui-kit/layout`…
- Réduire le coût d’import
- Segmenter l’API publique

Principe : créer un sous-dossier avec `ng-package.json`. Exemple :

```
projects/ui-kit/src/lib/forms/ng-package.json
projects/ui-kit/src/lib/forms/public-api.ts
```

> Selon la complexité, cela améliore l’organisation mais augmente le coût de maintenance.

---

# 5) Gestion des dépendances & packaging

## 5.1 Comprendre `dependencies` vs `peerDependencies`

- `dependencies` : installées automatiquement avec votre package.
- `peerDependencies` : **attendues chez le consommateur** (pas installées par défaut).

### Règle d’or en Angular

- `@angular/*`, `rxjs`, `zone.js` → généralement en **peerDependencies**

Pourquoi ?

- Éviter plusieurs versions d’Angular dans la même app
- Laisser l’app contrôler les versions compatibles

## 5.2 Exemple de `package.json` (lib)

Dans `dist/ui-kit/package.json` (généré), on doit viser :

```json
{
  "name": "@acme/ui-kit",
  "version": "1.2.0",
  "peerDependencies": {
    "@angular/common": ">=17 <20",
    "@angular/core": ">=17 <20",
    "rxjs": ">=7 <9"
  }
}
```

> Les ranges dépendent de votre politique de compatibilité.

## 5.3 `ng-package.json`

Fichier clé pour `ng-packagr`.

Exemple :

```json
{
  "$schema": "../../node_modules/ng-packagr/ng-package.schema.json",
  "dest": "../../dist/ui-kit",
  "lib": {
    "entryFile": "src/public-api.ts"
  }
}
```

## 5.4 Publier du code propre : side effects & tree-shaking

- Éviter du code exécuté à l’import
- Préférer la configuration par providers
- Garder les modules « minces »

### Éviter

```ts
// Anti-pattern: exécute du code au chargement du module
console.log('ui-kit loaded');
```

---

# 6) Styles, assets, théming, i18n (selon besoin)

## 6.1 Stratégies de styles

- Styles encapsulés au composant (`styleUrls`)
- Styles globaux optionnels livrés par la lib
- Variables CSS / design tokens

### Recommandation théming

- Exposer des **CSS variables** : facilement surchargeables

Exemple :

```css
:host {
  --ui-primary: #3f51b5;
}

.button {
  background: var(--ui-primary);
}
```

## 6.2 Assets (icônes, images)

Approches :

- livrer des SVG inline (via composants)
- exposer un chemin d’assets (nécessite config côté app)

Toujours documenter :

- où placer les assets
- comment configurer `angular.json` (si nécessaire)

## 6.3 i18n

- Si composants de lib gèrent des labels :
  - injection d’un service de traduction
  - ou `InjectionToken` pour textes

---

# 7) Tests, QA, CI/CD

## 7.1 Tests unitaires

Tester :

- composants (inputs/outputs)
- directives (interaction DOM)
- pipes (pure functions)
- services (DI, HttpTestingController)

Commandes :

```bash
ng test ui-kit
ng test
```

## 7.2 Tests d’intégration via application de démo

Créer une app `demo` (ou utiliser une existante) :

- importer la lib
- écrire quelques pages de showcase
- vérifier les scénarios critiques

Cette app est un excellent support de documentation vivante.

## 7.3 Lint, format, qualité

- ESLint (règles Angular)
- Prettier (optionnel)
- tests en CI
- build de la lib en CI (assure que le packaging est OK)

## 7.4 Pipeline CI “minimum viable”

Étapes :

1. `npm ci`
2. `npm run lint`
3. `npm test -- --watch=false`
4. `ng build ui-kit`

---

# 8) Versioning, release, compatibilité Angular

## 8.1 SemVer (Semantic Versioning)

- **MAJOR** : breaking changes (API publique cassée)
- **MINOR** : nouvelles features compatibles
- **PATCH** : corrections sans impact

Exemples :

- `1.3.0` : ajout d’un composant
- `1.3.1` : fix CSS
- `2.0.0` : rename d’input public, suppression d’exports

## 8.2 Définir la compatibilité Angular

Définir les ranges peerDeps:

- Si vous suivez le rythme Angular :
  - mettre à jour à chaque major Angular
- Ou maintenir une compatibilité multi-majors si possible

Mettre dans README :

| Version lib | Angular supporté |
|------------|------------------|
| 1.x        | 17–18            |
| 2.x        | 19               |

## 8.3 Changelog & releases

Recommandations :

- `CHANGELOG.md` avec sections : Added/Changed/Deprecated/Removed/Fixed
- Conventional Commits (optionnel)
- Tag git : `v1.2.0`

---

# 9) Documentation & adoption : DX pour les consommateurs

## 9.1 README de qualité

Doit contenir :

- Installation
- Compatibilité Angular
- Getting started
- Exemples (copy/paste)
- API : Inputs/Outputs, services, tokens
- FAQ / Troubleshooting
- Guide de contribution

## 9.2 Exemples d’utilisation

### Installation

```bash
npm i @acme/ui-kit
```

### Import module (approche module)

```ts
import { UiKitModule } from '@acme/ui-kit';

@NgModule({
  imports: [UiKitModule]
})
export class AppModule {}
```

### Exemple composant

```html
<acme-button variant="primary" (clicked)="save()">
  Enregistrer
</acme-button>
```

## 9.3 Documentation API générée (optionnel)

- Compodoc (génère doc des composants/services)
- Storybook (showcase UI, interactions)

> Pour UI libraries, **Storybook** est souvent le standard.

---

# 10) Publication & consommation

## 10.1 Publication npm (public)

Dans `dist/ui-kit` :

```bash
cd dist/ui-kit
npm publish --access public
```

Si package scoped privé :

```bash
npm publish
```

## 10.2 Registry privé (GitHub Packages, GitLab, Nexus)

Points d’attention :

- `.npmrc` (auth)
- scopes (`@acme/*`)
- droits d’accès
- cache CI

## 10.3 Consommation dans une application externe

```bash
npm i @acme/ui-kit
```

Puis utiliser selon docs.

## 10.4 Monorepo vs multirepo

- Monorepo : Nx, Angular workspace, changes groupés, tooling.
- Multirepo : releases indépendantes, mais coordination plus difficile.

---

# Ateliers (guidés)

## Atelier 1 — Créer une librairie `ui-kit` et un composant button

### Objectif

- Générer une lib
- Créer un composant
- L’exposer via `public-api.ts`
- Le consommer dans une app

### Étapes

```bash
ng new acme-workspace
cd acme-workspace
ng g library ui-kit
ng g component button --project ui-kit
ng build ui-kit
```

Dans `projects/ui-kit/src/public-api.ts` :

```ts
export * from './lib/button/button.component';
```

Consommer dans `app.component.html` :

```html
<lib-button></lib-button>
```

> Selon le selector généré, adapter (`lib-button` par défaut).

---

## Atelier 2 — Définir une API stable (Inputs/Outputs)

### Objectif

- Concevoir des inputs/outputs cohérents

Exemple :

```ts
@Component({
  selector: 'acme-button',
  template: `<button (click)="clicked.emit()"><ng-content /></button>`
})
export class ButtonComponent {
  @Input() variant: 'primary' | 'secondary' = 'primary';
  @Output() clicked = new EventEmitter<void>();
}
```

Points à valider :

- types stricts
- valeurs par défaut
- noms d’événements : verbe au passé (`clicked`, `changed`) ou intent (`submit`)

---

## Atelier 3 — Gérer `peerDependencies` et publier une version

### Objectif

- Vérifier `peerDependencies`
- Construire et simuler une publication

Étapes :

1. lancer `ng build ui-kit`
2. inspecter `dist/ui-kit/package.json`
3. ajuster stratégie de version
4. `npm pack` (crée un tarball local)

```bash
cd dist/ui-kit
npm pack
```

Puis installer le `.tgz` dans une app de test :

```bash
npm i ../dist/ui-kit/acme-ui-kit-1.0.0.tgz
```

---

## Atelier 4 — Documentation minimale (README)

### Objectif

- Rédiger une doc exploitable

Template README conseillé :

```md
# @acme/ui-kit

## Installation
npm i @acme/ui-kit

## Usage
...

## Compatibility
...
```

---

# Annexes

## A) Checklist “Ready to publish”

- [ ] API publique claire (`public-api.ts`)
- [ ] Pas d’imports internes utilisés par les apps
- [ ] `peerDependencies` correctement définies
- [ ] Build OK (`ng build lib`)
- [ ] Tests OK
- [ ] README + exemples
- [ ] Version bump (SemVer)
- [ ] Changelog mis à jour

## B) Anti-patterns fréquents

- Exposer des chemins internes
- Mettre `@angular/core` dans `dependencies`
- Modifier des signatures publiques sans major
- Absence de doc et d’exemples
- Surcouplage de la lib avec l’app consommatrice

## C) Glossaire

- **Public API** : surface d’import officielle d’une lib.
- **Peer dependency** : dépendance fournie par l’hôte.
- **Tree-shaking** : élimination du code mort au build.
- **Secondary entry point** : point d’entrée additionnel d’un package.

---

## Conclusion

Une bonne bibliothèque Angular n’est pas seulement du code réutilisable : c’est un **produit** destiné à d’autres développeurs. La réussite dépend surtout de la **qualité de l’API publique**, d’une **stratégie de versioning**, d’une **gestion saine des dépendances**, d’un **packaging fiable** et d’une **documentation orientée usage**.
