# Formation Angular — Change Detection

> Objectif : maîtriser en profondeur le mécanisme de *Change Detection* (CD) d’Angular pour réduire les surcoûts de rendu et optimiser les performances via **OnPush**, **l’immutabilité** et la maîtrise des **déclencheurs**.

---

## Public visé & prérequis

### Public visé
- Développeurs Angular (intermédiaire à avancé)
- Formateurs/tech leads souhaitant expliquer et industrialiser les bonnes pratiques de performance

### Prérequis
- Connaissances Angular : composants, templates, directives, services, RxJS de base
- Typescript (interfaces, classes, accessoires)
- Avoir déjà utilisé le CLI et un navigateur avec DevTools

---

## Durée & format

- Durée conseillée : **1 journée (6–7h)** ou **2 demi-journées**
- Alternance : théorie (40%) / pratique (60%)

---

## Objectifs pédagogiques

À l’issue de la formation, le participant saura :
1. Expliquer comment Angular déclenche et exécute une boucle de Change Detection.
2. Identifier ce qui déclenche des cycles de CD (événements, async, timers, HTTP…).
3. Utiliser **ChangeDetectionStrategy.OnPush** correctement.
4. Mettre en place une stratégie d’**immutabilité** pour rendre OnPush efficace.
5. Réduire drastiquement le travail du CD via `trackBy`, `async` pipe, découpage de composants, et bonnes pratiques de template.
6. Diagnostiquer les problèmes avec des outils de profiling et des hooks.

---

# Plan de cours

1. **Fondations : qu’est-ce que le Change Detection ?**
2. **Le cycle de CD et les déclencheurs (Zone.js)**
3. **Default vs OnPush : modèles mentaux et impacts**
4. **Immutabilité : rendre l’UI prévisible et performante**
5. **Optimiser les templates : éviter les pièges courants**
6. **Maîtriser explicitement le CD : ChangeDetectorRef & signaux avancés**
7. **Ateliers pratiques : audit, corrections, mesures**
8. **Check-list & patterns industriels**

---

# 1) Fondations : qu’est-ce que le Change Detection ?

## 1.1 Définition
Le *Change Detection* d’Angular est le mécanisme qui :
- **évalue** les expressions de template,
- **compare** les valeurs précédentes et courantes,
- **applique** les mises à jour DOM nécessaires (via le rendu Angular).

**But** : maintenir la vue synchronisée avec l’état applicatif.

## 1.2 Pourquoi c’est critique pour les performances ?
En Angular, *le coût* vient souvent de :
- la fréquence des cycles de CD,
- le nombre de composants vérifiés,
- la complexité des templates (fonctions, pipes, boucles),
- les structures de données mutées qui empêchent des optimisations.

Un cycle de CD peut être déclenché très souvent : clavier, scroll, WebSocket, timers, etc.

## 1.3 Modèle mental
- Angular organise l’application en **arbre de composants**.
- Une détection peut :
  - parcourir une grande partie (ou tout) de l’arbre,
  - ou s’arrêter plus tôt si on utilise **OnPush** et une architecture adaptée.

---

# 2) Le cycle de CD et les déclencheurs (Zone.js)

## 2.1 Comment un cycle démarre ?
Historiquement (et majoritairement en Angular), la boucle de CD est orchestrée par **Zone.js**.

### Déclencheurs fréquents
Angular lance généralement une détection après :
- un événement DOM géré par Angular (`click`, `input`, `keyup`…),
- la résolution d’une promesse (`Promise.then`),
- un `setTimeout` / `setInterval`,
- une requête HTTP (observables complétés),
- un événement provenant de certaines APIs asynchrones patchées par Zone.js.

> Idée clé : *ce n’est pas votre `setState` qui déclenche*, c’est la fin d’un “macro/micro-task” observée par Zone.

## 2.2 Détection et tree traversal
Par défaut, Angular effectue une **vérification** des bindings pour une grande portion de l’arbre.

Pseudo-processus (simplifié) :
1. Un évènement survient (ex: click).
2. Zone.js notifie Angular qu’un traitement async est terminé.
3. Angular exécute `ApplicationRef.tick()`.
4. Angular parcourt l’arbre, réévalue les bindings.
5. Angular met à jour le DOM si nécessaire.

