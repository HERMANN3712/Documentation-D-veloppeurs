# 08 — Stratégie **OnPush** (Angular)

> **Objectif** : comprendre et appliquer `ChangeDetectionStrategy.OnPush` pour réduire le coût de la détection de changements, structurer des composants performants et maîtriser les cas où il faut déclencher/forcer un rendu.

---

## Plan de la formation

1. **Rappels sur la détection de changements Angular**
   - Zone.js et le cycle de CD
   - Arbre de composants, vérification et rendu
   - Coûts typiques (templates lourds, listes, pipes, bindings, etc.)

2. **Pourquoi `OnPush` ?**
   - Ce que change réellement `OnPush`
   - Cas d’usage (listes, dashboards, composants “présentation”)
   - Gains et limites

3. **Quand Angular relance le CD avec `OnPush` ? (les 4 déclencheurs)**
   - Changement de **référence** des `@Input()`
   - **Événements** du composant (DOM / Output / handlers)
   - **Émissions asynchrones** (AsyncPipe, Observables/Promises)
   - **Marquage manuel** (`markForCheck`, `detectChanges`, `NgZone.run`)

4. **Immutabilité : la règle d’or**
   - Mutations vs changements de référence
   - Patterns immuables (spread, map/filter/reduce)
   - Attention aux objets imbriqués

5. **OnPush + flux asynchrones**
   - `AsyncPipe` comme stratégie par défaut
   - Composants “smart” vs “presentational”
   - Common pitfalls avec `subscribe()` manuel

6. **API de contrôle : `ChangeDetectorRef` et `NgZone`**
   - `markForCheck()` vs `detectChanges()`
   - `detach()` / `reattach()` (cas avancés)
   - Optimisations avec `NgZone.runOutsideAngular()`

7. **Bonnes pratiques et architecture**
   - `trackBy` sur `*ngFor`
   - Pure pipes, signaux (si utilisés), memoization
   - State management (RxJS, NgRx, services, etc.)

8. **Ateliers / exercices**
   - Convertir un composant Default → OnPush
   - Corriger un bug de mutation
   - Tableau dynamique performant avec `AsyncPipe` + `trackBy`

9. **Checklist de prod & dépannage**
   - Symptômes typiques (UI qui ne se met pas à jour)
   - Débogage (Angular DevTools, logs, profiling)

---

## 1) Rappels : comment Angular détecte les changements

Angular doit décider **quand** et **quoi** re-rendre dans le DOM.

- La **détection de changements** (change detection, CD) est un mécanisme qui :
  1. parcourt l’arbre des composants,
  2. évalue les expressions du template,
  3. met à jour le DOM si nécessaire.

Par défaut (`ChangeDetectionStrategy.Default`) :
- Angular vérifie **très souvent** (à chaque “tick” déclenché par Zone.js : événement user, timer, promesse, XHR, etc.).
- Cela peut devenir coûteux si vous avez :
  - gros arbres de composants,
  - listes longues,
  - templates riches (beaucoup de bindings),
  - calculs dans le template,
  - défilements / animations / timers.

**But de OnPush** : réduire le nombre de composants vérifiés à chaque tick.

---

## 2) `ChangeDetectionStrategy.OnPush` : ce que ça change

### Définition
Avec `OnPush`, Angular considère qu’un composant est **stable** tant qu’il n’a pas de raison explicite de penser le contraire.

- En `Default`, Angular “passe” souvent sur le composant.
- En `OnPush`, Angular **saute** le composant (et sa sous-arborescence) tant qu’aucun déclencheur n’arrive.

### Activation

```ts
import { ChangeDetectionStrategy, Component } from '@angular/core';

@Component({
  selector: 'app-user-card',
  templateUrl: './user-card.component.html',
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class UserCardComponent {}
```

### Bénéfice principal
- Moins de travail = **meilleures performances**.
- Les composants deviennent plus facilement **prévisibles** si vous adoptez l’immuabilité.

### Limite principale
- Si vous **mutez** des objets/arrays passés en `@Input`, la UI peut **ne pas** être mise à jour, car la référence ne change pas.

---

## 3) Les 4 déclencheurs de rendu avec `OnPush`

`OnPush` relance la vérification du composant (et potentiellement du sous-arbre) dans ces cas :

### 3.1 Changement de **référence** des `@Input()`

Angular compare les `@Input()` via une logique de **référence** (pas une comparaison deep).

✅ Déclenche :

