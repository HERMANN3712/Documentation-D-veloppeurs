# 13 — Directives personnalisées (Angular)

> **Objectif** : concevoir, tester et industrialiser des **directives personnalisées** pour factoriser du comportement réutilisable (focus, permissions, raccourcis clavier, styles dynamiques…), en distinguant clairement **directives attributaires** et **directives structurelles**.

---

## Plan de la formation

1. **Pourquoi des directives personnalisées ?**
2. **Rappels : le modèle des directives Angular**
3. **Directives attributaires**
   - API (`@Input`, `@HostBinding`, `@HostListener`)
   - Accès au DOM : `ElementRef`, `Renderer2`
   - Exemples : focus, styles dynamiques, permissions (attributaire), accessibilité
4. **Directives structurelles**
   - `TemplateRef`, `ViewContainerRef`
   - Microsyntaxe (`*directive="..."`) et `ng-template`
   - Exemples : permissions (structurelle), rendu conditionnel enrichi
5. **Patterns avancés et bonnes pratiques**
   - Typage, `exportAs`, alias d’inputs, gestion des changements
   - Zones, perf, SSR, sécurité
   - Architecture & distribution (standalone, libraries)
6. **Tests (unitaires & intégration)**
7. **Atelier de synthèse + checklists**

---

## 1) Pourquoi des directives personnalisées ?

En Angular, une **directive** est une classe qui peut **attacher un comportement** à un élément du DOM (ou à une vue). Les directives personnalisées servent à :

- **Factoriser** un comportement réutilisable (éviter copier/coller).
- **Uniformiser** des règles transverses (UX, sécurité, accessibilité).
- **Encapsuler** de la manipulation DOM de façon sûre et testable.
- **Améliorer la lisibilité** des templates (un attribut vaut mieux qu’un bloc de logique).

Exemples typiques :

- Autofocus sur un champ / focus conditionnel.
- Permissions : afficher/masquer/désactiver selon un rôle.
- Raccourcis clavier globaux ou limités à un composant.
- Styles dynamiques (surbrillance, validation, drag, etc.).

---

## 2) Rappels : le modèle des directives Angular

Angular distingue principalement :

- **Directives attributaires** : s’appliquent à un élément existant et **enrichissent son comportement**.
  - Exemple : `[appHighlight]="'yellow'"`.
- **Directives structurelles** : **modifient la structure du DOM** en ajoutant/supprimant des vues.
  - Exemple : `*appHasRole="'admin'"`.

Tout composant Angular est “une directive spéciale” avec un template. Mais ici on se concentre sur des directives **sans template**.

### Directive minimale

```ts
import { Directive } from '@angular/core';

@Directive({
  selector: '[appDemo]'
})
export class DemoDirective {}
```

### Standalone vs NgModule

Dans Angular moderne, privilégiez les **directives standalone** :

```ts
import { Directive } from '@angular/core';

@Directive({
  selector: '[appDemo]',
  standalone: true,
})
export class DemoDirective {}
```

Puis import dans un composant :

```ts
@Component({
  standalone: true,
  imports: [DemoDirective],
  template: `<div appDemo></div>`
})
export class MyComponent {}
```

---

## 3) Directives attributaires

### 3.1 API de base

- `@Input()` : paramétrer la directive.
- `@HostBinding()` : binder des propriétés/attributs/classes/styles sur l’élément hôte.
- `@HostListener()` : écouter des événements sur l’élément hôte (ou sur `document`, `window`).

> Bonne pratique : privilégier `@HostBinding` pour les styles/classes plutôt que de manipuler directement le DOM.

---

### 3.2 Manipuler le DOM en sécurité : `ElementRef` et `Renderer2`

- `ElementRef` donne accès à l’élément natif (`nativeElement`).
- `Renderer2` fournit des méthodes compatibles (et plus sûres) pour manipuler le DOM (utile pour SSR, web workers, etc.).

> Évitez `nativeElement` quand cela est possible, surtout pour des modifications de style/attributs. Utilisez `Renderer2`.

---

### 3.3 Exemple 1 — Directive de focus (autofocus et focus conditionnel)

#### Besoin

- Donner le focus automatiquement à l’affichage.
- Optionnel : ne focaliser que si une condition est vraie.

#### Code

```ts
import { AfterViewInit, Directive, ElementRef, Input } from '@angular/core';

@Directive({
  selector: '[appFocus]',
  standalone: true,
})
export class FocusDirective implements AfterViewInit {
  /**
   * - `true`/`''` : focus
   * - `false` : pas de focus
   */
  @Input('appFocus') enabled: boolean | '' = true;

  constructor(private readonly el: ElementRef<HTMLElement>) {}

  ngAfterViewInit(): void {
    const shouldFocus = this.enabled === '' ? true : !!this.enabled;
    if (shouldFocus) {
      // microtask pour éviter certains cas où l'élément n'est pas encore focusable
      queueMicrotask(() => this.el.nativeElement.focus());
    }
  }
}
```

