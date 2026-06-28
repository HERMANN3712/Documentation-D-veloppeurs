# Formation Angular — Performance et optimisation

- **Référence** : 19
- **Intitulé** : Performance et optimisation
- **Public** : Développeurs Angular (intermédiaire → avancé)
- **Pré-requis** : TypeScript, Angular (components, services, routing), RxJS de base
- **Durée conseillée** : 1 jour (7h) ou 2 demi-journées
- **Format** : cours + démos + ateliers guidés + exercices

---

## Objectifs pédagogiques

À l’issue de la formation, le participant saura :

1. **Diagnostiquer** les causes courantes de lenteur dans une application Angular.
2. **Maîtriser le Change Detection** (détection de changements) et ses impacts.
3. **Appliquer la stratégie `OnPush`** correctement et de manière robuste.
4. **Mettre en place le lazy loading** (chargement différé) de manière efficace.
5. **Optimiser les templates** (boucles, pipes, binding, rendering) pour réduire les re-renders.
6. Utiliser des **bonnes pratiques** de mesure (profiling, audits) et de prévention des régressions.

---

## Plan détaillé

1. **Introduction & fondamentaux de performance Angular**
2. **Change Detection (détection de changements)**
   - Mécanisme, zones, cycles CD
   - Réduction du coût de CD
3. **Stratégie `ChangeDetectionStrategy.OnPush`**
   - Principes, contraintes, patterns
   - Cas d’usage réels
4. **Lazy loading**
   - Modules vs routes standalone
   - Préchargement, découpage et stratégie
5. **Optimisation des templates**
   - `trackBy`, pipes, `async`, `ng-container`
   - Déférer l’affichage, réduire le DOM, éviter les recalculs
6. **Mesure, outillage et checklist**
   - DevTools, Angular DevTools, Lighthouse
   - Indicateurs, budget de performance, bonnes pratiques
7. **Atelier de synthèse** : refactor d’un écran lent

---

# 1) Introduction & fondamentaux

## 1.1. Qu’est-ce que “la performance” côté front ?

Trois axes principaux :

- **Temps de chargement** : combien de temps avant de voir/ utiliser le contenu.
- **Réactivité** : latence entre interaction (clic, scroll) et réponse UI.
- **Fluidité** : animation, scroll, rendu — éviter les “janks”.

### Métriques utiles

- **TTFB** (Time To First Byte)
- **FCP** (First Contentful Paint)
- **LCP** (Largest Contentful Paint)
- **TBT** (Total Blocking Time)
- **CLS** (Cumulative Layout Shift)
- **INP** (Interaction to Next Paint)

> Objectif : maximiser la perception utilisateur. Un bundle plus petit aide, mais une UI stable et réactive est souvent prioritaire.

## 1.2. D’où viennent les lenteurs dans Angular ?

Causes fréquentes :

- Trop de **Change Detection** (CD) déclenché trop souvent.
- Templates avec gros volumes : `*ngFor` massifs, DOM lourd.
- Fonctions appelées dans le template (réévaluées très souvent).
- Pipes non “pure” (ou calculs coûteux) déclenchés trop fréquemment.
- Chargement initial trop gros (pas de lazy loading, dépendances lourdes).
- Multiples subscriptions, flux RxJS mal maîtrisés.

---

# 2) Change Detection : comprendre et optimiser

## 2.1. Comment Angular sait qu’il doit mettre à jour l’UI ?

Angular met à jour l’interface en parcourant l’arbre des composants et en comparant l’état courant (modèle) avec ce qui est affiché.

- Par défaut, Angular utilise une stratégie de CD **“CheckAlways”** (implicite).
- Au déclenchement, Angular vérifie le composant et ses enfants.

### Déclencheurs courants du Change Detection

- Événement utilisateur : click, input, scroll (selon binding)
- `setTimeout`, `setInterval`
- Promises résolues
- Requêtes HTTP terminées
- Émissions RxJS (via `async` pipe ou subscriptions)

