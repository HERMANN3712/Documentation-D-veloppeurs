# Formation Angular — Accessibilité (a11y)

**Public visé :** développeurs Angular (intermédiaire → avancé)  
**Durée conseillée :** 1 journée (7h) ou 2×3h30  
**Pré-requis :** Angular (components, templates, forms), TypeScript, notions HTML/CSS  
**Objectif général :** concevoir et développer des composants Angular accessibles dès la conception, conformément aux principes WCAG (perceptible, utilisable, compréhensible, robuste).

---

## Plan de la formation

1. **Introduction à l’accessibilité web**
   - Pourquoi l’accessibilité (inclusion, conformité, qualité)
   - WCAG, niveaux A/AA/AAA, obligations fréquentes
   - A11y = UX + qualité front + robustesse

2. **Structure sémantique et HTML accessible**
   - Landmarks, titres, listes, tableaux, formulaires
   - Ordre de lecture, hiérarchie de titres
   - Pièges : div-soup, éléments cliquables non sémantiques

3. **Navigation clavier**
   - Tab order, focus management
   - Éléments focusables, `tabindex`, pièges clavier
   - Raccourcis, interactions (menus, modales, tabs)

4. **Focus visible et états d’interaction**
   - Styles focus, `:focus-visible`
   - Cohérence design system
   - Cas complexes : modales, overlays, popovers

5. **Rôles ARIA et attributs ARIA**
   - Quand utiliser ARIA (et quand ne pas)
   - Rôles, propriétés et états
   - `aria-label`, `aria-labelledby`, `aria-describedby`
   - Live regions (annonces), `aria-live`, `role="status"`

6. **Contrastes, couleurs et contenus perceptibles**
   - Contraste texte / UI components
   - Ne pas s’appuyer uniquement sur la couleur
   - Modes sombre/clair, thèmes

7. **Compatibilité lecteurs d’écran**
   - Modèle d’accessibilité : DOM, arbre d’accessibilité
   - Patterns pour formulaires, erreurs, feedback
   - Bonnes pratiques d’annonce et d’update du contenu

8. **Accessibilité dans Angular : patterns et outils**
   - Templates, bindings, directives
   - CDK A11y (FocusTrap, LiveAnnouncer, ListKeyManager…)
   - Angular Material (a11y intégrée + points d’attention)

9. **Tests et audit d’accessibilité**
   - Tests manuels clavier + lecteurs d’écran
   - Outils : Lighthouse, axe DevTools, WAVE
   - Tests automatisés (unit/e2e) : jest + axe, Cypress + axe

10. **Atelier de mise en pratique (guidé)**
   - Rendre une modale accessible
   - Rendre un composant “tabs” ou “menu” accessible
   - Accessibiliser un formulaire (labels, erreurs, messages)

---

# 1) Introduction à l’accessibilité web

## 1.1 Définition
L’**accessibilité web** consiste à permettre à **tous** les utilisateurs, y compris ceux en situation de handicap (visuel, auditif, moteur, cognitif, temporaire, contextuel), d’utiliser une application.

### Exemples concrets
- Personne malvoyante utilisant **zoom** + contrastes élevés.
- Personne non-voyante utilisant un **lecteur d’écran** (NVDA, JAWS, VoiceOver).
- Personne avec difficulté motrice naviguant **uniquement au clavier**.
- Utilisateur sur mobile en plein soleil → contraste critique.

## 1.2 Référentiels
- **WCAG 2.1/2.2** : recommandations internationales.
- Niveaux : **A**, **AA** (souvent requis), **AAA**.

### Les 4 principes (POUR)
- **Perceptible** : l’info doit être percevable.
- **Operable (Utilisable)** : l’interface doit être utilisable (clavier, focus).
- **Understandable (Compréhensible)** : libellés, erreurs, comportements.
- **Robust** : compatible avec user agents/AT (assistive technologies).

## 1.3 Règle d’or
> **Utiliser HTML natif** dès que possible. ARIA vient **en complément**.

---

# 2) Structure sémantique et HTML accessible

## 2.1 Landmarks (repères)
Utiliser les structures sémantiques pour aider la navigation (lecteurs d’écran, raccourcis) :

```html
<header>…</header>
<nav>…</nav>
<main>…</main>
<aside>…</aside>
<footer>…</footer>
```

