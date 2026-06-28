# Formation Angular — Gestion d’état locale (Signals, services scopés, mini-stores)

> Objectif : maîtriser la gestion d’état **locale** dans Angular en utilisant **Signals**, des **services fournis au niveau composant** (scoped) et de **petits stores dédiés à une feature**, afin d’éviter la complexité d’un state management global (NgRx/NGXS/Akita, etc.) quand ce n’est pas justifié.

---

## Public & Pré-requis

- **Public** : développeurs Angular (test, TS, RxJS basique).
- **Pré-requis** :
  - Connaissances composants, templates, DI, Services.
  - RxJS : `Observable`, `Subject`, `BehaviorSubject` (optionnel mais utile).
  - Angular v16+ conseillé (Signals).

---

## Durée & format

- Durée type : **1 jour (7h)** ou **2 demi-journées**.
- Alternance : **70% pratique** / **30% théorie**.

---

## Plan global

1. **Pourquoi la gestion d’état locale ?**
2. **Identifier et modéliser l’état local**
3. **Approche 1 : état local dans le composant avec Signals**
4. **Approche 2 : services scopés au composant (Component-scoped services)**
5. **Approche 3 : mini-store de feature (Signal store ou service store)**
6. **Synchronisation UI ↔ API : loading/error, cache local, retries**
7. **Interopérations Signals ↔ RxJS**
8. **Tests (unitaires) et bonnes pratiques**
9. **Quand basculer vers un state management global ?**
10. **Atelier final : refactor d’une feature**

---

# 1) Pourquoi la gestion d’état locale ?

## 1.1 Définition rapide
L’**état (state)** est l’ensemble des données qui influencent l’UI et la logique :
- données chargées (liste, détail),
- sélection, filtres, pagination,
- état de formulaire (dirty, valid, valeurs),
- états UI (modale ouverte, onglet actif),
- états réseau (loading, error).

**Gestion d’état locale** : l’état vit **au plus près** de l’endroit où il est utilisé.

## 1.2 Pourquoi éviter le global par défaut
Un store global peut :
- ajouter du **boilerplate** (actions, reducers, selectors, effects),
- augmenter la **surface mentale** du projet,
- générer des **dépendances transverses**,
- sur-concevoir des features simples.

## 1.3 Bénéfices du local
- **Lisibilité** : l’état est proche du composant/feature.
- **Isolation** : moins d’effets de bord.
- **Facilité de test** : plus simple à mocker.
- **Évolutivité** : on peut toujours migrer vers du global quand nécessaire.

---

# 2) Identifier et modéliser l’état local

## 2.1 Questions de cadrage
Pour décider d’une stratégie locale, répondre :
1. **Qui consomme l’état ?** (un composant, un sous-arbre, une feature)
2. **Combien de temps vit l’état ?** (durée d’un écran ? navigation ?)
3. **Qui le modifie ?** (UI seule ? effets API ?)
4. **Partage requis ?** (entre routes/écrans ? entre features ?)

> Règle pratique : si l’état ne dépasse pas un **sous-arbre** de composants ou une **feature**, commencer local.

## 2.2 Typologie d’état
- **État UI** : toggles, tabs, modals.
- **État de view-model** : données transformées prêtes pour l’affichage.
- **État métier local** : panier temporaire d’un wizard, brouillon.
- **État serveur “caché”** : résultat d’un appel API mis en cache localement.

## 2.3 Modèle “state slice” minimal
Exemple de modèle d’état pour une liste :

```ts
export type LoadStatus = 'idle' | 'loading' | 'success' | 'error';

export interface ProductsState {
  status: LoadStatus;
  error: string | null;
  products: Product[];
  query: {
    search: string;
    page: number;
    pageSize: number;
  };
  selectedId: string | null;
}
```

Bonnes pratiques :
- inclure `status` + `error` quand il y a des appels,
- regrouper les paramètres en sous-objets (ex: `query`),
- éviter les dérivations stockées (les calculer via `computed`).

