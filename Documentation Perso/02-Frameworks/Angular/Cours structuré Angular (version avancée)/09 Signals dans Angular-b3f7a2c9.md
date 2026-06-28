# Formation — Signals dans Angular

## Métadonnées
- **Public** : développeurs Angular (niveau intermédiaire)
- **Objectif** : maîtriser la réactivité fine avec `signal`, `computed`, `effect`, et leur intégration dans une architecture moderne Angular
- **Durée indicative** : 1 jour (7h) ou 2 demi-journées
- **Pré-requis** : Angular standalone (ou modules), TypeScript, RxJS notions de base
- **Version** : Angular 16+ (Signals), avec remarques sur Angular 17/18 quand utile

---

## Plan de la formation
1. Introduction : pourquoi Signals ? quelle place dans Angular moderne
2. Concepts fondamentaux : `signal`, lecture, écriture, typage
3. Dérivation : `computed` et la programmation déclarative
4. Effets : `effect`, effets secondaires, cycle de vie et nettoyage
5. Intégration UI : templates, change detection, perf et bonnes pratiques
6. Gestion d’état local : patterns (component store, view-model, forms)
7. Interopérabilité : RxJS ↔ Signals et points d’attention
8. Architecture : services, injection, scopes, partage d’état
9. Tests : tester signals/computed/effects proprement
10. Atelier guidé : mini-app (panier / todo) en signals
11. Cheatsheet + checklists + anti-patterns

---

# 1) Introduction : pourquoi Signals ?

## 1.1 Problème historique
Angular a longtemps reposé sur :
- **Bindings** + détection de changement (Change Detection)
- **RxJS** pour la réactivité asynchrone

Mais pour l’état **local** (état de composant, UI state) on utilisait parfois :
- des champs simples + méthodes
- un `BehaviorSubject`
- un store externe

Cela peut mener à :
- du code plus verbeux
- des difficultés à tracer les dépendances
- des recalculs inutiles

## 1.2 Ce qu’apportent les Signals
Les **signals** introduisent une **réactivité fine** :
- un signal contient une valeur
- la lecture crée une **dépendance**
- la mutation déclenche la mise à jour des dépendants

En pratique :
- moins de boilerplate que les sujets RxJS pour du synchro
- des calculs dérivés plus lisibles via `computed`
- des effets secondaires encapsulés via `effect`

## 1.3 Quand utiliser Signals vs RxJS ?
- **Signals** : état synchrone local, dérivations, UI state, composition simple.
- **RxJS** : flux asynchrones, événements, opérateurs temporels (debounce, switchMap), websockets.

Angular fournit des ponts :
- `toSignal(observable$)`
- `toObservable(signal)`

---

# 2) Concepts fondamentaux : `signal`

## 2.1 Définition
Un **signal** représente une valeur réactive.

- **Lecture** : en appelant le signal comme une fonction `count()`
- **Écriture** : `set(...)` ou `update(fn)`

### Exemple minimal
```ts
import { signal } from '@angular/core';

const count = signal(0);

console.log(count()); // lecture => 0
count.set(1);         // écriture
count.update(v => v + 1);
```

## 2.2 Typage TypeScript
```ts
const username = signal<string>('');
const loggedIn = signal(false); // inféré boolean
```

## 2.3 `set` vs `update`
- `set(value)` : remplace la valeur
- `update(updater)` : calcule la nouvelle valeur depuis l’ancienne

```ts
const items = signal<string[]>([]);

items.set(['A']);
items.update(list => [...list, 'B']);
```

## 2.4 Mutabilité : attention aux objets
Les signals suivent une logique « valeur », mais si vous **mutez** un objet/array sans changer la référence, vous risquez de ne pas déclencher de mise à jour.

✅ Bon (immutable) :
```ts
const user = signal({ name: 'Ada', role: 'admin' });
user.update(u => ({ ...u, role: 'user' }));
```