### Bonnes pratiques
- Un seul `<main>` par page.
- Les `<nav>` peuvent être multiples (menu principal, fil d’ariane…), mais doivent être **distinguables** (via `aria-label`).

```html
<nav aria-label="Navigation principale">…</nav>
<nav aria-label="Fil d’ariane">…</nav>
```

## 2.2 Hiérarchie de titres
- Respecter l’ordre `<h1>` → `<h2>` → `<h3>` …
- Un titre décrit un **bloc de contenu**.
- Éviter de sauter des niveaux.

## 2.3 Boutons, liens et contrôles
### Anti-pattern : div cliquable
```html
<div (click)="save()">Enregistrer</div>
```
Problèmes : pas focusable, pas activable au clavier, pas annoncé comme bouton.

### Correct : bouton
```html
<button type="button" (click)="save()">Enregistrer</button>
```

### Lien vs bouton
- **Lien** : navigation vers une autre page/route.
- **Bouton** : action (ouvrir une modale, envoyer, filtrer…).

---

# 3) Navigation clavier

## 3.1 Principes
Toute fonctionnalité doit être utilisable au clavier :
- **Tab / Shift+Tab** : navigation focus
- **Entrée / Espace** : activation (selon contrôle)
- **Flèches** : navigation interne (menus, tabs, listes)
- **Échap** : fermeture (modales, dropdowns)

## 3.2 Ordre de tabulation
Le focus suit l’ordre du DOM. On évite de “réparer” avec `tabindex`.

### `tabindex`
- `tabindex="0"` : rend focusable et suit l’ordre.
- `tabindex="-1"` : focusable **par code** seulement.
- `tabindex="1+"` : à éviter (ordre artificiel, difficile à maintenir).

## 3.3 Cartographier les interactions
Pour chaque composant, définir :
- Entrée clavier (comment on y arrive)
- Navigation interne (flèches, tab)
- Sortie (où va le focus)
- Raccourcis/fermeture

---

# 4) Focus visible et états d’interaction

## 4.1 Le focus doit être visible
Beaucoup de design systems suppriment `outline`. C’est un problème.

### Recommandation CSS
```css
:focus {
  outline: 2px solid #1a73e8;
  outline-offset: 2px;
}

:focus:not(:focus-visible) {
  outline: none;
}

:focus-visible {
  outline: 3px solid #1a73e8;
  outline-offset: 2px;
}
```

## 4.2 Gestion du focus dans les composants
Cas typiques :
- Ouverture d’une modale → focus sur le titre ou premier champ.
- Fermeture → focus revient sur l’élément déclencheur.
- Popover/dropdown → focus piégé tant que l’overlay est ouvert.

---

# 5) Rôles ARIA et attributs ARIA

## 5.1 Règles d’utilisation
1. **Préférer HTML natif** (button, input, select).
2. Ne pas ajouter de rôle redondant (ex. `role="button"` sur `<button>`).
3. ARIA ne “rend pas” un composant accessible si le clavier/focus ne suit pas.

## 5.2 Nom accessible (accessible name)
Un contrôle interactif doit avoir un nom.

### Méthodes
- Texte visible (ex. `<button>Supprimer</button>`)
- `aria-label` (sans texte visible)
- `aria-labelledby` (référence un élément)

```html
<button type="button" aria-label="Supprimer la ligne">🗑</button>
```

```html
<h2 id="dialog-title">Détails du compte</h2>
<section role="dialog" aria-labelledby="dialog-title">…</section>
```

## 5.3 Description (aide contextualisée)
```html
<input id="email" aria-describedby="email-help" />
<p id="email-help">Nous n’utiliserons jamais votre email à des fins commerciales.</p>
```

## 5.4 États ARIA
- `aria-expanded` (accordéon, dropdown)
- `aria-selected` (tabs)
- `aria-checked` (switch)
- `aria-disabled` (si pas possible d’utiliser `disabled`)

---

# 6) Contrastes, couleurs et contenus perceptibles

## 6.1 Contrast ratio (rappels)
- Texte normal : **≥ 4.5:1** (AA)
- Grand texte (≥ 24px ou 18.66px gras) : **≥ 3:1**
- Composants UI / icônes informatives : viser **≥ 3:1**

