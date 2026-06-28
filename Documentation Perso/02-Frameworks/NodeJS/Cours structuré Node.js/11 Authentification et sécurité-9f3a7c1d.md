# 11 — Authentification et sécurité (Node.js / React)

> **Public visé** : développeurs (niveau intermédiaire) Node.js / React
>
> **Prérequis** : JavaScript moderne, HTTP/REST, bases d’Express, notions de React
>
> **Durée suggérée** : 1 jour (6–7h) ou 2 demi-journées
>
> **Objectifs pédagogiques**
> - Comprendre les principes d’authentification/autorisation et les menaces courantes.
> - Implémenter une auth **JWT** robuste (access + refresh tokens).
> - Stocker les mots de passe de façon sûre avec **bcrypt** (salage, coût, bonnes pratiques).
> - Sécuriser une API Node/Express via des **middlewares** (auth, rôles, rate limiting, headers).
> - Protéger le front React (flux login, stockage token, renouvellement, guard de routes).
> - Mettre en place des contrôles de sécurité (CORS, Helmet, validation, logs, rotation des secrets).

---

## Plan de la formation

1. **Fondamentaux : authentification vs autorisation**
   - Identité, sessions, tokens
   - Menaces : brute force, credential stuffing, XSS, CSRF, injection, leakage de tokens
2. **Gestion des mots de passe : bcrypt**
   - Hash vs chiffrement
   - Salage, cost factor, timing attacks
   - Politique de mot de passe et protections
3. **JWT en pratique**
   - Structure JWT, signature, claims
   - Access token vs refresh token
   - Expiration, rotation, révocation
4. **Architecture d’une API sécurisée avec Express**
   - Patterns de middlewares
   - Validation d’entrées, gestion d’erreurs, réponses cohérentes
   - Sécurisation HTTP (Helmet), CORS, rate limiting
5. **Implémentation pas à pas (Node.js + Express)**
   - Endpoints: register, login, refresh, logout, me
   - Middlewares: authenticateJWT, authorizeRoles
   - Stockage refresh tokens (DB) et rotation
6. **Intégration React**
   - Formulaires login/register
   - Gestion des tokens (mémoire vs storage vs cookies httpOnly)
   - Intercepteurs (axios/fetch) et renouvellement automatique
   - Protection des routes
7. **Bonnes pratiques et checklist**
   - Secrets, env vars, rotation
   - Observabilité (logs, audit)
   - Tests (unitaires, intégration)
   - Déploiement (HTTPS, proxy, headers)

---

# 1) Fondamentaux : Authentification vs Autorisation

## 1.1 Définitions

- **Authentification (AuthN)** : prouver qui vous êtes (ex: email + mot de passe, OAuth).
- **Autorisation (AuthZ)** : vérifier ce que vous avez le droit de faire (ex: rôle admin, permissions).

## 1.2 Sessions vs Tokens

### Sessions (stateful)
- Le serveur stocke une session (ex: en mémoire, Redis).
- Le client garde un **cookie de session**.
- Avantages : révocation simple, contrôle central.
- Inconvénients : scalabilité (store central), complexité multi-services.

### Tokens (stateless)
- Le serveur signe un token (ex: JWT). Le serveur ne stocke pas nécessairement l’état.
- Avantages : scalabilité et microservices.
- Inconvénients : révocation plus complexe, risques si le token fuit.

## 1.3 Menaces courantes

- **Brute force** : tentatives massives sur login.
- **Credential stuffing** : réutilisation de credentials volés.
- **XSS** : exfiltration de tokens stockés en localStorage.
- **CSRF** : attaques via cookies si pas de protection.
- **JWT leakage** : logs, URL, stockage non sécurisé.
- **Injection** : SQL/NoSQL injection si absence de validation.

---

# 2) Gestion des mots de passe : bcrypt

## 2.1 Hash vs Chiffrement

- **Chiffrement** : réversible (avec une clé). Inadapté pour stocker des mots de passe.
- **Hash** : fonction à sens unique. On compare en re-hashant l’entrée.

## 2.2 Pourquoi bcrypt

- Conçu pour être **lent** (coût configurable) et résister aux attaques par GPU.
- Intègre un **sel** (salt) automatiquement.

## 2.3 Paramétrage : cost factor (salt rounds)

