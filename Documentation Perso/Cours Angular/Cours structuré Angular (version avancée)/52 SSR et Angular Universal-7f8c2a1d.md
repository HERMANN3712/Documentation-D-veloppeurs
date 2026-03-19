# Formation Angular — SSR et Angular Universal

> **Public** : développeurs Angular (intermédiaire à avancé)
>
> **Objectif** : comprendre, mettre en place et industrialiser le **Server‑Side Rendering (SSR)** avec **Angular Universal**, tout en gérant les différences d’exécution **navigateur vs serveur**, l’impact SEO, la performance, ainsi que les dépendances non compatibles SSR.
>
> **Durée recommandée** : 1 jour (7h) ou 2 demi‑journées
>
> **Pré‑requis** : Angular CLI, TypeScript, RxJS, notions de routing, HTTP, architecture d’applications Angular.

---

## Table des matières

1. [Introduction : pourquoi le SSR ?](#1-introduction--pourquoi-le-ssr-)
2. [Rappels : CSR, SSR, SSG, Hydration](#2-rappels--csr-ssr-ssg-hydration)
3. [Angular Universal : principes et architecture](#3-angular-universal--principes-et-architecture)
4. [Mettre en place Angular Universal (Angular CLI)](#4-mettre-en-place-angular-universal-angular-cli)
5. [Cycle de rendu SSR et partage de code](#5-cycle-de-rendu-ssr-et-partage-de-code)
6. [Browser-only vs Server-only : détecter la plateforme](#6-browser-only-vs-server-only--détecter-la-plateforme)
7. [Gérer le DOM, Window, Document, localStorage… sans casser le SSR](#7-gérer-le-dom-window-document-localstorage-sans-casser-le-ssr)
8. [HTTP, transfert d’état et performances](#8-http-transfert-détat-et-performances)
9. [Routage, guards, resolvers et SSR](#9-routage-guards-resolvers-et-ssr)
10. [SEO : meta, title, balisage, OpenGraph](#10-seo--meta-title-balisage-opengraph)
11. [Dépendances et bibliothèques non compatibles SSR : stratégies](#11-dépendances-et-bibliothèques-non-compatibles-ssr--stratégies)
12. [Déploiement : Node, reverse proxy, cache, CDN](#12-déploiement--node-reverse-proxy-cache-cdn)
13. [Sécurité et bonnes pratiques](#13-sécurité-et-bonnes-pratiques)
14. [Debug, logs et tests](#14-debug-logs-et-tests)
15. [Ateliers pratiques (guidés)](#15-ateliers-pratiques-guidés)
16. [Checklist de mise en production](#16-checklist-de-mise-en-production)
17. [Références et ressources](#17-références-et-ressources)

---

## 1. Introduction : pourquoi le SSR ?

Angular est historiquement orienté **Client‑Side Rendering (CSR)** : le navigateur charge une page minimale, puis JavaScript construit l’UI. Cela fonctionne très bien pour des apps internes ou des SPAs où le SEO est secondaire.

Le **Server‑Side Rendering (SSR)** consiste à **rendre la première page HTML côté serveur**, puis à laisser Angular reprendre la main côté navigateur.

### Bénéfices clés

- **SEO** : les robots voient un HTML déjà rendu (contenu, titres, liens).
- **Temps au premier affichage (First Contentful Paint / Largest Contentful Paint)** : le contenu est visible plus tôt.
- **Partage social** : meta tags (OpenGraph/Twitter) dynamiques.
- **Accessibilité et périphériques faibles** : un HTML initial complet aide l’affichage.

### Contraintes

- Besoin de distinguer :
  - **code compatible navigateur** (DOM, APIs Web, stockage local)
  - **code exécuté côté serveur** (Node.js)
- Certaines bibliothèques sont **non SSR‑friendly** (accès au DOM au moment du chargement, dépendance à `window`, etc.).
- Coût serveur : rendu à la demande, cadence et cache.

---

## 2. Rappels : CSR, SSR, SSG, Hydration

### CSR (Client‑Side Rendering)
- HTML initial minimal.
- JS télécharge les bundles, puis construit la page.
- **Problèmes** : SEO et « blank screen » initial.

### SSR (Server‑Side Rendering)
- Le serveur (Node) exécute Angular et produit un HTML initial.
- Le navigateur charge la page déjà rendue.

### SSG (Static Site Generation / prerender)
- Les pages sont rendues **à l’avance** en HTML statique.
- Très rapide, excellent SEO, mais moins adapté au contenu très dynamique.

### Hydration
- Étape où le client « réutilise » le DOM SSR existant au lieu de le reconstruire.
- Réduit les clignotements et accélère l’interactivité.

---

## 3. Angular Universal : principes et architecture

Angular Universal est l’ensemble de briques permettant le SSR.

### Composants typiques

- **Application Angular** (code commun)
- **Bundle serveur** (exécuté sur Node)
- **Serveur Express** (souvent) qui :
  - sert les assets statiques
  - rend les routes via Universal
- Optionnel : **prerender** (SSG) pour certains chemins.

### Point fondamental

Votre code Angular doit devenir **isomorphe/compatible universel** :
- ne pas dépendre d’APIs navigateur lors de l’exécution serveur
- isoler les effets de bord
- éviter le code qui s’exécute dès l’import (top-level side effects) avec accès DOM.

---

## 4. Mettre en place Angular Universal (Angular CLI)

> Les commandes exactes peuvent varier selon la version d’Angular. L’approche reste la même : ajouter le support SSR, puis lancer un serveur Node.

### 4.1 Ajouter Universal

Dans un projet Angular existant :

```bash
ng add @nguniversal/express-engine
```

Cela ajoute typiquement :
- un **main.server.ts**
- une **app.server.module.ts**
- un serveur **Express** : `server.ts`
- des configurations build : `build:ssr`, `serve:ssr`...

### 4.2 Builder et servir en SSR

```bash
npm run build:ssr
npm run serve:ssr
```

Vous devez obtenir :
- un build navigateur (assets + JS)
- un build serveur (bundle Node)
- un serveur qui rend les routes.

### 4.3 Structure de sortie (indicative)

- `dist/<app>/browser/` : build client
- `dist/<app>/server/` : build serveur

### 4.4 Vérifier que c’est bien du SSR

- Ouvrir le HTML côté navigateur, mais surtout :
  - `curl http://localhost:4000/` et vérifier que le HTML contient déjà le contenu.
  - Désactiver JS dans le navigateur : la page doit rester lisible.

---

## 5. Cycle de rendu SSR et partage de code

### 5.1 Rendu SSR
1. Requête HTTP sur une route `/produits/42`
2. Express déclenche Angular Universal
3. Angular exécute :
   - router
   - resolvers/guards
   - services
   - templates
4. Angular renvoie un HTML string
5. Réponse HTTP avec HTML rendu

### 5.2 Reprise côté navigateur
- Le navigateur charge `index.html` déjà rendu
- Télécharge les bundles
- Angular démarre et « attache » l’interactivité

### 5.3 Implication pratique
- Un composant peut être instancié **deux fois** :
  - côté serveur
  - côté client

Il faut donc :
- éviter les effets de bord non idempotents
- contrôler quand et où on déclenche certaines actions.

---

## 6. Browser-only vs Server-only : détecter la plateforme

### 6.1 Injection de `PLATFORM_ID`

```ts
import { Inject, PLATFORM_ID } from '@angular/core';
import { isPlatformBrowser, isPlatformServer } from '@angular/common';

constructor(@Inject(PLATFORM_ID) private platformId: object) {}

get isBrowser(): boolean {
  return isPlatformBrowser(this.platformId);
}

get isServer(): boolean {
  return isPlatformServer(this.platformId);
}
```

### 6.2 Exemple : n’exécuter un code DOM que dans le navigateur

```ts
ngAfterViewInit() {
  if (!isPlatformBrowser(this.platformId)) return;

  // code manipulant le DOM, charts, etc.
}
```

### 6.3 Séparer les implémentations via DI (pattern recommandé)

Créer un token d’abstraction :

```ts
export abstract class StorageService {
  abstract get(key: string): string | null;
  abstract set(key: string, value: string): void;
}
```

- Implémentation Browser : `localStorage`
- Implémentation Server : mémoire/Noop

```ts
@Injectable()
export class BrowserStorageService implements StorageService {
  get(key: string) { return localStorage.getItem(key); }
  set(key: string, value: string) { localStorage.setItem(key, value); }
}

@Injectable()
export class ServerStorageService implements StorageService {
  private map = new Map<string, string>();
  get(key: string) { return this.map.get(key) ?? null; }
  set(key: string, value: string) { this.map.set(key, value); }
}
```

Puis fournir selon la plateforme (approche simple : provider conditionnel dans un module, ou factory basée sur `PLATFORM_ID`).

---

## 7. Gérer le DOM, Window, Document, localStorage… sans casser le SSR

### 7.1 Problème classique

Le code suivant **casse** côté serveur :

```ts
const width = window.innerWidth;
```

Côté Node : `window` n’existe pas → crash.

### 7.2 Injecter `DOCUMENT`

Angular fournit un token `DOCUMENT`.

```ts
import { DOCUMENT } from '@angular/common';

constructor(@Inject(DOCUMENT) private document: Document) {}
```

Mais attention : `Document` en SSR est un document « simulé ». Certaines opérations doivent rester browser-only.

### 7.3 Abstraire `window`

Créer un token `WINDOW` :

```ts
import { InjectionToken } from '@angular/core';

export const WINDOW = new InjectionToken<Window | null>('WINDOW');

export function windowFactory(platformId: object): Window | null {
  return isPlatformBrowser(platformId) ? window : null;
}
```

Provider :

```ts
{ provide: WINDOW, useFactory: windowFactory, deps: [PLATFORM_ID] }
```

Usage :

```ts
constructor(@Inject(WINDOW) private win: Window | null) {}

ngOnInit() {
  if (!this.win) return;
  console.log(this.win.innerWidth);
}
```

### 7.4 localStorage / sessionStorage
- Non disponibles côté serveur.
- Approche : wrap via service (cf. section 6.3).

### 7.5 setTimeout / requestAnimationFrame
- `setTimeout` existe côté Node, mais son usage en SSR peut produire :
  - rendu non déterministe
  - fuite de ressources
- `requestAnimationFrame` n’existe pas côté serveur.
- Solution : exécuter ces actions uniquement côté navigateur.

---

## 8. HTTP, transfert d’état et performances

### 8.1 Problème : double-fetch

Sans précaution, une page SSR peut :
- faire un HTTP côté serveur pour rendre le HTML
- refaire le même HTTP côté client lors du bootstrap

→ coût multiplié, ralentissement.

### 8.2 Transfert d’état (concept)

Le serveur récupère les données, les place dans un état sérialisé dans la page, puis le client les réutilise.

Dans Angular, historiquement : `TransferState`.

#### Exemple conceptuel avec `TransferState`

```ts
import { TransferState, makeStateKey } from '@angular/platform-browser';

const PRODUCTS_KEY = makeStateKey<any>('products');

constructor(
  private http: HttpClient,
  private transferState: TransferState,
  @Inject(PLATFORM_ID) private platformId: object
) {}

loadProducts() {
  const cached = this.transferState.get(PRODUCTS_KEY, null as any);
  if (cached) {
    this.transferState.remove(PRODUCTS_KEY);
    return of(cached);
  }

  return this.http.get('/api/products').pipe(
    tap(data => {
      if (isPlatformServer(this.platformId)) {
        this.transferState.set(PRODUCTS_KEY, data);
      }
    })
  );
}
```

### 8.3 Cache serveur

Même avec TransferState, le SSR rend « à la demande ». Pour les pages peu personnalisées :
- cache mémoire (court)
- cache reverse-proxy (Nginx, Varnish)
- CDN

Objectif : éviter de recalculer la même page à chaque requête.

### 8.4 Mesures de perf à suivre
- TTFB (Time To First Byte)
- FCP / LCP
- INP (Interaction to Next Paint)
- Taille HTML SSR renvoyé
- CPU serveur par rendu

---

## 9. Routage, guards, resolvers et SSR

### 9.1 Routage SSR
- Universal exécute le routeur.
- Les resolvers peuvent être utiles pour précharger les données.

### 9.2 Guards
- Un guard qui lit `localStorage` cassera SSR.
- Un guard dépendant de cookies (via headers) doit être adapté : côté serveur, l’info vient de la requête HTTP.

### 9.3 Résolution des données pour SSR

Approche :
- mettre la logique de fetch dans un service
- typiquement utiliser un resolver pour s’assurer que les données sont disponibles au moment du rendu.

Exemple de resolver (simplifié) :

```ts
@Injectable({ providedIn: 'root' })
export class ProductResolver implements Resolve<Product> {
  constructor(private api: ProductsApi) {}

  resolve(route: ActivatedRouteSnapshot) {
    return this.api.getProduct(route.paramMap.get('id')!);
  }
}
```

---

## 10. SEO : meta, title, balisage, OpenGraph

SSR rend possible un SEO plus fiable, mais il faut aussi **mettre à jour les meta**.

### 10.1 Title et Meta service

```ts
import { Title, Meta } from '@angular/platform-browser';

constructor(private title: Title, private meta: Meta) {}

setSeo(product: Product) {
  this.title.setTitle(`${product.name} | Ma boutique`);
  this.meta.updateTag({ name: 'description', content: product.summary });
  this.meta.updateTag({ property: 'og:title', content: product.name });
  this.meta.updateTag({ property: 'og:description', content: product.summary });
}
```

### 10.2 Canonical, robots, hreflang
- Ajouter des liens `canonical` si nécessaire.
- Gérer `robots` selon les environnements.

### 10.3 Données structurées (JSON-LD)
- SSR permet d’injecter du JSON-LD directement dans le HTML initial.

---

## 11. Dépendances et bibliothèques non compatibles SSR : stratégies

### 11.1 Symptômes
- `ReferenceError: window is not defined`
- `document is not defined`
- crash à l’import d’un module

### 11.2 Stratégies

#### A) Import dynamique (lazy import) côté navigateur

```ts
async ngAfterViewInit() {
  if (!isPlatformBrowser(this.platformId)) return;

  const { Chart } = await import('chart.js');
  // init chart
}
```

#### B) Wrapper service + DI
- Encapsuler la lib dans un service.
- Fournir un stub côté serveur.

#### C) Vérifier les side effects au chargement
- Éviter que la lib exécute du code DOM dès l’import.
- Parfois, une alternative SSR-friendly est nécessaire.

#### D) Conditional rendering
- `*ngIf="isBrowser"` pour ne pas rendre certains composants côté serveur.

---

## 12. Déploiement : Node, reverse proxy, cache, CDN

### 12.1 Architecture typique

- Nginx/Ingress (TLS, compression, headers)
- Node SSR (Express)
- API backend
- CDN pour assets statiques

### 12.2 Concepts importants
- **Compression** : gzip/brotli
- **Cache-control** :
  - assets fingerprintés : cache long
  - HTML SSR : cache court/conditionnel
- **Timeouts** : SSR ne doit pas bloquer indéfiniment
- **Observabilité** : logs, métriques, traces

### 12.3 Règles de cache de base
- `/assets/*` → cache très long
- `/*.js,/*.css` fingerprintées → cache long
- routes HTML → cache court (ou surrogate keys / invalidation)

---

## 13. Sécurité et bonnes pratiques

- Attention au rendu de contenu utilisateur : XSS.
- Ne pas exposer de secrets côté client.
- Ne pas sérialiser d’informations sensibles dans `TransferState`.
- Sur le serveur SSR :
  - headers sécurité (CSP, X-Frame-Options, etc.)
  - limiter la charge (rate limiting)
  - isoler les environnements.

---

## 14. Debug, logs et tests

### 14.1 Debug SSR
- Logger côté serveur (Node) vs console navigateur.
- Utiliser des messages préfixés : `[SSR]`.

### 14.2 Tester rapidement le SSR
- `curl` sur les routes et vérifier le HTML.
- Désactiver JS pour valider la lisibilité.

### 14.3 Tests
- unit tests : privilégier l’absence de dépendance DOM.
- e2e : vérifier méta tags, HTML initial, contenu.

---

## 15. Ateliers pratiques (guidés)

### Atelier 1 — Activer Angular Universal
**But** : ajouter SSR à un projet existant.

1. `ng add @nguniversal/express-engine`
2. `npm run serve:ssr`
3. Vérifier via `curl` que le HTML est rendu.

**Livrable** : application accessible en SSR.

---

### Atelier 2 — Corriger une erreur `window is not defined`
**But** : rendre un composant compatible SSR.

1. Identifier la ligne fautive.
2. Mettre en place `PLATFORM_ID` + `isPlatformBrowser`.
3. Optionnel : introduire un token `WINDOW`.

**Livrable** : composant stable SSR + navigateur.

---

### Atelier 3 — Éviter le double-fetch avec TransferState
**But** : ne pas refaire un appel API côté client.

1. Ajouter `TransferState`.
2. Placer la donnée côté serveur.
3. Réutiliser côté client.

**Livrable** : un seul fetch effectif pour la première navigation.

---

### Atelier 4 — SEO : title/meta dynamiques par route
**But** : améliorer l’indexabilité.

1. Sur une page produit/blog, définir title/description.
2. Ajouter OpenGraph.
3. Vérifier dans le HTML SSR.

**Livrable** : meta tags corrects selon la route.

---

## 16. Checklist de mise en production

- [ ] HTML SSR contient le contenu principal (test `curl`).
- [ ] Aucun accès direct à `window/document/localStorage` sans garde plateforme.
- [ ] Bibliothèques UI/Charts importées de manière SSR-safe (lazy import / wrappers).
- [ ] Éviter le double-fetch (TransferState ou cache).
- [ ] Stratégie cache claire (assets vs HTML).
- [ ] Observabilité (logs SSR + métriques).
- [ ] Meta SEO : title/description/Og + canonical.
- [ ] Paramètres Node : mémoire, timeouts, gestion erreurs.
- [ ] Vérification performance (TTFB, FCP/LCP, charge CPU SSR).

---

## 17. Références et ressources

- Documentation Angular : SSR / Angular Universal
- Guides de performance Web (Core Web Vitals)
- Bonnes pratiques SEO pour SPAs/SSR

---

# Annexes

## A. Anti‑patterns SSR fréquents

1. **Accéder à `window` au top‑level**
   ```ts
   // Mauvais : exécuté dès l'import
   const ua = window.navigator.userAgent;
   ```

2. **Initialiser une lib DOM dans un service singleton**
   - S’exécute server-side aussi.

3. **Assumer que `ngAfterViewInit` ne tourne que côté navigateur**
   - Il tourne aussi en SSR (selon les cas), donc gardez une condition plateforme.

## B. Stratégie pédagogique (pour formateur)

- Démarrer par un cas concret : route produit/blog.
- Montrer le gain SEO + `curl`.
- Provoquer volontairement une erreur `window is not defined` puis corriger.
- Introduire TransferState pour le coût réseau.
- Conclure sur déploiement et cache.