```ts
this.user = { ...this.user, name: 'Alice' }; // nouvelle référence
```

❌ Ne déclenche pas (mutation) :

```ts
this.user.name = 'Alice'; // même objet => souvent pas de CD
```

> Important : même si une propriété interne change, **si la référence de l’input reste identique**, `OnPush` peut ne rien re-rendre.

---

### 3.2 Événements internes du composant

Si un événement provient de la vue du composant (click, input, etc.), Angular déclenche la CD pour ce composant.

```html
<button (click)="inc()">+1</button>
<p>{{ count }}</p>
```

```ts
count = 0;
inc() { this.count++; }
```

Ici, même en `OnPush`, le click déclenche une mise à jour.

---

### 3.3 Émissions asynchrones

Quand une **source async** produit une nouvelle valeur, Angular met à jour la vue si vous utilisez un mécanisme intégré comme `AsyncPipe`.

```html
<p>Users: {{ users$ | async | json }}</p>
```

- `AsyncPipe` s’abonne, se désabonne automatiquement et marque la vue pour vérification.

⚠️ Si vous faites un `subscribe()` manuel et modifiez un champ, vous devrez parfois appeler `markForCheck()` selon le contexte.

---

### 3.4 Marquage manuel

Vous pouvez forcer/planifier une vérification via `ChangeDetectorRef` :

- `markForCheck()` : marque ce composant comme “à vérifier” au prochain cycle.
- `detectChanges()` : exécute immédiatement une détection sur ce composant (et ses enfants) (use with care).

```ts
import { ChangeDetectorRef, Component, ChangeDetectionStrategy } from '@angular/core';

@Component({
  selector: 'app-clock',
  template: '{{ time }}',
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class ClockComponent {
  time = new Date().toISOString();

  constructor(private cdr: ChangeDetectorRef) {
    setInterval(() => {
      this.time = new Date().toISOString();
      this.cdr.markForCheck();
    }, 1000);
  }
}
```

---

## 4) Immutabilité : la règle d’or avec `OnPush`

### Pourquoi ?
`OnPush` s’appuie sur l’idée : “si la **référence** ne change pas, alors l’état effectif est probablement identique”.
Cela ne fonctionne correctement que si :
- vous **n’altérez pas** les objets/arrays existants,
- vous produisez des **nouvelles références** lors des mises à jour.

### Exemples pratiques

#### Mettre à jour un objet

```ts
// ✅ immuable
this.settings = { ...this.settings, theme: 'dark' };

// ❌ mutation
this.settings.theme = 'dark';
```

#### Mettre à jour un tableau

```ts
// ✅ ajouter
this.items = [...this.items, newItem];

// ✅ modifier un élément
this.items = this.items.map(i => i.id === id ? { ...i, done: true } : i);

// ✅ supprimer
this.items = this.items.filter(i => i.id !== id);

// ❌ mutation
this.items.push(newItem);
```

### Attention aux structures imbriquées

```ts
// ✅ nouvelle référence à chaque niveau modifié
this.user = {
  ...this.user,
  address: {
    ...this.user.address,
    city: 'Paris'
  }
};
```

---

## 5) OnPush + RxJS / AsyncPipe : approche recommandée

### Pourquoi privilégier `AsyncPipe`
- Déclenche correctement le rafraîchissement.
- Gère les abonnements (moins de fuites mémoire).
- Encourage des templates déclaratifs.

Exemple “smart component” :