- Plus le coût est élevé, plus le hash est lent.
- Valeurs courantes : **10–12** en dev / prod selon contraintes.

> Recommandation : mesurer le temps de hash dans votre infra (objectif often 100–250 ms par hash) et ajuster.

## 2.4 Bonnes pratiques

- Ne jamais stocker le mot de passe en clair.
- Ne jamais logger les mots de passe.
- Appliquer un **rate limiting** sur login.
- Utiliser une politique (longueur min, listes de mots de passe compromis via API HIBP si souhaité).
- Protéger contre le **user enumeration** : messages d’erreur identiques ("identifiants invalides").

## 2.5 Exemple Node.js (bcrypt)

```js
import bcrypt from "bcrypt";

const SALT_ROUNDS = 12;

export async function hashPassword(plainPassword) {
  return bcrypt.hash(plainPassword, SALT_ROUNDS);
}

export async function verifyPassword(plainPassword, passwordHash) {
  return bcrypt.compare(plainPassword, passwordHash);
}
```

---

# 3) JWT en pratique

## 3.1 Structure d’un JWT

Un JWT est composé de :

- **Header** : algorithme (ex: HS256)
- **Payload** : claims (sub, exp, iat, roles, etc.)
- **Signature** : garantit l’intégrité

Format : `header.payload.signature`

## 3.2 Claims importants

- `sub` : subject = identifiant utilisateur
- `exp` : expiration
- `iat` : émis à
- `iss` : issuer
- `aud` : audience

> Évitez de mettre des données sensibles dans le payload : il est **encodé** en Base64URL, pas chiffré.

## 3.3 Access vs Refresh tokens

### Access token
- Durée courte (ex: 5–15 min)
- Envoyé sur chaque requête API (Authorization: Bearer)

### Refresh token
- Durée plus longue (ex: 7–30 jours)
- Sert à obtenir un nouvel access token
- Doit être **mieux protégé** (idéalement cookie httpOnly) et **roté**

## 3.4 Révocation / rotation

- Un JWT stateless n’est pas trivial à révoquer.
- Solutions :
  - **Blacklist** côté serveur (Redis) jusqu’à expiration
  - **Refresh token rotation** + stockage DB des refresh tokens actifs
  - Sessions stateful + cookies

---

# 4) Architecture d’une API sécurisée (Express)

## 4.1 Middlewares de sécurité (à connaître)

- **helmet** : en-têtes HTTP de sécurité
- **cors** : contrôle des origines
- **express-rate-limit** : limitation du nombre de requêtes
- **cookie-parser** : lecture des cookies
- **validation** (zod/joi/yup) : validation des payloads
- Middleware d’authentification : vérifie JWT
- Middleware d’autorisation : vérifie rôles/permissions

## 4.2 Gestion d’erreurs

- Centraliser les erreurs dans `app.use((err, req, res, next) => ...)`.
- Ne pas exposer de stack trace en prod.
- Normaliser les réponses.

Exemple de format d’erreur :

```json
{ "error": { "code": "UNAUTHORIZED", "message": "Authentication required" } }
```

---

# 5) Implémentation pas à pas (Node.js + Express)

> Le but ici est pédagogique : vous pouvez adapter à votre stack (Mongo/SQL/Prisma, etc.).

## 5.1 Dépendances

```bash
npm i express jsonwebtoken bcrypt helmet cors cookie-parser express-rate-limit zod
```

## 5.2 Variables d’environnement

Créez un `.env` (via dotenv si souhaité) :

- `JWT_ACCESS_SECRET` : secret de signature access
- `JWT_REFRESH_SECRET` : secret de signature refresh
- `JWT_ACCESS_TTL` : ex `15m`
- `JWT_REFRESH_TTL` : ex `7d`
- `CORS_ORIGIN` : ex `https://app.example.com`

> En production : secrets longs, rotation, stockage dans un secret manager.

## 5.3 Modèle de données minimal

Conceptuellement :

- `User { id, email, passwordHash, roles[] }`
- `RefreshToken { id, userId, tokenHash, expiresAt, revokedAt, createdAt }`

> Important : stocker un **hash** du refresh token en base (si DB compromise, l’attaquant ne peut pas l’utiliser directement).

