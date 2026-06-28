# Formation Angular — Modules (NgModule)

- **Public** : développeurs front, intégrateurs, formateurs, développeurs Angular (débutant à intermédiaire)
- **Pré-requis** : TypeScript (bases), ES modules, notions de composants/services Angular, CLI Angular
- **Durée indicative** : 1 journée (6–7h) ou 2 demi-journées
- **Version Angular** : principes applicables toutes versions ; remarques spécifiques indiquées si besoin

---

## Objectifs pédagogiques

À l’issue de la formation, vous serez capable de :

1. Expliquer le rôle d’un **NgModule** et son impact sur l’organisation d’une application Angular.
2. Construire des modules **fonctionnels** (Feature Modules) et **partagés** (Shared) de manière cohérente.
3. Maîtriser le trio **declarations / imports / exports** et les règles de visibilité.
4. Configurer l’injection de dépendances via **providers** (racine, module, composant) et comprendre la portée (scope) des services.
5. Mettre en place du **routing modulaire** et du **lazy loading**.
6. Identifier les bonnes pratiques modernes (standalone, migration progressive) tout en sachant maintenir un code basé sur NgModule.

---

## Plan de la formation

1. Introduction : pourquoi des modules ?
2. Anatomie d’un NgModule
3. Règles de compilation et de visibilité (declarations, imports, exports)
4. Modules Angular courants : AppModule, CoreModule, SharedModule, FeatureModule
5. Providers & Injection de dépendances : scope, patterns et pièges
6. Modules et formulaires / HTTP / fonctionnalités framework
7. Routing modulaire & Lazy Loading
8. Architecture et bonnes pratiques (organisation du dossier, conventions)
9. Exercices guidés (fil rouge)
10. Checklist & récapitulatif

---

# 1) Introduction : pourquoi des modules ?

Les modules (**NgModule**) permettent d’organiser une application Angular en **blocs fonctionnels**. Un module regroupe :

- des **composants** (components), **directives** et **pipes**
- des **services** (via `providers`)
- d’autres dépendances importées (modules Angular ou modules maison)

### Pourquoi regrouper ?

- **Lisibilité** : une application découpée par fonctionnalités est plus simple à comprendre.
- **Réutilisabilité** : un module partagé peut être importé dans plusieurs parties.
- **Performance** : via **lazy loading**, on charge seulement ce qui est nécessaire.
- **Encapsulation** : règles de visibilité contrôlées via `exports`.

> Remarque : depuis Angular 14+, les **standalone components** permettent de construire des apps sans NgModule. Néanmoins, NgModule reste central dans de nombreux projets existants, bibliothèques, patterns d’architecture et migrations progressives.

---

# 2) Anatomie d’un NgModule

Un NgModule est une classe TypeScript décorée par `@NgModule`.

```ts
import { NgModule } from '@angular/core';
import { CommonModule } from '@angular/common';

@NgModule({
  declarations: [],
  imports: [],
  exports: [],
  providers: [],
  bootstrap: []
})
export class ExampleModule {}
```

## 2.1 Les métadonnées principales

### `declarations`
Déclare ce que **le module possède** côté template :

- composants
- directives
- pipes

> Une déclaration ne peut appartenir qu’à **un seul NgModule**.

### `imports`
Liste les modules dont ce module a besoin pour :

- utiliser leurs composants/directives/pipes exportés
- accéder à certaines fonctionnalités (ex: formulaires, routes, etc.)

Exemples : `CommonModule`, `FormsModule`, `ReactiveFormsModule`, `RouterModule`.

### `exports`
Expose certaines déclarations (ou modules importés) pour qu’un autre module qui importe celui-ci puisse les utiliser.

> Exporter = rendre visible vers l’extérieur.

### `providers`
Déclare des services fournis par ce module (injection de dépendances). Important pour la **portée** (scope) si on ne déclare pas les services via `providedIn: 'root'`.

### `bootstrap`
Uniquement dans le module de bootstrap (souvent `AppModule`) : liste les composants root à démarrer (`AppComponent`).

---

# 3) Règles de compilation et de visibilité

## 3.1 Règle n°1 : une déclaration appartient à un seul module

Si un composant est déclaré dans `ModuleA`, vous ne pouvez pas le déclarer dans `ModuleB`.

**Symptôme classique** :

- *“Type X is part of the declarations of 2 modules…”*

**Solution** :

- déclarer le composant dans un module (souvent `SharedModule` ou un module feature) puis **l’exporter**.

## 3.2 Règle n°2 : pour utiliser un composant/directive/pipe, il faut l’avoir dans le scope

Vous pouvez utiliser une directive/pipes/composant dans :

- le module où il est **déclaré**
- un module qui **importe** un module qui l’**exporte**

### Exemple

```ts
@NgModule({
  declarations: [BadgeComponent],
  exports: [BadgeComponent]
})
export class UiModule {}

@NgModule({
  imports: [UiModule]
})
export class OrdersModule {}
```

Dans les templates d’`OrdersModule`, vous pouvez utiliser `<app-badge>`.

## 3.3 `CommonModule` vs `BrowserModule`

