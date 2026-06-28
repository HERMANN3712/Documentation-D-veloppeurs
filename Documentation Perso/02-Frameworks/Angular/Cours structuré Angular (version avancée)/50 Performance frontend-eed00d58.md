# Formation Angular — Performance Frontend

> **Public visé** : développeurs Angular (intermédiaires → avancés), équipes produit/tech.
>
> **Format** : 1 journée (7h) ou 2 demi‑journées. Adaptable.
>
> **Objectifs pédagogiques**
>
> - Comprendre d’où viennent les lenteurs (chargement initial, rendu, change detection, runtime JS, DOM).
> - Mettre en œuvre les leviers clés : **lazy loading**, **ChangeDetectionStrategy.OnPush**, **suivi des bundles**, **réduction du JavaScript initial**, **limitation des watchers implicites**, **usage raisonnable des pipes**, **rendu performant de listes longues**.
> - Savoir mesurer (profiling) avant/après et éviter les régressions.
>
> **Pré‑requis**
>
> - Angular CLI, modules/standalone, routing, RxJS de base.
> - Node + Chrome DevTools.

---

## Plan de la formation

1. **Comprendre la performance frontend**
   - 1.1 Notions clés : TTI, FCP/LCP/CLS, long tasks, interaction latency
   - 1.2 Les goulots d’étranglement typiques Angular
   - 1.3 Méthodologie mesure → hypothèse → optimisation → validation
2. **Mesurer et diagnostiquer**
   - 2.1 Chrome DevTools (Performance, Network, Coverage)
   - 2.2 Angular DevTools (Change detection, profiler)
   - 2.3 Budgets Angular + Lighthouse
3. **Lazy loading : réduire le coût du chargement initial**
   - 3.1 Lazy loading via Router
   - 3.2 Préchargement (PreloadingStrategy)
   - 3.3 Boundary design : découper par routes/feature
4. **OnPush : accélérer le rendu en contrôlant la change detection**
   - 4.1 Change detection : comment Angular décide de re‑rendre
   - 4.2 OnPush et immutabilité
   - 4.3 AsyncPipe, Signals (si applicable), et déclenchement contrôlé
   - 4.4 Cas difficiles : inputs mutés, événements, zones, CD manuelle
5. **Suivi des bundles : comprendre ce qu’on envoie au navigateur**
   - 5.1 Stats Webpack/Vite et sources de dépendances
   - 5.2 Analyse (source-map-explorer / webpack-bundle-analyzer)
   - 5.3 Déduplication, tree-shaking, gestion des polyfills
6. **Réduction du JavaScript initial**
   - 6.1 Stratégies : code splitting, features optionnelles, chargement conditionnel
   - 6.2 Optimiser les dépendances (moment → date-fns, lodash-es, etc.)
   - 6.3 SSR/Prerender (option) et impact sur performance perçue
7. **Limiter les watchers implicites**
   - 7.1 Ce que recouvrent “watchers”/observateurs côté template
   - 7.2 Éviter les expressions coûteuses en template
   - 7.3 `*ngIf`, `*ngFor`, `ngClass`/`ngStyle` : bonnes pratiques
   - 7.4 RxJS : éviter les subscriptions multiples & recalculs
8. **Usage raisonnable des pipes**
   - 8.1 Pipes purs vs impurs
   - 8.2 Quand un pipe devient un goulet
   - 8.3 Alternatives : pré-calcul, memoization, computed signals
9. **Maîtriser le rendu des listes longues**
   - 9.1 `trackBy` indispensable
   - 9.2 Virtual scrolling (CDK)
   - 9.3 Pagination, incremental rendering, skeleton UI
10. **Atelier final : audit et plan d’amélioration**
   - 10.1 Checklist d’audit
   - 10.2 Plan d’action par ROI
   - 10.3 Mise en place anti-régression (budgets, CI)

---

## 1) Comprendre la performance frontend

### 1.1 Indicateurs clés

- **FCP (First Contentful Paint)** : premier affichage de contenu.
- **LCP (Largest Contentful Paint)** : affichage du “gros” contenu (souvent héro/entête). Important pour perception.
- **TTI (Time To Interactive)** : moment où l’app est réellement utilisable (plus difficile à définir précisément, mais utile comme notion).
- **TBT (Total Blocking Time)** : temps où le thread principal est bloqué (long tasks JS).
- **INP (Interaction to Next Paint)** : latence d’interaction (remplace progressivement FID).
- **CLS** : stabilité visuelle.

