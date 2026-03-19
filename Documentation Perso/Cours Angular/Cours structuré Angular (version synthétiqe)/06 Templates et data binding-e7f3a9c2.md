# Angular — Templates et Data Binding (Module 06)

## Informations générales
- **Public** : développeurs ayant des bases TypeScript/JS et notions Angular (composants/modules)
- **Prérequis** : Node.js, Angular CLI, notions de composants, bindings simples appréciées
- **Durée indicative** : 3h à 1 journée (selon profondeur et ateliers)
- **Objectifs pédagogiques** :
  - Comprendre le rôle du **template** Angular et du moteur de rendu
  - Maîtriser les 4 familles de **data binding** : interpolation, property, event, two-way
  - Savoir choisir le bon type de binding selon le cas d’usage
  - Écrire des templates lisibles, sûrs et performants
  - Mettre en place du two-way binding avec `[(ngModel)]` en respectant la structure Angular

---

## Plan de la formation
1. **Rappels : Templates Angular et cycle de rendu**
2. **Interpolation `{{ }}`**
3. **Property binding `[ ]`**
4. **Event binding `( )`**
5. **Two-way binding `[(ngModel)]`**
6. **Combinaisons, priorités, cas particuliers et bonnes pratiques**
7. **Ateliers pratiques (progressifs)**
8. **Synthèse & checklist**

---

## 1) Rappels : Templates Angular et cycle de rendu

### 1.1 Qu’est-ce qu’un template Angular ?
Un **template** Angular est du HTML enrichi par Angular, dans lequel on peut :
- afficher des **données** du composant
- réagir à des **événements** DOM
- lier des **propriétés** DOM/inputs de composants
- utiliser des **directives** (structurelles et attributaires)

Exemple de composant :

```ts
import { Component } from '@angular/core';

@Component({
  selector: 'app-demo',
  templateUrl: './demo.component.html'
})
export class DemoComponent {
  title = 'Templates & Data Binding';
}
```

Dans `demo.component.html`, on peut consommer `title`.

### 1.2 Unidirectionnel vs bidirectionnel
Angular propose principalement des bindings **unidirectionnels** :
- du composant vers la vue (affichage) : interpolation, property binding
- de la vue vers le composant (interaction) : event binding

Le **two-way binding** combine les deux directions.

### 1.3 Change detection (notions utiles)
Angular met à jour l’UI via la **change detection** :
- à chaque événement, promesse résolue, timer, etc., Angular vérifie les valeurs utilisées dans le template
- si une valeur a changé, l’UI est mise à jour

Conséquence pédagogique importante :
- éviter des **fonctions coûteuses** directement dans le template (elles peuvent être réévaluées souvent)

---

## 2) Interpolation `{{ }}`

### 2.1 Définition
L’**interpolation** permet d’insérer une expression Angular dans du texte HTML.

```html
<h1>{{ title }}</h1>
```

Ici, `title` est une propriété du composant.

### 2.2 Expressions autorisées
Dans une interpolation, on peut utiliser :
- propriétés du composant : `{{ user.name }}`
- opérateurs JS : `{{ count + 1 }}`
- ternaires : `{{ isAdmin ? 'Admin' : 'User' }}`
- appel de méthodes (à utiliser avec prudence) : `{{ fullName() }}`

Exemple :

```ts
export class DemoComponent {
  user = { firstName: 'Ada', lastName: 'Lovelace' };

  fullName(): string {
    return `${this.user.firstName} ${this.user.lastName}`;
  }
}
```

```html
<p>Bonjour {{ fullName() }}</p>
```

### 2.3 Interpolation et sécurité
Angular **échappe** les valeurs interpolées (protection XSS) lorsqu’elles sont rendues comme texte.

- ✅ `{{ userInput }}` est affiché en texte
- ⚠️ pour injecter du HTML, on utiliserait plutôt `[innerHTML]` (voir property binding) avec attention

### 2.4 Limites et erreurs fréquentes
- **Pas d’instructions** (pas de `if`, `for`, affectations `=`) : seulement des expressions.
- Éviter les appels de fonctions lourdes : préférer des propriétés calculées ou des `pipes`.

---

## 3) Property binding `[ ]`

### 3.1 Définition
Le **property binding** lie une **propriété DOM** (ou un **@Input** d’un composant enfant) à une expression Angular.

Syntaxe :
```html
<img [src]="imageUrl" />
```

### 3.2 Propriété DOM vs attribut HTML
Différence importante :
- **Attribut** (HTML) : valeur dans le markup initial
- **Propriété** (DOM) : valeur au runtime (celle qui pilote réellement l’état du composant DOM)