Ces triggers passent souvent via **Zone.js**, qui intercepte les tâches async et informe Angular.

## 2.2. Pourquoi le CD coûte-t-il cher ?

Le CD peut devenir coûteux si :

- il se déclenche trop fréquemment
- chaque cycle vérifie un grand arbre de composants
- des expressions lourdes sont évaluées dans les templates

> Un cycle de CD doit rester “cheap”. Le rendre cheap est souvent plus efficace que de le faire disparaître.

## 2.3. Réduire le coût du CD

### Bonnes pratiques générales

- **Éviter les fonctions dans les templates**
  - Pré-calculer dans le composant, ou utiliser un pipe **pure**.
- **Limiter le DOM** (surtout dans les listes)
- Favoriser `async` pipe plutôt que `subscribe()` manuel (réduit le risque de triggering inutile et gère le teardown)

### Démo (anti-pattern) : fonction dans le template

```ts
// component.ts
items = [...];

getTotalPrice(): number {
  // Calcul coûteux
  return this.items.reduce((acc, i) => acc + i.price, 0);
}
```

```html
<!-- component.html -->
<p>Total: {{ getTotalPrice() }}</p>
```

Problème : `getTotalPrice()` peut être appelé à chaque cycle CD.

**Amélioration**

```ts
totalPrice = 0;

ngOnInit() {
  this.totalPrice = this.items.reduce((acc, i) => acc + i.price, 0);
}
```

ou pipe pure :

```ts
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({ name: 'totalPrice', standalone: true, pure: true })
export class TotalPricePipe implements PipeTransform {
  transform(items: Array<{price: number}>): number {
    return items.reduce((acc, i) => acc + i.price, 0);
  }
}
```

```html
<p>Total: {{ items | totalPrice }}</p>
```

---

# 3) Stratégie OnPush : rendre le CD prévisible

## 3.1. Principe

`OnPush` demande à Angular de ne vérifier le composant **que lorsque c’est nécessaire**.

```ts
import { ChangeDetectionStrategy, Component } from '@angular/core';

@Component({
  selector: 'app-user-card',
  templateUrl: './user-card.component.html',
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class UserCardComponent {
  user!: { id: number; name: string };
}
```

Avec `OnPush`, Angular vérifie le composant quand :

1. Un **@Input** change de **référence** (immutabilité)
2. Un événement émis par le composant (click, input…) survient
3. Un observable lié par `async` pipe émet
4. On déclenche manuellement via `markForCheck()` / `detectChanges()`

## 3.2. Immutabilité : clé de voûte

### Erreur classique

```ts
// parent.ts
this.user.name = 'Nouveau nom'; // Mutation → même référence
```

En `OnPush`, le child peut ne pas se rafraîchir.

### Correct

```ts
this.user = { ...this.user, name: 'Nouveau nom' }; // Nouvelle référence
```

### Recommandations

- Utiliser des patterns immutables : spread, `map`, `filter`
- Pour des états complexes : envisager un store (NgRx, Akita, SignalStore) ou signaux selon contexte

## 3.3. `markForCheck()` vs `detectChanges()`

- `markForCheck()` : marque le composant pour vérification au prochain cycle CD.
- `detectChanges()` : déclenche immédiatement une vérification locale.

Exemple (cas rare) : callback externe non gérée par zone.

```ts
import { ChangeDetectorRef, Component } from '@angular/core';

@Component({
  selector: 'app-external-widget',
  template: `{{ value }}`,
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class ExternalWidgetComponent {
  value = 0;

  constructor(private cdr: ChangeDetectorRef) {}

  ngOnInit() {
    someExternalLibrary.onTick((v: number) => {
      this.value = v;
      this.cdr.markForCheck();
    });
  }
}
```

## 3.4. Pattern de composition : OnPush partout ?

