# Formation – Introduction à Angular

**Public visé :** développeurs web (débutants à intermédiaires) souhaitant découvrir Angular et sa philosophie.

**Pré-requis :**
- HTML/CSS (bases)
- JavaScript (ES6+)
- Notions de programmation orientée objet (recommandé)

**Durée indicative :** 1 jour (7h) ou 2 demi-journées (2×3h30)

**Objectifs pédagogiques :**
- Comprendre ce qu’est Angular et dans quels cas l’utiliser
- Installer et démarrer un projet Angular avec l’Angular CLI
- Maîtriser les concepts fondamentaux : composants, templates, binding, directives, services, DI
- Comprendre la structuration d’une application Angular (modules, routing)
- Mettre en place un mini-projet fil rouge

---

## Plan de la formation

1. **Panorama d’Angular**
   - Qu’est-ce qu’Angular ?
   - Différences avec d’autres frameworks (React/Vue) – vue d’ensemble
   - Cas d’usage et avantages

2. **Préparer l’environnement**
   - Node.js, npm, TypeScript
   - Angular CLI
   - Structure d’un projet Angular

3. **Architecture Angular : la logique par composants**
   - Composant : rôle et responsabilités
   - Template, styles, classe TS
   - Cycle de vie (aperçu)

4. **Templates et Data Binding**
   - Interpolation
   - Property Binding
   - Event Binding
   - Two-way binding

5. **Directives et composants intégrés**
   - Directives structurelles (`*ngIf`, `*ngFor`, `*ngSwitch`)
   - Directives d’attributs (`[ngClass]`, `[ngStyle]`)
   - Pipes (intro)

6. **Services et Injection de Dépendances (DI)**
   - Pourquoi des services ?
   - `@Injectable()` et `providedIn`
   - Injectable vs singleton
   - Bonnes pratiques

7. **Modules (NgModules) et organisation**
   - Rôle des modules
   - Module racine et modules de fonctionnalité
   - Composants déclarés vs importés

8. **Routing (Introduction)**
   - Configuration des routes
   - Navigation (routerLink)
   - Route paramétrée (aperçu)

9. **HTTP & Observables (Introduction)**
   - `HttpClient`
   - Observable : notion, subscription
   - Gestion simple des erreurs

10. **Mini-projet fil rouge**
   - Construction d’une petite application (liste + détail)
   - Mise en place de composants, services, routing
   - Récupération de données simulées

11. **Synthèse & suite du parcours**
   - Récapitulatif
   - Bonnes pratiques
   - Ressources et axes d’approfondissement

---

# 1. Panorama d’Angular

## 1.1 Qu’est-ce qu’Angular ?
Angular est un **framework frontend** développé par **Google** pour construire des applications web **dynamiques**, **modulaires** et **scalables**.

Caractéristiques clés :
- Basé sur **TypeScript** (langage typé, superset de JavaScript)
- Architecture **orientée composants**
- Utilise l’**injection de dépendances** comme mécanisme standard
- Fournit un ensemble complet : **routing**, **HTTP**, **forms**, **outillage (CLI)**, tests

## 1.2 Angular vs bibliothèque
Angular est un *framework* : il propose une manière structurée de faire, avec des conventions et une architecture.

Exemples :
- **React** est souvent considéré comme une bibliothèque (UI) complétée par d’autres libs.
- **Vue** peut être utilisé comme framework progressif.

Angular arrive avec “tout le nécessaire” :
- Router officiel
- Gestion avancée de formulaires
- HttpClient
- Outils de build, tests, lint, etc.

## 1.3 Quand choisir Angular ?
Angular est adapté lorsque :
- L’application est **moyenne à grande**
- On veut une **structure** et de la **maintenabilité**
- Les équipes sont plus grandes, ou le projet à longue durée
- On veut bénéficier d’un écosystème stable et documenté

---

# 2. Préparer l’environnement

## 2.1 Prérequis techniques
- **Node.js** (LTS recommandé)
- **npm** (ou pnpm/yarn)
- Un éditeur : **VS Code** (recommandé)

Vérifier :
```bash
node -v
npm -v
```

## 2.2 Installer l’Angular CLI
L’Angular CLI (Command Line Interface) permet de :
- créer un projet
- générer des composants/services
- lancer un serveur de dev
- builder pour la production

Installation globale :
```bash
npm install -g @angular/cli
```

