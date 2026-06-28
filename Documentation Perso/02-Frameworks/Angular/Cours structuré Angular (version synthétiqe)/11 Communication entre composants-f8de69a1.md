# Formation Angular — Communication entre composants

> **Objectif** : maîtriser la communication **descendante** (parent → enfant) via `@Input()` et la communication **remontante** (enfant → parent) via `@Output()` + `EventEmitter`.

---

## 1) Public visé & prérequis

### Public visé
- Développeurs front-end (débutant à intermédiaire) travaillant avec Angular.
- Formateurs ou tech leads souhaitant standardiser de bonnes pratiques.

### Prérequis
- Bases TypeScript (types, interfaces, classes).
- Bases Angular : composants, templates, data-binding, CLI.
- Compréhension de la notion de **hiérarchie de composants**.

---

## 2) Objectifs pédagogiques

À la fin de cette formation, vous saurez :
- Expliquer le flux de données **unidirectionnel** dans Angular.
- Transmettre des données d’un parent vers un enfant avec `@Input()`.
- Remonter un événement de l’enfant vers le parent avec `@Output()` et `EventEmitter`.
- Gérer les cas fréquents : alias, valeurs par défaut, setters `@Input`, et typage des événements.
- Appliquer des bonnes pratiques (immutabilité, nommage, limites des `EventEmitter`).

---

## 3) Plan de formation

1. Introduction : pourquoi et quand communiquer entre composants
2. Communication descendante : `@Input()`
   - Déclaration, liaison, typage
   - Alias, valeurs par défaut, `required`, et `@Input` setter
   - Détection de changements et bonnes pratiques
3. Communication remontante : `@Output()` + `EventEmitter`
   - Déclaration, émission (`emit`), réception
   - Typage d’événements et conventions de nommage
   - Cas d’usage : clic, sélection, suppression, formulaires
4. Atelier guidé : Parent/Enfant (liste + sélection + suppression)
5. FAQ & erreurs fréquentes
6. Récapitulatif

---

## 4) Introduction : flux de données et responsabilités

Dans une application Angular, l’interface est composée de **composants** organisés en **arbre** :
- Un **composant parent** instancie et configure un **composant enfant**.
- Angular encourage un flux :
  - **Données descendantes** : parent → enfant (paramétrage, affichage).
  - **Événements remontants** : enfant → parent (actions utilisateur, interactions).

### Pourquoi cette séparation ?
- **Lisibilité** : le parent pilote l’état, l’enfant affiche et déclenche des actions.
- **Testabilité** : l’enfant est plus facilement testable si ses entrées/sorties sont claires.
- **Réutilisabilité** : un composant enfant devient un « widget » générique.

---

## 5) Communication descendante avec `@Input()`

### 5.1 Principe
Un `@Input()` est une **propriété publique** de l’enfant que le parent peut lier depuis son template.

#### Parent (template)
```html
<app-user-card
  [user]="selectedUser"
  [isHighlighted]="true"
></app-user-card>
```

#### Enfant (classe)
```ts
import { Component, Input } from '@angular/core';

export interface User {
  id: number;
  name: string;
  email: string;
}

@Component({
  selector: 'app-user-card',
  templateUrl: './user-card.component.html'
})
export class UserCardComponent {
  @Input() user!: User;
  @Input() isHighlighted = false;
}
```

#### Enfant (template)
```html
<article [class.highlight]="isHighlighted">
  <h3>{{ user.name }}</h3>
  <p>{{ user.email }}</p>
</article>
```

---

### 5.2 Typage et valeurs par défaut

- **Toujours typer** les `@Input` pour éviter les erreurs à l’exécution.
- Fournissez des **valeurs par défaut** quand c’est pertinent.

Exemple :
```ts
@Input() title = 'Sans titre';
@Input() maxItems: number | null = null;
```

---

### 5.3 Alias d’Input

On peut exposer un nom différent dans le template parent, souvent pour :
- éviter un conflit de nom,
- harmoniser une API publique.

```ts
@Input('data') user!: User;
```

Utilisation côté parent :
```html
<app-user-card [data]="selectedUser"></app-user-card>
```

---

### 5.4 `@Input` requis (Angular récent)

Selon votre version Angular, vous pouvez rendre un input **obligatoire**.

```ts
import { Input } from '@angular/core';

@Input({ required: true }) user!: User;
```

Cela améliore la DX (developer experience) : Angular signale un input manquant.

---

### 5.5 Réagir aux changements : setter `@Input`