#### Utilisation

```html
<input type="text" appFocus />
<input type="text" [appFocus]="isEditing" />
```

#### Points d’attention

- Le focus peut échouer si l’élément est désactivé (`disabled`) ou non focusable.
- En SSR, `focus()` n’a pas de sens : dans ce cas, conditionnez via `isPlatformBrowser` si nécessaire.

---

### 3.4 Exemple 2 — Directive de styles dynamiques (highlight)

#### Objectif

Mettre en surbrillance un élément au survol ou selon une valeur.

#### Variante A : via `@HostBinding`

```ts
import { Directive, HostBinding, HostListener, Input } from '@angular/core';

@Directive({
  selector: '[appHighlight]',
  standalone: true,
})
export class HighlightDirective {
  @Input('appHighlight') color = 'gold';
  @Input() appHighlightDefault = 'transparent';

  @HostBinding('style.backgroundColor') bg = this.appHighlightDefault;

  @HostListener('mouseenter') onEnter() {
    this.bg = this.color;
  }

  @HostListener('mouseleave') onLeave() {
    this.bg = this.appHighlightDefault;
  }
}
```

#### Utilisation

```html
<p appHighlight>Surbrillance par défaut</p>
<p [appHighlight]="'lightblue'" [appHighlightDefault]="'white'">Custom</p>
```

#### Variante B : classes (souvent préférable)

```ts
@HostBinding('class.is-highlighted') highlighted = false;

@HostListener('mouseenter') onEnter() { this.highlighted = true; }
@HostListener('mouseleave') onLeave() { this.highlighted = false; }
```

Et côté CSS :

```css
.is-highlighted { background: gold; }
```

---

### 3.5 Exemple 3 — Permissions en directive attributaire (désactiver / rendre non-interactif)

#### Objectif

Au lieu de masquer, on peut :

- désactiver un bouton,
- empêcher les clics,
- ajouter un tooltip explicatif,
- améliorer l’accessibilité avec `aria-disabled`.

#### Service de permissions (exemple)

```ts
import { Injectable, signal } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class AuthzService {
  // Exemple simple : rôles courants
  private rolesSig = signal<string[]>(['user']);

  setRoles(roles: string[]) { this.rolesSig.set(roles); }
  has(role: string): boolean { return this.rolesSig().includes(role); }
  hasAny(roles: string[]): boolean { return roles.some(r => this.has(r)); }
}
```

#### Directive

```ts
import { Directive, HostBinding, HostListener, Input } from '@angular/core';
import { AuthzService } from './authz.service';

@Directive({
  selector: '[appCanInteract]',
  standalone: true,
})
export class CanInteractDirective {
  @Input('appCanInteract') roles: string | string[] = [];
  @Input() appCanInteractMode: 'disable' | 'block' = 'disable';

  @HostBinding('attr.aria-disabled') ariaDisabled: 'true' | null = null;
  @HostBinding('attr.disabled') disabledAttr: '' | null = null; // fonctionne surtout pour form controls
  @HostBinding('style.pointer-events') pointerEvents: string | null = null;
  @HostBinding('style.opacity') opacity: string | null = null;

  constructor(private readonly authz: AuthzService) {}

  private get required(): string[] {
    return Array.isArray(this.roles) ? this.roles : [this.roles];
  }

  private get allowed(): boolean {
    return this.required.length === 0 ? true : this.authz.hasAny(this.required);
  }

  ngOnChanges() {
    this.apply();
  }

  ngOnInit() {
    this.apply();
  }

  private apply() {
    if (this.allowed) {
      this.ariaDisabled = null;
      this.disabledAttr = null;
      this.pointerEvents = null;
      this.opacity = null;
      return;
    }

    this.ariaDisabled = 'true';

    if (this.appCanInteractMode === 'block') {
      this.pointerEvents = 'none';
      this.opacity = '0.6';
      this.disabledAttr = null;
    } else {
      // mode disable
      this.disabledAttr = '';
      this.opacity = '0.6';
      this.pointerEvents = null;
    }
  }

  @HostListener('click', ['$event'])
  onClick(ev: MouseEvent) {
    if (!this.allowed) {
      ev.preventDefault();
      ev.stopImmediatePropagation();
    }
  }
}
```

#### Utilisation

```html
<button [appCanInteract]="['admin','manager']">Action réservée</button>
<button [appCanInteract]="'admin'" appCanInteractMode="block">Bloquée</button>
```

> Remarque : `disabled` est pertinent pour `<button>`, `<input>`, etc. Pour des éléments non-form (ex : `<a>`), préférez `pointer-events: none` + gestion du click.

---