**À retenir** : Angular peut être rapide, mais le coût principal peut venir de :
1) **trop de JavaScript initial** (bundle trop gros),
2) **rendu trop fréquemment** (change detection),
3) **DOM trop lourd** (listes),
4) **calculs répétés** dans les templates/pipes.

### 1.2 Goulots d’étranglement typiques Angular

- **Chargement initial** : gros bundle, dépendances lourdes, polyfills inutiles.
- **Runtime** : change detection déclenchée trop souvent, templates qui font des calculs.
- **Rendu** : listes, composants imbriqués, CSS/layout reflow.

### 1.3 Méthodologie

1. **Mesurer** (baseline) : Lighthouse + DevTools.
2. **Identifier** : Network (poids), Performance (long tasks), Angular DevTools (CD cycles).
3. **Optimiser** une hypothèse à la fois.
4. **Valider** : comparaison avant/après, surveillance en CI.

---

## 2) Mesurer et diagnostiquer

### 2.1 Chrome DevTools

- **Network** : poids des JS, nombre de requêtes, cache.
- **Performance** : long tasks, scripting vs rendering.
- **Coverage** : JS/CSS non utilisés (excellent pour repérer du code chargé “pour rien”).

**Exercice** :
- Ouvrir un build production (`ng build --configuration production`).
- Servir localement (`npx http-server dist/...`).
- Lancez Lighthouse puis inspectez Coverage.

### 2.2 Angular DevTools

- **Profiler** : temps de rendering, détecter les composants qui re‑rendent souvent.
- **Change detection** : voir si un arbre se met à jour trop fréquemment.

### 2.3 Budgets et Lighthouse

Dans `angular.json`, définir des **budgets** (exemples à adapter) :

```json
"budgets": [
  {
    "type": "initial",
    "maximumWarning": "250kb",
    "maximumError": "350kb"
  },
  {
    "type": "anyComponentStyle",
    "maximumWarning": "6kb",
    "maximumError": "10kb"
  }
]
```

**But** : échouer la build CI si le bundle explose.

---

## 3) Lazy loading : réduire le coût du chargement initial

### 3.1 Lazy loading via Router

**Objectif** : ne charger que le strict nécessaire au démarrage.

Exemple (routes lazy) :

```ts
import { Routes } from '@angular/router';

export const routes: Routes = [
  {
    path: 'admin',
    loadChildren: () => import('./admin/admin.routes').then(m => m.ADMIN_ROUTES)
  },
  {
    path: '',
    loadComponent: () => import('./home/home.component').then(c => c.HomeComponent)
  }
];
```

**Bonnes pratiques** :
- Lazy load **les sections** (admin, reporting, back-office, wizard, etc.).
- Éviter le lazy loading trop fin (micro‑chunks) : trop de requêtes + overhead.

### 3.2 Préchargement (PreloadingStrategy)

**Objectif** : combiner TTI rapide + navigation fluide.

```ts
import { provideRouter, withPreloading, PreloadAllModules } from '@angular/router';

bootstrapApplication(AppComponent, {
  providers: [
    provideRouter(routes, withPreloading(PreloadAllModules))
  ]
});
```

**Approche avancée** : préchargement conditionnel (wifi, idle time) via stratégie custom.

### 3.3 Boundary design : découper par feature

- Un “feature boundary” regroupe : routes, composants, services, assets.
- Les dépendances lourdes (charts, editor WYSIWYG) doivent être confinées dans un bundle lazy.

**Anti‑patterns** :
- Importer une lib lourde dans un util partagé consommé par l’écran d’accueil.
- Avoir un module partagé qui ré-exporte tout (le “God SharedModule”).

---

## 4) OnPush : accélérer le rendu en contrôlant la change detection

### 4.1 Comment Angular met à jour l’UI

- Par défaut, Angular vérifie fréquemment les composants (change detection) suite à :
  - événements (click, input),
  - timers,
  - HTTP promises,
  - émissions RxJS,
  - etc.

**Coût** : plus l’arbre de composants est grand, plus chaque cycle peut coûter.

### 4.2 Passer en `ChangeDetectionStrategy.OnPush`

```ts
import { ChangeDetectionStrategy, Component, input } from '@angular/core';

@Component({
  selector: 'app-user-card',
  templateUrl: './user-card.component.html',
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class UserCardComponent {
  user = input.required<{ id: number; name: string }>();
}
```

**Principe** : avec OnPush, Angular re‑rend le composant si :
- un **Input change de référence**,
- un événement survient **dans** le composant,
- un `async` pipe émet,
- on déclenche manuellement via `markForCheck()`.

### 4.3 Immutabilité : condition de réussite

