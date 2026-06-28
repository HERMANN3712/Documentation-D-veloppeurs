# Formation Node.js (et React) — Déploiement

## Objectifs pédagogiques
À la fin de cette formation, vous saurez :

- Préparer une application Node.js (et éventuellement un front React) pour la production.
- Déployer sur un serveur Linux (VM/VPS) avec **Nginx + systemd** ou **PM2**.
- Déployer sur des plateformes cloud courantes (approche générique et exemples).
- Conteneuriser et déployer avec **Docker** (et Docker Compose).
- Mettre en place une observabilité minimale (logs, healthchecks) et des bonnes pratiques de sécurité.

## Pré-requis

- Connaissances de base en Node.js, npm, scripts, variables d’environnement.
- Connaissances de base Linux (shell, SSH, fichiers, permissions).
- Optionnel : notions de réseaux (DNS, ports, reverse proxy).

## Durée suggérée

- 1 journée (7h) ou 2 demi-journées.

## Matériel

- Une application Node.js (API Express/Nest/Fastify…) et éventuellement un front React.
- Un serveur Linux (Ubuntu/Debian) accessible en SSH **ou** un environnement local Docker.
- Un nom de domaine (optionnel mais recommandé pour HTTPS).

---

# 1) Comprendre les environnements de déploiement

## 1.1 Développement vs Production

**Développement** :
- Rechargement à chaud (nodemon, vite).
- Logs verbeux.
- Dépendances de dev.

**Production** :
- Process stable (redémarrage auto, uptime).
- Logs structurés.
- Configuration via variables d’environnement.
- Performance et sécurité.

## 1.2 Stratégies de déploiement courantes

1. **Serveur Linux** (classique) :
   - Contrôle total.
   - Maintenance, sécurité, mises à jour.
2. **Plateformes Cloud / PaaS** :
   - Déploiement simplifié.
   - Autoscaling possible.
   - Moins de contrôle sur l’OS.
3. **Docker** :
   - Reproductibilité.
   - Portabilité (local → CI → prod).

---

# 2) Préparer une application Node.js pour la production

## 2.1 Structure et scripts npm

Exemple standard :

```json
{
  "scripts": {
    "dev": "nodemon src/index.js",
    "build": "tsc -p tsconfig.json",
    "start": "node dist/index.js",
    "lint": "eslint ."
  }
}
```

Bonnes pratiques :
- `start` doit lancer **le serveur en mode production**.
- `build` doit produire des artefacts (ex: `dist/`).

## 2.2 Variables d’environnement

- Utiliser `process.env` (et `.env` seulement en local).
- En production, injecter via systemd/PM2/Docker/plateforme.

Exemples de variables :

- `NODE_ENV=production`
- `PORT=3000`
- `DATABASE_URL=...`

**⚠️ Ne jamais committer** les secrets.

## 2.3 Healthcheck et readiness

Ajoutez un endpoint de santé :

```js
app.get('/health', (req, res) => {
  res.status(200).json({ ok: true, uptime: process.uptime() })
})
```

Optionnel : vérifiez la connexion DB avant de répondre `ok`.

## 2.4 Logs et gestion des erreurs

- Loguer en JSON (pino) ou format clair.
- Capturer les erreurs globales :

```js
process.on('unhandledRejection', (err) => {
  console.error('unhandledRejection', err)
  process.exit(1)
})

process.on('uncaughtException', (err) => {
  console.error('uncaughtException', err)
  process.exit(1)
})
```

---

# 3) Déployer sur un serveur Linux (approche recommandée)

Objectif : appliquer une méthode “robuste” et standard :

- Application Node.js écoute sur `127.0.0.1:3000`.
- **Nginx** expose `https://votre-domaine` et reverse-proxy vers Node.
- Process géré par **systemd** (ou PM2).

## 3.1 Préparer le serveur

### Connexion SSH

```bash
ssh ubuntu@IP_DU_SERVEUR
```

### Installer Node.js

Sur Ubuntu/Debian, privilégier NodeSource ou `nvm`.

Exemple NodeSource (adapter la version) :

```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs
node -v
npm -v
```

### Créer un utilisateur applicatif