## 6.2 Ne pas s’appuyer sur la couleur seule
Exemples :
- Erreur champ → ajouter texte + icône + message.
- Statut (vert/rouge) → ajouter libellé “Succès/Échec”.

## 6.3 Thèmes et variables CSS
Soutenir modes clair/sombre via variables :

```css
:root {
  --text: #1c1c1c;
  --bg: #ffffff;
  --focus: #1a73e8;
}

[data-theme="dark"] {
  --text: #f1f1f1;
  --bg: #0f0f0f;
  --focus: #8ab4f8;
}
```

---

# 7) Compatibilité lecteurs d’écran

## 7.1 Comment un lecteur d’écran “voit” la page
Il s’appuie sur :
- Le **DOM**
- Les rôles/états (ARIA)
- L’ordre de focus

## 7.2 Annoncer les changements dynamiques
Dans une SPA, beaucoup d’updates ne sont pas “vus” automatiquement.

### Live region (annonce)
- `aria-live="polite"` : annonce non urgente
- `aria-live="assertive"` : urgente (à utiliser avec parcimonie)

Exemple simple :
```html
<p aria-live="polite">{{ statusMessage }}</p>
```

### Angular CDK: LiveAnnouncer
```ts
import { LiveAnnouncer } from '@angular/cdk/a11y';

constructor(private liveAnnouncer: LiveAnnouncer) {}

save() {
  // … appel API
  this.liveAnnouncer.announce('Enregistrement terminé', 'polite');
}
```

## 7.3 Formulaires : labels, erreurs, aide
### Labels
Toujours un `<label>` associé :
```html
<label for="email">Email</label>
<input id="email" type="email" />
```

### Erreurs accessibles
Objectifs :
- Message lisible
- Message lié au champ (`aria-describedby`)
- Champ invalidé (`aria-invalid="true"`)

Exemple Angular (template-driven ou reactive) :
```html
<label for="email">Email</label>
<input
  id="email"
  type="email"
  [formControl]="email"
  [attr.aria-invalid]="email.invalid && email.touched ? 'true' : null"
  [attr.aria-describedby]="email.invalid && email.touched ? 'email-error' : null"
/>

<p *ngIf="email.invalid && email.touched" id="email-error" role="alert">
  L’email est invalide.
</p>
```
`role="alert"` déclenche une annonce immédiate.

---

# 8) Accessibilité dans Angular : patterns et outils

## 8.1 Templates Angular : points d’attention
### Attributs ARIA dynamiques
Toujours utiliser `attr.` :
```html
<button
  type="button"
  [attr.aria-expanded]="isOpen"
  [attr.aria-controls]="panelId"
  (click)="toggle()"
>
  Détails
</button>

<div [id]="panelId" [hidden]="!isOpen">…</div>
```

### `hidden` vs `*ngIf`
- `*ngIf` retire du DOM → plus accessible si l’élément ne doit pas exister.
- `[hidden]` garde dans le DOM mais non visible → peut être utile, mais attention aux AT selon cas. En général, `hidden` est OK car il retire de l’arbre d’accessibilité.

## 8.2 Router et annonce de changement de page
Dans une SPA, annoncer le changement de “page” (route) améliore l’expérience.

Pattern : déplacer le focus sur un `<h1>` de page à chaque navigation.

```html
<h1 tabindex="-1" #pageTitle>{{ title }}</h1>
```

```ts
@ViewChild('pageTitle', { static: true }) pageTitle!: ElementRef<HTMLElement>;

ngAfterViewInit() {
  this.pageTitle.nativeElement.focus();
}
```
Alternative : combiner avec `LiveAnnouncer`.

## 8.3 Angular CDK A11y (recommandé)
### FocusTrap (modales, panneaux)
```ts
import { FocusTrapFactory } from '@angular/cdk/a11y';

private focusTrap?: any;

constructor(private focusTrapFactory: FocusTrapFactory,
            private host: ElementRef<HTMLElement>) {}

open() {
  this.focusTrap = this.focusTrapFactory.create(this.host.nativeElement);
  this.focusTrap.focusInitialElementWhenReady();
}

close() {
  this.focusTrap?.destroy();
}
```

### ListKeyManager (navigation clavier dans des listes)
Idéal pour menus, listes d’options custom.

