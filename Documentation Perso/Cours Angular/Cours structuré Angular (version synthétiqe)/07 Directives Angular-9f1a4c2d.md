# Formation Angular — Directives Angular

## Objectifs pédagogiques
À l’issue de cette formation, le participant sera capable de :

- Expliquer le rôle des **directives** dans Angular et leur lien avec le **DOM**.
- Distinguer **directives structurelles** et **directives attributaires**.
- Utiliser efficacement les directives natives (`*ngIf`, `*ngFor`, `ngClass`, `ngStyle`, etc.).
- Comprendre la **syntaxe étoile** (`*`) et son **désucrage** en `<ng-template>`.
- Éviter les pièges courants (performance, lisibilité, accessibilité).
- (Option avancée) Créer une directive **attributaire** simple et comprendre un cas d’usage courant.

## Prérequis
- Connaissances de base en Angular (composants, templates, bindings).
- Environnement de dev fonctionnel (Node.js, Angular CLI) et notions TypeScript.

## Public
- Développeurs Angular (débutant → intermédiaire)
- Formateurs / mentors Angular

## Durée recommandée
- 3h à 1 jour selon le niveau et la profondeur (avec ateliers).

---

# 1. Introduction : qu’est-ce qu’une directive Angular ?

Une **directive** est une classe Angular qui permet d’indiquer au framework **comment manipuler le DOM**.

Dans Angular, tout ce qui "vit" dans le template est orchestré via :

- des **components** (qui sont en réalité des directives avec un template)
- des **directives** (qui ajoutent/retirent des éléments ou modifient leur comportement)

### 1.1 Pourquoi les directives sont centrales ?

- Elles rendent le template **déclaratif** : on décrit une intention (`*ngIf="isLoggedIn"`) plutôt que de manipuler le DOM à la main.
- Elles encapsulent des comportements réutilisables (ex : afficher/masquer, boucler, ajouter une classe, gérer un rôle, etc.).

---

# 2. Les deux grandes familles de directives

## 2.1 Directives structurelles

**Rôle :** modifier la **structure du DOM** : ajout/retrait d’éléments, changement de contexte de template, création de vues.

Caractéristiques :

- Souvent reconnaissables au préfixe `*` dans le template.
- Elles **créent ou détruisent** des portions de DOM.

Exemples :

- `*ngIf`
- `*ngFor`
- `*ngSwitchCase`, `*ngSwitchDefault` (avec `[ngSwitch]`)

## 2.2 Directives attributaires

**Rôle :** modifier l’**apparence** ou le **comportement** d’un élément existant.

Caractéristiques :

- Elles s’utilisent comme un attribut HTML.
- Elles n’ajoutent pas/retirent pas l’élément du DOM ; elles modifient son style, ses classes, ses propriétés, son comportement.

Exemples :

- `[ngClass]`
- `[ngStyle]`
- `ngModel` (Forms)

---

# 3. Directives structurelles — pratique et compréhension

## 3.1 `*ngIf` — afficher/masquer une portion du DOM

### 3.1.1 Cas simple

```html
<div *ngIf="isAdmin">Bienvenue, admin.</div>
```

- Si `isAdmin === true` : l’élément est **présent** dans le DOM.
- Si `false` : l’élément est **retiré** du DOM (ce n’est pas simplement `display: none`).

### 3.1.2 Variante `else`

```html
<div *ngIf="isLoggedIn; else guestTpl">
  Bonjour {{ user.name }}
</div>

<ng-template #guestTpl>
  <a routerLink="/login">Se connecter</a>
</ng-template>
```

### 3.1.3 Variante `then/else`

```html
<ng-container *ngIf="loaded; then contentTpl; else loadingTpl"></ng-container>

<ng-template #contentTpl>
  <app-dashboard></app-dashboard>
</ng-template>

<ng-template #loadingTpl>
  Chargement...
</ng-template>
```

