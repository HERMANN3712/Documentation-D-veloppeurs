# 20 — Build et déploiement (Angular)

## Objectifs pédagogiques
À la fin de ce module, vous serez capable de :

- Comprendre ce que produit **Angular CLI** lors d’un build.
- Construire une application Angular avec `ng build` selon plusieurs environnements.
- Optimiser le build (AOT, optimisation, budgets, source maps, hashing).
- Préparer et vérifier un livrable de déploiement.
- Déployer les fichiers statiques sur un **serveur web** ou une **plateforme cloud**.

## Prérequis
- Connaissances de base Angular (composants, modules, routes).
- Node.js + npm installés.
- Angular CLI installé (`npm i -g @angular/cli`) ou via `npx`.

## Durée estimée
- 2h à 3h (selon le niveau d’approfondissement et les exercices)

---

## 1) Le principe : build = compilation + bundling + optimisation
Une application Angular est écrite en TypeScript et composée de nombreux fichiers (TS, HTML, SCSS, assets). Pour être déployée sur le Web, elle doit être transformée en **fichiers statiques** consommables par un navigateur.

Le build Angular consiste principalement à :

- **Compiler** TypeScript → JavaScript.
- Compiler les templates Angular et exécuter l’**AOT (Ahead-of-Time)**.
- **Assembler** (bundler) le code en bundles (chunks) optimisés.
- Minifier, tree-shaker, optimiser les CSS.
- Copier les **assets**.
- Générer un `index.html` prêt à charger les bundles.

La sortie du build est un dossier (par défaut) :

- `dist/<nom-du-projet>/` contenant des fichiers statiques.

---

## 2) La commande centrale : `ng build`
### 2.1 Commande minimum
```bash
ng build
```
- Construit l’application avec la configuration par défaut.
- Produit un dossier `dist/`.

### 2.2 Choisir une configuration (environnement)
Les builds sont généralement différenciés par **environnement** (dev, staging, prod).

Exemple :
```bash
ng build --configuration production
```
ou (alias courant) :
```bash
ng build -c production
```

#### Où sont définies les configurations ?
Dans `angular.json`, section `architect > build > configurations`.

### 2.3 Flags courants
Quelques options utiles (selon version Angular/CLI) :

- `--configuration <name>` : utiliser une configuration.
- `--output-path <path>` : changer le répertoire de sortie.
- `--base-href <href>` : base URL de l’application (cas de sous-chemin).
- `--deploy-url <url>` : URL de chargement des assets.
- `--source-map` : activer/désactiver les source maps.

Exemple :
```bash
ng build -c production --output-path dist/prod --base-href /monapp/
```

---

## 3) Comprendre le contenu de `dist/`
Après un build, vous trouverez typiquement :

- `index.html` : point d’entrée.
- `main.<hash>.js` : bundle principal Angular.
- `polyfills.<hash>.js` : compatibilité navigateur.
- `runtime.<hash>.js` : runtime du bundler.
- `styles.<hash>.css` : styles globaux.
- éventuels chunks : `chunk-XXXX.<hash>.js` (lazy-loading).
- `assets/` : images, icônes, fichiers statiques copiés.

### 3.1 Pourquoi des hashes dans les noms de fichiers ?
Le **cache busting** :
- En production, on active le *file hashing* pour que le navigateur recharge les fichiers quand ils changent.

---

## 4) Build production : objectifs et optimisations
### 4.1 Ce que vise un build production
- Temps de chargement réduit
- Poids des bundles minimisé
- Moins de code mort (tree-shaking)
- Performances runtime améliorées

### 4.2 Paramètres typiques d’un build production
Selon la configuration `production`, Angular CLI active généralement :
- Optimisations et minification
- AOT
- Build optimizer
- Hashing des fichiers
- Budgets (alertes si bundles trop gros)

> Remarque : les détails varient légèrement selon la version Angular/CLI, mais l’objectif reste identique.

---

## 5) Gestion des environnements (dev/staging/prod)
### 5.1 Fichiers d’environnements
Angular utilise souvent :
- `src/environments/environment.ts`
- `src/environments/environment.prod.ts`

Ces fichiers contiennent des valeurs (ex : URL d’API) remplacées selon la configuration.

Exemple :
```ts
// environment.ts
export const environment = {
  production: false,
  apiUrl: 'http://localhost:3000'
};
```

```ts
// environment.prod.ts
export const environment = {
  production: true,
  apiUrl: 'https://api.mondomaine.com'
};
```

### 5.2 File replacements
Dans `angular.json`, configuration `production` :
- Remplacement automatique de `environment.ts` par `environment.prod.ts`.

---

## 6) Déployer : ce que vous livrez réellement
### 6.1 Le livrable
Pour une application Angular **SPA** classique, vous livrez :

- Le contenu du dossier `dist/<app>/`

Ce sont **des fichiers statiques** (HTML/CSS/JS/assets). Il n’y a pas besoin d’un serveur Node pour exécuter Angular côté client, sauf cas spécifiques (SSR, API, etc.).

### 6.2 Rôle du serveur web
Le serveur web doit :

1. Servir `index.html`
2. Servir les fichiers `*.js`, `*.css`, `assets/*`
3. Gérer le routing côté client (Angular Router)

Le point 3 est crucial : sur une route `/users/42`, le serveur doit renvoyer `index.html` (fallback) pour laisser Angular router la page.

