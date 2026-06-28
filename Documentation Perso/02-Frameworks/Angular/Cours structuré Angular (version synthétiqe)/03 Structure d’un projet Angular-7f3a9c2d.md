# Formation (03) — Structure d’un projet Angular

> Public : développeurs et/ou apprenants Angular (débutant → intermédiaire)
>
> Objectif : comprendre **où se trouve quoi** dans un projet Angular, **comment l’application démarre**, et **comment organiser / maintenir** une base de code.

---

## 1. Objectifs pédagogiques

À l’issue de cette formation, vous saurez :

- Expliquer l’arborescence standard d’un projet Angular.
- Identifier le rôle des dossiers clés : **src**, **app**, **assets**, **environments**.
- Décrire le chemin d’exécution au démarrage : **main.ts → AppModule → AppComponent**.
- Différencier les fichiers de configuration (workspace) des fichiers applicatifs.
- Appliquer quelques bonnes pratiques d’organisation (features, core, shared).

---

## 2. Pré-requis

- Connaissances de base en TypeScript.
- Connaissances minimales d’Angular (components, modules).
- Node.js + npm/yarn installés.

---

## 3. Vocabulaire essentiel

- **Workspace Angular** : le dossier racine qui contient le code + configurations (Angular CLI).
- **Build** : compilation + bundling pour produire des fichiers déployables.
- **Environment** : variables / paramètres selon le contexte (dev, prod, etc.).
- **Bootstrap** : étape où Angular instancie le module racine et affiche le premier composant.

---

## 4. Vue d’ensemble d’un projet Angular

Un projet Angular, généré typiquement via :

```bash
ng new mon-projet
```

…produit une structure proche de :

```text
mon-projet/
├─ src/
│  ├─ app/
│  ├─ assets/
│  ├─ environments/
│  ├─ index.html
│  ├─ main.ts
│  ├─ styles.css (ou .scss)
│  └─ ...
├─ angular.json
├─ package.json
├─ tsconfig.json
└─ ...
```

> Note : selon la version d’Angular, vous pouvez voir des variations (ex. `app.config.ts` / `main.ts` avec `bootstrapApplication` pour des projets "standalone"). Dans ce cours, on se concentre sur la structure classique mettant en avant **AppModule**.

---

## 5. Le dossier `src/` : le cœur de l’application

### 5.1 Rôle de `src/`

Le dossier **`src/`** contient l’essentiel de ce qui sera **compilé** et **servi** par l’application :

- le code Angular (components, services, modules)
- le point d’entrée de l’application (`main.ts`)
- la page hôte (`index.html`)
- les styles globaux
- les ressources statiques (via `assets/`)
- les paramètres d’environnements (via `environments/`)

### 5.2 Fichiers fréquents dans `src/`

- **`index.html`** : la page qui héberge votre SPA (Single Page Application).
  - On y trouve généralement une balise racine comme :

    ```html
    <app-root></app-root>
    ```

- **`styles.(css|scss)`** : styles globaux.
- **`main.ts`** : point d’entrée TypeScript — c’est ici qu’Angular démarre.

---

## 6. Le dossier `src/app/` : le code applicatif Angular

### 6.1 Rôle de `app/`

Le dossier **`src/app/`** contient la logique **Angular** de votre application :

- composants (UI)
- services (logique métier, accès API)
- modules (regroupements fonctionnels)
- routing (navigation)
- guards/interceptors/pipes/directives

Arborescence typique (exemple) :

```text
src/app/
├─ app.component.ts
├─ app.component.html
├─ app.component.css
├─ app.module.ts
├─ app-routing.module.ts
└─ ...
```

### 6.2 Les éléments clés

#### `AppComponent`

- C’est le **composant racine**.
- Son sélecteur (souvent `app-root`) se retrouve dans `index.html`.
- Il sert généralement de conteneur principal : layout, header, router-outlet.

Exemple simplifié :

```ts
import { Component } from '@angular/core';

@Component({
  selector: 'app-root',
  template: `
    <h1>Mon application</h1>
    <router-outlet></router-outlet>
  `
})
export class AppComponent {}
```

#### `AppModule` (module principal)

- C’est le **module racine** dans une architecture Angular "classique".
- Il déclare les composants de base, importe d’autres modules, et définit le composant de bootstrap.

Exemple simplifié :

```ts
import { NgModule } from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';

import { AppComponent } from './app.component';

@NgModule({
  declarations: [AppComponent],
  imports: [BrowserModule],
  providers: [],
  bootstrap: [AppComponent]
})
export class AppModule {}
```

> Le **module principal** mentionné dans l’énoncé est donc **`AppModule`**.

---

## 7. Le dossier `src/assets/` : ressources statiques

### 7.1 Rôle de `assets/`

Le dossier **`src/assets/`** contient des fichiers statiques copiés tels quels lors du build :

- images (`.png`, `.svg`, `.jpg`)
- fichiers JSON statiques
- polices (`.woff`, `.ttf`)
- documents (PDF)

Exemple :

```text
src/assets/
├─ logo.svg
├─ i18n/
│  ├─ fr.json
│  └─ en.json
└─ data/
   └─ demo.json
```

### 7.2 Accéder à une ressource depuis le code

Dans un template :

```html
<img src="assets/logo.svg" alt="Logo" />
```

Ou via HTTP (si vous chargez un JSON statique) :

```ts
this.http.get('assets/data/demo.json');
```

> Astuce : `assets/` est idéal pour des éléments versionnés et distribués avec l’application.

---

## 8. Le dossier `src/environments/` : configuration par environnement

### 8.1 Rôle de `environments/`

Le dossier **`src/environments/`** sert à stocker des fichiers de configuration **différenciés** selon le contexte :

