# Concepts Angular — Fiche de révision

## Table des matières

- [1. Angular, c’est quoi ?](#1-angular-cest-quoi)
- [2. Vue globale d’une application Angular](#2-vue-globale-dune-application-angular)
- [3. Component](#3-component)
- [4. Template](#4-template)
- [5. Data Binding](#5-data-binding)
- [6. Directive](#6-directive)
- [7. Pipe](#7-pipe)
- [8. Service](#8-service)
- [9. Dependency Injection](#9-dependency-injection)
- [10. Module Angular / NgModule](#10-module-angular-ngmodule)
- [11. Standalone Component](#11-standalone-component)
- [12. Routing](#12-routing)
- [13. Lazy Loading](#13-lazy-loading)
- [14. HTTP Client](#14-http-client)
- [15. Observable / RxJS](#15-observable-rxjs)
- [16. Signal](#16-signal)
- [17. Forms](#17-forms)
- [18. Guards](#18-guards)
- [19. Interceptor](#19-interceptor)
- [20. Lifecycle Hooks](#20-lifecycle-hooks)
- [21. Input / Output](#21-input-output)
- [22. State Management](#22-state-management)
- [23. Architecture Angular classique](#23-architecture-angular-classique)
- [24. Résumé des concepts](#24-resume-des-concepts)
- [25. Exemple complet simplifié](#25-exemple-complet-simplifie)
- [26. À retenir pour un entretien technique](#26-a-retenir-pour-un-entretien-technique)
- [27. Questions fréquentes en entretien](#27-questions-frequentes-en-entretien)

---

## 1. Angular, c’est quoi ?

**Angular** est un framework frontend basé sur **TypeScript** permettant de créer des applications web structurées, dynamiques et maintenables.

Il est souvent utilisé pour créer des **SPA** : *Single Page Applications*.

Angular permet de gérer :

- les interfaces utilisateur ;
- les composants ;
- les formulaires ;
- la navigation ;
- les appels API ;
- la sécurité côté frontend ;
- la réactivité ;
- l’organisation d’une application frontend complète.

---

## 2. Vue globale d’une application Angular

```text
Application Angular
│
├── Components
│   ├── Template HTML
│   ├── Classe TypeScript
│   └── CSS / SCSS
│
├── Services
│   ├── Logique métier frontend
│   ├── Appels API
│   └── Partage de données
│
├── Router
│   ├── Routes
│   ├── Pages
│   └── Lazy loading
│
├── Forms
│   ├── Template-driven forms
│   └── Reactive forms
│
├── HTTP Client
│   └── Communication backend
│
├── Dependency Injection
│   └── Injection de services
│
└── State Management
    ├── Signals
    ├── RxJS
    └── Store éventuel
```

---

## 3. Component

Le **component** est la brique principale d’Angular.

Il représente une partie de l’interface utilisateur.

Exemples :

```text
HeaderComponent
LoginComponent
UserListComponent
ProductCardComponent
DashboardComponent
```

Un composant contient généralement :

- une classe TypeScript ;
- un template HTML ;
- un fichier CSS ou SCSS ;
- un sélecteur permettant de l’utiliser dans une page.

Exemple :

```ts
@Component({
  selector: 'app-user',
  templateUrl: './user.component.html',
  styleUrl: './user.component.css'
})
export class UserComponent {
  name = 'Hermann';
}
```

Template associé :

```html
<h1>{{ name }}</h1>
```

### Rôle du component

Un composant sert à :

- afficher des données ;
- gérer les actions utilisateur ;
- appeler des services ;
- organiser l’écran en blocs réutilisables.

---

## 4. Template

Le **template** est la partie HTML du composant.

Il décrit ce que l’utilisateur voit à l’écran.

Exemple :

```html
<h1>{{ title }}</h1>

<button (click)="save()">
  Enregistrer
</button>
```

Dans un template Angular, on utilise notamment :

```text
{{ }}          interpolation
[property]    property binding
(event)       event binding
[(ngModel)]   two-way binding
@if           condition
@for          boucle
```

---

## 5. Data Binding

Le **data binding** permet de connecter la classe TypeScript au template HTML.

### Interpolation

```html
<p>{{ userName }}</p>
```

Permet d’afficher une valeur.

### Property Binding

```html
<img [src]="imageUrl">
```

Permet de lier une propriété HTML à une variable TypeScript.

### Event Binding

```html
<button (click)="onClick()">Clique</button>
```

Permet de réagir à une action utilisateur.

### Two-way Binding

```html
<input [(ngModel)]="email">
```

Permet une synchronisation dans les deux sens :

```text
TypeScript → HTML
HTML → TypeScript
```

---

## 6. Directive

Une **directive** modifie le comportement ou l’apparence d’un élément HTML.

Il existe deux grandes catégories :

- les directives structurelles ;
- les directives d’attribut.

### Directives structurelles

Elles modifient la structure du DOM.

Ancienne syntaxe :

```html
<div *ngIf="isVisible">Visible</div>
```

Nouvelle syntaxe Angular moderne :

```html
@if (isVisible) {
  <div>Visible</div>
}
```

Boucle :

```html
@for (user of users; track user.id) {
  <p>{{ user.name }}</p>
}
```

### Directives d’attribut

Elles modifient l’apparence ou le comportement d’un élément.

Exemple :

```html
<p [ngClass]="{ active: isActive }">
  Texte
</p>
```

---

## 7. Pipe

Un **pipe** transforme une valeur dans le template.

Exemples :

```html
{{ today | date }}
{{ price | currency:'EUR' }}
{{ name | uppercase }}
```

Exemple :

```html
<p>{{ user.name | uppercase }}</p>
```

Si `user.name = "hermann"`, l’affichage sera :

```text
HERMANN
```

On peut aussi créer ses propres pipes.

---

## 8. Service

Un **service** contient du code réutilisable, souvent indépendant de l’affichage.

Il sert à :

- appeler une API ;
- partager des données ;
- gérer une logique métier frontend ;
- centraliser du code commun.

Exemple :

```ts
@Injectable({
  providedIn: 'root'
})
export class UserService {
  getUsers() {
    return ['Alice', 'Bob'];
  }
}
```

---

## 9. Dependency Injection

La **Dependency Injection**, ou **injection de dépendances**, permet à Angular de fournir automatiquement les objets dont une classe a besoin.

Au lieu de faire :

```ts
const service = new UserService();
```

On fait :

```ts
private userService = inject(UserService);
```

Ou avec le constructeur :

```ts
constructor(private userService: UserService) {}
```

### Avantages

L’injection de dépendances permet de :

- réduire le couplage ;
- faciliter les tests ;
- centraliser la création des objets ;
- remplacer facilement une dépendance par une autre.

---

## 10. Module Angular / NgModule

Historiquement, Angular utilisait les **NgModules** pour regrouper :

- composants ;
- directives ;
- pipes ;
- services ;
- routes.

Exemple :

```ts
@NgModule({
  declarations: [AppComponent],
  imports: [BrowserModule],
  providers: [],
  bootstrap: [AppComponent]
})
export class AppModule {}
```

Aujourd’hui, Angular favorise les **standalone components**.

---

## 11. Standalone Component

Un **standalone component** est un composant qui peut fonctionner sans être déclaré dans un `NgModule`.

Exemple :

```ts
@Component({
  selector: 'app-home',
  standalone: true,
  imports: [],
  templateUrl: './home.component.html'
})
export class HomeComponent {}
```

### Avantages

```text
Moins de modules
Moins de configuration
Plus simple à comprendre
Meilleur lazy loading
Architecture plus directe
```

---

## 12. Routing

Le **routing** permet de naviguer entre plusieurs pages.

Exemples de routes :

```text
/login
/users
/products
/dashboard
```

Configuration :

```ts
export const routes: Routes = [
  { path: 'home', component: HomeComponent },
  { path: 'users', component: UsersComponent },
  { path: '', redirectTo: 'home', pathMatch: 'full' }
];
```

Dans le template principal :

```html
<router-outlet></router-outlet>
```

`router-outlet` est l’endroit où Angular affiche le composant correspondant à la route.

---

## 13. Lazy Loading

Le **lazy loading** consiste à charger une partie de l’application seulement quand l’utilisateur en a besoin.

Exemple :

```ts
{
  path: 'admin',
  loadComponent: () =>
    import('./admin/admin.component')
      .then(m => m.AdminComponent)
}
```

### Avantages

```text
Application plus rapide au démarrage
Chargement à la demande
Meilleure performance
```

---

## 14. HTTP Client

Le **HttpClient** permet d’appeler une API backend.

Exemple :

```ts
@Injectable({
  providedIn: 'root'
})
export class UserApiService {
  private http = inject(HttpClient);

  getUsers() {
    return this.http.get<User[]>('/api/users');
  }
}
```

Utilisation dans un composant :

```ts
users$ = this.userApiService.getUsers();
```

Angular utilise souvent **RxJS Observables** avec `HttpClient`.

---

## 15. Observable / RxJS

Un **Observable** représente un flux de données dans le temps.

Exemples :

```text
Réponse HTTP
Événement utilisateur
WebSocket
Recherche dynamique
Formulaire réactif
```

Exemple :

```ts
this.http.get<User[]>('/api/users')
  .subscribe(users => {
    this.users = users;
  });
```

### Opérateurs courants

```text
map
filter
switchMap
mergeMap
concatMap
debounceTime
catchError
tap
```

Exemple :

```ts
this.searchControl.valueChanges.pipe(
  debounceTime(300),
  switchMap(value => this.api.search(value))
);
```

---

## 16. Signal

Les **signals** sont un mécanisme moderne d’Angular pour gérer l’état réactif.

Exemple :

```ts
count = signal(0);

increment() {
  this.count.update(value => value + 1);
}
```

Template :

```html
<p>{{ count() }}</p>
<button (click)="increment()">+</button>
```

### Idée simple

```text
signal = donnée réactive
computed = donnée calculée
effect = réaction automatique
```

Exemple :

```ts
firstName = signal('Hermann');
lastName = signal('Durand');

fullName = computed(() => 
  `${this.firstName()} ${this.lastName()}`
);
```

---

## 17. Forms

Angular propose deux grandes approches pour les formulaires.

### Template-driven forms

Approche simple, basée sur le HTML.

```html
<input [(ngModel)]="user.name">
```

Adapté aux petits formulaires.

### Reactive forms

Approche plus robuste, basée sur TypeScript.

```ts
form = new FormGroup({
  email: new FormControl(''),
  password: new FormControl('')
});
```

Template :

```html
<form [formGroup]="form">
  <input formControlName="email">
  <input formControlName="password">
</form>
```

Adapté aux formulaires complexes, validations avancées, tests et grosses applications.

---

## 18. Guards

Les **guards** protègent les routes.

Exemple :

```ts
{
  path: 'admin',
  component: AdminComponent,
  canActivate: [authGuard]
}
```

Ils permettent de vérifier :

```text
L’utilisateur est-il connecté ?
A-t-il le bon rôle ?
Peut-il quitter la page ?
La donnée existe-t-elle ?
```

Types fréquents :

```text
CanActivate
CanDeactivate
CanMatch
Resolve
```

---

## 19. Interceptor

Un **interceptor** intercepte les requêtes HTTP.

Il sert à :

- ajouter un token JWT ;
- gérer les erreurs globales ;
- afficher un loader ;
- journaliser les appels HTTP.

Exemple :

```ts
export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const token = localStorage.getItem('token');

  const cloned = req.clone({
    setHeaders: {
      Authorization: `Bearer ${token}`
    }
  });

  return next(cloned);
};
```

---

## 20. Lifecycle Hooks

Les **lifecycle hooks** sont des méthodes appelées à différents moments de la vie d’un composant.

Les plus connues :

```text
ngOnInit
ngOnChanges
ngAfterViewInit
ngOnDestroy
```

Exemple :

```ts
ngOnInit() {
  this.loadUsers();
}
```

### Rôle des hooks

```text
ngOnInit         Initialisation
ngOnChanges      Changement d’Input
ngAfterViewInit  Accès à la vue
ngOnDestroy      Nettoyage mémoire
```

---

## 21. Input / Output

Les composants communiquent entre eux avec `@Input` et `@Output`.

### Parent vers enfant : Input

```ts
@Input() user!: User;
```

Utilisation :

```html
<app-user-card [user]="selectedUser"></app-user-card>
```

### Enfant vers parent : Output

```ts
@Output() deleted = new EventEmitter<number>();
```

Utilisation :

```html
<app-user-card (deleted)="onUserDeleted($event)">
</app-user-card>
```

---

## 22. State Management

Le **state management** correspond à la gestion de l’état de l’application.

Exemples d’état :

```text
Utilisateur connecté
Panier
Filtres de recherche
Liste de produits
Préférences utilisateur
```

Solutions possibles :

```text
Signals
Services Angular
RxJS BehaviorSubject
NgRx
Akita
Elf
```

Pour une petite application, un service avec `signal` ou `BehaviorSubject` suffit souvent.

Pour une grosse application, on peut utiliser NgRx.

---

## 23. Architecture Angular classique

Exemple d’organisation :

```text
src/app
│
├── core
│   ├── services
│   ├── interceptors
│   ├── guards
│   └── models
│
├── shared
│   ├── components
│   ├── pipes
│   └── directives
│
├── features
│   ├── users
│   ├── products
│   └── orders
│
├── app.routes.ts
└── app.config.ts
```

### Rôle des dossiers

```text
core      Services globaux, auth, guards, interceptors
shared    Composants réutilisables, pipes, directives
features  Fonctionnalités métier
```

---

## 24. Résumé des concepts

| Élément | Rôle |
|---|---|
| Component | Bloc d’interface utilisateur |
| Template | HTML du composant |
| Service | Logique réutilisable |
| Directive | Modifie le DOM ou le comportement |
| Pipe | Transforme une donnée affichée |
| Router | Navigation entre pages |
| Guard | Protection des routes |
| Interceptor | Interception des requêtes HTTP |
| HttpClient | Appels API |
| Observable | Flux de données asynchrone |
| Signal | État réactif moderne |
| Form | Gestion des formulaires |
| DI | Injection automatique des dépendances |
| Module | Ancienne organisation Angular |
| Standalone | Organisation moderne sans NgModule |

---

## 25. Exemple complet simplifié

### Service

```ts
@Injectable({
  providedIn: 'root'
})
export class UserService {
  private http = inject(HttpClient);

  getUsers() {
    return this.http.get<User[]>('/api/users');
  }
}
```

### Component

```ts
@Component({
  selector: 'app-users',
  standalone: true,
  templateUrl: './users.component.html'
})
export class UsersComponent implements OnInit {
  private userService = inject(UserService);

  users: User[] = [];

  ngOnInit() {
    this.userService.getUsers()
      .subscribe(data => this.users = data);
  }
}
```

### Template

```html
<h1>Liste des utilisateurs</h1>

@for (user of users; track user.id) {
  <p>{{ user.name }}</p>
}
```

---

## 26. À retenir pour un entretien technique

Angular repose principalement sur cette logique :

```text
Component = affiche et gère l’écran
Template = HTML dynamique
Service = logique réutilisable
DI = fournit les services
Router = navigation
HttpClient = communication API
RxJS / Signals = réactivité
Forms = saisie utilisateur
Guards / Interceptors = sécurité et contrôle
```

Phrase simple à dire en entretien :

> Angular est un framework frontend basé sur des composants. Chaque composant possède une classe TypeScript, un template HTML et éventuellement du style. La logique réutilisable est placée dans des services injectés grâce à la Dependency Injection. Le routing permet de naviguer entre les pages, HttpClient permet de communiquer avec une API backend, et la réactivité est gérée avec RxJS ou les Signals.

---

## 27. Questions fréquentes en entretien

### Quelle est la différence entre un component et un service ?

Un **component** gère l’affichage et les interactions utilisateur.  
Un **service** contient une logique réutilisable, comme les appels API ou le partage de données.

### Quelle est la différence entre Observable et Signal ?

Un **Observable** représente un flux asynchrone souvent utilisé avec RxJS, par exemple pour les appels HTTP ou les événements.  
Un **Signal** représente un état réactif plus simple à utiliser dans les composants Angular modernes.

### À quoi sert un interceptor ?

Un **interceptor** permet d’intercepter les requêtes ou réponses HTTP, par exemple pour ajouter un token JWT ou gérer les erreurs globales.

### À quoi sert un guard ?

Un **guard** permet de contrôler l’accès à une route, par exemple pour empêcher un utilisateur non connecté d’accéder à une page d’administration.

### Quelle est la différence entre template-driven forms et reactive forms ?

Les **template-driven forms** sont simples et pilotés par le HTML.  
Les **reactive forms** sont pilotés par TypeScript, plus adaptés aux formulaires complexes et aux tests.

### Pourquoi utiliser le lazy loading ?

Le **lazy loading** permet de charger certaines parties de l’application uniquement quand elles sont nécessaires, ce qui améliore les performances au démarrage.

