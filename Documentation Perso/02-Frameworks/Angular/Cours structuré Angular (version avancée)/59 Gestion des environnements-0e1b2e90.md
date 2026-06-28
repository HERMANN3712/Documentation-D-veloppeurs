# Formation Angular — Gestion des environnements

## Informations générales
- **Public** : développeurs Angular (niveau débutant à intermédiaire)
- **Pré‑requis** : TypeScript, bases Angular (CLI, modules/standalone, services), notions HTTP
- **Durée suggérée** : 1 journée (6–7h) ou 2 demi‑journées
- **Objectifs pédagogiques** :
  1. Comprendre comment Angular gère les environnements (dev/recette/prod).
  2. Maîtriser la configuration **build‑time** (remplacements de fichiers, options de build).
  3. Mettre en place une configuration **runtime** (chargée au démarrage) pour éviter de reconstruire.
  4. Implémenter une stratégie avancée **sans secrets** côté frontend.
  5. Savoir diagnostiquer et sécuriser la chaîne CI/CD (build, déploiement, variables).

---

## Plan de la formation
1. **Concepts clés : environnements, build‑time vs runtime, secrets**
2. **Mécanisme Angular classique : `environment.ts` + `fileReplacements`**
3. **Configurations Angular CLI : `angular.json`, `build`, `serve`, `configurations`**
4. **Bonnes pratiques : typage, structure, validations, tests**
5. **Stratégie avancée : configuration runtime (JSON) + `APP_INITIALIZER`**
6. **Sécurité : pourquoi les secrets ne doivent jamais être dans l’environnement frontend**
7. **Recette & prod : scénarios CI/CD et déploiements multi‑cibles**
8. **Atelier guidé : mise en place complète (build‑time + runtime) + check‑list**

---

# 1) Concepts clés

## 1.1 Qu’appelle‑t‑on “environnement” ?
Un **environnement** est un contexte d’exécution distinct, généralement :
- **Développement (dev)** : features en cours, logs verbeux, endpoints locaux.
- **Recette / staging / pré‑prod** : validation avant prod, données anonymisées, monitoring activé.
- **Production (prod)** : performances, sécurité, observabilité.

Dans une application Angular, “changer d’environnement” revient à changer **certaines valeurs de configuration** :
- URL d’API (`apiBaseUrl`)
- activation/désactivation de features (feature flags)
- niveau de logs
- identifiants de services externes **non sensibles** (ex: DSN Sentry public, ID Analytics)

## 1.2 Build‑time vs Runtime
### Build‑time
La configuration est décidée **au moment du build**. Exemple :
- `ng build -c production`
- remplacement de `environment.ts` par `environment.prod.ts`

**Avantages**
- Très simple à intégrer
- Optimisations (tree‑shaking, dead code elimination) possibles si la config pilote des branches

**Inconvénients**
- Il faut **rebuild** pour chaque environnement
- Risque de multiplier les artefacts

### Runtime
La configuration est chargée **au démarrage** (ex: `config.json` servie par le serveur). On build une seule fois et on déploie le même artefact partout.

**Avantages**
- Un seul build pour dev/recette/prod
- Paramétrage côté infra (Kubernetes, Nginx, CDN…)

**Inconvénients**
- Mise en place plus avancée
- Certaines optimisations liées à la config statique ne s’appliquent plus

## 1.3 Point crucial : les secrets
**Règle d’or** : tout ce qui est dans le frontend est **public**.
- `environment.ts` est inclus dans le bundle JavaScript.
- Un attaquant peut lire le code, les sources map, ou observer les requêtes.

Donc on ne met **jamais** :
- mots de passe, clés privées, tokens avec privilèges
- chaînes de connexion internes
- secrets d’API donnant accès à des ressources sensibles

À la place :
- utiliser un **backend** comme “proxy” sécurisé
- utiliser OAuth/OIDC avec token utilisateur
- limiter les clés “publiques” (ex: API key Google Maps restreinte par domaine)

---

# 2) Le mécanisme Angular classique : `environment.ts`

## 2.1 Structure standard
Dans un projet Angular CLI, on trouve souvent :

```
src/environments/environment.ts
src/environments/environment.prod.ts
```

Exemple :

```ts
// src/environments/environment.ts
export const environment = {
  production: false,
  apiBaseUrl: 'http://localhost:3000',
  enableDebugTools: true,
};
```

```ts
// src/environments/environment.prod.ts
export const environment = {
  production: true,
  apiBaseUrl: 'https://api.example.com',
  enableDebugTools: false,
};
```

Utilisation :

```ts
import { environment } from '../environments/environment';

console.log('API:', environment.apiBaseUrl);
```

## 2.2 Ajouter un environnement “recette”
Créer :

```ts
// src/environments/environment.staging.ts
export const environment = {
  production: false,
  apiBaseUrl: 'https://api-staging.example.com',
  enableDebugTools: false,
};
```