**Problème classique** : mutation d’objet.

```ts
// Mauvais : même référence, OnPush ne voit pas toujours la mise à jour
this.user.name = 'Nouveau nom';

// Bon : nouvelle référence
this.user = { ...this.user, name: 'Nouveau nom' };
```

Pour des listes :

```ts
// Bon
this.items = this.items.map(i => i.id === id ? { ...i, done: true } : i);
```

### 4.4 `AsyncPipe` : votre meilleur allié

```html
<section *ngIf="user$ | async as user">
  <app-user-card [user]="user" />
</section>
```

- Évite `subscribe()` manuel.
- Déclenche la CD au bon endroit.
- Gère l’unsubscribe automatiquement.

### 4.5 Forcer/contrôler la CD

Dans certains cas (lib externe, callback non géré) :

```ts
constructor(private cdr: ChangeDetectorRef) {}

updateFromOutside() {
  // ...mise à jour data
  this.cdr.markForCheck();
}
```

**Attention** : `detectChanges()` est plus “agressif” (sync) ; `markForCheck()` est souvent préférable.

---

## 5) Suivi des bundles : comprendre ce qu’on envoie

### 5.1 Pourquoi analyser

- La perf est souvent une histoire de **poids** et de **code inutile**.
- Un bundle gonfle par : dépendances, locales i18n, fonctions non tree‑shakées, polyfills.

### 5.2 Outils d’analyse

Options fréquentes :
- `source-map-explorer`
- `webpack-bundle-analyzer`

Exemple :

```bash
ng build --configuration production --source-map
npx source-map-explorer dist/**/browser/*.js
```

### 5.3 Actions typiques

- Remplacer libs lourdes :
  - `moment` → `date-fns` / `dayjs`
  - `lodash` → imports ciblés ou `lodash-es`
- Importer en mode “à la carte” au lieu de tout importer.
- Éviter les ré-export massifs.

---

## 6) Réduction du JavaScript initial

### 6.1 Stratégies

- **Lazy loading** des features (déjà vu).
- Charger conditionnellement :
  - éditeur riche uniquement dans `/admin/content`.
  - charts uniquement dans `/analytics`.

### 6.2 Dynamic import ciblé

```ts
async openEditor() {
  const { RichEditorComponent } = await import('./rich-editor/rich-editor.component');
  // ...utiliser ou router vers une route lazy
}
```

### 6.3 SSR / Prerender (option)

- **SSR** améliore souvent **FCP/LCP** (HTML déjà présent), mais ne supprime pas le coût JS.
- Peut réduire la perception de lenteur et améliorer SEO.

**Point de vigilance** : hydratation, libs non compatibles SSR, temps serveur.

---

## 7) Limiter les watchers implicites (templates)

Dans Angular, on parle moins de “watchers” que dans AngularJS, mais il existe des **observations implicites** :
- chaque binding `{{ expr }}` est ré‑évalué lors d’un cycle,
- chaque appel de fonction depuis le template est ré‑exécuté,
- certaines structures (`ngFor`, `ngClass`, etc.) comparent/itèrent.

### 7.1 Ne pas appeler de fonctions dans le template

```html
<!-- À éviter : appelée à chaque CD -->
<div>{{ computeTotal() }}</div>
```

Préférer :
- un champ calculé mis à jour quand il faut,
- un `computed` (signals) / memoization,
- un pipe **pur** simple.

```ts
total = 0;
ngOnChanges() {
  this.total = this.items.reduce((a, b) => a + b.price, 0);
}
```

### 7.2 Stabiliser les références

Si vous créez des objets inline :

```html
<!-- Évite : nouvel objet à chaque CD -->
<app-widget [config]="{ theme: 'dark' }" />
```

Préférer :

```ts
readonly config = { theme: 'dark' };
```

### 7.3 `*ngIf` / `*ngFor` : réduire la charge

- Éviter de rendre ce qui n’est pas visible.
- Découper la page : composants OnPush + sections conditionnelles.
- Pour `ngFor`, toujours réfléchir à `trackBy` (voir section 9).

### 7.4 RxJS : éviter les recomputations

- Partager un flux avec `shareReplay({ bufferSize: 1, refCount: true })` si plusieurs abonnés.
- Éviter de refaire des `map/filter` lourds à chaque souscription.

---

## 8) Usage raisonnable des pipes

### 8.1 Pipes purs vs impurs

- **Pipe pur (par défaut)** : recalcul uniquement si la **référence** des paramètres change.
- **Pipe impur** (`pure: false`) : recalcul à **chaque** cycle de CD → cher.

