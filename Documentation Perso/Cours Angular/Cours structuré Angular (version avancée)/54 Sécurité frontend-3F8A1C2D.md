# Formation Angular (Frontend)
## 54 — Sécurité frontend

> **Objectif** : comprendre et appliquer les bonnes pratiques de sécurité côté frontend avec Angular : prévention du XSS, rôle du *sanitizer*, prudence avec le HTML dynamique, gestion des tokens, protection des routes, sécurisation des échanges HTTP. 
>
> **Rappel essentiel** : **le frontend ne remplace jamais les contrôles de sécurité backend**. Toute règle de sécurité critique doit être validée côté serveur.

---

## Public visé
Développeurs Angular (niveau débutant à confirmé) souhaitant sécuriser leurs applications SPA.

## Prérequis
- Connaissances de base Angular (components, templates, services, routing)
- Notions HTTP/REST
- Compréhension des bases Web (DOM, cookies, localStorage)

## Durée suggérée
1 journée (6–7h) ou 2 demi-journées.

## Matériel
- Node.js + Angular CLI
- Un projet Angular de démonstration
- Un backend de test (mock API, JSON Server, ou API maison)

---

# Plan pédagogique

1. **Menaces et modèle de sécurité d’une SPA**
2. **XSS (Cross‑Site Scripting) et protections Angular**
3. **Sanitization : comprendre le *DomSanitizer***
4. **HTML dynamique : patterns sûrs / patterns dangereux**
5. **Gestion d’authentification et de tokens (JWT/OAuth2/OIDC)**
6. **Protection des routes (guards, résolveurs, checks côté serveur)**
7. **Sécuriser les échanges HTTP (interceptors, headers, CSRF, CORS, TLS)**
8. **Bonnes pratiques transverses + checklists**
9. **Ateliers / exercices**

---

# 1) Menaces et modèle de sécurité d’une SPA

## 1.1 Frontend vs Backend : responsabilités
Le frontend :
- **Affiche** des informations
- **Applique** des règles UX et des validations (confort utilisateur)
- **Stocke temporairement** des états
- **Orchestre** des appels API

Le backend :
- **Authentifie** et **autorise**
- **Valide** les données *de manière définitive*
- **Contrôle** les accès aux ressources
- **Journalise** et **détecte**

> Tout ce qui est sur le frontend peut être contourné (DevTools, proxy, script). Les contrôles de sécurité critiques (ACL, validation business, ownership des ressources) doivent vivre **côté serveur**.

## 1.2 Menaces fréquentes sur les apps Angular
- **XSS** (injection de script via HTML, URL, interpolation, bibliothèques…)
- **Vol de token** (localStorage exposé au XSS)
- **CSRF** (si authentification par cookies)
- **Fuite de données** (cache navigateur, logs, erreurs)
- **Open redirect / navigation non contrôlée**
- **Dépendances compromises** (supply chain)

## 1.3 Vocabulaire
- **Asset** : donnée ou ressource à protéger (token, PII, compte)
- **Threat** : menace
- **Attack surface** : surface d’attaque (toutes les entrées possibles)
- **Mitigation** : stratégie de réduction du risque

---

# 2) XSS : comprendre et prévenir

## 2.1 Rappels XSS
Le XSS arrive quand une entrée non fiable est injectée dans le DOM en tant que code exécutable.

### Types courants
- **Reflected XSS** : via URL / paramètres
- **Stored XSS** : persisté en base (commentaires, profils)
- **DOM-based XSS** : manipulation du DOM côté client

### Impacts
- Vol de session / token
- Défiguration
- Actions à la place de l’utilisateur
- Exfiltration de données

## 2.2 Protections natives d’Angular
Angular protège le template par défaut via :
- **Escaping automatique** en interpolation : `{{ userInput }}` est affiché en texte.
- **Sanitization** sur certaines propriétés sensibles (URL, styles, HTML) avant insertion.

### Exemple : interpolation (sûr)
```html
<p>{{ comment }}</p>
```
Même si `comment = '<img src=x onerror=alert(1)>'`, Angular l’affiche comme texte.