⚠️ Mauvais (mutation in-place) :
```ts
user().role = 'user'; // la référence ne change pas => risque d’UI non mise à jour
```

> Bon réflexe : travailler de manière immuable pour les structures.

---

# 3) Dérivation : `computed`

## 3.1 Objectif
Un `computed` est un signal dérivé :
- il **recalcule** sa valeur quand ses dépendances changent
- il **cache** (memoize) : pas de recalcul si dépendances inchangées

## 3.2 Exemple
```ts
import { signal, computed } from '@angular/core';

const firstName = signal('Ada');
const lastName = signal('Lovelace');

const fullName = computed(() => `${firstName()} ${lastName()}`);

console.log(fullName()); // "Ada Lovelace"
lastName.set('Byron');
console.log(fullName()); // "Ada Byron"
```

## 3.3 `computed` pour dériver l’état UI
```ts
const cartItems = signal([{ price: 10 }, { price: 25 }]);
const total = computed(() => cartItems().reduce((s, i) => s + i.price, 0));
const hasDiscount = computed(() => total() >= 50);
```

## 3.4 Bonnes pratiques
- gardez les `computed` **purs** (pas d’effets secondaires)
- évitez d’y déclencher des appels HTTP, des logs, etc.
- si la dérivation devient complexe, extraire une fonction pure

---

# 4) Effets : `effect`

## 4.1 Objectif
Un `effect` exécute du code quand les signaux lus à l’intérieur changent.

Usage typique :
- synchroniser avec `localStorage`
- déclencher un tracking analytics
- lancer une requête (avec précautions)

## 4.2 Exemple simple
```ts
import { signal, effect } from '@angular/core';

const theme = signal<'light' | 'dark'>('light');

effect(() => {
  document.body.dataset['theme'] = theme();
});
```

## 4.3 Nettoyage (cleanup)
Un effect peut retourner une fonction de nettoyage.

```ts
const query = signal('');

effect((onCleanup) => {
  const q = query();
  const controller = new AbortController();

  fetch(`/api/search?q=${encodeURIComponent(q)}`, { signal: controller.signal });

  onCleanup(() => controller.abort());
});
```

## 4.4 Contrôle de l’écriture dans un effect
Écrire dans un signal depuis un effect peut créer des boucles.
- préférez une architecture où l’effect observe et déclenche une action externe
- si vous devez écrire, faites-le de manière contrôlée et idempotente

Anti-pattern :
```ts
// Peut boucler si `count` est lu et écrit sans garde.
effect(() => {
  count.set(count() + 1);
});
```

---

# 5) Intégration UI : templates, change detection et performance

## 5.1 Lire un signal dans le template
Dans un composant :
```ts
import { Component, signal, computed } from '@angular/core';

@Component({
  selector: 'app-counter',
  standalone: true,
  template: `
    <h2>Counter</h2>
    <p>Valeur: {{ count() }}</p>
    <p>Double: {{ double() }}</p>
    <button (click)="inc()">+1</button>
  `
})
export class CounterComponent {
  count = signal(0);
  double = computed(() => this.count() * 2);

  inc() {
    this.count.update(v => v + 1);
  }
}
```

## 5.2 Signals et OnPush
Signals fonctionnent très bien avec `ChangeDetectionStrategy.OnPush`.
- Angular sait que la lecture d’un signal dans le template crée une dépendance.
- quand le signal change, Angular planifie la mise à jour nécessaire.

> En pratique : vous obtenez souvent les bénéfices d’OnPush avec moins de complexité.

## 5.3 Minimiser les recalculs
- placez les calculs dérivés dans `computed` plutôt que dans le template
- évitez les fonctions dans le template si elles font des calculs coûteux

---

# 6) Gestion d’état local : patterns

## 6.1 Pattern « ViewModel » dans le composant
Structure recommandée :
- état source : signals
- état dérivé : computed
- actions : méthodes

