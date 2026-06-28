# Formation Vue.js — Build et déploiement

> **Module 17** — Build et déploiement

## Objectifs pédagogiques
À l’issue de cette formation, vous saurez :

- Expliquer ce que fait `npm run build` dans un projet Vue.
- Identifier les artefacts générés et leur rôle (HTML/CSS/JS, assets, maps, manifest, etc.).
- Paramétrer un build de production (variables d’environnement, base URL, optimisation).
- Déployer une application Vue sur un serveur web (Nginx/Apache) et gérer le routage SPA.
- Déployer sur une plateforme cloud/static hosting (Netlify, Vercel, GitHub Pages, etc.).
- Mettre en place des bonnes pratiques : cache, headers, compression, versioning, CI/CD.

## Public cible
Développeurs et développeuses front-end ayant des bases en Vue.js (Vue 3 conseillé) et Node.js/npm.

## Pré-requis
- Savoir lancer un projet Vue et utiliser npm.
- Notions de base sur HTTP et sur le fonctionnement d’un serveur web.
- Accès à un terminal, Git, et un éditeur (VS Code).

## Durée indicative
2h à 3h (selon profondeur des ateliers).

## Plan détaillé
1. [Introduction : qu’est-ce qu’un build ?](#1-introduction--quest-ce-quun-build)
2. [Construire pour la production : `npm run build`](#2-construire-pour-la-production--npm-run-build)
3. [Comprendre le dossier de sortie (`dist`)](#3-comprendre-le-dossier-de-sortie-dist)
4. [Paramétrage : variables d’environnement et configuration](#4-paramétrage--variables-denvironnement-et-configuration)
5. [Routes et SPA : le point crucial du déploiement](#5-routes-et-spa--le-point-crucial-du-déploiement)
6. [Déploiement sur un serveur web (Nginx/Apache)](#6-déploiement-sur-un-serveur-web-nginxapache)
7. [Déploiement sur une plateforme cloud/static hosting](#7-déploiement-sur-une-plateforme-cloudstatic-hosting)
8. [Bonnes pratiques de production (perf, cache, sécurité)](#8-bonnes-pratiques-de-production-perf-cache-sécurité)
9. [CI/CD : automatiser build et déploiement](#9-cicd--automatiser-build-et-déploiement)
10. [Ateliers pratiques & checklist](#10-ateliers-pratiques--checklist)

---

## 1) Introduction : qu’est-ce qu’un build ?

Dans un projet Vue moderne, le code source (fichiers `.vue`, modules ES, SCSS, images, etc.) n’est **pas** directement optimisé pour le navigateur en production.

Le **build** consiste à transformer votre code en un ensemble de fichiers statiques optimisés :

- **Bundling** : regrouper les modules JS/CSS.
- **Transpilation** : rendre compatible avec les navigateurs ciblés.
- **Minification** : réduire la taille (suppression d’espaces, renommage, etc.).
- **Tree-shaking** : enlever le code non utilisé.
- **Hashing** : générer des noms de fichiers versionnés pour le cache navigateur.

En résumé : on passe de *code de dev* ➜ *artefacts statiques optimisés*.

---

## 2) Construire pour la production : `npm run build`

### 2.1 Où est définie la commande ?
Dans `package.json`, on trouve généralement :

```json
{
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview"
  }
}
```

- `npm run dev` : serveur de développement.
- `npm run build` : génère le build de production.
- `npm run preview` : sert localement le build `dist` pour validation.

> Remarque : avec Vue CLI (ancien), la commande était souvent `vue-cli-service build`. Le principe reste identique.

### 2.2 Exécuter le build

```bash
npm run build
```

Résultat attendu : création (ou mise à jour) d’un dossier de sortie, typiquement `dist/`.

### 2.3 Tester le build localement
Ne testez pas en ouvrant `dist/index.html` en `file://`, car le routage et les assets peuvent se comporter différemment.

Préférez :

```bash
npm run preview
```

- Cela sert `dist/` via un serveur HTTP local.
- Cela permet de vérifier la cohérence des chemins, du routing et des assets.

---

## 3) Comprendre le dossier de sortie (`dist`)

Après `npm run build`, vous obtenez généralement :

```
dist/
  index.html
  assets/
    index-xxxxxx.js
    index-xxxxxx.css
    logo-xxxxxx.svg
```

### 3.1 `index.html`
- Point d’entrée de l’application.
- Contient des balises `<script type="module" ...>` et `<link ...>` vers les bundles.

### 3.2 `assets/`
- Fichiers JS/CSS minifiés.
- Images et polices copiées/optimisées.
- Les noms incluent souvent un **hash** (ex: `index-a1b2c3.js`) pour permettre le **cache busting**.

### 3.3 Pourquoi les hashes sont importants
- Le navigateur peut garder un fichier en cache longtemps.
- Quand le contenu change, le **nom** change (hash différent), forçant le navigateur à télécharger la nouvelle version.

---

## 4) Paramétrage : variables d’environnement et configuration

### 4.1 Variables d’environnement

Avec Vite, les variables exposées au client doivent commencer par `VITE_`.

Exemple :

`.env.production`

```ini
VITE_API_BASE_URL=https://api.monsite.com
```

Utilisation dans le code :

```js
const baseUrl = import.meta.env.VITE_API_BASE_URL
```

### 4.2 Base URL (cas fréquent : sous-répertoire)
Déployer sur `https://monsite.com/app/` nécessite que les chemins générés pointent vers `/app/`.

Dans Vite (`vite.config.js/ts`) :

```js
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'

export default defineConfig({
  plugins: [vue()],
  base: '/app/'
})
```

> Si `base` est incorrect, vos fichiers JS/CSS peuvent être recherchés au mauvais endroit (404).

### 4.3 Mode production : implications
- Logs, source maps, optimisation…
- Selon la configuration, vous pouvez choisir d’activer/désactiver les source maps.

---

## 5) Routes et SPA : le point crucial du déploiement

### 5.1 Le problème typique
Application Vue en mode SPA + router en `history` :

- Sur `/` : OK.
- Sur `/about` après rafraîchissement : serveur répond 404.

Pourquoi ?
- Le serveur web cherche un fichier physique `/about`.
- Or c’est l’app Vue qui gère la route côté client.

### 5.2 La solution : fallback vers `index.html`
Il faut configurer le serveur pour renvoyer `index.html` pour toutes les routes non-fichier.

- Nginx : `try_files ... /index.html`
- Apache : rewrite rules
- Plateformes cloud : règles de rewrite (Netlify/Vercel, etc.)

---

## 6) Déploiement sur un serveur web (Nginx/Apache)

Les fichiers générés par `npm run build` sont **statiques** : un serveur web suffit.

### 6.1 Déploiement simple : copier `dist/`
Principe :

1. Builder en local ou CI : `npm run build`.
2. Envoyer le contenu de `dist/` vers le serveur.
3. Configurer le serveur pour servir `index.html` et `assets/`.

Sur serveur Linux, destination typique :

- `/var/www/monapp/`

### 6.2 Exemple Nginx (SPA + cache)

```nginx
server {
  listen 80;
  server_name monsite.com;

  root /var/www/monapp;
  index index.html;

  # Fallback SPA
  location / {
    try_files $uri $uri/ /index.html;
  }

  # Cache agressif pour les assets hashés
  location /assets/ {
    try_files $uri =404;
    expires 1y;
    add_header Cache-Control "public, immutable";
  }
}
```

Points clés :
- `try_files ... /index.html` : évite les 404 en refresh.
- Cache long pour `assets/` (hashés donc immuables).

### 6.3 Exemple Apache (.htaccess) pour SPA
Dans le répertoire de déploiement :

```apache
<IfModule mod_rewrite.c>
  RewriteEngine On
  RewriteBase /

  RewriteRule ^index\.html$ - [L]
  RewriteCond %{REQUEST_FILENAME} !-f
  RewriteCond %{REQUEST_FILENAME} !-d
  RewriteRule . /index.html [L]
</IfModule>
```

---

## 7) Déploiement sur une plateforme cloud/static hosting

Les plateformes “static hosting” gèrent :
- build en CI (optionnel),
- CDN,
- HTTPS,
- rollback,
- règles de rewrite.

### 7.1 Approche générique
- **Build command** : `npm run build`
- **Publish directory** : `dist`

### 7.2 Netlify (exemple)
- Commande : `npm run build`
- Dossier : `dist`

Pour le fallback SPA, ajouter `netlify.toml` :

```toml
[[redirects]]
  from = "/*"
  to = "/index.html"
  status = 200
```

### 7.3 Vercel (exemple)
En général, Vercel détecte Vite.

Fallback possible via `vercel.json` :

```json
{
  "rewrites": [{ "source": "/(.*)", "destination": "/index.html" }]
}
```

### 7.4 GitHub Pages (attention au base path)
GitHub Pages sert souvent sous :

- `https://username.github.io/mon-repo/`

Il faut généralement définir :

```js
// vite.config.js
export default defineConfig({
  base: '/mon-repo/'
})
```

---

## 8) Bonnes pratiques de production (perf, cache, sécurité)

### 8.1 Compression
Activer **gzip** ou **brotli** côté serveur/CDN améliore fortement le temps de chargement.

- Nginx : `gzip on;` (ou brotli via module)
- CDN/hosting : souvent activé par défaut

### 8.2 Cache HTTP
- HTML (`index.html`) : cache court (car il référence les nouvelles versions).
- Assets hashés : cache long (`immutable`).

Règle mentale :
- `index.html` change à chaque déploiement.
- `assets/index-<hash>.js` est immuable.

### 8.3 Source maps
- Utile pour debug et monitoring (Sentry, etc.).
- Peut exposer du code source : à décider selon le contexte.

### 8.4 Variables et secrets
- Tout ce qui est dans le build front est **public**.
- Ne mettez jamais de secrets (tokens privés, mots de passe).
- Utilisez un backend pour les opérations sensibles.

### 8.5 En-têtes de sécurité (minimum)
Selon l’app, envisagez :
- `Content-Security-Policy` (CSP)
- `X-Frame-Options`
- `Referrer-Policy`
- `Permissions-Policy`

---

## 9) CI/CD : automatiser build et déploiement

### 9.1 Pourquoi CI/CD ?
- Réduire les erreurs humaines.
- Reproductibilité.
- Déploiement plus fréquent et plus sûr.

### 9.2 Pipeline minimal (concept)
Étapes typiques :
1. Installer dépendances (`npm ci`).
2. Lancer tests/lint (optionnel mais recommandé).
3. Builder (`npm run build`).
4. Publier `dist/` (upload serveur, artifact, ou push vers hosting).

Exemple de commandes :

```bash
npm ci
npm run build
```

---

## 10) Ateliers pratiques & checklist

### Atelier 1 — Construire et prévisualiser
1. Lancer :

```bash
npm run build
npm run preview
```

2. Vérifier :
- Chargement de l’app.
- Absence de 404 sur les assets.
- Rafraîchissement sur une route interne (`/about`) sans erreur.

### Atelier 2 — Déployer sur Nginx (local ou VM)
Objectif : servir `dist/` et configurer le fallback SPA.

- Copier `dist/` vers `/var/www/monapp/`
- Appliquer la conf Nginx
- Tester :
  - `/` OK
  - `/une-route` + refresh OK

### Atelier 3 — Déploiement cloud
Objectif : configurer :
- Build command `npm run build`
- Publish directory `dist`
- Rewrite vers `/index.html`

### Checklist de déploiement (à réutiliser)
- [ ] `npm run build` réussi sans warnings critiques.
- [ ] `npm run preview` OK.
- [ ] `base` correct si sous-répertoire.
- [ ] Fallback SPA configuré.
- [ ] Cache : `index.html` court, assets hashés long.
- [ ] Compression activée.
- [ ] Variables d’environnement correctes (`VITE_`).
- [ ] Aucun secret côté frontend.
- [ ] Monitoring (optionnel) : Sentry / logs / analytics.

---

## Résumé
- Une application Vue se **construit** pour la production via `npm run build`.
- Le build produit des **fichiers statiques** (souvent dans `dist/`) déployables partout : serveur web ou plateforme cloud.
- Le point le plus critique en SPA est la prise en charge des routes (fallback vers `index.html`).
- En production, soignez cache, compression, variables d’environnement et automatisation CI/CD.