- `environment.ts` (souvent pour le dev)
- `environment.prod.ts` (pour la production)

Exemple typique :

`src/environments/environment.ts`

```ts
export const environment = {
  production: false,
  apiUrl: 'http://localhost:3000'
};
```

`src/environments/environment.prod.ts`

```ts
export const environment = {
  production: true,
  apiUrl: 'https://api.monsite.com'
};
```

### 8.2 Utilisation côté code

Dans un service :

```ts
import { environment } from '../environments/environment';

console.log(environment.apiUrl);
```

> Le remplacement entre `environment.ts` et `environment.prod.ts` est généralement orchestré par Angular CLI (via la configuration de build).

---

## 9. Le point d’entrée `main.ts` : démarrage de l’application

### 9.1 Pourquoi `main.ts` est central

Le fichier **`src/main.ts`** est le **point d’entrée** de l’application Angular :

- il initialise l’environnement Angular
- il démarre le framework
- il lance le **bootstrap** du module racine (`AppModule`)

### 9.2 Chaîne de démarrage : `main.ts → AppModule → AppComponent`

Le flux est généralement :

1. Le navigateur charge `index.html`.
2. Les scripts générés par Angular (bundles) sont chargés.
3. `main.ts` exécute le bootstrap.
4. Angular instancie `AppModule`.
5. `AppModule` bootstrape `AppComponent`.
6. `AppComponent` est rendu dans `<app-root></app-root>`.

Représentation :

```text
index.html
  └─ <app-root></app-root>
        ▲
        │
AppComponent  ← bootstrap par AppModule
        ▲
        │
AppModule     ← bootstrap par main.ts
        ▲
        │
main.ts
```

### 9.3 Exemple classique de `main.ts`

```ts
import { platformBrowserDynamic } from '@angular/platform-browser-dynamic';

import { AppModule } from './app/app.module';

platformBrowserDynamic()
  .bootstrapModule(AppModule)
  .catch(err => console.error(err));
```

---

## 10. Compléments utiles : fichiers de configuration à la racine

Même si le contenu demandé se concentre sur `src/app/assets/environments` et `main.ts/AppModule`, il est important de situer les principaux fichiers racine (workspace) :

- **`package.json`** : dépendances + scripts npm.
- **`angular.json`** : configuration Angular CLI (build options, assets, styles, etc.).
- **`tsconfig.json`** : configuration TypeScript.

> Ce ne sont pas des fichiers "métier" mais ils gouvernent compilation, bundling et exécution.

---

## 11. Bonnes pratiques d’organisation (recommandations)

### 11.1 Structurer `src/app/` par domaines fonctionnels

Au-delà de la structure minimale, une approche maintenable consiste à séparer :

- `core/` : singletons (services globaux), interceptors, guards, layout
- `shared/` : composants/pipes/directives réutilisables
- `features/` : modules/sections par fonctionnalités (ex. `users/`, `orders/`)

Exemple :

```text
src/app/
├─ core/
│  ├─ services/
│  └─ interceptors/
├─ shared/
│  ├─ components/
│  └─ pipes/
├─ features/
│  ├─ users/
│  └─ admin/
├─ app-routing.module.ts
└─ app.module.ts
```

### 11.2 Garder `assets/` propre

- Regrouper par type (images/fonts/i18n).
- Éviter d’y mettre de la logique (pas de TS).
- Conserver des noms cohérents.

### 11.3 Sécuriser les variables d’environnement

- `environment.ts` n’est **pas** un coffre-fort.
- N’y mettez jamais de secrets (tokens privés, clés non publiques).
- Utilisez un backend ou un système de secrets (CI/CD) quand nécessaire.

---

## 12. Exercices (avec corrigés)

### Exercice 1 — Retrouver le point d’entrée

**Énoncé :** Ouvrez un projet Angular et identifiez le fichier qui démarre l’application. Expliquez ce qu’il fait en 2 phrases.

**Correction :** Le point d’entrée est `src/main.ts`. Il lance Angular et déclenche le bootstrap du module principal (souvent `AppModule`), ce qui aboutit au rendu du composant racine.

### Exercice 2 — Localiser `AppModule`

**Énoncé :** Trouvez le module principal et citez son rôle.

**Correction :** `src/app/app.module.ts`. Il centralise les imports, déclarations et le bootstrap du composant racine.

### Exercice 3 — Ajouter un asset

**Énoncé :** Placez une image `banner.png` dans `src/assets/images/` puis affichez-la dans `AppComponent`.

**Correction :**

```html
<img src="assets/images/banner.png" alt="Banner" />
```

### Exercice 4 — Utiliser un environment

**Énoncé :** Ajoutez `apiUrl` dans `environment.ts` et affichez-le dans la console au démarrage.

**Correction (exemple dans un service ou composant) :**

```ts
import { environment } from '../environments/environment';

console.log('API =', environment.apiUrl);
```

---

## 13. Résumé

- **`src/`** : contient le code et les ressources de l’application.
- **`src/app/`** : logique Angular (components, services, modules) dont **`AppModule`**.
- **`src/assets/`** : ressources statiques copiées au build.
- **`src/environments/`** : configurations selon l’environnement.
- **`main.ts`** : **point d’entrée** qui bootstrape l’application.
- **`AppModule`** : **module principal** qui bootstrape `AppComponent`.

---

## 14. Annexes — Checklist de lecture rapide

- [ ] Je sais où trouver `main.ts`.
- [ ] Je sais où se trouve `AppModule`.
- [ ] Je sais où mettre une image (assets).
- [ ] Je sais où configurer `apiUrl` par environnement.
- [ ] Je comprends le flux `main.ts → AppModule → AppComponent`.