Recommandation pragmatique :

- Mettre `OnPush` sur les **composants de présentation** (dumb components).
- Garder une stratégie par défaut sur certains **containers** si la complexité d’état est élevée…
- …ou accepter `OnPush` partout en appliquant rigoureusement l’immutabilité + `async`.

---

# 4) Lazy loading : réduire le coût initial

## 4.1. Objectif

- Charger moins de code au démarrage
- Découper l’app selon routes / features
- Réduire le “time to interactive”

## 4.2. Lazy loading avec le Router

### Cas 1 : Lazy loading d’un module (approche historique)

```ts
const routes: Routes = [
  {
    path: 'admin',
    loadChildren: () =>
      import('./admin/admin.module').then(m => m.AdminModule),
  },
];
```

### Cas 2 : Lazy loading avec composants standalone (recommandé)

```ts
const routes: Routes = [
  {
    path: 'admin',
    loadComponent: () =>
      import('./admin/admin.page').then(m => m.AdminPage),
  },
];
```

### Lazy loading d’un ensemble de routes

```ts
const routes: Routes = [
  {
    path: 'products',
    loadChildren: () =>
      import('./products/products.routes').then(r => r.PRODUCTS_ROUTES),
  },
];
```

## 4.3. Découpage (chunking) : bonnes pratiques

- Chunk par **feature** (ex: admin, billing, reporting)
- Éviter le lazy trop fin (trop de requêtes, complexité)
- Surveiller les dépendances partagées (vendor chunk)

## 4.4. Préchargement (Preloading)

Précharger les routes en arrière-plan après le chargement initial.

```ts
import { PreloadAllModules, provideRouter, withPreloading } from '@angular/router';

bootstrapApplication(AppComponent, {
  providers: [
    provideRouter(routes, withPreloading(PreloadAllModules)),
  ]
});
```

Stratégie personnalisée (précharger seulement certaines routes) :

```ts
import { Injectable } from '@angular/core';
import { PreloadingStrategy, Route } from '@angular/router';
import { Observable, of } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class SelectivePreloadingStrategy implements PreloadingStrategy {
  preload(route: Route, load: () => Observable<any>): Observable<any> {
    return route.data?.['preload'] ? load() : of(null);
  }
}
```

---

# 5) Optimisation des templates

## 5.1. `*ngFor` : `trackBy` obligatoire sur listes dynamiques

Sans `trackBy`, Angular peut recréer inutilement des éléments DOM.

```html
<li *ngFor="let item of items; trackBy: trackById">
  {{ item.name }}
</li>
```

```ts
trackById(index: number, item: { id: number }) {
  return item.id;
}
```

Effet : améliore performance sur tri, pagination, mise à jour partielle.

## 5.2. Éviter les expressions coûteuses

### Anti-pattern

```html
<div *ngFor="let u of users">
  {{ u.firstName + ' ' + u.lastName }}
  <span>{{ formatAddress(u.address) }}</span>
</div>
```

### Better

- Préparer un champ `fullName`
- Utiliser un pipe pure `addressFormat`
- Ou calculer dans un selector/store

## 5.3. Pipes : pure vs impure

- **Pure pipe** (par défaut) : recalculée seulement si la référence d’entrée change.
- **Impure pipe** (`pure: false`) : recalculée à chaque cycle CD → à éviter sauf nécessité.

Exemple impure (souvent problématique) :

```ts
@Pipe({ name: 'now', pure: false, standalone: true })
export class NowPipe implements PipeTransform {
  transform(): number { return Date.now(); }
}
```

Si vous devez afficher une horloge : mieux via `interval` + `async`.

## 5.4. `async` pipe : performance + gestion mémoire

Préférer :

```html
<div *ngIf="user$ | async as user">
  {{ user.name }}
</div>
```

Plutôt que :

```ts
user?: User;
sub = this.user$.subscribe(u => this.user = u);
```