Puis configurer le remplacement dans `angular.json`.

---

# 3) Angular CLI : `angular.json` et `configurations`

## 3.1 Comprendre `build` vs `serve`
- `build` produit un dossier `dist/...` prêt à déployer
- `serve` lance un serveur de dev (souvent en s’appuyant sur une configuration de build)

## 3.2 Exemple de configuration avec `fileReplacements`
Dans `angular.json` (extrait typique) :

```jsonc
{
  "projects": {
    "my-app": {
      "architect": {
        "build": {
          "configurations": {
            "production": {
              "fileReplacements": [
                {
                  "replace": "src/environments/environment.ts",
                  "with": "src/environments/environment.prod.ts"
                }
              ],
              "optimization": true,
              "outputHashing": "all",
              "sourceMap": false
            },
            "staging": {
              "fileReplacements": [
                {
                  "replace": "src/environments/environment.ts",
                  "with": "src/environments/environment.staging.ts"
                }
              ],
              "optimization": true,
              "sourceMap": true
            }
          }
        },
        "serve": {
          "configurations": {
            "production": {
              "buildTarget": "my-app:build:production"
            },
            "staging": {
              "buildTarget": "my-app:build:staging"
            }
          }
        }
      }
    }
  }
}
```

Commandes :
- `ng serve` (dev par défaut)
- `ng serve -c staging`
- `ng build -c production`

## 3.3 Bonnes pratiques de configurations CLI
- Nommer clairement : `development`, `staging`, `production`
- Éviter de dupliquer trop d’options : partir d’une base commune
- Verrouiller `sourceMap` en prod (ou gérer l’upload sécurisé des sourcemaps via CI)

---

# 4) Structuration et typage de la config

## 4.1 Créer une interface (ou type) de configuration
Evite les fautes de frappe, facilite l’auto‑complétion.

```ts
// src/app/core/config/app-config.model.ts
export interface AppConfig {
  production: boolean;
  apiBaseUrl: string;
  enableDebugTools: boolean;
  sentryDsn?: string; // valeur publique possible
}
```

Puis :

```ts
// src/environments/environment.ts
import { AppConfig } from '../app/core/config/app-config.model';

export const environment: AppConfig = {
  production: false,
  apiBaseUrl: 'http://localhost:3000',
  enableDebugTools: true,
};
```

## 4.2 Validation minimale au runtime (optionnel)
Même en build‑time, il est utile de vérifier des valeurs à l’exécution.

```ts
export function assertConfig(c: AppConfig): void {
  if (!c.apiBaseUrl) throw new Error('apiBaseUrl manquant');
}
```

## 4.3 Centraliser l’accès à la config
Créer un service `ConfigService` (même si la source est build‑time) pour limiter le couplage.

---

# 5) Stratégie avancée : configuration runtime

## 5.1 Objectif
- **Construire une fois** l’application (un seul bundle)
- **Paramétrer** l’URL d’API, flags, etc. **au déploiement**
- Garder les secrets hors du frontend

## 5.2 Approche recommandée : fichier `config.json` servi par le serveur
### Arborescence
- `src/assets/config/config.json` (valeurs par défaut dev)
- en recette/prod, le serveur déploie **son** `config.json` dans `assets/config/` (ou via un endpoint `/config`)

Exemple `config.json` :

```json
{
  "apiBaseUrl": "https://api.example.com",
  "enableDebugTools": false,
  "sentryDsn": "https://public@sentry.io/123"
}
```

## 5.3 Charger la config au démarrage avec `APP_INITIALIZER`
### Service de configuration

```ts
// src/app/core/config/config.service.ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { firstValueFrom } from 'rxjs';

export interface RuntimeConfig {
  apiBaseUrl: string;
  enableDebugTools: boolean;
  sentryDsn?: string;
}

@Injectable({ providedIn: 'root' })
export class ConfigService {
  private config!: RuntimeConfig;

  constructor(private http: HttpClient) {}

  async load(): Promise<void> {
    // chemin versionné possible : /assets/config/config.json?v=...
    this.config = await firstValueFrom(
      this.http.get<RuntimeConfig>('/assets/config/config.json')
    );
  }

  get(): RuntimeConfig {
    if (!this.config) {
      throw new Error('Config non chargée. Vérifiez APP_INITIALIZER.');
    }
    return this.config;
  }
}
```

### Enregistrement `APP_INITIALIZER`

#### Avec des providers (Angular standalone)

```ts
// src/app/app.config.ts
import { APP_INITIALIZER } from '@angular/core';
import { provideHttpClient } from '@angular/common/http';
import { ConfigService } from './core/config/config.service';

export function initConfig(config: ConfigService) {
  return () => config.load();
}

export const appConfig = {
  providers: [
    provideHttpClient(),
    {
      provide: APP_INITIALIZER,
      useFactory: initConfig,
      deps: [ConfigService],
      multi: true,
    },
  ],
};
```

