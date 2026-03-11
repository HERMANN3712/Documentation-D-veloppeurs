# 19 — Build et déploiement (React)

## Objectifs pédagogiques
À la fin de ce module, vous serez capable de :

- Expliquer la différence entre **développement** et **production**.
- Générer un **build de production** d’une application React avec `npm run build`.
- Comprendre la structure des fichiers générés (dossier `build/` ou `dist/` selon l’outillage).
- Déployer un build statique sur :
  - un **serveur web** (Nginx/Apache)
  - un **service cloud** (Netlify, Vercel, GitHub Pages, etc.)
- Éviter les pièges courants (routes SPA, cache, variables d’environnement, base path).

---

## Pré-requis
- Node.js et npm installés
- Une application React fonctionnelle (CRA, Vite ou autre)
- Connaissances de base sur HTTP et les serveurs web

---

## 1) Développement vs Production : ce qui change

### 1.1 Mode développement
En développement, vous lancez généralement :

```bash
npm start
```

ou avec Vite :

```bash
npm run dev
```

Caractéristiques :
- Serveur de développement avec **rechargement à chaud** (HMR)
- Messages d’erreurs et warnings plus verbeux
- Code non optimisé, destiné à itérer rapidement

### 1.2 Mode production
En production, l’objectif est de livrer :
- Des fichiers **optimisés et compressés**
- Des assets **minifiés** (JS/CSS)
- Une application **stable, performante** et prête à être servie à grande échelle

Le serveur ne “compile” pas React à la volée : il sert un ensemble de fichiers statiques issus du build.

---

## 2) Construire une application React pour la production

### 2.1 La commande standard
Dans la plupart des projets React, la commande de build est :

```bash
npm run build
```

Elle exécute un script défini dans le `package.json`, généralement associé à l’outil de build.

#### Ce que fait le build (vue d’ensemble)
- Résolution des imports et bundling
- Minification du JavaScript et du CSS
- Hashing des fichiers (cache busting)
- Génération des fichiers statiques (HTML, JS, CSS, médias)

### 2.2 Où sont générés les fichiers ?
Selon votre stack :
- Create React App (CRA) : `build/`
- Vite : `dist/`

Dans ce cours, on parle de `build/` par convention, mais le principe est identique.

### 2.3 Vérifier le résultat du build
Après `npm run build`, vous devez voir un dossier comme :

```text
build/
  index.html
  static/
    css/
    js/
    media/
```

Points clés :
- `index.html` est la page d’entrée
- `static/js` contient les bundles de l’application
- Les fichiers sont souvent nommés avec un **hash** (ex: `main.3a1f2c.js`) pour gérer le cache

---

## 3) Tester localement un build de production

Il est recommandé de tester le build avec un serveur statique au lieu d’ouvrir `index.html` en double-cliquant (ce qui peut casser les routes et le chargement des assets).

### 3.1 Installer un serveur statique simple
Exemple avec `serve` :

```bash
npm i -g serve
serve -s build
```

- `-s` signifie “single page app” : toutes les routes renvoient vers `index.html`.

> Alternative : `npx serve -s build` (sans installation globale)

### 3.2 Ce qu’il faut vérifier
- L’app se charge correctement
- Les routes fonctionnent en navigation directe (ex: `/dashboard`)
- Les assets (JS/CSS/images) se chargent sans erreurs 404

---

## 4) Déployer les fichiers sur un serveur web

Une application React buildée est une **application statique** : vous pouvez la déployer sur n’importe quel serveur web capable de servir des fichiers.

### 4.1 Déploiement via copie de fichiers
Stratégie la plus simple :
1. Exécuter `npm run build`
2. Copier le contenu de `build/` sur le serveur (SCP, SFTP, rsync, pipeline CI/CD)
3. Configurer le serveur web pour servir ce dossier comme racine

Exemple de structure sur un serveur :

```text
/var/www/mon-app/
  index.html
  static/
```

### 4.2 Cas particulier des routes SPA (React Router)
Si vous utilisez des routes côté client (`react-router-dom`) alors :
- En navigation interne : OK
- En rafraîchissement (F5) sur `/profil` : le serveur cherche un fichier `/profil` et renvoie 404

Solution : configurer la **réécriture (rewrite) vers `index.html`**.

