# Formation — Angular Material et Design System

## Informations générales
- **Niveau** : avancé (Angular)
- **Public** : développeurs Angular, référents front, leads techniques, formateurs
- **Pré-requis** :
  - Angular ≥ 16/17 (standalone, router, forms)
  - TypeScript, RxJS
  - Connaissances CSS/SCSS (variables, mixins), notions d’accessibilité
  - Git + notion de monorepo (optionnel mais recommandé)
- **Durée cible** : 2 jours (14 h) — adaptable en 1 jour (7 h) ou 3 jours (21 h)
- **Objectif global** : maîtriser Angular Material **au-delà de l’usage** : personnalisation, theming, intégration à un **design system**, encapsulation dans une **bibliothèque interne**, et **maintien de la cohérence** (tokens, styles, a11y, tests).

---

## Objectifs pédagogiques
À l’issue de la formation, le participant sera capable de :
1. Choisir et configurer Angular Material dans un projet moderne (standalone, build, M3/M2 selon version).
2. Mettre en place une stratégie de **theming** (typo, couleurs, densité) et la faire évoluer.
3. Convertir des décisions de design en **design tokens** (couleurs, typographies, espacements) et les appliquer.
4. Intégrer Angular Material dans un **design system** existant (Figma/Specs) en gardant la cohérence.
5. Encapsuler Angular Material dans une librairie interne (wrapper + API stable) et gérer versioning.
6. Gérer l’accessibilité (ARIA, focus, contrastes) et la validation UX.
7. Industrialiser : tests, CI, documentation (Storybook/Docs), linting, budgets CSS.

---

## Fil rouge (projet pédagogique)
On construit progressivement un mini design system "**Acme DS**" :
- Un thème (clair/sombre) + palettes de marque
- Des composants internes : `acme-button`, `acme-input`, `acme-dialog`, `acme-table`
- Une librairie Angular (monorepo conseillé) avec documentation et tests

**Livrables** :
- Un thème Material personnalisé (M3 ou M2)
- Un jeu de tokens documentés
- Une librairie interne `@acme/ui` encapsulant Material
- Une application de démonstration + documentation

---

## Plan détaillé (2 jours)

### Jour 1 — Maîtrise avancée d’Angular Material et theming
1. **Rappels & architecture Material** (CDK, composants, a11y)
2. **Theming avancé** : palettes, typographies, densité, light/dark
3. **Design tokens** : stratégie, structure, mapping vers Material
4. **Personnalisation** : styles, surcharges, API, slots, composition

### Jour 2 — Design system, encapsulation, gouvernance
5. **Construire une librairie interne** : wrappers, API stable, schematics
6. **Cohérence & gouvernance** : lint, CI, review, versioning, migration
7. **Qualité** : accessibilité, performance, tests unitaires/visuels
8. **Documentation & adoption** : Storybook, guidelines, exemples

---

# Module 1 — Angular Material : composants, CDK et architecture

## 1.1. Angular Material vs CDK
- **Angular Material** : composants UI prêts à l’emploi (boutons, inputs, tables…), alignés Material Design.
- **Angular CDK** : briques bas niveau (overlay, focus, a11y, drag-drop, virtual scroll). Indispensable pour créer des composants sur-mesure.

### Pourquoi c’est critique en contexte design system
- Un design system n’est pas qu’une bibliothèque de composants : c’est **un ensemble de décisions**.
- Le CDK permet de concevoir des composants **cohérents** sans être bloqué par les opinions visuelles de Material.

## 1.2. Choix de version et implications (M2/M3)
Selon versions Angular Material :
- **M2 (Material Design 2)** : theming stable, beaucoup d’exemples historiques.
- **M3 (Material Design 3)** : tokens plus centrés sur « design decisions », dynamique de couleurs, etc.

> Recommandation formation : adopter la stratégie de theming compatible avec votre stack et prévoir une couche de tokens interne pour limiter l’impact des migrations.

## 1.3. Installation et baseline
Commandes (indicatives) :
```bash
ng add @angular/material
```
Bonnes pratiques :
- Activer la typographie Material **uniquement si alignée** avec votre design.
- Centraliser les imports Material via une librairie interne (ou un barrel) pour maîtriser l’usage.

### Structure recommandée
- `apps/demo` : application de démonstration
- `libs/ui` : bibliothèque UI (wrappers)
- `libs/tokens` : tokens et styles partagés
- `libs/theme` : thèmes Material / mapping tokens → Material

---

# Module 2 — Theming avancé : couleurs, typographies, densité, modes

## 2.1. Concepts clefs
- **Palette** : ensemble de teintes (primary/accent/warn ou plus riche en M3).
- **Theme** : regroupement palette + typographie + densité.
- **Density** : compacité (taille des surfaces, paddings, hauteurs).