Vérifier :
```bash
ng version
```

## 2.3 Créer un projet
```bash
ng new intro-angular
cd intro-angular
ng serve
```

Ouvrir : `http://localhost:4200`

## 2.4 Structure d’un projet Angular (vue d’ensemble)
Dossiers importants :
- `src/` : code applicatif
- `src/main.ts` : bootstrap de l’app
- `src/app/` : composants/services/modules
- `angular.json` : configuration CLI
- `package.json` : dépendances et scripts

---

# 3. Architecture Angular : la logique par composants

## 3.1 Le composant : unité de base
Un **composant** combine :
- une **classe TypeScript** (logique)
- un **template HTML** (vue)
- des **styles** (CSS/SCSS)

Créer un composant :
```bash
ng generate component components/hello
```

### Exemple de composant
`hello.component.ts`
```ts
import { Component } from '@angular/core';

@Component({
  selector: 'app-hello',
  templateUrl: './hello.component.html',
  styleUrls: ['./hello.component.css']
})
export class HelloComponent {
  name = 'Angular';
}
```

`hello.component.html`
```html
<h2>Bonjour {{ name }} !</h2>
```

## 3.2 Utiliser un composant
Dans un parent :
```html
<app-hello></app-hello>
```

## 3.3 Cycle de vie (aperçu)
Angular expose des hooks :
- `ngOnInit()` : initialisation
- `ngOnChanges()` : changements d’inputs
- `ngOnDestroy()` : nettoyage

Ex :
```ts
import { Component, OnInit } from '@angular/core';

export class HelloComponent implements OnInit {
  ngOnInit(): void {
    // appelé une fois après construction
  }
}
```

---

# 4. Templates et Data Binding

Angular offre plusieurs façons de lier les données.

## 4.1 Interpolation `{{ }}`
Permet d’afficher une valeur dans le template.
```html
<p>Nom : {{ name }}</p>
```

## 4.2 Property binding `[prop]`
Lie une propriété DOM à une expression.
```html
<img [src]="avatarUrl" />
<button [disabled]="isLoading">Valider</button>
```

## 4.3 Event binding `(event)`
Réagit à un événement utilisateur.
```html
<button (click)="increment()">+</button>
```

`component.ts`
```ts
count = 0;

increment() {
  this.count++;
}
```

## 4.4 Two-way binding `[(ngModel)]`
Synchronisation bidirectionnelle (souvent pour les formulaires).

> Nécessite `FormsModule`.

Template :
```html
<input [(ngModel)]="name" />
<p>Vous avez saisi : {{ name }}</p>
```

---

# 5. Directives et pipes

## 5.1 Directives structurelles
Elles modifient la structure du DOM.

### `*ngIf`
```html
<p *ngIf="isLoggedIn">Bienvenue !</p>
```

### `*ngFor`
```html
<ul>
  <li *ngFor="let item of items">{{ item }}</li>
</ul>
```

### `*ngSwitch`
```html
<div [ngSwitch]="status">
  <p *ngSwitchCase="'loading'">Chargement...</p>
  <p *ngSwitchCase="'done'">Terminé</p>
  <p *ngSwitchDefault>Inconnu</p>
</div>
```

## 5.2 Directives d’attributs
Elles modifient l’apparence/comportement sans changer la structure.

### `ngClass`
```html
<p [ngClass]="{ active: isActive, error: hasError }">Texte</p>
```

### `ngStyle`
```html
<p [ngStyle]="{ color: color, 'font-weight': 'bold' }">Stylé</p>
```

## 5.3 Pipes (introduction)
Les pipes transforment l’affichage d’une valeur.

Exemples natifs :
```html
<p>{{ today | date:'dd/MM/yyyy' }}</p>
<p>{{ price | currency:'EUR' }}</p>
<p>{{ name | uppercase }}</p>
```

---

# 6. Services et Injection de Dépendances (DI)

## 6.1 Pourquoi des services ?
Les services servent à :
- factoriser la logique métier
- partager des données entre composants
- gérer des appels HTTP
- isoler les responsabilités (clean architecture)

## 6.2 Créer un service
```bash
ng generate service services/user
```

`user.service.ts`
```ts
import { Injectable } from '@angular/core';

@Injectable({
  providedIn: 'root'
})
export class UserService {
  private users = ['Alice', 'Bob', 'Chloé'];

  getUsers(): string[] {
    return this.users;
  }
}
```