---

## 7) Déploiement sur un serveur web (Nginx/Apache)
### 7.1 Déploiement conceptuel
Étapes génériques :

1. Faire le build :
   ```bash
   ng build -c production
   ```
2. Copier le contenu de `dist/<app>/` vers le répertoire servi par le serveur web (ex : `/var/www/monapp`).
3. Configurer le serveur pour :
   - servir les fichiers statiques
   - renvoyer `index.html` pour toutes les routes non-fichiers

### 7.2 Exemple Nginx (SPA fallback)
Exemple minimal :

```nginx
server {
  listen 80;
  server_name mondomaine.com;

  root /var/www/monapp;
  index index.html;

  location / {
    try_files $uri $uri/ /index.html;
  }
}
```

### 7.3 Exemple Apache (SPA fallback)
Avec `.htaccess` (si `AllowOverride` activé) :

```apache
<IfModule mod_rewrite.c>
  RewriteEngine On
  RewriteBase /

  RewriteCond %{REQUEST_FILENAME} -f [OR]
  RewriteCond %{REQUEST_FILENAME} -d
  RewriteRule ^ - [L]

  RewriteRule ^ index.html [L]
</IfModule>
```

---

## 8) Déploiement sur une plateforme cloud (concepts)
Angular produit des fichiers statiques, donc de nombreux hébergeurs conviennent :

- Hébergement statique (object storage + CDN)
- Hosting spécialisé SPA
- PaaS avec serveur web

### 8.1 Points à vérifier
- **Fallback SPA** (toutes les routes → `index.html`)
- Gestion du **cache** (hashing + headers)
- HTTPS / domaine / redirections
- Variables d’environnement :
  - soit via build (file replacements)
  - soit via configuration runtime (fichier JSON chargé au démarrage) si nécessaire

### 8.2 Stratégie cache recommandée (générale)
- `index.html` : cache court ou no-cache (car il référence les nouveaux bundles)
- `*.js`/`*.css` hashés : cache long (immutable)

---

## 9) Cas fréquent : déploiement sur un sous-chemin
Si l’app n’est pas à la racine du domaine mais sous `/monapp/` :

```bash
ng build -c production --base-href /monapp/
```

- `base-href` doit correspondre au chemin public réel.
- Le serveur web doit servir l’app depuis ce sous-répertoire.

---

## 10) Vérifications avant mise en production
### 10.1 Vérifier localement le build
Faire un build prod puis servir le dossier `dist/` avec un serveur statique local.

Exemples :

- Avec `npx` :
  ```bash
  npx http-server dist/<app> -p 8080
  ```
- Ou tout serveur statique équivalent.

Ensuite :
- Tester les routes profondes (ex : `/users/42`) en rafraîchissant la page.
- Vérifier la console (erreurs CORS, erreurs 404 assets).
- Vérifier le chargement des bundles (onglet Network).

### 10.2 Vérifier les tailles de bundles
- Surveiller les avertissements de **budgets**.
- Identifier les dépendances lourdes.

---

## 11) Bonnes pratiques
- Toujours builder en **production** pour livrer.
- Activer le hashing de fichiers pour bénéficier du cache navigateur.
- Mettre en place un fallback SPA afin que le routing Angular fonctionne.
- Distinguer configuration build-time (environments) vs runtime (config externe) si votre déploiement doit changer sans rebuild.
- Automatiser via CI/CD : build → tests → artefact → déploiement.

---

## 12) Exercices (avec corrigés)
### Exercice 1 — Produire un build production
**Énoncé** : produire un build production et retrouver les fichiers générés.

1. Lancer :
   ```bash
   ng build -c production
   ```
2. Ouvrir `dist/<app>/` et identifier `index.html`, `main.*.js`, `styles.*.css`, `assets/`.

**Corrigé attendu** :
- Les fichiers ont des noms avec hash (selon config) et sont minifiés.

### Exercice 2 — Déployer sous un sous-chemin
**Énoncé** : supposons que l’application sera accessible via `https://mondomaine.com/monapp/`.

1. Builder avec :
   ```bash
   ng build -c production --base-href /monapp/
   ```
2. Copier `dist/<app>/` sur le serveur dans le répertoire correspondant.

**Corrigé attendu** :
- `index.html` contient un `<base href="/monapp/">`.
- Les assets chargent correctement sous `/monapp/`.

### Exercice 3 — Vérifier le fallback SPA
**Énoncé** : simuler un rafraîchissement sur une route Angular `/users/42`.

1. Servir le dossier `dist/` localement.
2. Naviguer vers une route profonde.
3. Rafraîchir.

**Corrigé attendu** :
- Si le serveur est configuré pour fallback vers `index.html`, la route se charge.
- Sinon, le serveur renvoie un 404 (mauvaise config de rewrite).

---

## 13) Synthèse
- `ng build` est l’outil standard pour **construire** une application Angular.
- Le résultat est un ensemble de **fichiers statiques** prêts à être servis.
- Le déploiement consiste à **héberger** le dossier `dist/` sur un serveur web ou une plateforme cloud.
- La configuration du serveur doit gérer le **routing SPA** et le **cache**.

---

## Références
- Documentation Angular CLI : https://angular.dev/tools/cli
- Déploiement Angular (guide) : https://angular.dev/tools/cli/deployment