Exemple classique :
```html
<button [disabled]="isDisabled">Valider</button>
```
Si `isDisabled` vaut `true`, le bouton devient réellement désactivé.

### 3.3 Binding sur classes et styles
Angular fournit des syntaxes pratiques :

#### 3.3.1 Class binding
```html
<div [class.active]="isActive">...</div>
```
Ou via `ngClass` :
```html
<div [ngClass]="{ active: isActive, error: hasError }">...</div>
```

#### 3.3.2 Style binding
```html
<div [style.width.px]="width">...</div>
```
Ou via `ngStyle` :
```html
<div [ngStyle]="{ width: width + 'px', backgroundColor: color }">...</div>
```

### 3.4 Binding vers un composant enfant (@Input)
Exemple :

**Composant enfant**
```ts
import { Component, Input } from '@angular/core';

@Component({
  selector: 'app-user-card',
  template: `
    <article>
      <h3>{{ name }}</h3>
      <p>Rôle : {{ role }}</p>
    </article>
  `
})
export class UserCardComponent {
  @Input() name = '';
  @Input() role = '';
}
```

**Composant parent**
```html
<app-user-card [name]="user.name" [role]="user.role"></app-user-card>
```

### 3.5 Erreurs fréquentes
- Confondre string literal et expression :
  - ✅ `[src]="imageUrl"`
  - ❌ `src="imageUrl"` (met juste le texte "imageUrl")
- Utiliser l’interpolation dans un property binding (inutile) :
  - ❌ `[src]="{{imageUrl}}"`
  - ✅ `[src]="imageUrl"`

---

## 4) Event binding `( )`

### 4.1 Définition
L’**event binding** écoute un événement DOM (ou un événement custom d’un composant enfant via `@Output`) et exécute une expression.

```html
<button (click)="increment()">+</button>
```

### 4.2 Récupérer l’objet événement `$event`
```html
<input (input)="onInput($event)" />
```

```ts
onInput(event: Event) {
  const target = event.target as HTMLInputElement;
  console.log('Valeur:', target.value);
}
```

### 4.3 Passer des paramètres
```html
<button (click)="addToCart(product.id)">Ajouter</button>
```

### 4.4 Prévenir le comportement par défaut et propagation
On peut :
- appeler `preventDefault()` et `stopPropagation()` dans le handler TS
- ou utiliser les **modifiers** Angular sur certains événements (selon contexte)

Exemple :
```html
<a href="/home" (click)="onLinkClick($event)">Accueil</a>
```

```ts
onLinkClick(event: MouseEvent) {
  event.preventDefault();
  // navigation contrôlée par Angular Router
}
```

### 4.5 Événements custom via @Output
**Enfant** :
```ts
import { Component, EventEmitter, Output } from '@angular/core';

@Component({
  selector: 'app-like-button',
  template: `<button (click)="like()">J'aime</button>`
})
export class LikeButtonComponent {
  @Output() liked = new EventEmitter<void>();

  like() {
    this.liked.emit();
  }
}
```

**Parent** :
```html
<app-like-button (liked)="onLiked()"></app-like-button>
```

---

## 5) Two-way binding `[(ngModel)]`

### 5.1 Définition
Le **two-way binding** synchronise :
- une valeur du composant → affichage dans l’input
- les modifications utilisateur → mise à jour dans le composant

Syntaxe :
```html
<input [(ngModel)]="name" />
```

C’est un **sucre syntaxique** équivalent à :
- property binding `[ngModel]="name"`
- + event binding `(ngModelChange)="name = $event"`

### 5.2 Prérequis : importer FormsModule
Pour utiliser `ngModel` (approche **template-driven forms**), il faut importer `FormsModule`.

Avec des modules :
```ts
import { NgModule } from '@angular/core';
import { FormsModule } from '@angular/forms';

@NgModule({
  imports: [FormsModule]
})
export class AppModule {}
```

Avec un composant standalone :
```ts
import { Component } from '@angular/core';
import { FormsModule } from '@angular/forms';

@Component({
  standalone: true,
  selector: 'app-demo',
  imports: [FormsModule],
  templateUrl: './demo.component.html'
})
export class DemoComponent {}
```

### 5.3 Exemple complet
```ts
export class DemoComponent {
  name = 'Ada';
}
```

```html
<label>
  Nom:
  <input [(ngModel)]="name" />
</label>

<p>Vous avez saisi : {{ name }}</p>
```

### 5.4 Two-way binding sur composants custom
On peut créer un two-way binding custom en respectant la convention :
- `@Input() value` et `@Output() valueChange`
- puis utilisation : `[(value)]="..."`

**Composant enfant** :
```ts
import { Component, EventEmitter, Input, Output } from '@angular/core';