Quand un input change, vous pouvez :
- utiliser un **setter**
- ou `ngOnChanges` (hors scope détaillé ici, mais mention utile)

Setter :
```ts
private _user!: User;

@Input()
set user(value: User) {
  this._user = value;
  // Exemple : recalculer une valeur dérivée
  this.initials = value.name
    .split(' ')
    .map(p => p[0])
    .join('')
    .toUpperCase();
}
get user(): User {
  return this._user;
}

initials = '';
```

---

### 5.6 Bonnes pratiques `@Input`

- L’enfant **ne doit pas muter** directement un objet d’entrée, surtout si le parent gère l’état.
  - Préférez émettre une intention via `@Output` plutôt que modifier `user.name` dans l’enfant.
- Évitez de passer des fonctions volumineuses, préférez :
  - `@Output` pour les événements,
  - ou des services / state management pour des cas complexes.
- Gardez l’API du composant claire : peu d’inputs, bien nommés.

---

## 6) Communication remontante avec `@Output()` + `EventEmitter`

### 6.1 Principe
Un `@Output()` expose un **événement** que le parent peut écouter.

#### Enfant (classe)
```ts
import { Component, EventEmitter, Input, Output } from '@angular/core';
import { User } from '../models/user.model';

@Component({
  selector: 'app-user-card',
  templateUrl: './user-card.component.html'
})
export class UserCardComponent {
  @Input({ required: true }) user!: User;

  @Output() selected = new EventEmitter<User>();

  onSelect(): void {
    this.selected.emit(this.user);
  }
}
```

#### Enfant (template)
```html
<article (click)="onSelect()">
  <h3>{{ user.name }}</h3>
</article>
```

#### Parent (template)
```html
<app-user-card
  [user]="u"
  (selected)="onUserSelected($event)"
></app-user-card>
```

#### Parent (classe)
```ts
onUserSelected(user: User): void {
  this.selectedUser = user;
}
```

> `$event` correspond à la valeur passée à `emit(...)`.

---

### 6.2 Conventions de nommage

- Les outputs représentent des **événements** : utilisez des noms au **participe passé** ou orientés action.
  - `selected`, `deleted`, `submitted`, `valueChanged`…
- Évitez les préfixes comme `onSelected` pour l’output (souvent réservé au handler côté parent).

Exemple cohérent :
- Enfant : `@Output() deleted = new EventEmitter<number>();`
- Parent : `(deleted)="onDeleted($event)"`

---

### 6.3 Typage des événements

Vous pouvez émettre :
- une primitive (`number`, `string`, `boolean`),
- un objet métier (`User`),
- un payload structuré.

Exemple payload structuré :
```ts
export interface DeleteUserEvent {
  id: number;
  reason?: 'user-click' | 'timeout' | 'admin';
}

@Output() deleted = new EventEmitter<DeleteUserEvent>();

requestDelete(): void {
  this.deleted.emit({ id: this.user.id, reason: 'user-click' });
}
```

Parent :
```html
<app-user-card (deleted)="onUserDeleted($event)"></app-user-card>
```

```ts
onUserDeleted(event: DeleteUserEvent) {
  this.removeUser(event.id);
}
```

---

### 6.4 `EventEmitter` : bonnes pratiques

- Utilisez `EventEmitter` **uniquement** pour la communication composant → parent (outputs).
- Ne l’utilisez pas comme bus global d’événements entre composants non liés.
- Émettez des données **minimales** :
  - `id` plutôt que l’objet complet, si le parent peut le retrouver.

---

## 7) Atelier guidé : Liste d’utilisateurs (parent) + carte utilisateur (enfant)

### 7.1 Objectif
- Le parent affiche une liste.
- L’enfant (`UserCardComponent`) reçoit un user via `@Input`.
- L’enfant remonte `selected` et `deleted` via `@Output`.

---

### 7.2 Modèle
`user.model.ts`
```ts
export interface User {
  id: number;
  name: string;
  email: string;
}
```

---

### 7.3 Composant enfant : `UserCardComponent`
`user-card.component.ts`
```ts
import { Component, EventEmitter, Input, Output } from '@angular/core';
import { User } from '../models/user.model';

@Component({
  selector: 'app-user-card',
  templateUrl: './user-card.component.html',
  styleUrls: ['./user-card.component.css']
})
export class UserCardComponent {
  @Input({ required: true }) user!: User;
  @Input() selectedId: number | null = null;

  @Output() selected = new EventEmitter<number>();
  @Output() deleted = new EventEmitter<number>();

  get isSelected(): boolean {
    return this.selectedId === this.user.id;
  }

  select(): void {
    this.selected.emit(this.user.id);
  }

  delete(): void {
    this.deleted.emit(this.user.id);
  }
}
```

