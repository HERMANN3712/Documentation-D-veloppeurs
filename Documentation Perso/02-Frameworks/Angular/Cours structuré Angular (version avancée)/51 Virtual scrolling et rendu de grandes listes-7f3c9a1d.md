# Formation Angular : Virtual Scrolling et rendu de grandes listes

## Objectifs pédagogiques
À l’issue de cette formation, vous serez capable de :

- Comprendre pourquoi le rendu d’une longue liste coûte cher (DOM, change detection, layout/reflow).
- Mettre en place le **virtual scrolling** avec **Angular CDK**.
- Choisir les bons patterns (trackBy, OnPush, pagination vs virtualisation).
- Gérer des cas réels : items à hauteur variable, chargement infini, filtres/tri, accessibilité.
- Mesurer et diagnostiquer les performances (profiling, FPS, timings, mémoire).

## Public visé & prérequis
- Développeurs Angular intermédiaires
- Connaissances requises : composants, templates, RxJS de base, services, modules/standalone.

## Durée suggérée
- 1 journée (6–7h) ou 2 demi-journées

## Matériel / environnement
- Angular 16+ (idéalement 17+)
- Node LTS
- Angular CDK

---

# Plan de cours détaillé

1. **Contexte performance : pourquoi les grandes listes posent problème**
2. **Approches possibles** : pagination, infinite scroll, virtual scrolling (comparatif)
3. **Angular CDK Virtual Scroll : prise en main**
4. **Optimisations indispensables** : trackBy, OnPush, templates simples, réduction des watchers
5. **Cas avancés** : hauteurs variables, viewport dynamique, sticky headers
6. **Données et UX** : chargement progressif, skeletons, gestion du scroll, restauration de position
7. **Accessibilité & interactions** : clavier, focus, lecteurs d’écran
8. **Mesure & debugging** : DevTools, Angular DevTools, profiling, budgets
9. **Ateliers pratiques** (guidés) + corrigés

---

# 1) Contexte performance : pourquoi les grandes listes posent problème

## 1.1 Le coût de rendu d’une liste
Afficher 5 000–50 000 éléments dans le DOM entraîne :

- **Coût DOM** : création de milliers de nœuds, mémoire, GC.
- **Coût Angular** : binding, change detection, directives structurelles.
- **Coût navigateur** : style recalculation, layout, paint.

⚠️ Symptômes courants :
- Scroll saccadé (FPS qui chute)
- TTI (Time To Interactive) plus long
- blocages lors des filtres/tri

## 1.2 Règle d’or
> Ne rendez que ce qui est visible (et un petit buffer).

C’est le principe de la **virtualisation** : on simule une grande hauteur de contenu, mais on ne garde en DOM qu’un sous-ensemble d’items.

---

# 2) Approches possibles : comparatif

## 2.1 Pagination (server-side)
- ✅ Simple à implémenter, très robuste
- ✅ Adaptée aux API paginées
- ❌ UX parfois moins fluide (changement de page)

## 2.2 Infinite scroll (chargement progressif)
- ✅ UX fluide (défilement continu)
- ✅ Réduit le nombre d’éléments au départ
- ❌ Le DOM grossit quand même si on ne virtualise pas

## 2.3 Virtual scrolling
- ✅ DOM constant (ou quasi)
- ✅ Super fluide même avec de très grosses collections
- ❌ Complexifie certains cas (hauteurs variables, focus, SEO)

👉 En pratique : **virtual scrolling + chargement progressif** est souvent le meilleur combo.

---

# 3) Angular CDK Virtual Scroll : prise en main

## 3.1 Installation
```bash
ng add @angular/cdk
```

Import (NG modules) :
```ts
import { ScrollingModule } from '@angular/cdk/scrolling';

@NgModule({
  imports: [ScrollingModule]
})
export class AppModule {}
```

En standalone :
```ts
import { ScrollingModule } from '@angular/cdk/scrolling';

@Component({
  standalone: true,
  imports: [ScrollingModule],
  templateUrl: './users.component.html',
})
export class UsersComponent {}
```

