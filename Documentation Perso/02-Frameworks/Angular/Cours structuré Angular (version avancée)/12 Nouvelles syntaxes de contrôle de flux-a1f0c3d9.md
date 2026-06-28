# Formation Angular — 12. Nouvelles syntaxes de contrôle de flux (@if, @for, @switch)

> **Public visé** : développeurs Angular (intermédiaire) et formateurs / référents techniques
>
> **Objectif** : maîtriser les **nouvelles syntaxes de contrôle de flux** introduites dans Angular (ex. `@if`, `@for`, `@switch`) afin d’écrire des templates **plus lisibles**, **plus déclaratifs** et parfois **plus performants** que leurs équivalents historiques (`*ngIf`, `*ngFor`, `*ngSwitch`).

---

## Sommaire

1. [Pré-requis et contexte](#1-pré-requis-et-contexte)
2. [Pourquoi une nouvelle syntaxe ?](#2-pourquoi-une-nouvelle-syntaxe-)
3. [@if : conditions modernes](#3-if--conditions-modernes)
4. [@for : itérations modernes](#4-for--itérations-modernes)
5. [@switch : branchements modernes](#5-switch--branchements-modernes)
6. [Migration et cohabitation avec les anciennes directives](#6-migration-et-cohabitation-avec-les-anciennes-directives)
7. [Bonnes pratiques : lisibilité, perf, maintenance](#7-bonnes-pratiques--lisibilité-perf-maintenance)
8. [Ateliers / exercices](#8-ateliers--exercices)
9. [Corrections proposées](#9-corrections-proposées)
10. [Récapitulatif et checklist](#10-récapitulatif-et-checklist)

---

## 1. Pré-requis et contexte

### Pré-requis
- Connaître la base d’Angular (components, binding, services, RxJS non obligatoire).
- Avoir déjà utilisé :
  - `*ngIf`, `*ngFor`, `ngSwitch`
  - `ng-container`, `ng-template`
- Connaître le concept de **templates** Angular et le rendu conditionnel.

### Contexte technique
Les syntaxes `@if`, `@for`, `@switch` sont des **contrôles de flux** pour templates Angular.
Elles s’inscrivent dans la modernisation du templating Angular (notamment autour d’une approche plus déclarative et plus claire, proche de certains langages / frameworks modernes).

> **Note** : selon les versions, l’activation et la disponibilité peuvent varier. Dans un contexte de formation, on se concentre sur **la syntaxe**, les **cas d’usage** et les **impacts** (lisibilité / perf / maintenance). Vérifiez la version d’Angular de votre projet et la configuration recommandée par la documentation officielle.

---

## 2. Pourquoi une nouvelle syntaxe ?

### 2.1 Limites (ou irritants) des anciennes directives structurelles
Les directives `*ngIf` et `*ngFor` fonctionnent très bien, mais :
- elles reposent sur une **désucrage** (`*`) vers `ng-template`, parfois difficile à expliquer.
- certains templates deviennent vite verbeux avec :
  - `ng-container`
  - `ng-template` avec `else`
  - alias `as`, variables locales, `trackBy`, etc.
- la lecture “de haut niveau” (logique de flux) est moins immédiate.

### 2.2 Objectifs des syntaxes modernes
Les syntaxes `@...` visent à :
- **rendre explicite** le contrôle de flux dans le template ;
- réduire la “cérémonie” des `ng-template` ;
- améliorer la **lisibilité** ;
- offrir des opportunités d’optimisation (selon les cas) en étant mieux comprises par le compilateur/template engine.

### 2.3 Cohérence et style
Adopter `@if/@for/@switch` permet d’avoir des templates plus “code-like” :
- blocs avec accolades `{ ... }`
- branches explicites (équivalentes à `if/else`, `for`, `switch/case`)

---

## 3. @if : conditions modernes

### 3.1 Syntaxe de base

```html
@if (isLoading) {
  <app-spinner />
}
```

Équivalent historique :

```html
<app-spinner *ngIf="isLoading" />
```

### 3.2 Branches `@else if` et `@else`

```html
@if (error) {
  <app-error [error]="error" />
} @else if (items.length === 0) {
  <p>Aucun résultat.</p>
} @else {
  <app-list [items]="items" />
}
```

**Points clés**
- Enchaînement naturel des branches.
- Lisibilité améliorée : tout est au même “niveau logique”.

### 3.3 Déclaration d’alias (pattern “as”)
Un cas classique : on veut éviter d’évaluer plusieurs fois une expression (ex. un `async`, un calcul).

> Selon version et conventions, l’alias peut se faire via une syntaxe dédiée. Le principe est : **capturer une valeur** et la réutiliser dans le bloc.

Exemple (conceptuel) :

```html
@if (user$ | async; as user) {
  <h2>{{ user.name }}</h2>
  <p>{{ user.email }}</p>
} @else {
  <p>Utilisateur non chargé.</p>
}
```

Le but est de :
- réduire la duplication
- améliorer la stabilité visuelle (éviter la répétition de pipes)
- clarifier l’intention

### 3.4 `@if` et templates propres
Avec `@if`, vous évitez souvent :
- `ng-container` uniquement utilisé comme wrapper
- `ng-template #elseBlock`

Avant :

```html
<ng-container *ngIf="user$ | async as user; else empty">
  <h2>{{ user.name }}</h2>
</ng-container>
<ng-template #empty>
  <p>Pas d’utilisateur</p>
</ng-template>
```

Après :

```html
@if (user$ | async; as user) {
  <h2>{{ user.name }}</h2>
} @else {
  <p>Pas d’utilisateur</p>
}
```

### 3.5 Erreurs courantes
- Mettre trop de logique dans la condition (préférez des getters / computed / variables).
- Chaîner des branches longues : parfois extraire en composants.

---

## 4. @for : itérations modernes

### 4.1 Syntaxe de base

```html
@for (item of items) {
  <li>{{ item.label }}</li>
}
```

Équivalent historique :

```html
<li *ngFor="let item of items">{{ item.label }}</li>
```

### 4.2 Traçage / identité : `track`
Le point le plus important en performance sur des listes : **identifier les éléments**.

Avec la syntaxe moderne :

```html
@for (item of items; track item.id) {
  <li>{{ item.label }}</li>
}
```

**Pourquoi ?**
- Angular peut **réutiliser** le DOM existant plutôt que recréer.
- Réduit les clignotements et améliore les performances.

Équivalent avec `trackBy` :

```ts
trackById = (_: number, item: { id: string }) => item.id;
```

```html
<li *ngFor="let item of items; trackBy: trackById">
  {{ item.label }}
</li>
```

### 4.3 Variables de contexte (index, first, last…)
Vous avez souvent besoin de : index, pair/impair, premier/dernier.

Exemple :

```html
@for (item of items; track item.id; let i = $index) {
  <div class="row">
    <span class="index">#{{ i + 1 }}</span>
    <span>{{ item.label }}</span>
  </div>
}
```

> Selon version, les variables exposées sont souvent `$index`, `$count`, `$first`, `$last`, `$even`, `$odd`.

### 4.4 Bloc `@empty`
Très pratique : gérer le cas “aucun élément” directement dans la boucle.

```html
@for (item of items; track item.id) {
  <li>{{ item.label }}</li>
} @empty {
  <li class="muted">Aucun élément</li>
}
```

Cela évite de combiner `*ngIf` + `*ngFor` ou d’ajouter un `ng-template`.

### 4.5 Erreurs et anti-patterns
- Oublier `track` sur des collections dynamiques.
- Utiliser `track $index` quand l’ordre peut changer (tri, insertion au début) : cela cause des réaffectations DOM imprévues.

---

## 5. @switch : branchements modernes

### 5.1 Syntaxe de base

```html
@switch (status) {
  @case ('loading') {
    <app-spinner />
  }
  @case ('error') {
    <app-error />
  }
  @default {
    <app-content />
  }
}
```

Équivalent historique :

```html
<div [ngSwitch]="status">
  <app-spinner *ngSwitchCase="'loading'" />
  <app-error *ngSwitchCase="'error'" />
  <app-content *ngSwitchDefault />
</div>
```

### 5.2 Utiliser `@switch` pour clarifier un état d’UI
Cas typique : `loading | error | empty | success`.

```html
@switch (vm.state) {
  @case ('loading') {
    <app-spinner />
  }
  @case ('empty') {
    <p>Aucun résultat.</p>
  }
  @case ('error') {
    <app-error [error]="vm.error" />
  }
  @default {
    <app-list [items]="vm.items" />
  }
}
```

**Bénéfices**
- Regroupe la logique d’affichage.
- Limite les `@if` imbriqués.
- Mise au même niveau des cas.

### 5.3 Bonnes pratiques
- Préférer un **state machine** simple (string union, enum) plutôt que des conditions dispersées.
- Toujours prévoir `@default` pour les cas inattendus.

---

## 6. Migration et cohabitation avec les anciennes directives

### 6.1 Cohabitation
Vous pouvez rencontrer dans un même projet :
- des anciens templates en `*ngIf/*ngFor`
- des nouveaux templates en `@if/@for`

Recommandation pédagogique :
- définir une règle d’équipe (ex. « nouveaux composants => nouvelle syntaxe »)
- migrer progressivement sur les zones refactorées

### 6.2 Guide de migration rapide (mapping)

| Ancien | Nouveau |
|---|---|
| `*ngIf="cond"` | `@if (cond) { ... }` |
| `*ngIf="cond; else tpl"` | `@if (cond) { ... } @else { ... }` |
| `*ngFor="let x of xs"` | `@for (x of xs) { ... }` |
| `trackBy: fn` | `track expr` (ex. `track x.id`) |
| `ngSwitch/ngSwitchCase` | `@switch/@case/@default` |

### 6.3 Points de vigilance
- Revoir les `trackBy` : passer d’une **fonction** à une **expression** demande de s’assurer que l’expression est stable.
- Attention aux templates complexes : profitez-en pour **extraire des sous-composants**.

---

## 7. Bonnes pratiques : lisibilité, perf, maintenance

### 7.1 Lisibilité
- Éviter les conditions trop longues :
  - calculez dans le composant (`readonly`, getters, signaux/computed si utilisés)
  - utilisez des variables de vue (vm)
- Favoriser des blocs courts :
  - si un bloc dépasse ~30-40 lignes => envisager un composant.

### 7.2 Performance
- Utiliser `track` systématiquement sur `@for` quand la liste est dynamique.
- Préférer `@switch` à des `@if` imbriqués quand il s’agit d’un choix parmi plusieurs états.
- Éviter de réévaluer des expressions coûteuses : alias + préparations côté TS.

### 7.3 Robustesse
- `@default` dans `@switch`.
- `@empty` dans `@for` quand c’est pertinent.
- Tests : vérifier les états `loading/error/empty/success`.

---

## 8. Ateliers / exercices

### Exercice 1 — Convertir un `*ngIf` avec `else`
**Objectif** : remplacer `*ngIf` + `ng-template` par `@if/@else`.

Code initial :

```html
<ng-container *ngIf="user$ | async as user; else noUser">
  <h2>{{ user.name }}</h2>
</ng-container>
<ng-template #noUser>
  <p>Pas d’utilisateur</p>
</ng-template>
```

Attendu : version `@if`.

---

### Exercice 2 — Ajouter `@empty` et `track`
**Objectif** : itérer proprement et gérer le cas vide.

Code initial :

```html
<ul>
  <li *ngFor="let p of products">{{ p.name }}</li>
</ul>
<p *ngIf="products.length === 0">Aucun produit</p>
```

Attendu : version `@for` avec `track p.id` + `@empty`.

---

### Exercice 3 — Transformer un `ngSwitch` en `@switch`
**Objectif** : clarifier un affichage multi-états.

Code initial :

```html
<div [ngSwitch]="state">
  <p *ngSwitchCase="'loading'">Chargement…</p>
  <p *ngSwitchCase="'empty'">Aucun résultat</p>
  <p *ngSwitchCase="'error'">Erreur</p>
  <app-list *ngSwitchDefault [items]="items" />
</div>
```

Attendu : version `@switch/@case/@default`.

---

## 9. Corrections proposées

### Correction Exercice 1

```html
@if (user$ | async; as user) {
  <h2>{{ user.name }}</h2>
} @else {
  <p>Pas d’utilisateur</p>
}
```

### Correction Exercice 2

```html
<ul>
  @for (p of products; track p.id) {
    <li>{{ p.name }}</li>
  } @empty {
    <li class="muted">Aucun produit</li>
  }
</ul>
```

### Correction Exercice 3

```html
@switch (state) {
  @case ('loading') {
    <p>Chargement…</p>
  }
  @case ('empty') {
    <p>Aucun résultat</p>
  }
  @case ('error') {
    <p>Erreur</p>
  }
  @default {
    <app-list [items]="items" />
  }
}
```

---

## 10. Récapitulatif et checklist

### À retenir
- `@if` remplace proprement beaucoup de cas `*ngIf` + `ng-template`.
- `@for` apporte :
  - syntaxe claire
  - `track` simple
  - `@empty` intégré
- `@switch` simplifie les affichages multi-états.

### Checklist avant de valider un template
- [ ] Mes conditions `@if` sont-elles lisibles (pas trop de logique inline) ?
- [ ] Ai-je utilisé un `@switch` lorsqu’il y a 3+ branches exclusives ?
- [ ] Ai-je un `track` stable dans mes `@for` ?
- [ ] Ai-je géré le cas vide avec `@empty` quand nécessaire ?
- [ ] Mes blocs ne sont-ils pas trop longs (extraction composant) ?

---

## Annexes

### A. Exemple complet (mini-composant)

**TypeScript (exemple)**

```ts
export type State = 'loading' | 'error' | 'empty' | 'success';

export interface Product {
  id: string;
  name: string;
}

export class ProductsComponent {
  state: State = 'loading';
  products: Product[] = [];
  error?: unknown;

  // Exemple : après chargement
  setProducts(products: Product[]) {
    this.products = products;
    this.state = products.length ? 'success' : 'empty';
  }
}
```

**Template**

```html
@switch (state) {
  @case ('loading') {
    <p>Chargement…</p>
  }

  @case ('error') {
    <p>Une erreur est survenue.</p>
  }

  @case ('empty') {
    <p>Aucun produit.</p>
  }

  @default {
    <ul>
      @for (p of products; track p.id) {
        <li>{{ p.name }}</li>
      }
    </ul>
  }
}
```