Bénéfices :

- unsubscribe automatique
- intégration naturelle avec `OnPush`

## 5.5. Réduire le DOM : `ng-container`, `ng-template`

- `ng-container` n’ajoute pas de nœud au DOM
- utile pour éviter des wrappers inutiles

```html
<ng-container *ngIf="vm$ | async as vm">
  <app-header [title]="vm.title" />
  <app-list [items]="vm.items" />
</ng-container>
```

## 5.6. Optimiser les images et listes longues (bonus utile)

Même si hors du strict Angular, fortement impactant :

- Utiliser `loading="lazy"` sur images
- Pagination / virtual scrolling (CDK Virtual Scroll) pour longues listes

---

# 6) Mesure, outillage et checklist

## 6.1. Mesurer avant d’optimiser

Approche recommandée :

1. Observer symptôme (lenteur, freeze, jank)
2. Identifier la zone concernée
3. Mesurer (profiling)
4. Appliquer une optimisation ciblée
5. Mesurer à nouveau

## 6.2. Outils

- **Chrome DevTools Performance** : CPU profile, frames, scripting time
- **Network** : tailles des chunks, waterfalls
- **Angular DevTools** : profiler, CD cycles, component tree
- **Lighthouse** : audits, métriques web

## 6.3. Checklist de performance Angular

### Change Detection

- [ ] Éviter fonctions dans template
- [ ] `OnPush` sur composants de présentation
- [ ] Immutabilité sur `@Input()`
- [ ] `async` pipe privilégié

### Templates

- [ ] `trackBy` sur les listes
- [ ] Pipes pure pour calculs
- [ ] DOM minimal (`ng-container`)

### Chargement

- [ ] Lazy loading par feature
- [ ] Préchargement contrôlé
- [ ] Analyse bundles (taille, dépendances)

---

# 7) Ateliers (guidés)

## Atelier 1 — Diagnostiquer une page lente

**Contexte** : une page liste 1 000 éléments, filtre en temps réel, et affiche des totaux.

### Étapes

1. Mesurer avec Angular DevTools : nombre de CD/s, temps de rendu
2. Identifier les hotspots : fonction dans template, absence de `trackBy`
3. Appliquer optimisations :
   - `trackBy`
   - suppression fonctions template
   - `OnPush` sur items
4. Re-mesurer et comparer

**Résultat attendu** : réduction du temps de scripting et rendu plus fluide.

## Atelier 2 — Passer un feature en Lazy loading

1. Identifier feature “admin”
2. Créer une route lazy (standalone ou routes)
3. Vérifier que le bundle principal diminue
4. Option : preloading sélectif

---

# Annexes

## A) Exemple complet : liste optimisée avec OnPush + async + trackBy

```ts
import { ChangeDetectionStrategy, Component, Input } from '@angular/core';
import { CommonModule } from '@angular/common';

export interface Product {
  id: number;
  name: string;
  price: number;
}

@Component({
  selector: 'app-product-list',
  standalone: true,
  imports: [CommonModule],
  template: `
    <ul>
      <li *ngFor="let p of products; trackBy: trackById">
        {{ p.name }} — {{ p.price | number:'1.2-2' }}
      </li>
    </ul>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class ProductListComponent {
  @Input() products: Product[] = [];

  trackById = (_: number, p: Product) => p.id;
}
```

## B) Pièges fréquents

- Utiliser `OnPush` sans immutabilité → UI “bloquée”
- Pipes impures pour des calculs → recalcul permanent
- Tout lazy-load en micro-chunks → trop de requêtes, UX dégradée

---

## Conclusion

L’optimisation Angular se résume souvent à :

1. **Rendre la CD moins fréquente** (OnPush, async)
2. **Rendre chaque cycle plus cheap** (template minimal, trackBy, éviter fonctions)
3. **Réduire le coût initial** (lazy loading + stratégie de préchargement)

Fin de formation.