`user-card.component.html`
```html
<div class="card" [class.selected]="isSelected">
  <div class="content" (click)="select()">
    <h4>{{ user.name }}</h4>
    <small>{{ user.email }}</small>
  </div>

  <button type="button" (click)="delete()">Supprimer</button>
</div>
```

`user-card.component.css`
```css
.card {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 12px;
  border: 1px solid #ddd;
  border-radius: 8px;
  margin-bottom: 8px;
}

.card.selected {
  border-color: #1976d2;
  background: #e3f2fd;
}

.content {
  cursor: pointer;
}
```

Points clés :
- `@Input() selectedId` est une donnée **descendante** : le parent indique quel user est sélectionné.
- `@Output() selected` et `deleted` sont des événements **remontants**.
- On émet des `number` (IDs) : payload minimal.

---

### 7.4 Composant parent : `UserListComponent`
`user-list.component.ts`
```ts
import { Component } from '@angular/core';
import { User } from '../models/user.model';

@Component({
  selector: 'app-user-list',
  templateUrl: './user-list.component.html'
})
export class UserListComponent {
  users: User[] = [
    { id: 1, name: 'Ada Lovelace', email: 'ada@history.dev' },
    { id: 2, name: 'Alan Turing', email: 'alan@history.dev' },
    { id: 3, name: 'Grace Hopper', email: 'grace@history.dev' }
  ];

  selectedId: number | null = null;

  onSelected(id: number): void {
    this.selectedId = id;
  }

  onDeleted(id: number): void {
    this.users = this.users.filter(u => u.id !== id);
    if (this.selectedId === id) {
      this.selectedId = null;
    }
  }
}
```

`user-list.component.html`
```html
<h2>Utilisateurs</h2>

<app-user-card
  *ngFor="let u of users"
  [user]="u"
  [selectedId]="selectedId"
  (selected)="onSelected($event)"
  (deleted)="onDeleted($event)"
></app-user-card>

<p *ngIf="selectedId !== null">
  Sélection : {{ selectedId }}
</p>
```

Ce que vous venez d’implémenter :
- Un **état** (`selectedId`, `users`) géré par le parent.
- Des **entrées** (`user`, `selectedId`) pour configurer l’enfant.
- Des **sorties** (`selected`, `deleted`) pour remonter les intentions.

---

## 8) FAQ & erreurs fréquentes

### 8.1 « Mon `@Input()` vaut `undefined` »
Causes courantes :
- L’input n’est pas bindé côté parent.
- L’input arrive plus tard (données asynchrones).

Solutions :
- Utiliser `@Input({required:true})` si possible.
- Gérer le template avec du rendu conditionnel :

```html
<app-user-card *ngIf="selectedUser" [user]="selectedUser"></app-user-card>
```

---

### 8.2 « Mon `(output)` n’est jamais appelé »
Vérifiez :
- Le nom de l’output : `(selected)` vs `(select)`…
- Que la fonction est bien déclenchée (clic sur le bon élément).
- Que vous appelez bien `emit(...)`.

---

### 8.3 « Je modifie l’objet reçu et ça cause des effets de bord »
Règle :
- L’enfant ne doit pas modifier directement un input.

Préférez :
- émettre un événement au parent,
- ou travailler sur une copie locale si besoin.

---

## 9) Récapitulatif

- **Parent → Enfant** : `@Input()` pour la donnée descendante.
- **Enfant → Parent** : `@Output()` + `EventEmitter` pour les événements remontants.
- Typage, conventions de nommage, et payload minimal rendent l’API composant robuste.

---

## 10) Exercices (pour aller plus loin)

1. Ajouter un `@Output() edited` qui émet `{ id, name }` et mettre à jour la liste côté parent.
2. Ajouter un input `disabled` et empêcher la sélection/suppression quand `true`.
3. Ajouter un composant `UserDetailsComponent` (enfant) qui reçoit l’utilisateur sélectionné via `@Input()`.

---

### Annexes — Exemples rapides

#### Exemple d’input avec alias
```ts
@Input('value') count = 0;
```
```html
<app-counter [value]="42"></app-counter>
```

#### Exemple d’output événement simple
```ts
@Output() closed = new EventEmitter<void>();
close() { this.closed.emit(); }
```
```html
<app-modal (closed)="onClosed()"></app-modal>
```
