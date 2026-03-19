# Formation Angular — 56. Interopérabilité avec des bibliothèques tierces

> **Objectif** : intégrer proprement des bibliothèques JavaScript non-Angular (DOM-based, jQuery-like, widgets UI, charting, editors, SDKs) en respectant les bonnes pratiques Angular (encapsulation, testabilité, performances, SSR, sécurité).

---

## Public & prérequis

- **Public** : développeurs Angular (intermédiaire à avancé), formateurs, tech leads.
- **Prérequis** :
  - Composants, directives, services, DI.
  - `@ViewChild`, templates, data binding.
  - Notions de cycle de vie Angular.
  - Notions de TypeScript et npm.

---

## Compétences visées

À l’issue de la formation, vous serez capable de :

1. **Choisir une stratégie d’intégration** (wrapper composant, directive, service, dynamic import).
2. Manipuler le DOM de manière **safe** via `ElementRef` et surtout `Renderer2`.
3. Exploiter les **lifecycle hooks** pour initialiser/mettre à jour/détruire la lib.
4. **Encapsuler** la dépendance pour limiter l’impact sur le reste de l’application.
5. Gérer les cas avancés : **Zone.js**, performance, SSR/hydration, accessibilité.
6. Réduire les risques : **sécurité**, mémoire, compatibilité, typings.

---

## Durée & format (suggestion)

- **Durée** : 1 journée (6–7h) ou 2 demi-journées.
- **Format** : alternance théorie + ateliers.
- **Livrables** : wrappers réutilisables, checklist d’intégration.

---

# Plan détaillé

1. **Comprendre le problème** : pourquoi certaines libs nécessitent une intégration spécifique
2. **Stratégies d’intégration** (composant, directive, service, module, façade)
3. **Accès au DOM en Angular** : `ElementRef`, `Renderer2`, `DOCUMENT`, injections
4. **Lifecycle hooks** : init, update, destroy (et pourquoi)
5. **Encapsulation de la dépendance** : wrappers dédiés, API Angular-friendly
6. **Gestion des événements et des callbacks** : `NgZone`, change detection
7. **Chargement et bundling** : ESM/CommonJS, dynamic import, lazy, styles/assets
8. **SSR / Angular Universal** : rendre l’intégration compatible serveur
9. **Qualité & robustesse** : typings, tests, erreurs, mémoire (cleanup)
10. **Ateliers** : intégrer une librairie « DOM mutant » et construire un wrapper propre

---

# 1) Comprendre le problème : pourquoi certaines libs exigent une intégration spécifique

De nombreuses bibliothèques JavaScript classiques :

- **réécrivent le DOM** (innerHTML, appendChild, querySelector…),
- attachent des listeners directement,
- maintiennent un état interne non synchronisé avec Angular,
- utilisent parfois jQuery,
- exposent une API orientée « impérative » (init/destroy/update) plutôt qu’« déclarative ».

Or Angular fonctionne selon un modèle où :

- le template est *déclaratif*,
- le rendu est piloté par le framework (change detection),
- la manipulation directe du DOM est **déconseillée** car source de bugs, failles, incompatibilités SSR, et rupture d’encapsulation.

**Conséquence** : l’intégration doit passer par des points de contact maîtrisés (`ElementRef`, `Renderer2`, hooks, wrappers), et surtout **encapsuler** la lib pour éviter que toute l’application ne dépende de détails d’implémentation.

---

# 2) Stratégies d’intégration

## 2.1 Wrapper **composant** (recommandé pour widgets UI)

Idéal quand la lib rend un composant visuel : éditeur WYSIWYG, chart, datepicker, map…

- Le composant Angular possède :
  - des `@Input()` pour configurer,
  - des `@Output()` pour émettre des événements,
  - une zone DOM dédiée.

**Avantages** : encapsulation forte, API stable, réutilisable, facile à documenter.

## 2.2 Wrapper **directive** (recommandé pour ajouter un comportement)

Idéal quand la lib « enrichit » un élément existant (tooltip, mask input, drag/drop…).

- La directive s’attache à un élément et installe la lib.

**Avantages** : léger, composable.

## 2.3 Service/facade (recommandé pour SDKs non DOM)

Idéal pour libs : analytics, auth, feature flags, paiement, notifications push…

- Le service encapsule le SDK et expose des méthodes Angular-friendly.

## 2.4 Intégration « brute » (à éviter)

Appeler la lib dans un composant sans encapsuler (ou manipuler le DOM depuis `ngOnInit`) est fragile.

**Règle** : si vous devez écrire du code impératif d’intégration, faites-le **dans un wrapper**.

---

# 3) Accès au DOM : `ElementRef`, `Renderer2`, `DOCUMENT`

## 3.1 `ElementRef`

- `ElementRef` donne accès à `nativeElement`.
- **Attention** : l’usage direct de `nativeElement` vous pousse à manipuler le DOM en direct.

```ts
@ViewChild('host', { static: true }) host!: ElementRef<HTMLElement>;
```