## 8.4 Angular Material
- Beaucoup de composants Material sont déjà a11y-friendly.
- À vérifier :
  - libellés (`mat-icon-button` → `aria-label`)
  - erreurs de formulaire
  - contrastes du thème
  - focus style (peut être custom)

---

# 9) Tests et audit d’accessibilité

## 9.1 Checklist manuelle rapide
- Tout est utilisable **sans souris**.
- Focus visible partout.
- Pas de piège clavier.
- Ordre de focus logique.
- Contrastes OK.
- Labels + erreurs compréhensibles.

## 9.2 Outils
- **Lighthouse (Chrome)** : audit rapide, mais incomplet.
- **axe DevTools** : règles robustes, recommandations.
- **WAVE** : inspection visuelle.

## 9.3 Tests automatisés (exemples)
### Unit tests avec axe (Jest)
Principe : rendre un composant, exécuter axe, vérifier 0 violations.

### E2E avec Cypress + axe
Ajoute une étape dans CI.

> Note : les tests automatiques ne couvrent pas tout (navigation clavier fine, qualité des libellés, pertinence UX).

---

# 10) Atelier guidé — rendre une modale accessible

## Objectifs
- Gérer le focus à l’ouverture/fermeture
- Piéger le focus dans la modale
- Ajouter rôle/label ARIA
- Fermer via Échap

## 10.1 Structure recommandée
```html
<button type="button" (click)="openDialog()" #trigger>
  Ouvrir la modale
</button>

<div
  *ngIf="open"
  class="backdrop"
  (click)="closeDialog()"
></div>

<section
  *ngIf="open"
  class="dialog"
  role="dialog"
  aria-modal="true"
  [attr.aria-labelledby]="titleId"
  (keydown.escape)="closeDialog()"
  cdkTrapFocus
  [cdkTrapFocusAutoCapture]="true"
>
  <h2 [id]="titleId">Paramètres</h2>

  <p id="dialog-desc">Modifiez vos préférences puis validez.</p>

  <button type="button" (click)="closeDialog()">Fermer</button>
  <button type="button" (click)="save()">Enregistrer</button>
</section>
```

### Points clés
- `role="dialog"` + `aria-modal="true"`
- `aria-labelledby` vers le titre
- `cdkTrapFocus` pour piéger le focus
- `(keydown.escape)` pour fermer
- Restituer le focus au bouton déclencheur

## 10.2 Composant Angular
```ts
import { Component, ElementRef, ViewChild } from '@angular/core';

@Component({
  selector: 'app-settings-dialog-demo',
  templateUrl: './settings-dialog-demo.component.html'
})
export class SettingsDialogDemoComponent {
  open = false;
  titleId = 'settings-title';

  @ViewChild('trigger', { static: true }) trigger!: ElementRef<HTMLButtonElement>;

  openDialog() {
    this.open = true;
  }

  closeDialog() {
    this.open = false;
    // rendre le focus au déclencheur
    queueMicrotask(() => this.trigger.nativeElement.focus());
  }

  save() {
    // …
    this.closeDialog();
  }
}
```

---

# Annexes

## A) Checklist “Accessible by default” pour composants Angular
- [ ] HTML sémantique (button, a, input, label)
- [ ] Contrôles interactifs ont un nom accessible
- [ ] Navigation clavier complète
- [ ] Focus visible et cohérent
- [ ] Gestion du focus (ouverture/fermeture overlay)
- [ ] ARIA minimal et correct (états, relationships)
- [ ] Contrastes respectés
- [ ] Messages et erreurs annoncés
- [ ] Tests manuels + outils (axe/Lighthouse)

## B) Ressources
- WCAG Quick Reference : https://www.w3.org/WAI/WCAG21/quickref/
- ARIA Authoring Practices : https://www.w3.org/WAI/ARIA/apg/
- Angular CDK a11y : https://material.angular.io/cdk/a11y/overview

---

## Fin de formation — livrables attendus
À l’issue, le participant sait :
- concevoir des composants accessibles (pattern + sémantique)
- implémenter navigation clavier et focus management
- utiliser ARIA correctement
- vérifier contrastes et compatibilité lecteurs d’écran
- mettre en place une stratégie d’audit et de tests a11y