## 5.4 Utilitaires JWT

```js
import jwt from "jsonwebtoken";

const accessSecret = process.env.JWT_ACCESS_SECRET;
const refreshSecret = process.env.JWT_REFRESH_SECRET;

const accessTtl = process.env.JWT_ACCESS_TTL || "15m";
const refreshTtl = process.env.JWT_REFRESH_TTL || "7d";

export function signAccessToken(user) {
  return jwt.sign(
    { roles: user.roles },
    accessSecret,
    { subject: String(user.id), expiresIn: accessTtl, issuer: "api" }
  );
}

export function signRefreshToken(user) {
  // payload minimal
  return jwt.sign(
    {},
    refreshSecret,
    { subject: String(user.id), expiresIn: refreshTtl, issuer: "api" }
  );
}

export function verifyAccessToken(token) {
  return jwt.verify(token, accessSecret, { issuer: "api" });
}

export function verifyRefreshToken(token) {
  return jwt.verify(token, refreshSecret, { issuer: "api" });
}
```

## 5.5 Middleware d’authentification (JWT)

```js
export function authenticateJWT(req, res, next) {
  const auth = req.headers.authorization;
  if (!auth?.startsWith("Bearer ")) {
    return res.status(401).json({ error: { code: "UNAUTHORIZED", message: "Authentication required" } });
  }

  const token = auth.slice("Bearer ".length);

  try {
    const payload = verifyAccessToken(token);
    // payload.sub contient userId (string)
    req.user = {
      id: payload.sub,
      roles: payload.roles || [],
    };
    next();
  } catch {
    return res.status(401).json({ error: { code: "INVALID_TOKEN", message: "Invalid or expired token" } });
  }
}
```

## 5.6 Middleware d’autorisation (rôles)

```js
export function authorizeRoles(...allowedRoles) {
  return (req, res, next) => {
    const roles = req.user?.roles || [];
    const ok = allowedRoles.some(r => roles.includes(r));

    if (!ok) {
      return res.status(403).json({ error: { code: "FORBIDDEN", message: "Insufficient permissions" } });
    }

    next();
  };
}
```

## 5.7 Validation des entrées (zod)

```js
import { z } from "zod";

export const registerSchema = z.object({
  email: z.string().email(),
  password: z.string().min(10),
});

export const loginSchema = z.object({
  email: z.string().email(),
  password: z.string().min(1),
});
```

Utilisation :

```js
function validate(schema) {
  return (req, res, next) => {
    const result = schema.safeParse(req.body);
    if (!result.success) {
      return res.status(400).json({ error: { code: "VALIDATION_ERROR", message: result.error.message } });
    }
    req.body = result.data;
    next();
  };
}
```

## 5.8 Mise en place des middlewares globaux

```js
import express from "express";
import helmet from "helmet";
import cors from "cors";
import cookieParser from "cookie-parser";
import rateLimit from "express-rate-limit";

const app = express();

app.use(helmet());
app.use(express.json({ limit: "100kb" }));
app.use(cookieParser());

app.use(cors({
  origin: process.env.CORS_ORIGIN,
  credentials: true,
}));

// Rate limiting global (à ajuster)
app.use(rateLimit({
  windowMs: 15 * 60 * 1000,
  limit: 300,
}));
```

> Bonnes pratiques : un rate limit **plus strict** sur `/login`.

## 5.9 Endpoints Auth

### 5.9.1 Register

```js
import { hashPassword, verifyPassword } from "./password.js";
import { signAccessToken, signRefreshToken, verifyRefreshToken } from "./jwt.js";

const loginLimiter = rateLimit({ windowMs: 15 * 60 * 1000, limit: 20 });

app.post("/auth/register", validate(registerSchema), async (req, res) => {
  const { email, password } = req.body;

  // TODO: vérifier unicité email en DB
  const passwordHash = await hashPassword(password);

  // TODO: créer user en DB
  const user = { id: "123", email, roles: ["user"], passwordHash };

  return res.status(201).json({ id: user.id, email: user.email });
});
```

### 5.9.2 Login (émission access + refresh)

Deux approches pour le refresh token :

- **A**. Refresh token en **cookie httpOnly** (recommandé en navigateur)
- **B**. Refresh token renvoyé en JSON (plus simple, mais plus risqué si stocké)