### Exemple : binding HTML (nécessite vigilance)
```html
<div [innerHTML]="htmlFromApi"></div>
```
Angular va **sanitizer** le HTML, mais :
- vous devez comprendre ce qui est autorisé ou supprimé
- et éviter d’utiliser des contournements

## 2.3 Endroits à risque dans Angular
- `[innerHTML]`
- `bypassSecurityTrust...` (si mal utilisé)
- `[href]` / `[src]` / `[style]` / `background-image`
- Génération de templates/DOM via libs externes
- Affichage de Markdown/HTML rendu (WYSIWYG)

---

# 3) Le Sanitizer Angular (DomSanitizer)

## 3.1 Rôle
`DomSanitizer` sert à empêcher l’injection de valeurs dangereuses dans des contextes sensibles.

### Contextes pris en charge (conceptuellement)
- **HTML**
- **Style**
- **URL**
- **Resource URL** (ex. iframe src)

Angular applique automatiquement une sanitization à l’exécution selon le type de binding.

## 3.2 Ce que fait la sanitization HTML
Sur `[innerHTML]`, Angular supprime typiquement :
- Scripts (`<script>`)
- Attributs d’événements (`onclick`, `onerror`, …)
- Urls dangereuses (`javascript:`)

## 3.3 Les méthodes `bypassSecurityTrust...` (à éviter par défaut)
Angular propose :
- `bypassSecurityTrustHtml`
- `bypassSecurityTrustStyle`
- `bypassSecurityTrustUrl`
- `bypassSecurityTrustResourceUrl`

> Elles **désactivent** la protection pour la valeur fournie. À utiliser uniquement si vous êtes *certain* de la provenance et si vous maîtrisez le contenu.

### Exemple risqué
```ts
constructor(private sanitizer: DomSanitizer) {}

trusted = this.sanitizer.bypassSecurityTrustHtml(this.htmlFromApi);
```
Si `htmlFromApi` est manipulable par un attaquant, vous venez d’introduire un XSS.

## 3.4 Stratégie recommandée
1. **Ne pas accepter** du HTML arbitraire si non nécessaire.
2. Si besoin d’un rendu riche :
   - préférer un **format contrôlé** (ex. Markdown) + rendu via un parser configuré en mode sécurisé
   - appliquer une **sanitization côté serveur** (défense en profondeur)
   - conserver la sanitization Angular, **ne pas bypass**

---

# 4) HTML dynamique : usage prudent

## 4.1 Patterns sûrs
### 4.1.1 Afficher du texte
Toujours privilégier l’interpolation :
```html
<p>{{ value }}</p>
```

### 4.1.2 Binder des attributs via Angular (sans concat)
```html
<a [href]="safeLink">Lien</a>
<img [src]="avatarUrl" />
```
Éviter les concaténations de chaînes non maîtrisées dans le template.

### 4.1.3 Utiliser des composants contrôlés
Au lieu d’injecter des bouts de HTML, construire une UI via composants :
- liste d’items
- rendu conditionnel
- pipe de formatage

## 4.2 Patterns dangereux
### 4.2.1 `innerHTML` pour tout
```html
<div [innerHTML]="content"></div>
```
Surtout si `content` vient d’une API/modèle éditable.

### 4.2.2 Générer du HTML avec `string` + assignation DOM
Éviter `ElementRef.nativeElement.innerHTML = ...`.

### 4.2.3 Utiliser `eval` / `new Function`
En Angular moderne, ce n’est quasiment jamais justifié.

## 4.3 Cas particulier : afficher du Markdown
Approche recommandée :
- parser Markdown → HTML via une lib réputée
- configurer une **sanitization stricte** (ou utiliser le sanitizer d’Angular)
- bloquer les liens `javascript:`

Checklist :
- interdire les attributs `on...`
- limiter les tags autorisés
- valider les URLs (http/https)

---

# 5) Gestion des tokens et authentification