```ts
type Todo = { id: string; title: string; done: boolean };

todos = signal<Todo[]>([]);
filter = signal<'all' | 'open' | 'done'>('all');

visibleTodos = computed(() => {
  const f = this.filter();
  const list = this.todos();
  if (f === 'open') return list.filter(t => !t.done);
  if (f === 'done') return list.filter(t => t.done);
  return list;
});

add(title: string) {
  const todo: Todo = { id: crypto.randomUUID(), title, done: false };
  this.todos.update(t => [todo, ...t]);
}
```

## 6.2 Services avec state en Signals
Pour partager l’état entre composants, mettez les signals dans un service.

```ts
import { Injectable, signal, computed } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class CartStore {
  private readonly _items = signal<{ id: string; price: number }[]>([]);
  readonly items = this._items.asReadonly();

  readonly total = computed(() => this.items().reduce((s, i) => s + i.price, 0));

  add(item: { id: string; price: number }) {
    this._items.update(list => [...list, item]);
  }

  remove(id: string) {
    this._items.update(list => list.filter(i => i.id !== id));
  }
}
```

### `.asReadonly()`
Exposer un signal en lecture seule évite les écritures hors du store.

## 6.3 Signals + formulaires
- **Reactive Forms** conservent leur pertinence.
- Utiliser un signal pour refléter l’état du formulaire ou des paramètres UI.
- Pour synchroniser : un `effect` peut écouter une valeur de `FormControl` (via `valueChanges` + `toSignal`).

---

# 7) Interopérabilité RxJS ↔ Signals

## 7.1 `toSignal` : Observable → Signal
Cas d’usage : HTTP, route params, websockets.

```ts
import { toSignal } from '@angular/core/rxjs-interop';
import { inject } from '@angular/core';
import { ActivatedRoute } from '@angular/router';
import { map } from 'rxjs/operators';

const route = inject(ActivatedRoute);

const userId = toSignal(
  route.paramMap.pipe(map(p => p.get('id'))),
  { initialValue: null }
);
```

Points d’attention :
- fournir `initialValue` si l’Observable n’émet pas immédiatement
- gérer les erreurs côté RxJS avant conversion

## 7.2 `toObservable` : Signal → Observable
Utile pour réutiliser des opérateurs RxJS.

```ts
import { toObservable } from '@angular/core/rxjs-interop';

const query$ = toObservable(this.query);
```

## 7.3 Stratégie conseillée
- Garder RxJS pour les flux async et l’orchestration
- Convertir au bord (edge) vers signals pour l’état UI

---

# 8) Architecture : injection, scopes, partage d’état

## 8.1 Où stocker les signals ?
- **Composant** : état local, non partagé
- **Service `providedIn: root`** : état global
- **Service fourni au niveau d’une route/composant** : état par feature

## 8.2 Encapsulation
- signaux privés `_state`
- signaux publics `readonly` + méthodes d’action

Checklist :
- [ ] l’état ne doit pas être modifiable directement depuis l’extérieur
- [ ] les actions décrivent l’intention métier

---

# 9) Tests

## 9.1 Tester un `computed`
```ts
import { signal, computed } from '@angular/core';

describe('computed', () => {
  it('should derive value', () => {
    const a = signal(1);
    const b = signal(2);
    const sum = computed(() => a() + b());

    expect(sum()).toBe(3);
    a.set(5);
    expect(sum()).toBe(7);
  });
});
```

## 9.2 Tester un service store
- tester les actions
- vérifier les valeurs des signals exposés

```ts
it('adds item', () => {
  const store = new CartStore();
  store.add({ id: 'p1', price: 10 });
  expect(store.items().length).toBe(1);
  expect(store.total()).toBe(10);
});
```

## 9.3 Tester un `effect`
- évitez les effets globaux difficiles à isoler
- injectez des dépendances (ex: storage, http)
- vérifiez les appels via spies

---

# 10) Atelier guidé : mini-app « Panier » en Signals