```bash
sudo adduser nodeapp
sudo usermod -aG sudo nodeapp
```

## 3.2 Déployer le code (Git)

```bash
sudo -iu nodeapp
mkdir -p /home/nodeapp/apps
cd /home/nodeapp/apps
git clone https://github.com/votre-org/votre-repo.git
cd votre-repo
```

Installer les dépendances :

```bash
npm ci --omit=dev
```

Si build nécessaire (TS, bundling) :

```bash
npm run build
```

## 3.3 Configuration via `.env` (selon votre politique)

Option 1 : gérer via systemd/PM2 (recommandé)
Option 2 : fichier `.env` + `dotenv` (acceptable si permissions strictes)

Exemple :

```bash
nano .env
```

```env
NODE_ENV=production
PORT=3000
DATABASE_URL=postgres://...
```

**Sécurisez** :

```bash
chmod 600 .env
```

## 3.4 Exposer l’application derrière Nginx

Installer Nginx :

```bash
sudo apt-get update
sudo apt-get install -y nginx
```

Créer une conf Nginx :

```bash
sudo nano /etc/nginx/sites-available/votre-app
```

Exemple :

```nginx
server {
  listen 80;
  server_name votre-domaine.tld;

  location / {
    proxy_pass http://127.0.0.1:3000;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
    proxy_cache_bypass $http_upgrade;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
  }
}
```

Activer :

```bash
sudo ln -s /etc/nginx/sites-available/votre-app /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

## 3.5 HTTPS avec Let’s Encrypt

```bash
sudo apt-get install -y certbot python3-certbot-nginx
sudo certbot --nginx -d votre-domaine.tld
```

---

# 4) Gérer le process Node.js en production

Deux options principales : **systemd** ou **PM2**.

## 4.1 Option A — systemd (très standard)

Créez un service :

```bash
sudo nano /etc/systemd/system/votre-app.service
```

Exemple :

```ini
[Unit]
Description=Votre API Node.js
After=network.target

[Service]
Type=simple
User=nodeapp
WorkingDirectory=/home/nodeapp/apps/votre-repo
Environment=NODE_ENV=production
Environment=PORT=3000
# Si vous utilisez un fichier env:
EnvironmentFile=/home/nodeapp/apps/votre-repo/.env
ExecStart=/usr/bin/node dist/index.js
Restart=always
RestartSec=3

# Sécurité basique
NoNewPrivileges=true
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

Activer et démarrer :

```bash
sudo systemctl daemon-reload
sudo systemctl enable votre-app
sudo systemctl start votre-app
sudo systemctl status votre-app
```

Voir les logs :

```bash
journalctl -u votre-app -f
```

## 4.2 Option B — PM2 (gestionnaire de process Node)

**PM2** est utile si vous voulez :
- Redémarrage automatique.
- Mode cluster (multi-process).
- Gestion de logs.
- Démarrage au boot.

### Installer PM2

```bash
sudo npm i -g pm2
pm2 -v
```

### Démarrer l’app

```bash
cd /home/nodeapp/apps/votre-repo
pm2 start dist/index.js --name "votre-app" --env production
pm2 ls
pm2 logs votre-app
```

### Mode cluster

```bash
pm2 start dist/index.js -i max --name "votre-app" --env production
```

### Fichier ecosystem (recommandé)

Créez `ecosystem.config.cjs` :

```js
module.exports = {
  apps: [
    {
      name: 'votre-app',
      script: 'dist/index.js',
      instances: 'max',
      exec_mode: 'cluster',
      env: {
        NODE_ENV: 'production',
        PORT: 3000
      }
    }
  ]
}
```

Puis :

```bash
pm2 start ecosystem.config.cjs
pm2 save
```

### Démarrage au reboot

```bash
pm2 startup
# exécuter la commande affichée
pm2 save
```

---

# 5) Déploiement via Docker

Docker permet de livrer l’application avec son environnement.

## 5.1 Dockerfile (Node.js)

### Exemple simple

```Dockerfile
FROM node:20-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci --omit=dev

COPY . .

ENV NODE_ENV=production
EXPOSE 3000

CMD ["node", "dist/index.js"]
```

### Variante multi-stage (recommandée si build)