### 3.6 Exemple 4 — Raccourcis clavier (HostListener sur document)

#### Objectif

Déclencher une action quand l’utilisateur presse une combinaison (ex. `Ctrl+K`).

```ts
import { Directive, EventEmitter, HostListener, Input, Output } from '@angular/core';

@Directive({
  selector: '[appHotkey]',
  standalone: true,
})
export class HotkeyDirective {
  @Input('appHotkey') combo = 'Control.k';
  @Output() appHotkeyTriggered = new EventEmitter<KeyboardEvent>();

  @HostListener('document:keydown', ['$event'])
  onKeyDown(ev: KeyboardEvent) {
    if (this.matches(ev, this.combo)) {
      ev.preventDefault();
      this.appHotkeyTriggered.emit(ev);
    }
  }

  private matches(ev: KeyboardEvent, combo: string): boolean {
    // Combo simple: "Control.k", "Shift.Enter", "Alt.s" ...
    const parts = combo.split('.');
    const key = parts.at(-1)?.toLowerCase();

    const needsCtrl = parts.includes('Control');
    const needsShift = parts.includes('Shift');
    const needsAlt = parts.includes('Alt');
    const needsMeta = parts.includes('Meta');

    if (needsCtrl !== ev.ctrlKey) return false;
    if (needsShift !== ev.shiftKey) return false;
    if (needsAlt !== ev.altKey) return false;
    if (needsMeta !== ev.metaKey) return false;

    return (ev.key || '').toLowerCase() === key;
  }
}
```

Utilisation :

```html
<div [appHotkey]="'Control.k'" (appHotkeyTriggered)="openPalette()"></div>
```

Bonnes pratiques :

- Éviter les collisions avec le navigateur (ex. `Ctrl+L`, `Ctrl+T`).
- Permettre la désactivation via input (`[enabled]`).
- Restreindre à un contexte donné (quand le composant est visible/actif).

---

## 4) Directives structurelles

### 4.1 Concepts clés

Une directive structurelle :

- reçoit un **template** (`TemplateRef`) : le contenu à insérer,
- manipule un **conteneur de vues** (`ViewContainerRef`) : où insérer/supprimer.

```ts
constructor(
  private tpl: TemplateRef<unknown>,
  private vcr: ViewContainerRef
) {}
```

### 4.2 Microsyntaxe : `*`

La syntaxe :

```html
<div *appX="expr">...</div>
```

est un sucre syntaxique pour :

```html
<ng-template [appX]="expr">
  <div>...</div>
</ng-template>
```

Comprendre ceci est essentiel pour maîtriser les directives structurelles.

---

### 4.3 Exemple 1 — Permission structurelle `*appHasRole`

#### Objectif

Afficher un bloc uniquement si l’utilisateur a un rôle.

#### Code

```ts
import { Directive, Input, TemplateRef, ViewContainerRef } from '@angular/core';
import { AuthzService } from './authz.service';

@Directive({
  selector: '[appHasRole]',
  standalone: true,
})
export class HasRoleDirective {
  private hasView = false;

  @Input('appHasRole') role: string | string[] = [];

  constructor(
    private readonly tpl: TemplateRef<unknown>,
    private readonly vcr: ViewContainerRef,
    private readonly authz: AuthzService
  ) {}

  ngOnChanges(): void {
    this.update();
  }

  ngOnInit(): void {
    this.update();
  }

  private get required(): string[] {
    return Array.isArray(this.role) ? this.role : [this.role];
  }

  private update() {
    const allowed = this.required.length === 0 ? true : this.authz.hasAny(this.required);

    if (allowed && !this.hasView) {
      this.vcr.createEmbeddedView(this.tpl);
      this.hasView = true;
    } else if (!allowed && this.hasView) {
      this.vcr.clear();
      this.hasView = false;
    }
  }
}
```

#### Utilisation

```html
<section *appHasRole="'admin'">
  <h2>Administration</h2>
</section>

<section *appHasRole="['manager','admin']">
  <h2>Management</h2>
</section>
```

---

### 4.4 Exemple 2 — Bloc `else` (pattern type `ngIf`)

On peut enrichir votre directive pour accepter un template alternatif.

#### Utilisation cible

```html
<div *appHasRole="'admin'; else noAccess">
  Accès admin
</div>

<ng-template #noAccess>
  <p>Accès refusé.</p>
</ng-template>
```

#### Implémentation (version simplifiée)