## 10.1 Spécifications
- liste de produits
- ajout/retrait au panier
- affichage du total
- persistance dans `localStorage`

## 10.2 Store
```ts
import { Injectable, signal, computed, effect } from '@angular/core';

export type CartItem = { id: string; name: string; price: number; qty: number };

@Injectable({ providedIn: 'root' })
export class CartStore {
  private readonly _items = signal<CartItem[]>(this.load());
  readonly items = this._items.asReadonly();

  readonly total = computed(() =>
    this.items().reduce((sum, i) => sum + i.price * i.qty, 0)
  );

  constructor() {
    effect(() => {
      const items = this.items();
      localStorage.setItem('cart', JSON.stringify(items));
    });
  }

  add(p: { id: string; name: string; price: number }) {
    this._items.update(list => {
      const found = list.find(i => i.id === p.id);
      if (!found) return [{ ...p, qty: 1 }, ...list];
      return list.map(i => (i.id === p.id ? { ...i, qty: i.qty + 1 } : i));
    });
  }

  dec(id: string) {
    this._items.update(list =>
      list
        .map(i => (i.id === id ? { ...i, qty: i.qty - 1 } : i))
        .filter(i => i.qty > 0)
    );
  }

  clear() {
    this._items.set([]);
  }

  private load(): CartItem[] {
    try {
      const raw = localStorage.getItem('cart');
      return raw ? (JSON.parse(raw) as CartItem[]) : [];
    } catch {
      return [];
    }
  }
}
```

## 10.3 Composant
```ts
import { Component, ChangeDetectionStrategy, inject } from '@angular/core';
import { CommonModule } from '@angular/common';
import { CartStore } from './cart.store';

@Component({
  selector: 'app-cart',
  standalone: true,
  imports: [CommonModule],
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <h2>Panier</h2>

    <p>Total: {{ store.total() }} €</p>
    <button (click)="store.clear()" [disabled]="store.items().length === 0">
      Vider
    </button>

    <ul>
      <li *ngFor="let i of store.items()">
        <strong>{{ i.name }}</strong>
        — {{ i.price }} € x {{ i.qty }}
        <button (click)="store.dec(i.id)">-</button>
      </li>
    </ul>
  `
})
export class CartComponent {
  readonly store = inject(CartStore);
}
```

Exercices :
1. Ajouter un `computed` `itemsCount` (somme des quantités)
2. Ajouter un `computed` `isEmpty`
3. Ajouter une promo : si total > 100 alors -10%
4. Remplacer le `localStorage` par une abstraction `StorageService` injectable (testable)

---

# 11) Cheatsheet, checklists et anti-patterns

## 11.1 Cheatsheet
```ts
const a = signal(0);
a();                 // read

a.set(10);           // write

a.update(v => v+1);  // update

const b = computed(() => a() * 2);

effect(() => {
  console.log(a(), b());
});
```

## 11.2 Checklist bonnes pratiques
- [ ] Utiliser `computed` pour les valeurs dérivées
- [ ] Garder `computed` **pur**
- [ ] Mettre les side-effects dans `effect`
- [ ] Privilégier l’immuabilité (nouvelle référence)
- [ ] Encapsuler l’écriture (`private _state` + `readonly state`)
- [ ] Convertir RxJS → Signal aux bords (`toSignal`)

## 11.3 Anti-patterns fréquents
- Muter un objet directement (`obj().x = ...`)
- Mettre des appels HTTP dans un `computed`
- Écrire sans garde dans un `effect` (risque boucle)
- Exposer des signals modifiables publiquement dans un store partagé

---

## Conclusion
Les signals apportent une réactivité fine et lisible, particulièrement adaptée à la **gestion d’état local** et à la dérivation d’état dans Angular moderne. Avec `signal` (source), `computed` (dérivé) et `effect` (effets secondaires), on obtient un modèle simple, performant, et composable, qui s’intègre naturellement avec RxJS via les fonctions d’interopérabilité.