#### Exemple Nginx (conceptuel)
```nginx
location / {
  try_files $uri /index.html;
}
```

#### Exemple Apache (conceptuel via .htaccess)
```apache
RewriteEngine On
RewriteBase /
RewriteRule ^index\.html$ - [L]
RewriteCond %{REQUEST_FILENAME} !-f
RewriteCond %{REQUEST_FILENAME} !-d
RewriteRule . /index.html [L]
```

> L’idée : toute URL inconnue renvoie `index.html`, et React Router gère la route.

### 4.3 Déployer dans un sous-dossier (base path)
Si votre app est déployée dans `https://exemple.com/mon-app/`, vous devez gérer le “base path”.

- Selon l’outil (CRA/Vite), il existe un paramètre pour préfixer les chemins d’assets.
- Vérifiez que vos routes et vos assets sont compatibles avec ce sous-chemin.

---

## 5) Déployer sur un service cloud (approche moderne)

Les services cloud spécialisés (Netlify, Vercel, Cloudflare Pages, GitHub Pages, etc.) automatisent :
- Le build
- Le déploiement
- Les previews
- Le CDN

### 5.1 Principe général
1. Vous connectez le dépôt Git (GitHub/GitLab)
2. Le service exécute :
   - `npm install`
   - `npm run build`
3. Le service publie le dossier de sortie (`build/` ou `dist/`)

### 5.2 Points de configuration typiques
- **Build command** : `npm run build`
- **Publish directory** : `build` (ou `dist`)
- **Rewrite SPA** : “toutes les routes vers /index.html”

### 5.3 Avantages
- Déploiement continu (CI/CD)
- CDN mondial + HTTP/2 + compression
- Rollback et previews faciles

---

## 6) Variables d’environnement et configuration

En production, vous aurez souvent besoin de configurer :
- L’URL d’une API
- Une clé publique (ex: analytics)
- Un mode de fonctionnement

### 6.1 Attention : les variables sont “figées” au build
Pour les applications React statiques, les variables d’environnement sont généralement injectées **au moment du build**.

Conséquences :
- Changer une variable nécessite souvent de **rebuild** et **redéployer**.

### 6.2 Bonnes pratiques
- Ne jamais inclure de secrets (clé privée) dans le build front
- Utiliser un backend pour les secrets
- Centraliser la configuration et la documenter

---

## 7) Cache, versions et invalidation

Les bundles étant hashés, le navigateur peut les mettre en cache très longtemps.

### 7.1 Pourquoi c’est une bonne chose
- Gains de performance
- Réduction de bande passante

### 7.2 Piège classique
- Les utilisateurs peuvent rester sur une ancienne version du `index.html` si mal configuré.

### 7.3 Recommandations
- Configurer des en-têtes de cache :
  - `index.html` : cache court (revalidation fréquente)
  - assets hashés (`static/js/*.hash.js`) : cache long

---

## 8) Checklist de déploiement

Avant de livrer en production :

- [ ] `npm run build` passe sans erreurs
- [ ] Test local du build (`serve -s build`)
- [ ] Configuration SPA (rewrite vers `index.html`) en place
- [ ] Vérification des routes directes
- [ ] Vérification des variables d’environnement (API URL, etc.)
- [ ] Vérification du “base path” si sous-dossier
- [ ] Politique de cache correcte
- [ ] Monitoring basique (logs, erreurs, performance)

---

## 9) Atelier pratique (suggestion)

### Exercice A — Générer et tester un build
1. Lancer :
   ```bash
   npm run build
   ```
2. Servir le build :
   ```bash
   npx serve -s build
   ```
3. Tester plusieurs routes et un rafraîchissement sur une route interne.

### Exercice B — Simulation de déploiement
- Copier le dossier `build/` dans un dossier “serveur” local
- Servir via un serveur statique (ex: Nginx local ou `serve`)
- Mettre en place un rewrite SPA (selon l’outil serveur)

---

## Résumé
- Une application React est construite pour la production avec :
  ```bash
  npm run build
  ```
- Les fichiers générés (ex: dossier `build/`) sont des fichiers statiques : HTML/CSS/JS.
- Ils peuvent être déployés sur un **serveur web** ou un **service cloud**.
- Pour les SPA, il faut gérer les **rewrites** vers `index.html`.
- Pensez au cache, aux variables d’environnement et aux chemins de base.