Ici : approche A.

```js
app.post("/auth/login", loginLimiter, validate(loginSchema), async (req, res) => {
  const { email, password } = req.body;

  // TODO: chercher user en DB
  const user = null; // placeholder

  // Anti user enumeration : message identique
  if (!user) {
    return res.status(401).json({ error: { code: "INVALID_CREDENTIALS", message: "Invalid credentials" } });
  }

  const ok = await verifyPassword(password, user.passwordHash);
  if (!ok) {
    return res.status(401).json({ error: { code: "INVALID_CREDENTIALS", message: "Invalid credentials" } });
  }

  const accessToken = signAccessToken(user);
  const refreshToken = signRefreshToken(user);

  // TODO: stocker hash(refreshToken) en DB, associé à user

  res.cookie("refresh_token", refreshToken, {
    httpOnly: true,
    secure: true,      // true en prod (HTTPS)
    sameSite: "strict",// ou "lax" selon besoin
    path: "/auth/refresh",
    maxAge: 7 * 24 * 60 * 60 * 1000,
  });

  return res.json({ accessToken });
});
```

### 5.9.3 Refresh (rotation)

Objectif :
- le client envoie le cookie refresh
- le serveur vérifie, puis **émet un nouveau refresh** et invalide l’ancien

```js
app.post("/auth/refresh", async (req, res) => {
  const refreshToken = req.cookies?.refresh_token;
  if (!refreshToken) {
    return res.status(401).json({ error: { code: "UNAUTHORIZED", message: "Refresh token required" } });
  }

  try {
    const payload = verifyRefreshToken(refreshToken);
    const userId = payload.sub;

    // TODO: vérifier en DB que le refresh token est encore valide (hash match & non révoqué)

    // TODO: charger user en DB
    const user = { id: userId, roles: ["user"] };

    // Rotation : nouveau refresh
    const newRefreshToken = signRefreshToken(user);
    const newAccessToken = signAccessToken(user);

    // TODO: révoquer l’ancien en DB et enregistrer le nouveau (hash)

    res.cookie("refresh_token", newRefreshToken, {
      httpOnly: true,
      secure: true,
      sameSite: "strict",
      path: "/auth/refresh",
      maxAge: 7 * 24 * 60 * 60 * 1000,
    });

    return res.json({ accessToken: newAccessToken });
  } catch {
    return res.status(401).json({ error: { code: "INVALID_REFRESH", message: "Invalid or expired refresh token" } });
  }
});
```

### 5.9.4 Logout (révocation)

```js
app.post("/auth/logout", async (req, res) => {
  const refreshToken = req.cookies?.refresh_token;

  if (refreshToken) {
    // TODO: révoquer en DB (via hash) si vous stockez les refresh tokens
  }

  res.clearCookie("refresh_token", { path: "/auth/refresh" });
  return res.status(204).send();
});
```

### 5.9.5 Route protégée : /me

```js
app.get("/me", authenticateJWT, async (req, res) => {
  // req.user défini par le middleware
  // TODO: charger user en DB
  return res.json({ id: req.user.id, roles: req.user.roles });
});
```

### 5.9.6 Route admin

```js
app.get("/admin/stats", authenticateJWT, authorizeRoles("admin"), async (req, res) => {
  return res.json({ ok: true, message: "Secret admin stats" });
});
```

---

# 6) Intégration React

## 6.1 Stratégies de stockage des tokens

### Option recommandée (web)
- **Access token** : en mémoire (state) ou storage court terme (risque XSS)
- **Refresh token** : **cookie httpOnly** (non accessible via JS)

Avantages : limite l’exfiltration du refresh token via XSS.

## 6.2 Flux login

1. React envoie `POST /auth/login` avec email/password
2. API renvoie `accessToken` et pose cookie `refresh_token`
3. React conserve l’access token en mémoire
4. React appelle l’API avec `Authorization: Bearer <accessToken>`

## 6.3 Client HTTP avec renouvellement (fetch)

Pseudo-implémentation :