> `ng-container` ne génère pas de nœud HTML : pratique pour conditionner sans ajouter de wrapper.

---

## 3.2 `*ngFor` — itérer sur une collection

### 3.2.1 Boucle simple

```html
<ul>
  <li *ngFor="let p of products">{{ p.name }}</li>
</ul>
```

### 3.2.2 Variables locales utiles

`*ngFor` expose des variables locales :

- `index` : position
- `first`, `last` : booléens
- `even`, `odd` : booléens
- `count` (selon versions) : taille

```html
<li *ngFor="let user of users; let i = index; let isFirst = first">
  #{{ i }} — {{ user.name }} <strong *ngIf="isFirst">(Premier)</strong>
</li>
```

### 3.2.3 Performance : `trackBy`

Sans `trackBy`, Angular compare souvent par **référence**, ce qui peut provoquer des recréations inutiles.

```ts
trackById(index: number, item: { id: number }) {
  return item.id;
}
```

```html
<li *ngFor="let item of items; trackBy: trackById">
  {{ item.label }}
</li>
```

**Bénéfices** :

- Moins de destructions/recréations dans le DOM
- Meilleure performance sur grandes listes
- Conservation d’état (ex : focus, inputs)

---

## 3.3 `ngSwitch` — gérer des cas multiples

```html
<div [ngSwitch]="status">
  <p *ngSwitchCase="'success'">Succès</p>
  <p *ngSwitchCase="'error'">Erreur</p>
  <p *ngSwitchDefault>En attente</p>
</div>
```

---

# 4. Comprendre la syntaxe `*` (désucrage en `<ng-template>`)

La syntaxe `*` est un **sucre syntaxique**.

Exemple :

```html
<div *ngIf="visible">Salut</div>
```

Équivaut conceptuellement à :

```html
<ng-template [ngIf]="visible">
  <div>Salut</div>
</ng-template>
```

### Conséquences importantes

- Une directive structurelle agit sur une **vue** (template), pas directement sur un élément.
- Cela explique pourquoi on ne peut avoir **qu’une seule directive structurelle** avec `*` sur un même élément.

#### Contournement : `ng-container`

```html
<ng-container *ngIf="loggedIn">
  <div *ngFor="let n of notifications">{{ n.title }}</div>
</ng-container>
```

---

# 5. Directives attributaires — pratique

## 5.1 `[ngClass]` — classes conditionnelles et dynamiques

### 5.1.1 Avec objet

```html
<button
  [ngClass]="{ 'btn': true, 'btn-primary': isPrimary, 'btn-disabled': disabled }">
  Valider
</button>
```

### 5.1.2 Avec tableau

```html
<div [ngClass]="['card', theme, isActive ? 'active' : '']"></div>
```

### 5.1.3 Avec string

```html
<div [ngClass]="currentClass"></div>
```

---

## 5.2 `[ngStyle]` — styles embarqués

```html
<div [ngStyle]="{ 'background-color': bgColor, 'font-size.px': fontSize }">
  Texte stylé
</div>
```

Bonnes pratiques :

- Préférer CSS/SCSS et classes (maintenabilité).
- Utiliser `ngStyle` pour des styles réellement dynamiques.

---

## 5.3 Différence clé : structurelles vs attributaires

| Type | Effet sur le DOM | Exemple | Usage |
|------|-------------------|---------|-------|
| Structurelle | Ajoute/retire des nœuds | `*ngIf`, `*ngFor` | Conditionner/itérer |
| Attributaire | Modifie un nœud existant | `[ngClass]`, `[ngStyle]` | Style/comportement |

---

# 6. Ateliers (exercices guidés)

## Atelier A — Liste filtrée avec `*ngFor`, `*ngIf` et `trackBy`

**Objectif :** afficher une liste d’articles, filtrer par texte, et optimiser le rendu.

1. Créer une propriété `filterText` liée à un `<input>`.
2. Afficher seulement les items dont le titre contient `filterText`.
3. Ajouter un `trackBy` basé sur `id`.