---

# 3) Approche 1 — État local dans le composant avec Signals

## 3.1 Rappels Signals
- `signal(value)` : état mutable.
- `computed(() => ...)` : dérivé, recalculé automatiquement.
- `effect(() => ...)` : side-effects, attention aux écritures.

### Exemple minimal

```ts
import { Component, computed, effect, signal } from '@angular/core';

@Component({
  selector: 'app-counter',
  template: `
    <button (click)="dec()">-</button>
    <span>{{ count() }}</span>
    <button (click)="inc()">+</button>
    <p>Double: {{ double() }}</p>
  `,
})
export class CounterComponent {
  count = signal(0);
  double = computed(() => this.count() * 2);

  inc() { this.count.update(v => v + 1); }
  dec() { this.count.update(v => v - 1); }
}
```

## 3.2 Pattern ViewModel local
On regroupe l’état et les “actions” (méthodes) dans le composant.

### Exemple : filtre + pagination

```ts
import { Component, computed, signal } from '@angular/core';

@Component({
  selector: 'app-products-toolbar',
  template: `
    <input
      [value]="search()"
      (input)="setSearch($any($event.target).value)"
      placeholder="Rechercher..." />

    <button (click)="prev()" [disabled]="page() === 1">Précédent</button>
    <span>Page {{ page() }}</span>
    <button (click)="next()">Suivant</button>

    <p class="muted">Query: {{ query() | json }}</p>
  `,
})
export class ProductsToolbarComponent {
  search = signal('');
  page = signal(1);
  pageSize = signal(20);

  query = computed(() => ({
    search: this.search().trim(),
    page: this.page(),
    pageSize: this.pageSize(),
  }));

  setSearch(v: string) { this.search.set(v); this.page.set(1); }
  next() { this.page.update(p => p + 1); }
  prev() { this.page.update(p => Math.max(1, p - 1)); }
}
```

## 3.3 Gestion de “loading/error” en local

```ts
import { Component, computed, inject, signal } from '@angular/core';
import { ProductsApi } from './products.api';

@Component({
  selector: 'app-products',
  template: `
    @if (status() === 'loading') { <p>Chargement...</p> }
    @if (status() === 'error') { <p class="error">{{ error() }}</p> }

    <ul>
      @for (p of products(); track p.id) {
        <li (click)="select(p.id)" [class.selected]="p.id === selectedId()">
          {{ p.name }}
        </li>
      }
    </ul>

    <button (click)="reload()">Recharger</button>
  `,
  styles: [`.selected{font-weight:bold}.error{color:#b00020}`]
})
export class ProductsComponent {
  private api = inject(ProductsApi);

  status = signal<'idle'|'loading'|'success'|'error'>('idle');
  error = signal<string | null>(null);
  products = signal<Product[]>([]);
  selectedId = signal<string | null>(null);

  async reload() {
    this.status.set('loading');
    this.error.set(null);

    try {
      const data = await this.api.list();
      this.products.set(data);
      this.status.set('success');
    } catch (e: any) {
      this.status.set('error');
      this.error.set(e?.message ?? 'Erreur inconnue');
    }
  }

  select(id: string) { this.selectedId.set(id); }
}
```

### Points d’attention
- Le composant grossit vite si l’état + effets + transformations s’accumulent.
- À partir de **plusieurs composants** partageant le même état : préférer un **service scoped** ou mini-store.

---

# 4) Approche 2 — Services scopés au composant

## 4.1 Quand utiliser
- état partagé entre un composant parent et plusieurs enfants,
- besoin d’isoler l’état à une instance d’écran (une route),
- réduire la taille du composant (séparer UI et état).

## 4.2 Fournir un service au niveau composant
Dans Angular, `providers` dans le décorateur crée une instance par composant.

```ts
import { Injectable, signal, computed } from '@angular/core';