- `BrowserModule` : à importer **une seule fois** dans le module de démarrage (souvent `AppModule`).
- `CommonModule` : à importer dans les modules **feature** ou **shared** pour avoir accès à :

  - `*ngIf`, `*ngFor`, `ngClass`, `ngStyle`, etc.

> Erreur fréquente : importer `BrowserModule` dans un module feature → à éviter.

---

# 4) Les grands types de modules

## 4.1 AppModule (module racine)

Rôle : point d’entrée de l’application.

Exemple simplifié :

```ts
import { NgModule } from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';

import { AppComponent } from './app.component';
import { AppRoutingModule } from './app-routing.module';

@NgModule({
  declarations: [AppComponent],
  imports: [BrowserModule, AppRoutingModule],
  bootstrap: [AppComponent]
})
export class AppModule {}
```

## 4.2 CoreModule

Contient les **singletons** et éléments à instancier une fois :

- services « applicatifs » (auth, config, logger)
- interceptors HTTP
- guards globaux
- composants de shell (layout), éventuellement

**But** : éviter les duplications et centraliser les dépendances globales.

Pattern courant :

- `CoreModule` importé **uniquement** dans `AppModule`.
- `SharedModule` importé partout.

### Protéger CoreModule contre les imports multiples

```ts
import { NgModule, Optional, SkipSelf } from '@angular/core';

@NgModule({
  providers: [/* services singletons */]
})
export class CoreModule {
  constructor(@Optional() @SkipSelf() parentModule: CoreModule) {
    if (parentModule) {
      throw new Error('CoreModule is already loaded. Import it in the AppModule only');
    }
  }
}
```

## 4.3 SharedModule

Contient ce qui est **réutilisable** et **sans état global** :

- composants UI génériques (boutons, badges)
- directives utilitaires
- pipes
- modules tiers UI (Angular Material, etc.)

### Exemple

```ts
@NgModule({
  declarations: [CapitalizePipe, BadgeComponent],
  imports: [CommonModule],
  exports: [CommonModule, CapitalizePipe, BadgeComponent]
})
export class SharedModule {}
```

> Bonne pratique : ré-exporter `CommonModule` depuis `SharedModule` pour éviter de l’importer partout.

## 4.4 Feature Modules (modules fonctionnels)

Chaque fonctionnalité (ex: `Orders`, `Catalog`, `Admin`) peut avoir son module.

Caractéristiques :

- contient les pages (components) liées au domaine
- peut gérer son routing local
- idéalement **lazy-loadable**

Structure typique :

```
orders/
  pages/
  components/
  services/
  orders-routing.module.ts
  orders.module.ts
```

---

# 5) Providers & Injection de dépendances : scope, patterns et pièges

## 5.1 `providedIn: 'root'` (recommandé)

La plupart des services devraient être fournis au niveau racine :

```ts
import { Injectable } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class OrdersApi {
  // singleton app
}
```

Avantages :

- singleton global
- tree-shakeable (si non utilisé, peut être éliminé)
- configuration plus simple

## 5.2 Fournir un service dans un module

```ts
@NgModule({
  providers: [OrdersFacade]
})
export class OrdersModule {}
```

Conséquence :

- si le module est **lazy-loaded**, vous obtenez une instance **par lazy module**.
- si le module est importé plusieurs fois (ce qu’on évite), vous risquez plusieurs instances.

## 5.3 Fournir au niveau composant

```ts
@Component({
  selector: 'app-orders',
  templateUrl: './orders.component.html',
  providers: [OrdersVM]
})
export class OrdersComponent {}
```

Conséquence : nouvelle instance par composant (et sous-arbre).

## 5.4 Configuration via `forRoot()` / `forChild()`

Pattern utilisé par des libs et parfois en interne :

- `forRoot()` : configuration + providers singletons
- `forChild()` : sans providers singletons (ou providers spécifiques)

Exemple pédagogique :

```ts
export class LoggerModule {
  static forRoot(config: LoggerConfig): ModuleWithProviders<LoggerModule> {
    return {
      ngModule: LoggerModule,
      providers: [
        { provide: LOGGER_CONFIG, useValue: config },
        LoggerService
      ]
    };
  }
}
```

---

# 6) Modules et fonctionnalités framework

## 6.1 HTTP : `HttpClientModule`

À importer une seule fois (souvent `AppModule` ou `CoreModule`).

```ts
import { HttpClientModule } from '@angular/common/http';

@NgModule({
  imports: [HttpClientModule]
})
export class CoreModule {}
```

### Interceptors

```ts
import { HTTP_INTERCEPTORS } from '@angular/common/http';

providers: [
  {
    provide: HTTP_INTERCEPTORS,
    useClass: AuthInterceptor,
    multi: true
  }
]
```

## 6.2 Formulaires

- `FormsModule` : template-driven
- `ReactiveFormsModule` : reactive

Dans un feature module qui contient des formulaires, importer le(s) module(s) nécessaires.

---

# 7) Routing modulaire & Lazy Loading

## 7.1 Routing : module dédié

Deux fichiers fréquents :