### Principes
- Thème = **contrat** (API) entre design et dev.
- Éviter les surcharges globales au hasard : préférer `tokens` + `mixins`.

## 2.2. Stratégie : 1 thème Material + tokens internes
Objectif : ne pas coupler toute votre UI aux variables internes Material.

- **Tokens internes** (source de vérité) :
  - couleurs : `--acme-color-brand-500`, `--acme-color-surface`, `--acme-color-on-surface`
  - typo : `--acme-font-family-base`, `--acme-font-size-2`, etc.
  - spacing : `--acme-space-1`, `--acme-space-2`...
  - radius/ elevation

- **Mapping** vers Material, ex : primary = brand.

## 2.3. Mise en place de tokens CSS
Exemple de définition (CSS variables) :
```scss
:root {
  /* Brand */
  --acme-color-brand-50:  #eef3ff;
  --acme-color-brand-100: #d9e2ff;
  --acme-color-brand-500: #3b82f6;
  --acme-color-brand-700: #1d4ed8;

  /* Sémantiques */
  --acme-color-surface: #ffffff;
  --acme-color-on-surface: #111827;
  --acme-color-error: #dc2626;

  /* Typo */
  --acme-font-family-base: ui-sans-serif, system-ui, -apple-system, "Segoe UI", Roboto;
  --acme-font-size-1: 0.875rem;
  --acme-font-size-2: 1rem;
  --acme-font-size-3: 1.125rem;

  /* Spacing */
  --acme-space-1: 4px;
  --acme-space-2: 8px;
  --acme-space-3: 12px;
  --acme-space-4: 16px;

  /* Radius */
  --acme-radius-1: 6px;
  --acme-radius-2: 10px;
}

[data-theme="dark"] {
  --acme-color-surface: #0b1220;
  --acme-color-on-surface: #e5e7eb;
}
```

### Bonnes pratiques tokens
- Nommer en **2 niveaux** :
  1) tokens de base (brand/neutral) 
  2) tokens **sémantiques** (surface/on-surface/primary/on-primary)
- Les composants ne consomment **que** les tokens sémantiques.

## 2.4. Thème Material (approche SCSS)
Selon version Material, la syntaxe varie. Idée générale :
- Définir un thème Material
- Injecter la typography/density
- Appliquer `all-component-themes`.

Pseudo-exemple (M2) :
```scss
@use '@angular/material' as mat;

$primary: mat.define-palette(mat.$indigo-palette, 500); // à mapper vers brand
$accent:  mat.define-palette(mat.$blue-palette,  500);
$warn:    mat.define-palette(mat.$red-palette);

$typography: mat.define-typography-config(
  $font-family: var(--acme-font-family-base),
);

$theme: mat.define-light-theme((
  color: (
    primary: $primary,
    accent: $accent,
    warn: $warn,
  ),
  typography: $typography,
  density: 0,
));

@include mat.core();
@include mat.all-component-themes($theme);
```

> Note : on ne peut pas toujours injecter des CSS variables partout dans les API SCSS Material. D’où l’intérêt d’un **mapping** pragmatique : Material fournit une base, les tokens internes fixent les détails.

## 2.5. Mode clair/sombre
Approche recommandée :
- Un **set de tokens sémantiques** par mode.
- Les composants utilisent ces tokens.
- Le basculement se fait via attribut (`data-theme`) ou classe sur `html/body`.

Exemple de toggle :
```ts
// theme.service.ts
@Injectable({ providedIn: 'root' })
export class ThemeService {
  setDark(isDark: boolean) {
    document.documentElement.toggleAttribute('data-theme');
    if (isDark) document.documentElement.setAttribute('data-theme', 'dark');
    else document.documentElement.removeAttribute('data-theme');
  }
}
```

## 2.6. Densité (density) et responsive
- Densité = "toucher" de l’UI (desktop vs mobile vs data dense).
- Stratégie : créer des niveaux (comfortable/compact) doxés.

Suggestion :
- `data-density="compact"` sur une section/table
- variables CSS pour paddings/hauteurs

---

# Module 3 — Design system : tokens, règles et cohérence

## 3.1. Définir un design system (version dev)
Un design system comprend :
- **Tokens** (couleurs, typo, spacing, radius, elevation)
- **Composants** (API, states, variants)
- **Guidelines** (do/don’t, a11y, responsive)
- **Gouvernance** (versioning, contribution, revue)

Lorsqu’on intègre Material :
- Material apporte des patterns et comportements.
- Votre DS impose le visuel et les variantes.