## 6.3 Injecter un service dans un composant
```ts
import { Component } from '@angular/core';
import { UserService } from '../services/user.service';

@Component({
  selector: 'app-users',
  template: `
    <h3>Utilisateurs</h3>
    <ul>
      <li *ngFor="let u of users">{{ u }}</li>
    </ul>
  `
})
export class UsersComponent {
  users: string[] = [];

  constructor(private userService: UserService) {
    this.users = this.userService.getUsers();
  }
}
```

## 6.4 Comprendre l’injection de dépendances
Angular maintient un **injector** qui fournit les instances.

- `providedIn: 'root'` : service singleton au niveau application
- Possibilité de fournir un service au niveau module ou composant

Bonnes pratiques :
- Injecter par le **constructeur**
- Ne pas mettre de logique lourde dans le constructeur (préférer `ngOnInit`)

---

# 7. Modules (NgModules) et organisation

## 7.1 Pourquoi des modules ?
Les modules aident à :
- organiser l’application en domaines
- déclarer les composants/directives/pipes
- importer des dépendances (FormsModule, HttpClientModule)

## 7.2 Module racine
Le module racine est souvent `AppModule` (selon configuration).

Exemple typique :
```ts
import { NgModule } from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';
import { FormsModule } from '@angular/forms';

import { AppComponent } from './app.component';

@NgModule({
  declarations: [AppComponent],
  imports: [BrowserModule, FormsModule],
  bootstrap: [AppComponent]
})
export class AppModule {}
```

## 7.3 Modules de fonctionnalité (feature modules)
On peut créer des modules par domaine : `UsersModule`, `AdminModule`, etc.

Objectif : améliorer la lisibilité, faciliter le lazy loading.

---

# 8. Routing (Introduction)

## 8.1 Mettre en place le routing
Créer un projet avec routing :
```bash
ng new intro-angular --routing
```

Ou ajouter `AppRoutingModule` puis configurer :

`app-routing.module.ts`
```ts
import { NgModule } from '@angular/core';
import { RouterModule, Routes } from '@angular/router';
import { HomeComponent } from './pages/home/home.component';
import { UsersComponent } from './pages/users/users.component';

const routes: Routes = [
  { path: '', component: HomeComponent },
  { path: 'users', component: UsersComponent },
  { path: '**', redirectTo: '' }
];

@NgModule({
  imports: [RouterModule.forRoot(routes)],
  exports: [RouterModule]
})
export class AppRoutingModule {}
```

## 8.2 Afficher les pages : `<router-outlet>`
Dans `app.component.html` :
```html
<nav>
  <a routerLink="/">Accueil</a>
  <a routerLink="/users">Utilisateurs</a>
</nav>

<router-outlet></router-outlet>
```

---

# 9. HTTP & Observables (Introduction)

## 9.1 HttpClient
Pour faire des requêtes HTTP, on utilise `HttpClient`.

Dans `AppModule` (ou module équivalent) :
```ts
import { HttpClientModule } from '@angular/common/http';

imports: [HttpClientModule]
```

Service API :
```ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';

export interface Post {
  id: number;
  title: string;
}

@Injectable({ providedIn: 'root' })
export class ApiService {
  constructor(private http: HttpClient) {}

  getPosts(): Observable<Post[]> {
    return this.http.get<Post[]>('https://jsonplaceholder.typicode.com/posts');
  }
}
```

## 9.2 Observable : idée générale
Un `Observable` représente un flux de données asynchrones.

Dans un composant :
```ts
posts: Post[] = [];

constructor(private api: ApiService) {}

ngOnInit() {
  this.api.getPosts().subscribe({
    next: (data) => (this.posts = data.slice(0, 5)),
    error: (err) => console.error('Erreur HTTP', err)
  });
}
```

> Pour aller plus loin : `async pipe`, `map`, `switchMap`, gestion d’annulation.

---

# 10. Mini-projet fil rouge (guidé)

Objectif : bâtir une mini app “Catalogue” avec :
- une page liste
- une page détail
- un service fournisseur de données

## 10.1 Étape 1 — Génération des pages
```bash
ng g c pages/catalog
ng g c pages/product-detail
ng g s services/catalog
```

## 10.2 Étape 2 — Modèle de données
`models/product.ts`
```ts
export interface Product {
  id: number;
  name: string;
  price: number;
}
```