#### Avec `AppModule` (approche module)

```ts
providers: [
  {
    provide: APP_INITIALIZER,
    useFactory: (config: ConfigService) => () => config.load(),
    deps: [ConfigService],
    multi: true
  }
]
```

## 5.4 Utiliser la config runtime dans un service API

```ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { ConfigService } from '../config/config.service';

@Injectable({ providedIn: 'root' })
export class ApiClient {
  constructor(
    private http: HttpClient,
    private configService: ConfigService
  ) {}

  getUsers() {
    const { apiBaseUrl } = this.configService.get();
    return this.http.get(`${apiBaseUrl}/users`);
  }
}
```

## 5.5 Variante : runtime via `window.__env` (injection globale)
Autre option : générer un `env.js` servi par le serveur :

```js
// env.js
window.__env = {
  apiBaseUrl: 'https://api.example.com'
};
```

Avantage : pas d’appel HTTP initial, mais il faut gérer le chargement `<script src="/env.js"></script>` avant Angular.

---

# 6) Sécurité : éviter d’exposer des secrets

## 6.1 Pourquoi un secret front n’est pas un secret
- Le bundle est téléchargé par le navigateur
- Les utilisateurs peuvent inspecter le JS, les sources map, le stockage
- Toute valeur statique est récupérable

## 6.2 Solutions correctes
- **Backend** : stocker les secrets et effectuer les appels sensibles côté serveur
- **OAuth/OIDC** : utiliser un token utilisateur, scopes minimaux
- **Clés publiques restreintes** : si un fournisseur impose une API key côté client, restreindre par domaine, quotas, endpoints

## 6.3 Check‑list sécurité config
- [ ] Aucun secret dans `environment.*.ts`
- [ ] Aucun secret dans `config.json` runtime
- [ ] En prod, `sourceMap` géré explicitement (désactivé ou upload contrôlé)
- [ ] En-têtes sécurité sur le serveur (CSP, HSTS selon contexte)

---

# 7) CI/CD : stratégies de build et déploiement

## 7.1 Stratégie 1 — Build par environnement
- CI exécute : `ng build -c staging`, `ng build -c production`
- Déploie les artefacts distincts

**Pro** : simple, proche du standard
**Con** : multi-build, risque d’erreur de version

## 7.2 Stratégie 2 — Build unique + config runtime
- CI produit un seul build `ng build -c production` (ou une configuration optimisée)
- CD déploie le même `dist/` partout
- Le serveur injecte `config.json` spécifique à l’environnement

**Pro** : cohérence, rapidité
**Con** : nécessite discipline et bootstrapping runtime

## 7.3 Versionnage et cache
- Les fichiers Angular ont souvent `outputHashing: all` => cache long
- `config.json` doit être **non caché** ou faiblement caché (sinon changements non pris en compte)

Exemple Nginx (idée) :
- `assets/**` hashés : cache long
- `assets/config/config.json` : `Cache-Control: no-cache`

---

# 8) Atelier guidé (pas à pas)

## 8.1 Étape A — Ajouter une recette build‑time
1. Créer `environment.staging.ts`
2. Ajouter `fileReplacements` dans `angular.json`
3. Tester : `ng serve -c staging` puis vérifier `apiBaseUrl`

## 8.2 Étape B — Introduire un `ConfigService` et `APP_INITIALIZER`
1. Créer `ConfigService` (runtime)
2. Enregistrer `APP_INITIALIZER`
3. Créer `assets/config/config.json`
4. Remplacer les usages directs de `environment` dans les services critiques

## 8.3 Étape C — Déploiement type
- Build unique
- Déployer `dist/`
- Déployer/monter `config.json` par environnement

## 8.4 Points d’attention
- Gérer les erreurs de chargement config (fallback, page d’erreur)
- Ne pas bloquer longtemps le bootstrap (timeout)
- Garder la config minimale : ne pas transformer le runtime config en “backend bis”

---

# Annexes

## A) Exemples de clés de config acceptables côté frontend
- URL API publique (qui nécessite une auth utilisateur)
- DSN Sentry (public)
- Identifiant d’application (non secret)
- Feature flags non sensibles

## B) Anti‑patterns fréquents
- Mettre un token admin dans `environment.prod.ts`
- Croire qu’un repo privé suffit (bundle accessible)
- Confondre “clé publique” et “secret”

## C) Mini quiz de fin
1. Pourquoi `environment.ts` ne doit jamais contenir un mot de passe ?
2. Différence entre config build‑time et runtime ?
3. Dans quel cas choisir une configuration runtime ?

---

## Résultat attendu en fin de formation
À l’issue, vous saurez :
- configurer dev/recette/prod avec Angular CLI
- typer et valider votre configuration
- mettre en place un chargement runtime via `APP_INITIALIZER`
- construire une stratégie réaliste et sécurisée sans secrets côté frontend