```ts
@Pipe({ name: 'formatUser', pure: true })
export class FormatUserPipe {
  transform(user: { name: string; role: string }) {
    return `${user.name} (${user.role})`;
  }
}
```

### 8.2 Quand un pipe devient un goulet

- Pipe appliqué dans un `ngFor` sur 500 éléments.
- Pipe qui trie/filtre une liste à chaque CD.

**Anti‑pattern fréquent** :

```html
<li *ngFor="let u of users | sortBy:'name'">
  {{ u.name }}
</li>
```

### 8.3 Alternatives

- Trier/filtrer **dans le composant** au moment où les données changent.
- Utiliser des sélecteurs memoizés (NgRx) ou `computed` (signals).
- Pagination / virtual scroll.

---

## 9) Maîtriser le rendu des listes longues

### 9.1 `trackBy` : éviter le re‑création DOM

```html
<div *ngFor="let item of items; trackBy: trackById">
  <app-item-row [item]="item" />
</div>
```

```ts
trackById = (_: number, item: { id: number }) => item.id;
```

**Effet** : Angular réutilise les nœuds DOM au lieu de tout détruire/recréer.

### 9.2 Virtual scrolling (Angular CDK)

Quand vous affichez des centaines/milliers d’éléments : ne rendez que ce qui est visible.

```html
<cdk-virtual-scroll-viewport itemSize="48" class="viewport">
  <div *cdkVirtualFor="let item of items; trackBy: trackById" class="row">
    {{ item.label }}
  </div>
</cdk-virtual-scroll-viewport>
```

CSS minimal :

```css
.viewport { height: 400px; width: 100%; }
.row { height: 48px; display: flex; align-items: center; }
```

### 9.3 Stratégies complémentaires

- **Pagination** (serveur ou client) pour réduire la taille de rendu.
- **Incremental rendering** : afficher par paquets.
- **Skeleton / placeholder** : améliore la perception.

---

## 10) Atelier final : audit et plan d’amélioration

### 10.1 Checklist d’audit (rapide)

**Chargement**
- [ ] Route d’accueil : bundle initial sous budget
- [ ] Features lourdes lazy load
- [ ] Dépendances analysées et justifiées

**Rendu**
- [ ] OnPush utilisé largement sur composants “présentation”
- [ ] Pas de fonctions coûteuses dans template
- [ ] Pas d’objets/arrays inline dans template

**Listes**
- [ ] `trackBy` partout
- [ ] Virtual scroll si > ~200 items visibles

**Pipes**
- [ ] Pas de pipes impurs non justifiés
- [ ] Pas de tri/filtre en pipe sur grosses listes

**CI**
- [ ] Budgets Angular configurés
- [ ] Lighthouse CI (option) avec seuils

### 10.2 Plan d’action par ROI

1. **Lazy load** les zones secondaires + préchargement → gros gain UX.
2. **OnPush + AsyncPipe** sur composants denses → gros gain CPU.
3. **Bundles** : supprimer dépendances inutiles → gain réseau + parse.
4. **Listes longues** : trackBy + virtual scroll → gain DOM.
5. **Templates/pipes** : enlever calculs répétitifs → gain CPU.

### 10.3 Anti-régressions

- Activer budgets.
- Ajouter une étape CI : build prod + analyse taille.
- Option : Lighthouse CI sur pages clés.

---

## Annexes

### A) Mini guide de troubleshooting

- **Lent au démarrage** : regarder bundle initial, polyfills, lazy loading.
- **Freeze lors d’un click** : long task JS ; profiler ; vérifier pipes/fonctions templates.
- **Scroll saccadé** : listes longues ; virtual scroll ; CSS (box-shadow, layout thrash).
- **Navigation lente** : précharger routes, cache HTTP, resolver trop lourd.

### B) Exemple de “règles d’équipe”

- Composants UI → OnPush par défaut.
- Interdiction des pipes impurs sans validation performance.
- Pas de fonctions dans le template (sauf trivial et prouvé).
- `trackBy` obligatoire.
- Analyse bundle à chaque release.

### C) Exercices proposés (si atelier)

1. **Mesurer** une app (Lighthouse + Coverage) : identifier 3 pistes.
2. **Migrer** une feature en lazy loading et comparer (TTI, taille bundle initial).
3. **Passer** une page en OnPush + refactor immutabilité.
4. **Optimiser** une liste de 1000 éléments (trackBy + virtual scroll).

---

## Livrables attendus (fin de formation)

- Un audit initial + capture des métriques.
- Un plan d’optimisation priorisé.
- Une base de conventions performance Angular pour l’équipe.