## 3.2. Cartographie : Material vs DS
Créer une matrice :
| Besoin DS | Matériau existant ? | Décision |
|---|---:|---|
| Button (primary/secondary/ghost) | `mat-button` | Wrapper `acme-button` + variants |
| Input + error + hint | `mat-form-field` | Wrapper `acme-input` + règles |
| Modal | `MatDialog` | Service interne `AcmeDialog` |
| Toast | `MatSnackBar` | Wrapper + design tokens |
| Table data dense | `mat-table` + CDK | Composant interne `acme-table` |

## 3.3. Règles de cohérence
- **Interdire** l’usage direct de composants Material dans les apps (ou le limiter) pour éviter divergence.
- Exposer une API stable via `@acme/ui`.
- Centraliser styles et tokens.

### Techniques concrètes
- ESLint rule (custom) ou convention + code review
- Ports d’entrée : `@acme/ui/button`, `@acme/ui/form`
- Exemples obligatoires + Storybook

---

# Module 4 — Personnaliser Angular Material proprement

## 4.1. Les anti-patterns fréquents
- Surcharger `.mat-mdc-...` partout dans l’app
- Dépendre de classes internes Material (fragiles)
- Copier-coller des snippets sans stratégie

## 4.2. Hiérarchie des solutions (du meilleur au pire)
1. **Utiliser l’API prévue** (inputs, theming, typography, density)
2. **Composer** : wrapper + styles scoping + host bindings
3. **Tokens** : variables CSS sur le host pour contrôler visuel
4. Surcharge globale ciblée (rare, documentée)
5. Hack CSS sur classes internes (à éviter)

## 4.3. Exemple : bouton DS sur base Material
Objectif : `acme-button` avec variants : `primary`, `secondary`, `ghost`.

### API souhaitée
```html
<acme-button variant="primary">Enregistrer</acme-button>
<acme-button variant="secondary">Annuler</acme-button>
<acme-button variant="ghost">Aide</acme-button>
```

### Implémentation (wrapper)
```ts
@Component({
  selector: 'acme-button',
  standalone: true,
  imports: [MatButtonModule],
  template: `
    <button
      mat-button
      [class.acme-primary]="variant === 'primary'"
      [class.acme-secondary]="variant === 'secondary'"
      [class.acme-ghost]="variant === 'ghost'"
      [disabled]="disabled"
      type="button">
      <ng-content />
    </button>
  `,
  styleUrls: ['./acme-button.scss']
})
export class AcmeButtonComponent {
  @Input() variant: 'primary' | 'secondary' | 'ghost' = 'primary';
  @Input() disabled = false;
}
```

### Styles via tokens
```scss
:host button {
  border-radius: var(--acme-radius-2);
  font-family: var(--acme-font-family-base);
}

:host button.acme-primary {
  background: var(--acme-color-brand-500);
  color: white;
}

:host button.acme-secondary {
  background: transparent;
  border: 1px solid var(--acme-color-brand-500);
  color: var(--acme-color-brand-500);
}

:host button.acme-ghost {
  background: transparent;
  color: var(--acme-color-on-surface);
}
```

**Points d’attention** :
- Contrastes (AA/AAA) pour chaque mode
- États hover/active/focus-visible
- Désactivation (`disabled`) cohérente

## 4.4. Exemple : `acme-input` basé sur `mat-form-field`
Objectif : imposer :
- label, hint, erreurs
- tailles (sm/md/lg)
- cohérence des paddings et radius

API :
```html
<acme-input
  label="Email"
  [control]="email"
  placeholder="name@company.com"
  hint="Nous n'envoyons pas de spam" />
```

Idée d’implémentation :
- wrapper qui reçoit `FormControl`
- applique des tokens (density, radius)
- standardise la gestion des erreurs

---

# Module 5 — Encapsuler Material dans une bibliothèque interne (@acme/ui)

## 5.1. Pourquoi encapsuler
- Stabiliser l’API pour les apps
- Réduire l’impact des migrations Material
- Garantir accessibilité + cohérence
- Mutualiser patterns (ex. gestion erreurs forms)

## 5.2. Stratégies d’encapsulation
- **Wrapper components** : `acme-button` encapsule `mat-button`
- **Facades/services** : `AcmeDialogService` encapsule `MatDialog`
- **Directives** : standardiser comportements (ex. `acmeAutofocus`)
- **Theming package** : tokens + thèmes exportés

## 5.3. Architecture de librairie
Recommandations :
- 1 package UI principal : `@acme/ui`
- Sous-modules par domaine :
  - `@acme/ui/button`
  - `@acme/ui/form`
  - `@acme/ui/dialog`

### API publique
- Fichier `public-api.ts` strict
- Pas d’export de dépendances internes non voulues

## 5.4. Versioning et compatibilité
- SemVer
- Politique :
  - MAJOR : breaking API DS
  - MINOR : nouveaux composants/variants
  - PATCH : correctifs