## 5.1 Options usuelles
- **OAuth2/OIDC** (recommandé) via un fournisseur d’identité
- **JWT** (souvent utilisé)
- **Sessions cookies** (classique)

## 5.2 Où stocker un token ?
### localStorage / sessionStorage
✅ simple
❌ exposé au XSS (un XSS = vol de token)

### Cookie `HttpOnly` + `Secure`
✅ non accessible en JS (réduit l’impact XSS sur le vol)
✅ envoyé automatiquement
❌ expose à **CSRF** si pas de protection

> Choix guidé par votre architecture : 
> - Si vous pouvez utiliser des cookies HttpOnly, c’est souvent plus robuste contre le vol de token via XSS.
> - Si token en storage, alors l’effort anti-XSS doit être maximal et vous devez prévoir une stratégie de rotation/expiration.

## 5.3 Expiration, rotation et révocation
Bonnes pratiques :
- **Access token** court (minutes)
- **Refresh token** plus long, stocké de manière plus sûre
- Rotation du refresh pour limiter le replay
- Possibilité de révocation côté serveur

## 5.4 Ne jamais mettre des secrets dans le frontend
- Clés privées, secrets OAuth, clés API sensibles : **interdit**
- Tout code frontend peut être lu

## 5.5 Interceptor HTTP pour joindre le token (si usage Bearer)
```ts
import { HttpInterceptorFn } from '@angular/common/http';

export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const token = localStorage.getItem('access_token');
  if (!token) return next(req);

  return next(req.clone({
    setHeaders: { Authorization: `Bearer ${token}` }
  }));
};
```
Points d’attention :
- ne pas attacher le token sur des domaines tiers
- gérer proprement 401/403 (refresh, logout)

---

# 6) Protection des routes (Angular Router)

## 6.1 Objectif
Empêcher l’accès à des écrans si l’utilisateur ne remplit pas les critères.

> Attention : c’est une **barrière UX**. Les données doivent être protégées par l’API.

## 6.2 Guards
### Auth guard (exemple)
```ts
import { CanActivateFn, Router } from '@angular/router';
import { inject } from '@angular/core';

export const authGuard: CanActivateFn = () => {
  const router = inject(Router);
  const isLoggedIn = !!localStorage.getItem('access_token');

  if (!isLoggedIn) {
    return router.parseUrl('/login');
  }
  return true;
};
```

### Guard rôle/permission
- vérifier une *claim* dans le token (ex. `roles`)
- mais surtout : **l’API doit vérifier les droits** à chaque requête.

## 6.3 Route data et RBAC
Exemple :
```ts
{
  path: 'admin',
  canActivate: [adminGuard],
  data: { roles: ['ADMIN'] },
  loadComponent: () => import('./admin.component').then(m => m.AdminComponent)
}
```

## 6.4 Bonnes pratiques
- Toujours gérer le cas des routes deep link (URL directe)
- Rediriger vers login avec `returnUrl`
- Ne pas exposer d’infos sensibles dans le routing (ex. IDs séquentiels + pas de contrôle serveur)

---

# 7) Sécuriser les échanges HTTP

## 7.1 HTTPS/TLS obligatoire
- Toujours forcer HTTPS en production
- HSTS côté serveur

## 7.2 Interceptors pour sécurité et robustesse
- Ajout d’en-têtes applicatifs (si requis)
- Gestion centralisée des erreurs
- Ajout de corrélation d’ID (observabilité)

### Exemple : gérer 401 globalement
```ts
import { HttpInterceptorFn, HttpErrorResponse } from '@angular/common/http';
import { catchError, throwError } from 'rxjs';
import { inject } from '@angular/core';
import { Router } from '@angular/router';

export const errorInterceptor: HttpInterceptorFn = (req, next) => {
  const router = inject(Router);

  return next(req).pipe(
    catchError((err: HttpErrorResponse) => {
      if (err.status === 401) {
        router.navigateByUrl('/login');
      }
      return throwError(() => err);
    })
  );
};
```