```ts
@Component({
  selector: 'app-users-page',
  template: `
    <app-users-list
      [users]="users$ | async"
      (select)="select($event)">
    </app-users-list>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class UsersPageComponent {
  users$ = this.usersService.users$;
  select(userId: string) { /* ... */ }
  constructor(private usersService: UsersService) {}
}
```

Composant présentationnel :

```ts
@Component({
  selector: 'app-users-list',
  template: `
    <ul>
      <li *ngFor="let u of users; trackBy: trackById" (click)="select.emit(u.id)">
        {{ u.name }}
      </li>
    </ul>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class UsersListComponent {
  @Input() users: {id: string; name: string}[] | null = [];
  @Output() select = new EventEmitter<string>();
  trackById = (_: number, u: {id: string}) => u.id;
}
```

---

## 6) `ChangeDetectorRef` en détail

### `markForCheck()`
À utiliser quand vous changez un état **depuis un contexte async** ou externe (callback, lib tierce) et que la vue ne se met pas à jour.

- Ne déclenche pas immédiatement.
- Indique à Angular : “Pense à vérifier ce composant au prochain tick”.

### `detectChanges()`
- Lance immédiatement une vérification locale.
- À réserver aux cas particuliers (intégrations, composants détachés, etc.).

### `detach()` / `reattach()` (avancé)
Permet de **désactiver** la CD automatique sur un composant.
Cas typiques :
- rendu très coûteux,
- rafraîchissement contrôlé (ex : toutes les 250ms).

```ts
constructor(private cdr: ChangeDetectorRef) {}

ngOnInit() {
  this.cdr.detach();
  setInterval(() => {
    // mettre à jour l’état…
    this.cdr.detectChanges();
  }, 250);
}
```

---

## 7) `NgZone` et intégrations externes

Certaines bibliothèques (maps, charts, websockets custom) peuvent déclencher beaucoup d’événements.

### Réduire le bruit avec `runOutsideAngular`

```ts
constructor(private zone: NgZone, private cdr: ChangeDetectorRef) {}

ngAfterViewInit() {
  this.zone.runOutsideAngular(() => {
    window.addEventListener('mousemove', () => {
      // ne déclenche pas CD en continu
    });
  });
}
```

Puis, quand vous avez réellement besoin de mettre à jour la UI :

```ts
this.zone.run(() => {
  this.value = 123;
  // souvent inutile avec run(), mais markForCheck peut être utile selon les cas
  this.cdr.markForCheck();
});
```

---

## 8) Bonnes pratiques performance (complémentaires)

1. **`trackBy`** sur les listes
   - évite de recréer des nœuds DOM inutilement.
2. **Éviter les fonctions dans le template**
   - elles sont réévaluées souvent.
3. **Pipes purs**
   - un pipe pur ne s’exécute que si sa référence d’entrée change.
4. **Composants plus petits**
   - `OnPush` est plus efficace si les responsabilités sont bien découpées.
5. **État centralisé et immuable**
   - RxJS / store : updates prévisibles, références nouvelles.

---

## 9) Ateliers

### Atelier 1 — Convertir un composant Default → OnPush

**Énoncé** : vous avez un composant `ProductTableComponent` qui affiche une liste filtrée et triée. Objectif : passer en `OnPush` et mesurer la différence.

**Étapes** :
1. Ajouter `changeDetection: OnPush`.
2. Remplacer les mutations de tableau par des opérations immuables.
3. Ajouter `trackBy`.
4. Déplacer les calculs de tri/filtre hors template (RxJS ou computed côté TS).

**Critères de réussite** :
- UI toujours correcte.
- Moins de “checks” visibles dans Angular DevTools.

---

### Atelier 2 — Bug de mutation

**Énoncé** : un composant enfant `OnPush` reçoit `@Input() config`. Le parent fait `config.title = 'x'` et l’enfant ne se met pas à jour.

**Correction attendue** :
- dans le parent : `this.config = { ...this.config, title: 'x' }`.

---

### Atelier 3 — Live data + AsyncPipe

**Énoncé** : afficher une liste d’événements temps réel (Observable) avec pagination simple.

**Attendu** :
- `events$` alimente la vue via `AsyncPipe`.
- composant liste `OnPush`.
- `trackBy`.

---

## 10) Checklist de dépannage

### Symptômes fréquents
- “Mon composant OnPush ne se met pas à jour”
- “Une propriété interne change mais l’UI reste figée”

### Causes fréquentes
- Mutation d’un `@Input()` (même référence)
- `subscribe()` manuel sans `markForCheck()` alors que l’émission se fait hors zone / contexte particulier
- Calculs dans template qui masquent la source de vérité

### Méthode
1. Vérifier si la **référence** de l’input change.
2. Vérifier l’usage d’`AsyncPipe` vs subscribe manuel.
3. Inspecter les événements (est-ce bien un événement du composant ?).
4. Utiliser Angular DevTools pour voir le nombre de checks / cycles.

---

## Résumé (à retenir)

- `OnPush` améliore les performances en limitant les vérifications aux cas :
  1) **nouvelle référence** d’inputs, 2) **événements**, 3) **async** (ex : `AsyncPipe`), 4) **marquage manuel**.
- Pour que cela fonctionne : **immutabilité** + **flux maîtrisés**.
- Compléter avec : `AsyncPipe`, `trackBy`, découpage de composants, limitation des calculs dans les templates.