Exemple de template :

```html
<input [(ngModel)]="filterText" placeholder="Filtrer..." />

<ul>
  <li *ngFor="let a of articles; trackBy: trackById">
    <span *ngIf="a.title.toLowerCase().includes(filterText.toLowerCase())">
      {{ a.title }}
    </span>
  </li>
</ul>
```

> Variante plus propre : filtrer en amont via `computed` (signals) ou via getter/pipe (attention perf).

---

## Atelier B — Affichage conditionnel avec `then/else`

**Objectif :** afficher un état de chargement.

- `loading = true` au départ
- après chargement (simulation), `loading = false`

```html
<ng-container *ngIf="!loading; else loadingTpl">
  <app-profile></app-profile>
</ng-container>

<ng-template #loadingTpl>
  <p>Chargement du profil...</p>
</ng-template>
```

---

# 7. (Option avancée) Créer une directive attributaire simple

Même si l’essentiel de cette formation porte sur l’utilisation, savoir créer une directive aide à comprendre le mécanisme.

## 7.1 Exemple : directive `highlight`

### 7.1.1 Génération

```bash
ng generate directive shared/highlight
```

### 7.1.2 Implémentation

```ts
import { Directive, ElementRef, HostListener, Input } from '@angular/core';

@Directive({
  selector: '[appHighlight]'
})
export class HighlightDirective {
  @Input('appHighlight') color = 'yellow';

  constructor(private el: ElementRef<HTMLElement>) {}

  @HostListener('mouseenter') onEnter() {
    this.el.nativeElement.style.backgroundColor = this.color;
  }

  @HostListener('mouseleave') onLeave() {
    this.el.nativeElement.style.backgroundColor = '';
  }
}
```

### 7.1.3 Utilisation

```html
<p [appHighlight]="'lightblue'">
  Survolez-moi
</p>
```

**Points à retenir :**

- Une directive attributaire interagit typiquement via `ElementRef`, `Renderer2` (souvent recommandé) et des `HostListener`.
- Elle ne change pas la structure du DOM (contrairement à `*ngIf/*ngFor`).

---

# 8. Bonnes pratiques et pièges

## 8.1 Une seule directive structurelle `*` par élément

Incorrect :

```html
<div *ngIf="ok" *ngFor="let x of xs"></div>
```

Correct :

```html
<ng-container *ngIf="ok">
  <div *ngFor="let x of xs">{{ x }}</div>
</ng-container>
```

## 8.2 Attention aux performances

- Éviter de calculer lourdement dans le template à chaque détection de changements.
- Utiliser `trackBy` pour les listes.
- Préférer des structures de données stables.

## 8.3 Lisibilité

- Ne pas surcharger un attribut `ngClass/ngStyle`.
- Extraire des conditions complexes en méthodes pures / propriétés calculées.

## 8.4 Accessibilité

- `*ngIf` retirant le contenu, attention aux annonces lecteurs d’écran.
- Préférer parfois masquer visuellement (CSS) si la présence sémantique doit rester.

---

# 9. Récapitulatif

- Les **directives** manipulent le DOM de manière déclarative.
- Les **directives structurelles** (`*ngIf`, `*ngFor`) modifient la **structure** du DOM.
- Les **directives attributaires** (`[ngClass]`, `[ngStyle]`) modifient **apparence** ou **comportement**.
- La syntaxe `*` est un **sucre** basé sur `<ng-template>`.
- `trackBy` est un levier majeur de performance pour `*ngFor`.

---

# Annexes — Cheatsheet rapide

## Structurelles

- `*ngIf="cond"`
- `*ngIf="cond; else tpl"`
- `*ngFor="let x of xs; trackBy: fn"`
- `[ngSwitch]="expr"` + `*ngSwitchCase` / `*ngSwitchDefault`

## Attributaires

- `[ngClass]="{...}"`
- `[ngStyle]="{...}"`