## 2.3 Dev mode vs prod mode
En **mode dev**, Angular effectue des vérifications supplémentaires (ex : contrôle d’état stable) pour aider à détecter des erreurs (notamment `ExpressionChangedAfterItHasBeenCheckedError`).

En **prod**, ces vérifications sont réduites, mais les gros goulots (templates coûteux, cycles trop fréquents) restent.

## 2.4 Hooks liés au CD
- `ngOnChanges` : appelé lorsqu’un `@Input()` change (avec référence différente, ou changement détecté selon la stratégie).
- `ngDoCheck` : hook bas niveau, appelé durant la détection. À utiliser avec parcimonie.
- `ngAfterViewChecked` / `ngAfterContentChecked` : appelés après vérification. Attention aux boucles.

**Règle** : si vous mettez du traitement lourd dans ces hooks, vous *amplifiez* le coût du CD.

---

# 3) Default vs OnPush : modèles mentaux et impacts

## 3.1 Stratégie Default (par défaut)
```ts
@Component({
  selector: 'app-a',
  templateUrl: './a.html',
  changeDetection: ChangeDetectionStrategy.Default
})
export class AComponent {}
```
### Caractéristiques
- À chaque cycle, Angular **vérifie** le composant et (souvent) ses descendants.
- Tolère mieux les mutations d’objets/arrays car la vue est fréquemment re-vérifiée.
- Plus simple mais plus coûteux quand l’arbre grandit.

## 3.2 Stratégie OnPush
```ts
@Component({
  selector: 'app-b',
  templateUrl: './b.html',
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class BComponent {}
```
### Caractéristiques
Un composant OnPush est vérifié principalement quand :
1. **Un de ses `@Input()` reçoit une *nouvelle référence***
2. **Un événement** est émis depuis sa vue (ex : `(click)`)
3. **Un observable via `async` pipe** émet une nouvelle valeur
4. On force la vérification via `markForCheck()` / `detectChanges()`

### Impact
- Moins de vérifications inutiles.
- Nécessite une discipline : **immutabilité** et flux de données clairs.

## 3.3 Exemple : mutation vs nouvelle référence
### Cas problématique (mutation)
```ts
// parent.component.ts
items = [{ id: 1, label: 'A' }];

add() {
  this.items.push({ id: 2, label: 'B' }); // mutation
}
```
Si `items` est passé à un enfant OnPush :
```html
<app-list [items]="items"></app-list>
```
L’enfant **peut ne pas se rafraîchir**, car la référence `items` n’a pas changé.

### Correction (immutabilité)
```ts
add() {
  this.items = [...this.items, { id: 2, label: 'B' }]; // nouvelle référence
}
```

---

# 4) Immutabilité : rendre l’UI prévisible et performante

## 4.1 Pourquoi l’immutabilité aide OnPush ?
OnPush repose sur l’**identité** (référence) des objets.
- Si vous remplacez l’objet/array, Angular détecte facilement un changement.
- Si vous mutez en place, Angular ne voit pas forcément le changement dans un enfant OnPush.

## 4.2 Pratiques recommandées
- Utiliser des opérateurs immutables : `map`, `filter`, `slice`, spread `...`.
- Éviter `push`, `splice`, mutation de propriété profonde (ex: `user.profile.name = ...`).

### Mise à jour profonde : exemple
```ts
this.user = {
  ...this.user,
  profile: {
    ...this.user.profile,
    name: 'Nouveau nom'
  }
};
```

## 4.3 Modéliser l’état
Approches possibles :
- Services avec `BehaviorSubject` + `async` pipe
- State management (NgRx, Akita, NGXS, SignalStore, etc.)

### Exemple simple (service + async)
```ts
@Injectable({ providedIn: 'root' })
export class TodosStore {
  private readonly _todos = new BehaviorSubject<Todo[]>([]);
  readonly todos$ = this._todos.asObservable();

  add(todo: Todo) {
    this._todos.next([...this._todos.value, todo]);
  }
}
```