```Dockerfile
# Build
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Runtime
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY --from=build /app/dist ./dist
ENV NODE_ENV=production
EXPOSE 3000
CMD ["node", "dist/index.js"]
```

## 5.2 Build et run

```bash
docker build -t votre-app:1.0.0 .
docker run -p 3000:3000 --env-file .env votre-app:1.0.0
```

## 5.3 Docker Compose (API + DB)

`docker-compose.yml` :

```yaml
services:
  api:
    build: .
    ports:
      - "3000:3000"
    env_file:
      - .env
    depends_on:
      - db

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: example
      POSTGRES_USER: example
      POSTGRES_DB: example
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

Démarrer :

```bash
docker compose up -d --build
```

## 5.4 Reverse proxy et TLS en Docker

Approche courante :
- Nginx/Traefik en front.
- Certificats Let’s Encrypt automatisés.

(À adapter selon votre infrastructure.)

---

# 6) Déployer une application React (rappels utiles)

Deux approches :

1. **Build statique** (`npm run build`) servi par Nginx, CDN, ou plateforme statique.
2. **SSR/Next.js** : déploiement Node similaire à une API.

## 6.1 React SPA (Vite/CRA) → Nginx

- Construire :

```bash
npm run build
```

- Déployer le dossier `dist/` (Vite) ou `build/` (CRA) dans `/var/www/app`.
- Config Nginx avec fallback SPA :

```nginx
location / {
  try_files $uri /index.html;
}
```

---

# 7) CI/CD (pipeline minimal)

Objectif : automatiser tests + build + déploiement.

## 7.1 Étapes typiques

- **Install** (npm ci)
- **Lint/Test**
- **Build**
- **Package** (artefact ou image Docker)
- **Deploy** (SSH, registry, plateforme)

## 7.2 Déploiement via SSH (concept)

- Push sur main → pipeline → SSH sur serveur → pull repo → install/build → restart service.

Bonnes pratiques :
- clé SSH dédiée CI
- droits minimaux
- déploiement atomique (dossier releases) si nécessaire

---

# 8) Sécurité et bonnes pratiques

## 8.1 Ports et firewall

- Exposer uniquement 80/443.
- Node écoute sur `127.0.0.1` (derrière Nginx).

## 8.2 Secrets

- Utiliser un Secret Manager (cloud) ou variables injectées.
- Rotation des secrets.

## 8.3 Mises à jour et durcissement

- Mises à jour OS régulières.
- Éviter root pour lancer l’app.
- Permissions strictes.

---

# 9) Atelier guidé (proposition)

## Exercice A — Déploiement Linux + Nginx + systemd

1. Provisionner un VPS.
2. Installer Node + Nginx.
3. Déployer l’API.
4. Créer le service systemd.
5. Ajouter un domaine + HTTPS.

**Validation** :
- `curl https://votre-domaine/health` renvoie `{ ok: true }`.
- `systemctl status` OK.

## Exercice B — Déploiement Docker + Compose

1. Écrire un Dockerfile multi-stage.
2. Créer un `docker-compose.yml` avec DB.
3. Démarrer, tester, puis simuler un redémarrage.

---

# 10) Checklist de déploiement (à réutiliser)

- [ ] Variables d’env en place, secrets sécurisés
- [ ] `NODE_ENV=production`
- [ ] Healthcheck `/health`
- [ ] Reverse proxy + HTTPS
- [ ] Process manager (systemd ou PM2)
- [ ] Logs consultables (`journalctl` ou `pm2 logs`)
- [ ] Redémarrage auto après reboot
- [ ] Sauvegardes DB (si applicable)
- [ ] Monitoring minimal (CPU/RAM/uptime)

---

## Annexes — Commandes utiles

### PM2

```bash
pm2 restart votre-app
pm2 reload votre-app
pm2 monit
pm2 describe votre-app
```

### systemd

```bash
sudo systemctl restart votre-app
sudo systemctl status votre-app
journalctl -u votre-app -n 200 --no-pager
```

### Nginx

```bash
sudo nginx -t
sudo systemctl reload nginx
sudo tail -f /var/log/nginx/access.log
sudo tail -f /var/log/nginx/error.log
```