## 3.2 Template minimal avec `cdk-virtual-scroll-viewport`
```html
<cdk-virtual-scroll-viewport
  class="viewport"
  itemSize="56"
  minBufferPx="280"
  maxBufferPx="560">

  <div class="row" *cdkVirtualFor="let user of users; trackBy: trackById">
    <span class="name">{{ user.name }}</span>
    <span class="email">{{ user.email }}</span>
  </div>
</cdk-virtual-scroll-viewport>
```

CSS indispensable (le viewport doit avoir une hauteur) :
```css
.viewport {
  height: 70vh;
  width: 100%;
  border: 1px solid #ddd;
}

.row {
  height: 56px; /* cohérent avec itemSize */
  display: flex;
  align-items: center;
  padding: 0 12px;
  box-sizing: border-box;
  border-bottom: 1px solid #f2f2f2;
}
```

## 3.3 Paramètres clés
- `itemSize` : hauteur **fixe** (px) de chaque item. C’est la base de performance.
- `minBufferPx` / `maxBufferPx` : buffer avant/après la zone visible pour éviter le “pop-in”.

### Recommandations
- Commencez avec `minBufferPx = 5 * itemSize` et `maxBufferPx = 10 * itemSize`.
- Mesurez sur devices réels.

---

# 4) Optimisations indispensables

## 4.1 `trackBy` : obligatoire
Sans `trackBy`, Angular détruit/recrée des composants DOM lors des changements.

```ts
trackById = (_: number, item: { id: string }) => item.id;
```

## 4.2 Change detection : `OnPush`
Pour des items (lignes) réutilisables :

```ts
@Component({
  selector: 'app-user-row',
  template: `...`,
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class UserRowComponent {
  @Input() user!: User;
}
```

Dans le viewport :
```html
<app-user-row *cdkVirtualFor="let user of users; trackBy: trackById" [user]="user" />
```

## 4.3 Réduire le coût template
- Éviter les fonctions dans le template (`{{ compute(x) }}`)
- Éviter les pipes non purs sur de gros volumes
- Pré-calculer côté TS ou via `computed`/signals

## 4.4 “DOM light”
- Limiter la profondeur (moins de wrappers)
- Éviter les composants trop lourds pour chaque ligne

---

# 5) Cas avancés

## 5.1 Hauteurs variables
La stratégie la plus performante est **hauteur fixe**.

Si vous avez des hauteurs variables (wrap de texte, cards…), options :

### Option A : Normaliser la hauteur
- Fixer une hauteur de ligne
- Troncature (`text-overflow: ellipsis`) ou clamp

### Option B : `AutoSizeVirtualScrollStrategy` (selon version)
Angular CDK propose des stratégies, mais elles sont plus complexes et parfois moins stables selon les besoins.

**Recommandation** : standardiser la hauteur quand c’est possible.

## 5.2 Viewport dynamique (resize)
Quand la taille de la fenêtre change, le viewport doit recalculer.

```ts
@ViewChild(CdkVirtualScrollViewport) viewport!: CdkVirtualScrollViewport;

@HostListener('window:resize')
onResize() {
  this.viewport.checkViewportSize();
}
```

## 5.3 En-têtes “sticky”
Deux approches :
- En-tête hors du viewport (simple, robuste)
- “Grouped lists” avec section headers (plus complexe)

---

# 6) Données & UX

## 6.1 Chargement progressif (infinite loading)
Objectif : quand on approche de la fin du dataset chargé, on fetch la suite.

```ts
readonly pageSize = 200;
private page = 0;

users: User[] = [];
loading = false;

loadNextPage() {
  if (this.loading) return;
  this.loading = true;

  this.api.getUsers({ page: this.page, size: this.pageSize }).subscribe({
    next: (newUsers) => {
      this.users = [...this.users, ...newUsers];
      this.page++;
      this.loading = false;
    },
    error: () => (this.loading = false),
  });
}
```

Dans le composant, détecter la fin approchée :