```ts
import { Directive, Input, TemplateRef, ViewContainerRef } from '@angular/core';
import { AuthzService } from './authz.service';

@Directive({
  selector: '[appHasRole]',
  standalone: true,
})
export class HasRoleDirective {
  private thenTpl: TemplateRef<unknown>;
  private elseTpl: TemplateRef<unknown> | null = null;

  @Input('appHasRole') role: string | string[] = [];
  @Input('appHasRoleElse') set appHasRoleElse(tpl: TemplateRef<unknown> | null) {
    this.elseTpl = tpl;
    this.update();
  }

  constructor(
    tpl: TemplateRef<unknown>,
    private readonly vcr: ViewContainerRef,
    private readonly authz: AuthzService
  ) {
    this.thenTpl = tpl;
  }

  ngOnChanges() { this.update(); }
  ngOnInit() { this.update(); }

  private update() {
    const required = Array.isArray(this.role) ? this.role : [this.role];
    const allowed = required.length === 0 ? true : this.authz.hasAny(required);

    this.vcr.clear();
    this.vcr.createEmbeddedView(allowed ? this.thenTpl : (this.elseTpl ?? this.thenTpl));
  }
}
```

> Pour aller plus loin : gérer `then` + `else` séparément, éviter de recréer la vue si rien ne change, exposer un contexte (`let-...`).

---

## 5) Patterns avancés & bonnes pratiques

### 5.1 Nommage, cohérence et ergonomie

- Préfixe projet : `app...`.
- Inputs lisibles :
  - `@Input('appHasRole') role` (alias utile)
  - Inputs secondaires : `appHasRoleElse`, `appHasRoleThen`.
- Préférez des API **déclaratives** : `*appHasRole="['admin']"`.

### 5.2 Contexte des directives structurelles

Une directive structurelle peut fournir un contexte :

```html
<div *appHasRole="'admin' as isAdmin">
  Admin? {{ isAdmin }}
</div>
```

Cela nécessite de passer un objet contexte à `createEmbeddedView(tpl, context)`.

### 5.3 Gestion des changements

- Directives attributaires : implémenter `OnChanges` si l’input est susceptible de changer.
- Structurelles : éviter de détruire/recréer des vues inutilement.

### 5.4 SSR / DOM / sécurité

- Attention aux accès directs au DOM côté serveur.
- Ne jamais injecter du HTML non maîtrisé.
- Pour des effets visuels, privilégier classes + CSS.

### 5.5 Standalone, réutilisation et librairies

- Regrouper des directives dans un `index.ts` (barrel).
- Pour une librairie : documenter le sélecteur, inputs/outputs, exemples, compatibilité Angular.

---

## 6) Tests

### 6.1 Tests unitaires d’une directive attributaire (ex. Highlight)

#### Host component de test

```ts
import { Component } from '@angular/core';
import { HighlightDirective } from './highlight.directive';

@Component({
  standalone: true,
  imports: [HighlightDirective],
  template: `<p [appHighlight]="'red'">Hello</p>`
})
class HostComponent {}
```

#### Spec (Jasmine/Karma ou Jest)

```ts
import { TestBed } from '@angular/core/testing';

describe('HighlightDirective', () => {
  it('should set background on mouseenter', async () => {
    const fixture = TestBed.configureTestingModule({
      imports: [HostComponent]
    }).createComponent(HostComponent);

    fixture.detectChanges();
    const p: HTMLElement = fixture.nativeElement.querySelector('p');

    p.dispatchEvent(new Event('mouseenter'));
    fixture.detectChanges();

    expect(p.style.backgroundColor).toBe('red');
  });
});
```

### 6.2 Tests d’une directive structurelle

- Vérifier la présence/absence d’un élément en fonction des rôles.
- Mock du `AuthzService`.

Pseudo-exemple :

```ts
// Arrange roles -> detectChanges -> expect(querySelector(...))
```

---

## 7) Atelier de synthèse (proposé)

### Exercice A — `appAutofocusIf`

- Input : condition booléenne.
- Focus seulement si `true`.
- Bonus : retenter quand la condition passe de `false` à `true`.

### Exercice B — `*appIfAuthorized`

- Input : un ou plusieurs rôles.
- Support d’un `else` template.
- Bonus : exposer `let-allowed`.

### Exercice C — `appHotkey`

- Support de `enabled`.
- Support de `scope`: `document` vs élément.

---

## Checklists

### Attributaire

- [ ] API simple : un input principal + options.
- [ ] `@HostBinding`/`@HostListener` privilégiés.
- [ ] `Renderer2` si manipulation DOM.
- [ ] Comportement testable.

### Structurelle

- [ ] Compréhension `*` ↔ `ng-template`.
- [ ] `TemplateRef` + `ViewContainerRef`.
- [ ] Ne pas recréer les vues inutilement.
- [ ] Option `else` documentée.

---

## Annexes — Snippets utiles

### Alias d’input (rappel)

```ts
@Input('appX') value!: string;
```

### `exportAs` (référence template)

```ts
@Directive({ selector: '[appX]', exportAs: 'appX', standalone: true })
export class XDirective {
  doSomething() {}
}
```

```html
<div appX #x="appX"></div>
<button (click)="x.doSomething()">Run</button>
```