## 5.5. Schematics / generators (option)
- Générer un composant DS (scaffold)
- Ajouter automatiquement tokens/demos/tests

---

# Module 6 — Gouvernance et cohérence visuelle

## 6.1. Définir des règles
- Un composant DS doit :
  - avoir des variants documentés
  - gérer states (hover/active/disabled/focus/error)
  - être responsive si applicable
  - respecter a11y

## 6.2. Contrôler l’usage de Material
- Convention : l’app consomme `@acme/ui`.
- Tolérance : Material direct uniquement dans `@acme/ui`.

Techniques :
- Répertoires d’import (path mappings)
- Revue de code
- Lint (custom rule) : interdire `@angular/material/*` hors `libs/ui`

## 6.3. Design tokens : gouvernance
- Les tokens sont versionnés.
- Les changements de tokens sont relus par design + tech.
- Les tokens sémantiques sont prioritaires.

---

# Module 7 — Qualité : accessibilité, performance, tests

## 7.1. Accessibilité (a11y)
Angular Material aide, mais ne garantit pas tout.

Checklist :
- Focus visible (`:focus-visible`) + parcours clavier
- Labels explicites (form-field)
- Rôles ARIA uniquement si nécessaire
- Contrastes (WCAG)

Outils :
- Axe DevTools
- Lighthouse
- Tests e2e d’accessibilité (Playwright + axe)

## 7.2. Performance
Risques :
- CSS massif (tous les thèmes + tous les composants)
- Imports Material non tree-shakés si mal structurés

Bonnes pratiques :
- Importer uniquement les modules nécessaires
- Split : thème global + styles composants
- Budgets CSS dans Angular

## 7.3. Stratégie de tests
- **Unit tests** : API et comportements wrappers
- **Harnesses Material** : tester les composants Material de manière robuste
- **Visual regression** : Storybook + Chromatic (ou équivalent)

Exemple : utiliser `MatButtonHarness` via Testbed.

---

# Module 8 — Documentation et adoption (Storybook, guidelines)

## 8.1. Documentation produit
Pour chaque composant :
- Usage
- API (inputs/outputs)
- Variants
- Do/Don’t
- Accessibilité
- Exemples de code

## 8.2. Storybook (recommandé)
- Stories par variant
- Controls pour tokens
- Doc auto via compodoc ou doc blocks

## 8.3. Process d’adoption
- Définir un **catalogue** et un plan de migration
- Former l’équipe
- Introduire une politique de contribution (PR template)

---

# Exercices (guidés)

## Exercice 1 — Mettre en place tokens + theme switch
1. Créer `:root` tokens
2. Définir `data-theme="dark"`
3. Ajouter un toggle dans l’app demo

**Critères** :
- Tous les composants utilisent `--acme-color-surface` et `--acme-color-on-surface`
- Contraste correct

## Exercice 2 — Construire `acme-button`
- API `variant`
- États hover/focus/disabled
- Démo Storybook

## Exercice 3 — Standardiser les erreurs de formulaire
- Créer une fonction utilitaire `getErrorMessage(control)`
- Wrapper `acme-input` et `acme-select`

## Exercice 4 — Encapsuler `MatDialog`
- Créer `AcmeDialogService`
- Ajout d’un composant `AcmeConfirmDialog`

## Exercice 5 — Gouvernance (lint + conventions)
- Interdire import direct Material dans `apps/*`
- Documenter exceptions

---

# Annexes

## A. Checklist « composant DS »
- [ ] API stable et typée
- [ ] Variants documentés
- [ ] États (hover/active/disabled/focus/error/loading)
- [ ] Responsivité
- [ ] A11y validée (clavier + lecteurs)
- [ ] Tokens sémantiques utilisés
- [ ] Tests unitaires + story

## B. Modèle de contribution (PR)
- Motivation + lien vers spec design
- Captures avant/après
- Validation a11y
- Impact tokens/thème
- Breaking changes ?

## C. Glossaire
- **Token** : valeur atomique de design (couleur, typo, espace)
- **Semantic token** : token décrivant un usage (surface/primary/error)
- **Wrapper** : composant interne encapsulant un composant externe
- **Harness** : API de test fournie par Material pour manipuler les composants

---

# Proposition de découpage (1 jour)
Si format 1 jour (7 h) :
- Matin : Modules 1-2 + exercice tokens
- Après-midi : Modules 4-5 + exercice wrapper + gouvernance

# Proposition de découpage (3 jours)
- Jour 1 : Modules 1-2 + tokens complets
- Jour 2 : Modules 3-5 + librairie + schematics
- Jour 3 : Modules 6-8 + tests, a11y, storybook, industrialisation