**Bon usage** : fournir un point d’ancrage (host element) à une lib qui exige un élément DOM.

## 3.2 `Renderer2` (préféré)

`Renderer2` propose une abstraction Angular pour manipuler le DOM (compatible plateformes, SSR plus friendly, meilleures pratiques).

```ts
constructor(private renderer: Renderer2) {}

const el = this.renderer.createElement('div');
this.renderer.addClass(el, 'widget');
this.renderer.appendChild(this.host.nativeElement, el);
```

## 3.3 Injection de `DOCUMENT`

Pour des accès contrôlés à `document` (à condition de gérer SSR)

```ts
import { DOCUMENT } from '@angular/common';

constructor(@Inject(DOCUMENT) private document: Document) {}
```

**Éviter** : `window`/`document` globaux sans garde SSR.

---

# 4) Lifecycle hooks : init / update / destroy

## 4.1 Les hooks essentiels

- `ngAfterViewInit()` : le DOM du composant (view) est prêt.
- `ngOnChanges()` : les `@Input()` changent, idéal pour *update*.
- `ngOnDestroy()` : nettoyer (listeners, timers, instances).

> Beaucoup de libs exigent un élément host existant : **`ngAfterViewInit` est souvent le bon moment**.

## 4.2 Erreur fréquente

- Initialiser la lib dans `ngOnInit()` alors que l’élément DOM cible n’existe pas encore.

---

# 5) Encapsuler la dépendance : wrappers dédiés

## 5.1 Principes

- **Une seule porte d’entrée** côté Angular.
- L’extérieur ne parle pas à la lib directement.
- API Angular : `@Input`, `@Output`, `ControlValueAccessor` si c’est un champ.
- Possibilité de remplacer la librairie sans modifier tout le code applicatif.

## 5.2 Exemple : wrapper composant générique

Imaginons une lib impérative fictive `ThirdPartyWidget` :

```ts
// API typique d'une lib non-Angular
class ThirdPartyWidget {
  constructor(host: HTMLElement, options: any) {}
  update(options: any): void {}
  on(eventName: string, cb: (...args: any[]) => void): void {}
  destroy(): void {}
}
```

### Composant wrapper Angular

```ts
import {
  AfterViewInit,
  Component,
  ElementRef,
  EventEmitter,
  Input,
  NgZone,
  OnChanges,
  OnDestroy,
  Output,
  SimpleChanges,
  ViewChild,
} from '@angular/core';

type WidgetOptions = {
  theme?: 'light' | 'dark';
  value?: string;
};

@Component({
  selector: 'app-third-party-widget',
  template: `
    <div #host class="widget-host"></div>
  `,
})
export class ThirdPartyWidgetComponent
  implements AfterViewInit, OnChanges, OnDestroy
{
  @ViewChild('host', { static: true }) host!: ElementRef<HTMLElement>;

  @Input() options: WidgetOptions = {};

  @Output() changed = new EventEmitter<string>();

  private instance?: ThirdPartyWidget;

  constructor(private zone: NgZone) {}

  ngAfterViewInit(): void {
    // Évite de déclencher la change detection à chaque event interne.
    this.zone.runOutsideAngular(() => {
      this.instance = new ThirdPartyWidget(this.host.nativeElement, this.options);

      this.instance.on('change', (value: string) => {
        // On re-rentre dans Angular seulement si on doit notifier l'app.
        this.zone.run(() => this.changed.emit(value));
      });
    });
  }

  ngOnChanges(changes: SimpleChanges): void {
    if (changes['options'] && this.instance) {
      // Update impératif quand les inputs changent.
      this.instance.update(this.options);
    }
  }

  ngOnDestroy(): void {
    this.instance?.destroy();
    this.instance = undefined;
  }
}
```

Points importants :

- `ngAfterViewInit` pour initialiser.
- `ngOnChanges` pour synchroniser les modifications.
- `ngOnDestroy` pour éviter fuites mémoire.
- `NgZone` pour éviter une sur-activité de détection.

---

# 6) Éviter les manipulations DOM directes non maîtrisées

## 6.1 Risques

- Rupture de l’encapsulation Angular.
- Incohérences d’affichage (Angular vs DOM modifié).
- Problèmes SSR (pas de DOM côté serveur).
- Vulnérabilités : injection via `innerHTML`.
- Difficile à tester.

## 6.2 Bonnes pratiques

- Préférer `Renderer2` et des **zones d’ancrage** claires.
- Ne jamais modifier le DOM global (`document.body`…) sans raison.
- Encapsuler les effets de bord dans un wrapper.
- Expliciter un contrat :
  - « Angular fournit un host element »
  - « La lib gère ce sous-arbre uniquement »

---

# 7) Gestion des événements, callbacks, Change Detection

## 7.1 Problème

Certaines libs déclenchent fréquemment des callbacks (mousemove, scroll, input…). Si chaque callback repasse dans Angular, performance dégradée.

## 7.2 Solution : `NgZone.runOutsideAngular`

- Initialiser la lib **hors zone**.
- Revenir dans la zone uniquement si nécessaire.