```ts
@ViewChild(CdkVirtualScrollViewport) viewport!: CdkVirtualScrollViewport;

ngAfterViewInit() {
  this.viewport.elementScrolled().pipe(
    debounceTime(50),
    map(() => this.viewport.measureScrollOffset('bottom')),
    filter((distanceToBottom) => distanceToBottom < 400)
  ).subscribe(() => this.loadNextPage());
}
```

## 6.2 Indicateurs de chargement
- Loader en haut/bas
- Skeleton rows (important pour perception de fluidité)

Exemple simple :
```html
<div class="loading" *ngIf="loading">Chargement…</div>
```

## 6.3 Restauration de position de scroll
Utile lors de navigation vers détail puis retour.

```ts
const offset = this.viewport.measureScrollOffset();
// stocker offset (service / router state)
this.viewport.scrollToOffset(offset);
```

---

# 7) Accessibilité & interactions

## 7.1 Focus clavier
Le virtual scroll détruit des éléments hors écran : le focus peut “disparaître”.

Bonnes pratiques :
- Gérer le focus coté donnée : conserver `focusedId`.
- Au scroll/rafraîchissement, restaurer le focus sur l’item rendu.

## 7.2 Lecteurs d’écran
- Fournir une structure sémantique (listbox, ul/li si pertinent)
- Annoncer les chargements (ARIA live si nécessaire)
- Ne pas surcharger avec trop de régions live

---

# 8) Mesure & debugging performance

## 8.1 Chrome DevTools
- Performance tab : main thread, layout, paint
- Memory tab : vérifier fuites, pics

## 8.2 Angular DevTools
- Profiler change detection
- Repérer les composants “chauds”

## 8.3 Indicateurs pratiques
- Nombre d’éléments DOM (Elements panel)
- FPS pendant le scroll
- Temps de réponse lors d’un filtre

---

# 9) Ateliers pratiques (guidés)

## Atelier 1 : Transformer une liste naïve en virtual scroll
### Énoncé
Vous avez un `*ngFor` affichant 10 000 utilisateurs. Le scroll est lent.

### Étapes
1. Remplacer `*ngFor` par `*cdkVirtualFor`.
2. Ajouter `cdk-virtual-scroll-viewport` + CSS hauteur.
3. Ajouter `trackBy`.
4. Mesurer le gain.

### Critères de réussite
- DOM stable (quelques dizaines/centaines de lignes max)
- Scroll fluide

## Atelier 2 : Infinite loading + virtual scroll
### Énoncé
Charger par pages de 200 éléments en approchant du bas.

### Étapes
1. Détecter la distance au bas avec `measureScrollOffset('bottom')`.
2. Déclencher `loadNextPage()` avec un seuil.
3. Ajouter un indicateur `loading`.

### Critères de réussite
- Pas de double chargement
- Dataset grandissant sans dégrader le scroll

## Atelier 3 : Optimiser un item coûteux
### Énoncé
Chaque ligne affiche un composant avec beaucoup de bindings.

### Étapes
1. Passer le composant ligne en `OnPush`.
2. Éliminer fonctions dans le template.
3. Ajouter `trackBy`.

---

# Annexes

## A) Checklist performance grandes listes
- [ ] Virtual scroll activé
- [ ] `trackBy` présent
- [ ] Item height stable (ou stratégie adaptée)
- [ ] Composants de ligne en `OnPush`
- [ ] Templates simples, pas de calculs coûteux
- [ ] Chargement progressif si dataset distant
- [ ] Profiling effectué (DevTools + Angular DevTools)

## B) Erreurs fréquentes
- Oublier la hauteur du viewport (rien ne s’affiche)
- `itemSize` incohérent avec la hauteur CSS
- Modifier le tableau en place sans déclencher correctement les changements (selon stratégie)
- Mettre un `mat-table` lourd sans optimisation (préférer CDK table + virtual scroll si besoin)

---

# Conclusion
Le **virtual scrolling** permet de rendre des listes massives tout en gardant une UI fluide, en limitant drastiquement le nombre de nœuds DOM et la charge Angular. Avec **Angular CDK**, l’implémentation est accessible, mais les meilleurs résultats viennent des optimisations complémentaires : `trackBy`, `OnPush`, templates légers, et chargement progressif lorsque la donnée est distante.