@Injectable()
export class WizardState {
  step = signal(1);
  data = signal<{ name: string; email: string }>({ name: '', email: '' });

  canGoNext = computed(() => {
    const d = this.data();
    return this.step() === 1 ? d.name.trim().length > 0 : d.email.includes('@');
  });

  setName(name: string) {
    this.data.update(d => ({ ...d, name }));
  }

  setEmail(email: string) {
    this.data.update(d => ({ ...d, email }));
  }

  next() { this.step.update(s => Math.min(2, s + 1)); }
  prev() { this.step.update(s => Math.max(1, s - 1)); }
}
```

Utilisation dans un composant “host” :

```ts
import { Component } from '@angular/core';
import { WizardState } from './wizard.state';

@Component({
  selector: 'app-wizard',
  providers: [WizardState],
  template: `
    <h2>Étape {{ state.step() }}</h2>

    @if (state.step() === 1) {
      <input [value]="state.data().name" (input)="state.setName($any($event.target).value)" />
    }
    @if (state.step() === 2) {
      <input [value]="state.data().email" (input)="state.setEmail($any($event.target).value)" />
    }

    <button (click)="state.prev()" [disabled]="state.step()===1">Retour</button>
    <button (click)="state.next()" [disabled]="!state.canGoNext()">Suivant</button>
  `,
})
export class WizardComponent {
  constructor(public state: WizardState) {}
}
```

## 4.3 Partage avec des enfants
Les enfants injectent le service (même instance) car il est fourni par l’ancêtre.

```ts
@Component({
  selector: 'app-wizard-summary',
  template: `
    <pre>{{ state.data() | json }}</pre>
  `,
})
export class WizardSummaryComponent {
  constructor(public state: WizardState) {}
}
```

## 4.4 Avantages / limites
**Avantages** :
- état testable séparément,
- composant plus “presentational”,
- réutilisable dans une même feature.

**Limites** :
- risque de “service fourre-tout” si trop de responsabilités,
- attention aux effets (appels API) : garder une architecture claire.

---

# 5) Approche 3 — Mini-store de feature (dédié)

## 5.1 Pourquoi un mini-store
Quand l’état local :
- devient conséquent,
- a des transitions complexes,
- orchestre plusieurs sources (UI + API + cache),
- doit être réutilisé dans plusieurs composants d’une feature.

Un mini-store reste **local à la feature**, pas global à toute l’app.

## 5.2 Store basé sur Signals (pattern “store class”)

```ts
import { Injectable, computed, inject, signal } from '@angular/core';
import { ProductsApi } from './products.api';

type Status = 'idle'|'loading'|'success'|'error';

interface State {
  status: Status;
  error: string | null;
  products: Product[];
  selectedId: string | null;
  search: string;
}

const initialState: State = {
  status: 'idle',
  error: null,
  products: [],
  selectedId: null,
  search: '',
};

@Injectable()
export class ProductsStore {
  private api = inject(ProductsApi);
  private state = signal<State>(initialState);

  // selectors
  status = computed(() => this.state().status);
  error = computed(() => this.state().error);
  products = computed(() => this.state().products);
  selectedId = computed(() => this.state().selectedId);
  search = computed(() => this.state().search);

  filtered = computed(() => {
    const s = this.search().trim().toLowerCase();
    const items = this.products();
    if (!s) return items;
    return items.filter(p => p.name.toLowerCase().includes(s));
  });

  selected = computed(() => {
    const id = this.selectedId();
    return this.products().find(p => p.id === id) ?? null;
  });

  // actions
  setSearch(v: string) {
    this.patch({ search: v });
  }

  select(id: string | null) {
    this.patch({ selectedId: id });
  }

  async load() {
    this.patch({ status: 'loading', error: null });

    try {
      const data = await this.api.list();
      this.patch({ products: data, status: 'success' });
    } catch (e: any) {
      this.patch({ status: 'error', error: e?.message ?? 'Erreur inconnue' });
    }
  }

