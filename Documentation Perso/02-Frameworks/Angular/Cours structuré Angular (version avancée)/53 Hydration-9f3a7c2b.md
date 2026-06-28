# Formation Angular 53 — Hydration (SSR/SSG) 

> **Objectif général** : comprendre et maîtriser l’**hydration Angular** (réutilisation côté client du HTML généré côté serveur) afin d’améliorer les performances perçues, réduire le coût du rendu initial et fiabiliser les applications SSR/SSG modernes.

---

## Sommaire

1. [Public & prérequis](#public--prérequis)
2. [Objectifs pédagogiques](#objectifs-pédagogiques)
3. [Durée & format](#durée--format)
4. [Contexte : SSR, SSG et rôle de l’hydration](#contexte--ssr-ssg-et-rôle-de-lhydration)
5. [Concepts clés](#concepts-clés)
6. [Architecture Angular : ce qui se passe réellement](#architecture-angular--ce-qui-se-passe-réellement)
7. [Mettre en place l’hydration dans Angular](#mettre-en-place-lhydration-dans-angular)
8. [Cas pratiques & ateliers](#cas-pratiques--ateliers)
9. [Debug, métriques & performance](#debug-métriques--performance)
10. [Pièges fréquents & bonnes pratiques](#pièges-fréquents--bonnes-pratiques)
11. [Checklist de mise en production](#checklist-de-mise-en-production)
12. [Quiz de validation](#quiz-de-validation)
13. [Annexes : snippets & aide-mémoire](#annexes--snippets--aide-mémoire)

---

## Public & prérequis

### Public
- Développeurs Angular (intermédiaire à avancé)
- Formateurs / référents frontend
- Équipes travaillant sur des apps **SSR/SSG** (Angular + Node, plateformes serverless, etc.)

### Prérequis techniques
- Angular (components, DI, routing, forms) au quotidien
- Notions de base sur **SSR** (Server-Side Rendering) et/ou **SSG** (Static Site Generation)
- Confort avec TypeScript
- Connaissances utiles (non obligatoires) : performance web (LCP/TTFB/CLS), DOM events, change detection

---

## Objectifs pédagogiques

À l’issue de cette formation, vous saurez :

- Expliquer **ce qu’est l’hydration** et pourquoi elle améliore la performance perçue.
- Distinguer **SSR/SSG** de l’hydration (et comprendre leurs complémentarités).
- Mettre en place l’hydration dans une application Angular SSR.
- Identifier et résoudre les problèmes courants (mismatch DOM, double exécution, accès à `window/document`, timing des données, etc.).
- Mesurer les gains via des métriques et outils (Lighthouse, Web Vitals, Angular DevTools/Profiler).
- Appliquer des **bonnes pratiques de développement “hydration-friendly”**.

---

## Durée & format

- **Durée recommandée** : 1 journée (6–7h) ou 2 demi-journées
- **Pédagogie** : alternance entre théorie, démonstrations, exercices guidés, ateliers de diagnostic
- **Livrables** :
  - Application SSR/SSG “avant/après hydration”
  - Checklist production
  - Snippets réutilisables

---

## Contexte : SSR, SSG et rôle de l’hydration

### Pourquoi SSR/SSG ?
- **SSR** : le serveur renvoie du HTML déjà rendu pour la route demandée.
  - Avantages : améliore **TTFB**, **SEO**, contenu visible rapidement.
  - Inconvénients : sans hydration, la page n’est pas interactive immédiatement.
- **SSG** : génération des pages HTML à build-time (prérendu), puis servis statiquement.
  - Avantages : très bon TTFB (CDN), contenu instantané, stabilité.
  - Inconvénients : interactivité nécessite quand même le JS côté client.

### Le problème historique (sans hydration)
Dans un rendu SSR classique, si le client ne réutilise pas le HTML du serveur, il doit :
1. Télécharger le bundle JS
2. Recréer toute l’application Angular
3. **Rerendre** le DOM complet côté client

Conséquences :
- Gaspillage : double rendu (serveur + client)
- LCP / INP / TTI dégradés
- Flicker possible (éléments qui “bougent”)

### Définition : Hydration
**Hydrater**, c’est **attacher Angular** sur le DOM existant (déjà généré côté serveur) et **réutiliser** cette structure HTML, au lieu de la jeter/recréer.

> ✅ Résultat : la page devient interactive plus vite, avec moins de travail DOM.

---

## Concepts clés

### 1) Ce que réutilise l’hydration
- Le **DOM SSR** déjà présent dans le navigateur
- La structure d’éléments, attributs, texte
- Les nœuds correspondants à la vue Angular

Angular vient ensuite :
- Reconnecter les bindings
- Rebrancher les handlers d’événements
- Relancer la détection de changement selon les besoins

### 2) “Mismatch” : le risque principal
Un **mismatch** survient si le HTML SSR **ne correspond pas** à ce que le client calculerait au démarrage.

Exemples :
- Valeur aléatoire (`Math.random()`) utilisée dans le template
- Date/heure locale côté client et côté serveur divergentes
- Données arrivant dans un ordre différent
- Conditions `*ngIf` qui divergent entre SSR et client

Conséquences :
- Angular peut être forcé de “réparer” le DOM
- Perte du bénéfice performance
- Comportements inattendus

### 3) Hydration et interactivité
L’hydration n’est pas “de la magie” :
- Tant que le JS n’est pas chargé, la page est visible mais **pas interactive**
- Dès que l’app est hydratée : boutons/clique/inputs deviennent fonctionnels

### 4) Hydration vs “boot” classique
- **Boot classique** : Angular reconstruit la vue → DOM reconstruit
- **Hydration** : Angular connecte la vue existante → DOM réutilisé

---

## Architecture Angular : ce qui se passe réellement

### Cycle SSR → Client (simplifié)

1. **Serveur**
   - Reçoit la requête
   - Exécute Angular côté serveur
   - Rend HTML (avec le router + data préchargées si prévu)
   - Renvoie HTML au navigateur

2. **Navigateur**
   - Affiche immédiatement le HTML (contenu visible)
   - Télécharge CSS/JS

3. **Boot Angular côté client**
   - Angular démarre
   - Au lieu de rerendre : **hydrate le DOM**
   - Ajoute la couche d’interactivité

### Où se situent les coûts ?
- Parsing HTML (inévitable)
- Téléchargement JS (inévitable)
- Exécution JS (réductible via bundling, lazy loading)
- **Construction DOM** (réduite grâce à hydration)

---

## Mettre en place l’hydration dans Angular

> Les APIs et flags peuvent évoluer selon la version d’Angular. L’idée centrale : **activer l’hydration** côté client pour une app SSR/SSG.

### Étape 0 — Avoir une application SSR/SSG
Si vous n’avez pas encore SSR :
- Utiliser la stack SSR officielle (Angular + rendu serveur)
- Vérifier que la route renvoie du HTML complet

#### Critères de succès SSR
- En désactivant JS dans le navigateur, on doit quand même voir le contenu de la page.

### Étape 1 — Activer l’hydration côté client
Dans une configuration moderne Angular (bootstrap via `bootstrapApplication`), on active l’hydration via un provider.

Exemple (conceptuel) :

```ts
import { bootstrapApplication } from '@angular/platform-browser';
import { provideClientHydration } from '@angular/platform-browser';
import { AppComponent } from './app/app.component';

bootstrapApplication(AppComponent, {
  providers: [
    provideClientHydration(),
  ],
});
```

> **À retenir** : l’hydration doit être activée **côté client**, et nécessite un DOM SSR cohérent.

### Étape 2 — Garantir la cohérence SSR/Client
#### Règle d’or
> Tout ce qui influence le template lors du premier rendu doit produire le **même résultat** sur le serveur et sur le client.

#### Anti-patterns
- `Math.random()` dans un binding
- `new Date()` affichée en direct au SSR sans stratégie de normalisation
- Lecture de `localStorage`/`window` dans le constructeur/computed initial
- Données API arrivant côté client mais pas côté SSR

#### Pattern recommandé : différer le “client-only”
- Encapsuler dans un bloc exécuté seulement dans le navigateur
- Injecter un service “platform” (server vs browser)
- Déclencher certaines features après stabilisation

### Étape 3 — Données : éviter le “double fetch”
Problème : SSR appelle une API pour construire la page, puis le client refait l’appel.

Solutions possibles (selon stack) :
- Transfert d’état (state transfer) serveur → client
- Cache HTTP + headers + déduplication
- Résolveurs + stratégie d’hydratation

Objectif :
- À l’hydration, le client **réutilise** les données déjà connues

### Étape 4 — Router & navigation
- Vérifier que la route hydratée correspond bien à l’URL
- Attention aux redirections côté client au bootstrap (ex : guard qui redirige) :
  - Risque de mismatch
  - Risque d’affichage “flash”

### Étape 5 — Événements & formulaires
- Tester les éléments interactifs :
  - click, input, change, submit
- Pièges :
  - Valeurs initiales d’inputs calculées différemment SSR vs client
  - Validation déclenchée trop tôt

---

## Cas pratiques & ateliers

### Atelier 1 — Baseline et observation
**But** : comparer un boot SSR avec et sans hydration.

1. Démarrer l’app SSR
2. Mesurer :
   - LCP
   - JS execution time
   - Long tasks
3. Observer :
   - Le DOM est-il recréé ?
   - Y a-t-il un flicker ?

**Livrable** : captures Lighthouse avant/après.

### Atelier 2 — Reproduire un mismatch
**But** : comprendre les symptômes.

Créer un composant :

```ts
@Component({
  selector: 'app-mismatch',
  template: `
    <p>Jeton: {{ token }}</p>
  `,
})
export class MismatchComponent {
  token = Math.random().toString(16).slice(2);
}
```

**Constat** : SSR et client produisent des tokens différents → mismatch.

**Correction** :
- Générer côté serveur et transférer l’état
- Ou différer l’affichage côté client après hydration (si acceptable)

### Atelier 3 — Client-only API (window/localStorage)
**But** : rendre le code SSR-safe et hydration-friendly.

Mauvais exemple :

```ts
const theme = localStorage.getItem('theme');
```

Correction (pattern) :
- Créer un service `BrowserStorageService`
- Ne lire le storage que si on est dans le navigateur
- Prévoir une valeur par défaut SSR identique

### Atelier 4 — Double fetch et state transfer
**But** : éviter les appels API en doublon.

Scénario : composant “ProductList” qui charge via HTTP.

Axes de correction :
- Mettre en cache côté serveur
- Transférer les données au client
- Déduplication observable (shareReplay) et stratégie de réutilisation

---

## Debug, métriques & performance

### Outils
- **Lighthouse** (Performance + Web Vitals)
- **Chrome DevTools Performance** (long tasks, scripting)
- **Network panel** (double fetch, cache)
- **Angular DevTools** (profiling, change detection)

### Métriques à surveiller
- **TTFB** : SSR/SSG améliore souvent
- **LCP** : contenu visible plus vite avec SSR, encore mieux si hydration évite le rerender
- **CLS** : attention aux changements de layout entre SSR et client
- **INP** : interactivité; hydration doit aider si elle réduit les coûts initiaux

### Signes d’un problème d’hydration
- Re-rendu complet visible (flicker)
- DOM reconstruit (nœuds remplacés)
- Erreurs/avertissements liés à mismatch
- Appels API dupliqués

---

## Pièges fréquents & bonnes pratiques

### Pièges
1. **Données non déterministes**
2. **Accès au navigateur** en phase SSR (window/document)
3. **Différences de timezone/locale**
4. **Redirections router** au démarrage
5. **Animations** qui démarrent trop tôt et changent la structure DOM
6. **Hydration partielle non maîtrisée** (si certaines zones sont exclues)

### Bonnes pratiques
- Rendre le **premier rendu déterministe**
- Centraliser les accès “browser-only” via services
- Précharger les données côté SSR et réutiliser côté client
- Stabiliser l’UI : skeletons cohérents SSR/Client
- Utiliser `OnPush` et une architecture réactive pour limiter les recalculs

---

## Checklist de mise en production

- [ ] SSR/SSG déjà fonctionnel et testable avec JS désactivé
- [ ] Hydration activée côté client
- [ ] Aucun contenu non déterministe au premier rendu
- [ ] Pas d’accès direct à `window/document/localStorage` au bootstrap
- [ ] Données critiques préchargées côté SSR et réutilisées (pas de double fetch)
- [ ] Tests E2E : interactivité après hydration (formulaires, boutons)
- [ ] Mesures Lighthouse/Web Vitals avant/après
- [ ] Surveillance en prod (RUM) : LCP/INP/CLS

---

## Quiz de validation

1. Définissez l’hydration en une phrase.
2. Quel est le bénéfice principal sur le plan performance ?
3. Donnez 3 causes typiques de mismatch SSR/Client.
4. Pourquoi le double fetch est-il fréquent et comment l’éviter ?
5. Que se passe-t-il si le HTML SSR ne correspond pas à ce qu’Angular attend ?

---

## Annexes : snippets & aide-mémoire

### A) Règles “Hydration-friendly”
- Éviter les valeurs non déterministes dans les templates (ou les stabiliser)
- Différer tout comportement “client-only” après le bootstrap/hydration
- Garder la structure DOM stable entre SSR et client

### B) Exemple de pattern “platform-aware” (conceptuel)

```ts
import { inject, Injectable } from '@angular/core';
import { PLATFORM_ID } from '@angular/core';
import { isPlatformBrowser } from '@angular/common';

@Injectable({ providedIn: 'root' })
export class PlatformService {
  private platformId = inject(PLATFORM_ID);
  isBrowser(): boolean {
    return isPlatformBrowser(this.platformId);
  }
}
```

Usage :

```ts
if (this.platform.isBrowser()) {
  // code navigateur uniquement
}
```

### C) Guide de diagnostic rapide
1. Contenu visible SSR OK ?
2. Hydration activée ?
3. Mismatch : vérifier les valeurs affichées (dates, random, conditions)
4. Double fetch : inspecter l’onglet Network
5. CLS : vérifier le layout entre SSR et après bootstrap

---

## Conclusion

L’hydration est la pierre angulaire des optimisations modernes SSR/SSG : elle permet à Angular de **réutiliser le HTML serveur** et d’atteindre une interactivité plus rapide sans rerendu complet. Pour en tirer le maximum, il faut une application SSR cohérente, déterministe au premier rendu, avec une stratégie de données et de “client-only logic” maîtrisée.