## 10.3 Étape 3 — Service de catalogue (données simulées)
`catalog.service.ts`
```ts
import { Injectable } from '@angular/core';
import { Product } from '../models/product';

@Injectable({ providedIn: 'root' })
export class CatalogService {
  private products: Product[] = [
    { id: 1, name: 'Clavier', price: 49.9 },
    { id: 2, name: 'Souris', price: 19.9 },
    { id: 3, name: 'Écran', price: 199.9 }
  ];

  list(): Product[] {
    return this.products;
  }

  getById(id: number): Product | undefined {
    return this.products.find(p => p.id === id);
  }
}
```

## 10.4 Étape 4 — Routing
`app-routing.module.ts`
```ts
const routes: Routes = [
  { path: '', redirectTo: 'catalog', pathMatch: 'full' },
  { path: 'catalog', component: CatalogComponent },
  { path: 'products/:id', component: ProductDetailComponent },
  { path: '**', redirectTo: 'catalog' }
];
```

## 10.5 Étape 5 — Page liste
`catalog.component.ts`
```ts
import { Component } from '@angular/core';
import { CatalogService } from '../../services/catalog.service';
import { Product } from '../../models/product';

@Component({
  selector: 'app-catalog',
  templateUrl: './catalog.component.html'
})
export class CatalogComponent {
  products: Product[];

  constructor(private catalog: CatalogService) {
    this.products = this.catalog.list();
  }
}
```

`catalog.component.html`
```html
<h2>Catalogue</h2>
<ul>
  <li *ngFor="let p of products">
    <a [routerLink]="['/products', p.id]">
      {{ p.name }} – {{ p.price | currency:'EUR' }}
    </a>
  </li>
</ul>
```

## 10.6 Étape 6 — Page détail
`product-detail.component.ts`
```ts
import { Component } from '@angular/core';
import { ActivatedRoute } from '@angular/router';
import { CatalogService } from '../../services/catalog.service';
import { Product } from '../../models/product';

@Component({
  selector: 'app-product-detail',
  templateUrl: './product-detail.component.html'
})
export class ProductDetailComponent {
  product?: Product;

  constructor(
    private route: ActivatedRoute,
    private catalog: CatalogService
  ) {
    const id = Number(this.route.snapshot.paramMap.get('id'));
    this.product = this.catalog.getById(id);
  }
}
```

`product-detail.component.html`
```html
<h2>Détail produit</h2>

<ng-container *ngIf="product; else notFound">
  <p><strong>Nom :</strong> {{ product.name }}</p>
  <p><strong>Prix :</strong> {{ product.price | currency:'EUR' }}</p>
  <a routerLink="/catalog">← Retour</a>
</ng-container>

<ng-template #notFound>
  <p>Produit introuvable.</p>
  <a routerLink="/catalog">← Retour</a>
</ng-template>
```

### Points pédagogiques à discuter
- Découpage par pages et service
- Gestion des paramètres de route
- Utilisation de pipes
- Responsabilités : component (UI) vs service (données)

---

# 11. Synthèse & suite du parcours

## 11.1 Récapitulatif
Vous savez maintenant :
- créer et lancer un projet Angular avec la CLI
- comprendre la structure d’une app Angular
- créer des composants et utiliser le data binding
- utiliser directives et pipes
- créer un service et l’injecter (DI)
- configurer un routing simple
- faire un appel HTTP (introduction) et manipuler des Observables

## 11.2 Bonnes pratiques (premiers réflexes)
- Garder les composants simples et orientés affichage
- Mettre la logique métier et la communication API dans des services
- Créer des modèles TypeScript (`interface`, `type`)
- Structurer par fonctionnalités (feature-first)

## 11.3 Pour aller plus loin
- Forms avancés : Reactive Forms
- RxJS : opérateurs, `async` pipe, `Subject`, `BehaviorSubject`
- Optimisation : change detection, lazy loading
- Angular Material
- Tests : Jasmine/Karma, Jest, Testing Library

---

## Annexes – Cheatsheet

### Commandes CLI utiles
```bash
ng new my-app
ng serve
ng build
ng test
ng g c components/name
ng g s services/name
ng g m features/users --routing
```

### Raccourcis template
- Interpolation : `{{ value }}`
- Property binding : `[disabled]="expr"`
- Event binding : `(click)="handler()"`
- Two-way : `[(ngModel)]="field"`

---

**Fin de la formation**