  reset() {
    this.state.set(initialState);
  }

  private patch(partial: Partial<State>) {
    this.state.update(s => ({ ...s, ...partial }));
  }
}
```

Dans un composant feature :

```ts
import { Component } from '@angular/core';
import { ProductsStore } from './products.store';

@Component({
  selector: 'app-products-page',
  providers: [ProductsStore],
  template: `
    <header>
      <input
        [value]="store.search()"
        (input)="store.setSearch($any($event.target).value)"
        placeholder="Filtrer" />
      <button (click)="store.load()">Charger</button>
    </header>

    @if (store.status() === 'loading') { <p>Chargement...</p> }
    @if (store.status() === 'error') { <p class="error">{{ store.error() }}</p> }

    <ul>
      @for (p of store.filtered(); track p.id) {
        <li (click)="store.select(p.id)" [class.selected]="p.id === store.selectedId()">
          {{ p.name }}
        </li>
      }
    </ul>

    <section>
      <h3>Détail</h3>
      @if (store.selected(); as item) {
        <pre>{{ item | json }}</pre>
      } @else {
        <p>Aucun élément sélectionné.</p>
      }
    </section>
  `,
})
export class ProductsPageComponent {
  constructor(public store: ProductsStore) {}
}
```

## 5.3 Conseils de conception
- **State privé**, exposer des `computed` comme selectors.
- Actions explicites (`load`, `select`, `setSearch`) → transitions claires.
- Ne pas stocker ce qui est dérivable (`filtered`, `selected`).
- Fournir le store dans le composant route/feature pour limiter la portée.

---

# 6) Synchronisation UI ↔ API : loading/error, cache, rafraîchissement

## 6.1 Associer “status + error” à chaque ressource
Pattern :
- `status: 'idle'|'loading'|'success'|'error'`
- `error: string | null`
- `data: T`

Option : modéliser génériquement une ressource :

```ts
export interface ResourceState<T> {
  status: 'idle'|'loading'|'success'|'error';
  error: string | null;
  data: T;
}
```

## 6.2 Cache local simple (memoization)
Si une feature recharge trop souvent, vous pouvez stocker un timestamp.

```ts
interface State {
  status: Status;
  error: string | null;
  products: Product[];
  lastLoadedAt: number | null;
}

const TTL = 30_000;

async loadIfStale() {
  const now = Date.now();
  const last = this.state().lastLoadedAt;
  if (last && now - last < TTL) return;
  await this.load();
}
```

## 6.3 Annulation / concurrence
Pour les recherches (typeahead), vous voulez souvent annuler les requêtes précédentes.
- Avec RxJS : `switchMap`.
- Avec Signals : interop RxJS ou logique d’abandon.

---

# 7) Interop Signals ↔ RxJS

Même si Signals couvrent beaucoup, RxJS reste pertinent pour :
- streams, événements multiples,
- annulation, composition (switchMap, combineLatest),
- websockets.

## 7.1 Convertir un Observable en Signal
Angular fournit `toSignal`.

```ts
import { toSignal } from '@angular/core/rxjs-interop';
import { inject } from '@angular/core';
import { ActivatedRoute } from '@angular/router';
import { map } from 'rxjs/operators';

const route = inject(ActivatedRoute);
const productId = toSignal(route.paramMap.pipe(map(p => p.get('id'))), { initialValue: null });
```

## 7.2 Convertir un Signal en Observable
Angular fournit `toObservable`.

```ts
import { toObservable } from '@angular/core/rxjs-interop';
import { debounceTime, distinctUntilChanged, switchMap } from 'rxjs/operators';

const search$ = toObservable(this.search);