Déjà illustré dans l’exemple.

## 7.3 Cas avancé : composants `OnPush`

Si votre wrapper est en `ChangeDetectionStrategy.OnPush`, vous pouvez :

- émettre via `@Output()` (Angular gère), ou
- appeler `ChangeDetectorRef.markForCheck()` après une maj.

---

# 8) Chargement, bundling et dépendances (npm, ESM/CommonJS)

## 8.1 Installer et typer

- Installer via npm : `npm i lib`.
- Vérifier la présence de types :
  - si pas de types : `npm i -D @types/lib` ou déclarations manuelles.

### Déclaration minimale de types

```ts
// src/types/third-party-widget.d.ts
declare class ThirdPartyWidget {
  constructor(host: HTMLElement, options: any);
  update(options: any): void;
  on(eventName: string, cb: (...args: any[]) => void): void;
  destroy(): void;
}
```

Configurer `typeRoots` si nécessaire (selon setup).

## 8.2 Dynamic import (lazy / perf)

Charger une grosse lib uniquement quand nécessaire :

```ts
private async load(): Promise<typeof import('third-party-widget')> {
  return import('third-party-widget');
}
```

Puis instanciation dans `ngAfterViewInit` avec `await` (ou via signal/observable).

## 8.3 Styles et assets

Certaines libs exigent CSS/fonts/images.

- Import global via `styles` (Angular CLI) ou `@import`.
- Ou encapsuler via styles du composant si possible.

---

# 9) SSR / Angular Universal : rendre l’intégration compatible serveur

## 9.1 Le problème

En SSR, `window`, `document`, `HTMLElement`, etc. n’existent pas.

## 9.2 Garde via `isPlatformBrowser`

```ts
import { isPlatformBrowser } from '@angular/common';
import { Inject, PLATFORM_ID } from '@angular/core';

constructor(@Inject(PLATFORM_ID) private platformId: object) {}

ngAfterViewInit() {
  if (!isPlatformBrowser(this.platformId)) return;
  // init lib
}
```

## 9.3 Stratégie recommandée

- Éviter l’init côté serveur.
- Fournir un rendu fallback (placeholder) dans le template.

---

# 10) Qualité & robustesse

## 10.1 Nettoyage (memory leaks)

Checklist :

- `destroy()` / `dispose()` appelé dans `ngOnDestroy`.
- Stopper timers (`setInterval`), observers, subscriptions.
- Retirer listeners si la lib ne le fait pas.

## 10.2 Gestion d’erreurs et résilience

- Envelopper l’initialisation dans un `try/catch`.
- Exposer un `@Output() error` si besoin.
- Dégrader gracieusement : message si la lib ne charge pas.

## 10.3 Tests

- **Unit tests**: mocker la lib (façade injectable) ou mock global.
- **Integration tests**: vérifier le wrapper (création/destroy) et événements.

Exemple de façade injectable :

```ts
export abstract class WidgetFactory {
  abstract create(host: HTMLElement, options: any): ThirdPartyWidget;
}
```

Puis fournir une implémentation réelle en prod et une mock en test.

---

# Ateliers (proposés)

## Atelier A — Construire un wrapper composant propre

**Objectif** : encapsuler une lib UI impérative.

### Étapes

1. Créer `ThirdPartyWidgetComponent`.
2. Ajouter `@Input() options` et `@Output() changed`.
3. Init dans `ngAfterViewInit`.
4. Update dans `ngOnChanges`.
5. Destroy dans `ngOnDestroy`.
6. Optimiser avec `NgZone.runOutsideAngular`.

### Critères de réussite

- Pas d’accès DOM global.
- Aucune fuite mémoire.
- API Angular documentée.

## Atelier B — Transformer un widget en champ de formulaire (optionnel)

Quand la lib représente un input, implémenter `ControlValueAccessor`.

Pistes :

- `writeValue`, `registerOnChange`, `registerOnTouched`.
- Retours d’événements de la lib vers Angular.

---

# Checklist pratique (à réutiliser)

- [ ] La lib est-elle **DOM-based** ou **SDK pure** ?
- [ ] Choix : composant / directive / service.
- [ ] Point d’ancrage DOM local (`#host`) uniquement.
- [ ] Init en `ngAfterViewInit`.
- [ ] Update en `ngOnChanges` (ou via setter d’`@Input`).
- [ ] Nettoyage en `ngOnDestroy`.
- [ ] Éviter DOM direct : utiliser `Renderer2` si manipulation nécessaire.
- [ ] Performance : `NgZone.runOutsideAngular` pour événements fréquents.
- [ ] SSR : garder via `isPlatformBrowser`.
- [ ] Typings et façade pour test.

---

# Conclusion

L’intégration d’une bibliothèque tierce non Angular doit être considérée comme une **frontière** : on limite la surface de contact, on contrôle l’accès au DOM, on respecte le cycle de vie Angular et on encapsule la dépendance (composant/directive/service). En appliquant ces principes, on obtient une intégration plus stable, testable, performante et maintenable.