@Component({
  selector: 'app-rating',
  template: `
    <button (click)="set(1)">1</button>
    <button (click)="set(2)">2</button>
    <button (click)="set(3)">3</button>
    <span>Note: {{ value }}</span>
  `
})
export class RatingComponent {
  @Input() value = 0;
  @Output() valueChange = new EventEmitter<number>();

  set(v: number) {
    this.value = v;
    this.valueChange.emit(v);
  }
}
```

**Parent** :
```html
<app-rating [(value)]="rating"></app-rating>
<p>Note choisie : {{ rating }}</p>
```

### 5.5 Bonnes pratiques
- Privilégier `ngModel` pour des formulaires simples.
- Pour les formulaires complexes, préférer les **Reactive Forms** (hors scope ici), qui offrent plus de contrôle.

---

## 6) Combinaisons, priorités, cas particuliers et bonnes pratiques

### 6.1 Récapitulatif des syntaxes
| Besoin | Syntaxe | Sens |
|---|---|---|
| Afficher du texte | `{{ expr }}` | Composant → Vue |
| Lier une propriété DOM / Input | `[prop]="expr"` | Composant → Vue |
| Écouter un événement | `(event)="handler($event)"` | Vue → Composant |
| Synchroniser | `[(ngModel)]="prop"` | Deux sens |

### 6.2 Mélanger interpolation et bindings
Exemples valides :
```html
<h2>{{ user.name }}</h2>
<img [src]="user.avatarUrl" alt="{{ user.name }}" />
```

Ici, `alt` utilise l’interpolation car c’est une simple chaîne, mais on peut aussi écrire :
```html
<img [alt]="user.name" />
```

### 6.3 Binding et valeurs null/undefined
Utiliser l’opérateur de navigation sûre :
```html
<p>Ville : {{ user?.address?.city }}</p>
```

### 6.4 Performance : éviter les calculs répétés
Éviter :
```html
<p>Total: {{ computeTotal() }}</p>
```
Préférer :
- une propriété mise à jour quand nécessaire
- ou un `pipe` pur

### 6.5 Lisibilité
- Garder les templates simples
- Déporter la logique dans le composant
- Nommer clairement les handlers : `onSave()`, `onSearchChange()`

---

## 7) Ateliers pratiques (progressifs)

> Les ateliers peuvent être réalisés dans un projet Angular généré avec :
> ```bash
> ng new demo-binding --standalone
> ```

### Atelier 1 — Interpolation
**Objectif** : afficher des données simples et formatées.
1. Créer un composant `ProfileComponent`
2. Ajouter `firstName`, `lastName`, `age`
3. Afficher :
   - `{{ firstName }} {{ lastName }}`
   - `{{ age >= 18 ? 'Majeur' : 'Mineur' }}`

Critères de réussite : affichage correct et sans erreurs template.

### Atelier 2 — Property binding (état UI)
**Objectif** : activer/désactiver un bouton selon une condition.
- Champ input `email`
- Bouton "Envoyer" désactivé tant que `email` est vide

Indication : utiliser `[disabled]="email.length === 0"`.

### Atelier 3 — Event binding
**Objectif** : compter des clics et gérer `$event`.
- Bouton "Clique" incrémente un compteur
- Un champ input met à jour une propriété via `(input)`

### Atelier 4 — Two-way binding avec ngModel
**Objectif** : formulaire simple.
- Champ `name` en `[(ngModel)]`
- Select `role` en `[(ngModel)]`
- Afficher un résumé en dessous en interpolation

Rappel : importer `FormsModule`.

### Atelier 5 — Two-way binding custom
**Objectif** : créer un composant `app-toggle` avec `[(checked)]`.
- `@Input() checked` / `@Output() checkedChange`
- Un clic inverse l’état et émet

---

## 8) Synthèse & checklist

### Checklist de choix rapide
- **Afficher du texte** : `{{ }}`
- **Piloter une propriété/état DOM** : `[ ]`
- **Réagir aux actions utilisateur** : `( )`
- **Synchroniser un champ de formulaire** : `[(ngModel)]` (+ `FormsModule`)

### Points de vigilance
- Ne pas mettre de logique lourde dans le template
- Ne pas confondre attribut HTML et propriété DOM
- Pour `ngModel`, vérifier l’import du module et la déclaration du champ

---

## Annexes — Mini aide-mémoire

### Exemples rapides
```html
<!-- Interpolation -->
<p>{{ message }}</p>

<!-- Property binding -->
<img [src]="url" />
<button [disabled]="loading">Envoyer</button>

<!-- Event binding -->
<button (click)="onSave()">Save</button>
<input (input)="value = ($event.target as HTMLInputElement).value" />

<!-- Two-way binding -->
<input [(ngModel)]="query" />
```