```html
<!-- composant OnPush -->
<ul>
  <li *ngFor="let t of (store.todos$ | async); trackBy: trackById">
    {{ t.label }}
  </li>
</ul>
```

---

# 5) Optimiser les templates : éviter les pièges courants

## 5.1 Éviter les fonctions dans le template
### Problème
```html
<div>{{ computeTotal() }}</div>
```
`computeTotal()` peut être appelé à chaque CD.

### Solutions
- Pré-calculer dans le composant via observable/pipe
- Utiliser des *pure pipes* (pipes purs)
- Utiliser des signaux/mémorisation (selon version et architecture)

## 5.2 Pipes : purs vs impurs
- **Pure pipe (par défaut)** : recalculée seulement si la référence d’entrée change.
- **Impure pipe** : recalculée à chaque CD (coûteux).

> Éviter autant que possible les pipes impurs.

## 5.3 `*ngFor` et `trackBy`
Sans `trackBy`, Angular peut recréer beaucoup d’éléments DOM quand la liste change.

```ts
trackById = (_: number, item: { id: number }) => item.id;
```

```html
<div *ngFor="let item of items; trackBy: trackById">
  {{ item.label }}
</div>
```

## 5.4 Minimiser le binding et les watchers
- Réduire les interpolations complexes
- Découper en sous-composants OnPush (granularité)
- Utiliser `async` pipe plutôt que `subscribe()` manuel + assignations

## 5.5 Éviter les changements “invisibles”
Exemples :
- mutation d’objet passé à un enfant
- mise à jour outsider (lib externe) hors Angular

Solution : rendre les flux explicites (observables, signaux) ou notifier CD (`markForCheck`).

---

# 6) Maîtriser explicitement le CD : ChangeDetectorRef & techniques avancées

## 6.1 `ChangeDetectorRef`
### `markForCheck()`
Demande à Angular de re-vérifier ce composant (et la chaîne) lors du prochain tick.

```ts
constructor(private cdr: ChangeDetectorRef) {}

refreshLater() {
  setTimeout(() => {
    this.value = 123;
    this.cdr.markForCheck();
  }, 1000);
}
```

### `detectChanges()`
Lance immédiatement une détection pour ce composant et ses enfants.

```ts
this.value = 123;
this.cdr.detectChanges();
```

> À manier avec précaution : peut provoquer du travail non planifié et des effets de bord.

### `detach()` / `reattach()`
Permet de sortir un composant du CD automatique.

Cas d’usage : écran avec zone très coûteuse, mise à jour ponctuelle seulement.

```ts
ngOnInit() {
  this.cdr.detach();
}

updateManually() {
  // ... maj données
  this.cdr.detectChanges();
}
```

## 6.2 Exécuter hors Angular : `NgZone.runOutsideAngular`
Utile pour des événements extrêmement fréquents (scroll, mousemove) où l’on ne souhaite pas déclencher le CD à chaque occurrence.

```ts
constructor(private zone: NgZone) {}

ngAfterViewInit() {
  this.zone.runOutsideAngular(() => {
    window.addEventListener('scroll', () => {
      // traitement léger sans CD automatique
    });
  });
}
```

Quand une mise à jour UI est nécessaire :
```ts
this.zone.run(() => {
  this.value = compute();
});
```

## 6.3 Focus : `async` pipe
- Abonne/désabonne automatiquement
- Marque le composant pour vérification quand une nouvelle valeur arrive
- Contribue à un flux unidirectionnel et OnPush-friendly

---

# 7) Ateliers pratiques : audit, corrections, mesures

## Atelier 1 — Visualiser le coût de CD
### Objectif
Comprendre « combien de fois » l’app vérifie les composants.

### Étapes
1. Ajouter des logs temporaires dans `ngDoCheck()` d’un composant clé.
2. Déclencher actions UI (click, input) et observer.
3. Identifier les interactions qui déclenchent des rafales.

> Variante : utiliser des outils de profiling (Performance tab) et Angular DevTools (si disponible) pour repérer les composants les plus checkés.

## Atelier 2 — Passer un sous-arbre en OnPush
### Objectif
Réduire le nombre de vérifications.