```js
let accessToken = null;

export function setAccessToken(token) {
  accessToken = token;
}

async function refreshAccessToken() {
  const res = await fetch("/auth/refresh", {
    method: "POST",
    credentials: "include", // important pour cookie refresh
  });

  if (!res.ok) throw new Error("refresh failed");
  const data = await res.json();
  setAccessToken(data.accessToken);
  return data.accessToken;
}

export async function apiFetch(input, init = {}) {
  const headers = new Headers(init.headers || {});
  if (accessToken) headers.set("Authorization", `Bearer ${accessToken}`);

  const res = await fetch(input, {
    ...init,
    headers,
    credentials: "include",
  });

  if (res.status !== 401) return res;

  // tentative de refresh puis retry une fois
  const newToken = await refreshAccessToken();
  const retryHeaders = new Headers(init.headers || {});
  retryHeaders.set("Authorization", `Bearer ${newToken}`);

  return fetch(input, {
    ...init,
    headers: retryHeaders,
    credentials: "include",
  });
}
```

## 6.4 Protection des routes

Exemple (React Router) :

- Garder un état `auth` (token présent + `me` ok)
- Rediriger si non authentifié

Pseudo-code :

```js
function RequireAuth({ children }) {
  const { isAuthenticated } = useAuth();
  if (!isAuthenticated) return <Navigate to="/login" replace />;
  return children;
}
```

## 6.5 Prévention XSS (front)

- Éviter `dangerouslySetInnerHTML`
- Échapper/valider les contenus utilisateurs
- Mettre en place CSP côté serveur (Helmet peut aider)

---

# 7) Sécuriser l’API avec des middlewares : checklist pratique

## 7.1 Helmet
- Active des en-têtes utiles : `X-Content-Type-Options`, `Referrer-Policy`, etc.
- Considérer une **CSP** (Content Security Policy) adaptée à votre front.

## 7.2 CORS
- **Ne pas** utiliser `origin: "*"` avec `credentials: true`.
- Restreindre à votre domaine.

## 7.3 Rate limiting

- Global + spécifique sur :
  - `/auth/login`
  - `/auth/register`
  - `/auth/refresh`

## 7.4 Validation d’entrées

- Toujours valider : body, params, query.
- Pour NoSQL, attention aux opérateurs (`$gt`, `$ne`) si input non filtré.

## 7.5 Logging & audit

- Logger : tentative login (sans mots de passe), reset password, refresh, erreurs 401/403.
- Corréler via request id.

## 7.6 Secrets & configuration

- Secrets uniquement via env / secret manager.
- Rotation et révocation (au moins périodiquement).

## 7.7 HTTPS

- En prod : HTTPS obligatoire.
- Cookies : `secure: true`.

---

# Exercices (avec correction attendue)

## Exercice 1 — Ajouter un middleware `requireAuth`

**Énoncé** : protéger `/me` et `/admin/stats`.

**Attendu** : utilisation du header `Authorization` et retour 401 si manquant/expiré.

## Exercice 2 — Ajouter la rotation des refresh tokens

**Énoncé** : stocker en base le hash du refresh token et invalider l’ancien à chaque refresh.

**Attendu** :
- table/collection refresh
- au refresh : vérifier l’existant, révoquer, créer nouveau

## Exercice 3 — Ajouter un rate limit spécifique sur `/auth/login`

**Énoncé** : 10 tentatives / 15 minutes / IP.

**Attendu** : `express-rate-limit` au niveau de la route.

---

# Annexes

## A) Pourquoi éviter localStorage pour le refresh token

- `localStorage` est accessible au JavaScript : en cas de XSS, un attaquant peut exfiltrer le refresh token.
- Avec cookie **httpOnly**, le refresh token n’est pas lisible par JS.

## B) Anti-patterns fréquents

- JWT sans expiration.
- Mettre des informations sensibles dans le payload.
- Secret JWT faible ou commité.
- Pas de rate limiting sur login.
- Accepter `alg: none` (lib obsolète/mal config).

## C) Mini-checklist “Ready for production”

- [ ] HTTPS en prod
- [ ] `helmet()` + CSP (si possible)
- [ ] CORS restreint + `credentials` correct
- [ ] rate limit global + auth
- [ ] validation (body/params/query)
- [ ] mots de passe en bcrypt (cost contrôlé)
- [ ] access token court + refresh token rotation
- [ ] refresh token en cookie httpOnly
- [ ] logs + monitoring
- [ ] tests d’intégration sur `/auth/*`