- `orders.module.ts`
- `orders-routing.module.ts`

### Exemple : OrdersRoutingModule

```ts
import { NgModule } from '@angular/core';
import { RouterModule, Routes } from '@angular/router';
import { OrdersPageComponent } from './pages/orders-page/orders-page.component';

const routes: Routes = [
  { path: '', component: OrdersPageComponent },
];

@NgModule({
  imports: [RouterModule.forChild(routes)],
  exports: [RouterModule]
})
export class OrdersRoutingModule {}
```

### Feature module

```ts
@NgModule({
  declarations: [OrdersPageComponent],
  imports: [CommonModule, OrdersRoutingModule]
})
export class OrdersModule {}
```

## 7.2 Lazy loading (chargement à la demande)

Dans `AppRoutingModule` :

```ts
const routes: Routes = [
  {
    path: 'orders',
    loadChildren: () =>
      import('./orders/orders.module').then(m => m.OrdersModule)
  }
];
```

Bénéfices :

- bundle initial plus léger
- meilleures perfs au démarrage

Points d’attention :

- les providers déclarés dans le module lazy auront une **instance isolée**
- bien séparer `CoreModule` (global) et `FeatureModule` (local)

---

# 8) Architecture et bonnes pratiques

## 8.1 Conventions de nommage

- `OrdersModule`, `OrdersRoutingModule`, `SharedModule`, `CoreModule`
- composants : `OrdersPageComponent`, `OrderDetailComponent`

## 8.2 Organisation des imports

- Importer `CommonModule` dans tous les Feature Modules (ou via `SharedModule`).
- Éviter `BrowserModule` hors du module racine.
- Ré-exporter les modules UI depuis `SharedModule` (pratique), mais surveiller la taille (perfs).

## 8.3 Ce qu’on met (ou pas) dans Shared

Mettre :

- composants « bêtes » (présentation), pipes, directives

Éviter :

- services qui gèrent un état global (plutôt dans Core ou `providedIn: 'root'`)

## 8.4 Anti-patterns fréquents

1. **God SharedModule** : un Shared trop lourd importé partout ⇒ surcharge.
2. **Providers de singletons** dans un module importé plusieurs fois ⇒ multiples instances.
3. **Déclarations dupliquées** (même composant déclaré dans 2 modules).
4. **Feature module qui importe CoreModule** ⇒ duplication et incohérences.

---

# 9) Exercices guidés (fil rouge)

## Exercice 1 — Créer un SharedModule

Objectif : créer un module `SharedModule` ré-exportant `CommonModule` et déclarant un pipe.

1. Créer `capitalize.pipe.ts`.
2. Déclarer et exporter dans `SharedModule`.
3. Importer `SharedModule` dans `OrdersModule`.

Validation : utiliser `{{ 'angular' | capitalize }}` dans une page Orders.

## Exercice 2 — Créer un Feature Module Orders lazy-loadé

Objectif : isoler la fonctionnalité Orders.

1. Créer `OrdersModule` et `OrdersRoutingModule`.
2. Déclarer `OrdersPageComponent`.
3. Configurer `loadChildren` dans `AppRoutingModule`.

Validation : naviguer vers `/orders`.

## Exercice 3 — Providers : scope module lazy

Objectif : comprendre l’isolation des providers.

1. Créer `OrdersCounterService` avec un compteur interne.
2. Le fournir dans `OrdersModule` via `providers`.
3. Ajouter deux composants dans Orders qui l’injectent.

Questions :

- Le compteur est-il partagé entre composants ? (Oui, dans le même injector du module)
- Que se passe-t-il si le module est chargé/ déchargé (selon stratégie) ?

## Exercice 4 — CoreModule et singleton

Objectif : centraliser un `LoggerService` singleton.

1. Créer `CoreModule`.
2. Mettre `LoggerService` en `providedIn: 'root'` ou dans `CoreModule.providers`.
3. Protéger `CoreModule` contre les imports multiples.

---

# 10) Checklist & récapitulatif

## Checklist module

- [ ] Ai-je importé `CommonModule` (ou `SharedModule`) pour utiliser les directives/pipes Angular ?
- [ ] Mes composants/directives/pipes sont-ils **déclarés une seule fois** ?
- [ ] Ai-je exporté ce qui doit être réutilisé ailleurs ?
- [ ] Les services sont-ils en `providedIn: 'root'` (si singleton global) ?
- [ ] Mon `CoreModule` n’est-il importé qu’une seule fois ?
- [ ] Mes feature modules sont-ils lazy-loadables si pertinent ?

## À retenir

- **NgModule** = unité d’organisation et de compilation, contrôle la visibilité.
- **declarations** : ce que le module possède (components/directives/pipes).
- **imports** : ce dont le module a besoin.
- **exports** : ce que le module rend disponible.
- **providers** : scope des services (attention avec lazy loading).
- **Core / Shared / Feature** : découpage architectural classique.

---

## Annexes — Mini glossaire

- **Scope / Injector** : contexte d’injection de dépendances.
- **Lazy module** : module chargé à la demande via router.
- **Tree-shaking** : élimination de code non utilisé à la compilation.