const results$ = search$.pipe(
  debounceTime(300),
  distinctUntilChanged(),
  switchMap(q => this.api.search(q))
);
```

## 7.3 Pattern mixte recommandé
- UI + dérivations : **Signals**
- orchestration asynchrone complexe : **RxJS**, résultat converti en signal.

---

# 8) Tests & bonnes pratiques

## 8.1 Tester un store/service local
Exemple avec Jasmine/Karma ou Jest :

```ts
describe('ProductsStore', () => {
  it('should filter products by search', () => {
    const store = new ProductsStore(/* mock injectors in TestBed in real test */) as any;

    // si vous testez hors Angular, abstraire l'API et injecter via constructeur
  });
});
```

### Recommandation practical
Pour faciliter les tests, vous pouvez :
- injecter l’API via **constructeur** au lieu de `inject()` (ou utiliser TestBed),
- isoler les fonctions pures de transformation.

## 8.2 Bonnes pratiques de maintenance
- État minimal, dériver via `computed`.
- Actions nommées : `setX`, `load`, `reset`, `select`.
- Ne pas muter des objets/arrays directement : préférer update immuable.
- Scope : fournir le service/store au **plus près** (route/feature).
- Documenter les invariants (ex: `selectedId` doit appartenir à `products`).

## 8.3 Anti-patterns fréquents
- Tout mettre en store global “par standard”.
- Dupliquer le même état dans plusieurs endroits sans source de vérité.
- Stocker des états dérivés (risque de désynchronisation).
- Utiliser `effect` pour faire des écritures en cascade sans contrôle.

---

# 9) Quand basculer vers un state management global ?

Voici des signaux (sans jeu de mots) indiquant qu’un store global peut être justifié :

- état partagé entre **plusieurs routes** distantes,
- cohérence forte nécessaire entre features (ex: auth, permissions, panier global),
- besoin d’outillage avancé : time-travel, devtools, logs centralisés,
- équipe large, conventions strictes, architecture mature.

> Approche recommandée : commencer local, puis “remonter” seulement ce qui doit être partagé.

---

# 10) Atelier final (guidé)

## Objectif
Refactorer une feature “Products” initialement gérée en :
- variables éparpillées (UI),
- subscriptions manuelles,
- duplication de logique entre composants.

## Étapes
1. **Inventorier** l’état : data, search, selected, status, error.
2. Mettre en place un **mini-store** scoped à la page.
3. Remplacer la transformation dans le template par des `computed`.
4. Ajouter `load()` avec `status/error`.
5. (Option) Ajouter `loadIfStale()`.
6. Tests d’actions (au moins `setSearch`, `select`, `reset`).

## Critères de réussite
- Une seule source de vérité.
- Le template n’exécute pas de logique complexe.
- Les composants enfants n’ont pas de logique de récupération de données.
- Code plus lisible et plus testable.

---

# Annexes

## A) Matrice de décision rapide

| Besoin | Recommandation |
|---|---|
| État simple, utilisé dans un seul composant | Signals dans le composant |
| Partagé entre un parent et des enfants (même écran) | Service scoped au composant |
| Feature avec logique plus riche, plusieurs composants, API | Mini-store de feature |
| Partagé entre routes / cross-feature, invariants globaux | Store global (au cas par cas) |

## B) Checklist de revue de code (state local)
- [ ] L’état est-il au bon niveau de scope ?
- [ ] Les propriétés dérivées sont-elles en `computed` ?
- [ ] Les transitions d’état passent-elles par des actions claires ?
- [ ] `status/error` sont-ils cohérents ?
- [ ] Pas de duplication d’état entre plusieurs services/composants ?
- [ ] Les appels API sont-ils annulables si nécessaire ?

---

## Fin de formation — Résumé

Vous savez désormais :
- choisir une stratégie de state local adaptée,
- gérer un état **simple** avec `signal/computed/effect`,
- partager un état via **services scopés**,
- structurer un **mini-store de feature**,
- éviter la complexité d’un state management global quand elle n’est pas justifiée.