## 7.3 CSRF
Si auth en cookies :
- utiliser un mécanisme CSRF (token synchronizer, double submit, etc.)
- Angular supporte le header `X-XSRF-TOKEN` via `HttpClientXsrfModule` (selon version/config)

Points clés :
- cookie CSRF lisible (pas HttpOnly) + header envoyé
- vérifier côté serveur

## 7.4 CORS
- CORS est une politique navigateur, pas une sécurité serveur
- configurer le backend pour n’autoriser que les origines nécessaires
- ne jamais mettre `Access-Control-Allow-Origin: *` avec credentials

## 7.5 Validation des entrées côté client
Utilité :
- améliorer UX
- réduire erreurs

Limite :
- aucune valeur sécurité si non dupliquée côté serveur

---

# 8) Bonnes pratiques transverses (défense en profondeur)

## 8.1 Content Security Policy (CSP)
Objectif : limiter l’exécution de scripts non autorisés.
- `script-src 'self'` + nonce/hashes
- réduire les sources d’images, styles, frames

> CSP se configure côté serveur (en-têtes HTTP). Angular doit être compatible (attention aux inline scripts/styles).

## 8.2 Dépendances et supply chain
- lockfile (`package-lock.json`/`pnpm-lock`)
- audit de dépendances (`npm audit`, Snyk, Dependabot)
- éviter libs non maintenues

## 8.3 Gestion des erreurs et logs
- ne pas afficher d’infos sensibles (stack traces internes) à l’utilisateur
- logger côté client avec prudence (PII)

## 8.4 Données sensibles côté client
- minimiser ce qui est stocké
- expirer/effacer au logout
- éviter de mettre des PII dans les query params

## 8.5 Séparation des environnements
- variables d’environnement Angular (attention : visibles au build)
- endpoints distincts dev/staging/prod

---

# 9) Ateliers / exercices (suggestions)

## Atelier 1 — Identifier les surfaces XSS
- Prendre une page commentaire
- Tester : `"><img src=x onerror=alert(1)>`
- Analyser où ça passe / où c’est neutralisé

## Atelier 2 — Rendu riche contrôlé
- Remplacer `innerHTML` libre par un rendu Markdown
- Ajouter une sanitization stricte

## Atelier 3 — Interceptor d’auth
- Ajouter `Authorization: Bearer …`
- Exclure les domaines tiers
- Gérer 401/refresh (simulation)

## Atelier 4 — Guards + permissions
- Protéger `/admin`
- Simuler des rôles
- Vérifier que l’API refuse l’accès même si le guard est contourné

---

# Checklists finales

## Checklist anti‑XSS
- [ ] Éviter `innerHTML` quand possible
- [ ] Ne jamais utiliser `bypassSecurityTrust...` sur du contenu non totalement maîtrisé
- [ ] Ne pas manipuler le DOM via `nativeElement.innerHTML`
- [ ] Valider/sanitizer le contenu riche côté serveur (défense en profondeur)
- [ ] Mettre en place une CSP adaptée

## Checklist tokens
- [ ] Minimiser le stockage côté client
- [ ] Prévoir expiration courte + refresh
- [ ] Ne pas envoyer le token à des domaines non nécessaires
- [ ] Prévoir révocation/rotation

## Checklist routes & API
- [ ] Guards côté Angular pour UX
- [ ] Autorisation systématique côté backend
- [ ] Ne pas faire confiance aux claims seules sans contrôles serveur

## Checklist HTTP
- [ ] HTTPS uniquement
- [ ] CSRF si cookies
- [ ] CORS restrictif
- [ ] Gestion centralisée des erreurs (interceptors)

---

# Conclusion
Angular fournit des mécanismes solides (escaping, sanitization, routing, interceptors), mais la sécurité dépend principalement de vos choix d’architecture : **réduire la surface d’attaque (XSS), stocker les tokens de façon adaptée, protéger les routes pour l’UX, et sécuriser chaque échange HTTP**. Enfin, répétons-le : **le frontend n’est jamais une autorité de sécurité** ; tout accès aux données et toute action critique doivent être validés côté backend.