### Scénario
- Un parent gère un formulaire et une liste.
- La liste est indépendante et coûteuse.

### Actions
1. Mettre la liste en composant indépendant OnPush.
2. Remplacer mutations par immutabilité.
3. Ajouter `trackBy`.
4. Mesurer le gain.

## Atelier 3 — Remplacer `subscribe()` manuel par `async`
### Objectif
Éviter les assignations qui déclenchent inutilement.

### Actions
- Exposer un `vm$` (ViewModel observable)
- Utiliser `*ngIf="vm$ | async as vm"`

Exemple :
```ts
vm$ = combineLatest([
  this.store.todos$,
  this.userService.user$
]).pipe(
  map(([todos, user]) => ({ todos, user }))
);
```

```html
<ng-container *ngIf="vm$ | async as vm">
  <h3>{{ vm.user.name }}</h3>
  <app-list [items]="vm.todos"></app-list>
</ng-container>
```

## Atelier 4 — Événements haute fréquence
### Objectif
Empêcher les scroll/mousemove de déclencher CD.

### Actions
- `runOutsideAngular` + *throttle* / *debounce* via RxJS
- Réintégrer dans Angular seulement quand nécessaire (`zone.run`).

---

# 8) Check-list & patterns industriels

## 8.1 Check-list OnPush
- [ ] Les composants “feuilles” (présentation) sont en OnPush
- [ ] Les `@Input()` sont immuables (nouvelles références)
- [ ] Les listes ont `trackBy`
- [ ] Pas de fonctions coûteuses dans le template
- [ ] Préférence pour `async` pipe (moins de glue code)
- [ ] Gestion des événements haute fréquence hors Angular si besoin

## 8.2 Patterns
### Container / Presentational
- **Container** : compose les flux, interagit avec services/store
- **Presentational** : OnPush, inputs immuables, outputs explicites

### ViewModel observable (vm$)
- Un flux unique qui alimente le template
- Diminue la logique dispersée

### Découpage par responsabilité
- Découpler les zones qui doivent se rafraîchir de celles qui ne changent pas

---

# Annexes

## A) Mini FAQ

### Pourquoi OnPush ne met pas à jour mon composant ?
Le plus courant : vous mutez un objet/array passé en `@Input()`. Il faut créer une nouvelle référence ou déclencher manuellement `markForCheck()`.

### `detectChanges()` ou `markForCheck()` ?
- `markForCheck()` : plus “Angular-friendly”, déclenche au prochain tick.
- `detectChanges()` : immédiat, utile en cas de besoin ponctuel et maîtrisé.

### Le `async` pipe fonctionne-t-il bien avec OnPush ?
Oui. Il marque le composant pour vérification lorsqu’il reçoit une nouvelle valeur.

## B) Exemple récapitulatif : composant OnPush performant
```ts
@Component({
  selector: 'app-users',
  template: `
    <ng-container *ngIf="users$ | async as users">
      <input (input)="filter$.next($any($event.target).value)" placeholder="Filtrer" />

      <div *ngFor="let u of users; trackBy: trackById">
        {{ u.name }}
      </div>
    </ng-container>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class UsersComponent {
  readonly filter$ = new BehaviorSubject('');

  readonly users$ = combineLatest([
    this.api.users$,
    this.filter$
  ]).pipe(
    map(([users, f]) => {
      const q = f.toLowerCase().trim();
      return q ? users.filter(u => u.name.toLowerCase().includes(q)) : users;
    })
  );

  trackById = (_: number, u: { id: number }) => u.id;

  constructor(private api: UsersApi) {}
}
```

---

## Conclusion
Maîtriser le Change Detection Angular revient à :
- comprendre **quand** et **pourquoi** Angular vérifie l’arbre,
- diminuer le travail à chaque tick (templates propres, `trackBy`),
- limiter la propagation (OnPush),
- rendre les changements explicites (immutabilité, `async`),
- contrôler les cas extrêmes (`NgZone`, `ChangeDetectorRef`).

> Résultat attendu : interfaces plus fluides, code plus prévisible, et performance qui passe à l’échelle.
