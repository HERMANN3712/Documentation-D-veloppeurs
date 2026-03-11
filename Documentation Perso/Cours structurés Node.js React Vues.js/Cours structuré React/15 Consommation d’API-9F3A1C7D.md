# Formation React — Consommation d’API (REST) avec `fetch` et `axios`

**Référence :** 15 — Consommation d’API  
**Public :** développeurs React (débutant → intermédiaire)  
**Pré-requis :** JavaScript ES6+, Promesses, bases React (composants, hooks, props/state)  
**Durée suggérée :** 1 journée (6–7h) ou 2×3h  

---

## Objectifs pédagogiques

À la fin de cette formation, vous saurez :

- Comprendre le rôle et les contraintes des API REST dans une application React.
- Consommer une API avec `fetch` et avec `axios`.
- Stocker et afficher les données récupérées via l’état (`useState`) et déclencher les appels via `useEffect`.
- Gérer les états d’interface essentiels : **loading**, **error**, **success**.
- Mettre en place une couche d’accès API (client HTTP), factoriser la configuration et les erreurs.
- Aborder des sujets pratiques : pagination, recherche, debounce, annulation de requêtes, CORS, variables d’environnement.
- Écrire des tests de base et éviter les pièges courants.

---

## Plan de la formation

1. **Rappels : API REST et modèle client/serveur**
2. **Architecture React typique pour la consommation d’API**
3. **Consommer une API avec `fetch`**
4. **Consommer une API avec `axios`**
5. **Gérer l’état : loading / error / data**
6. **Structurer son code : services, client HTTP, configuration**
7. **Cas pratiques fréquents** : pagination, recherche, filtres, debounce
8. **Gestion des erreurs, timeouts, annulation, retries**
9. **Optimisations et bonnes pratiques** (cache, évitement des doubles appels, StrictMode)
10. **Sécurité & contraintes** (CORS, tokens, stockage)
11. **Mise en pratique guidée : mini-projet**
12. **Checklist finale & ressources**

---

# 1) Rappels : API REST et modèle client/serveur

## 1.1 Qu’est-ce qu’une API REST ?

Une API REST expose des **ressources** via des **URL** et utilise les verbes HTTP.

- `GET /users` → liste d’utilisateurs
- `GET /users/42` → un utilisateur
- `POST /users` → créer un utilisateur
- `PUT /users/42` / `PATCH /users/42` → mettre à jour
- `DELETE /users/42` → supprimer

Les réponses sont souvent en **JSON** (mais pas uniquement).

## 1.2 Codes HTTP essentiels

- `200 OK` : succès
- `201 Created` : création
- `204 No Content` : succès sans contenu
- `400 Bad Request` : requête invalide
- `401 Unauthorized` : authentification requise
- `403 Forbidden` : accès refusé
- `404 Not Found` : ressource inexistante
- `409 Conflict` : conflit
- `422 Unprocessable Entity` : validation
- `500`+ : erreur serveur

## 1.3 Contraintes typiques côté front

- **Asynchrone** : l’UI doit rester réactive.
- **Réseau incertain** : timeouts, erreurs, lenteurs.
- **Données** : transformation, normalisation, pagination.
- **Sécurité** : tokens, CORS.

---

# 2) Architecture React typique pour la consommation d’API

## 2.1 Où placer l’appel API ?

En React fonctionnel moderne, on met généralement la requête dans un `useEffect` (ou dans une action déclenchée par l’utilisateur), et on stocke le résultat dans un `useState`.

Schéma classique :

1. **Composant** monte
2. `useEffect` se déclenche
3. Appel réseau
4. `setState` met à jour
5. Rendu UI (liste, détails, messages)

## 2.2 Les 3 états UI minimum

Pour une UX correcte, pensez systématiquement :

- `loading`: booléen (ou status)
- `error`: objet/texte si échec
- `data`: résultat (liste/objet)

Exemple de modèle :

```js
const [data, setData] = useState(null);
const [loading, setLoading] = useState(false);
const [error, setError] = useState(null);
```

---

# 3) Consommer une API avec `fetch`

`fetch` est natif dans les navigateurs modernes.

## 3.1 GET simple

Exemple : récupérer des posts depuis `https://jsonplaceholder.typicode.com/posts`.

```jsx
import { useEffect, useState } from "react";

export function PostsFetch() {
  const [posts, setPosts] = useState([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);

  useEffect(() => {
    let ignore = false;

    async function load() {
      setLoading(true);
      setError(null);

      try {
        const res = await fetch("https://jsonplaceholder.typicode.com/posts");

        // fetch ne lève PAS d'erreur automatique sur 404/500
        if (!res.ok) {
          throw new Error(`HTTP ${res.status}`);
        }

        const json = await res.json();
        if (!ignore) setPosts(json);
      } catch (e) {
        if (!ignore) setError(e);
      } finally {
        if (!ignore) setLoading(false);
      }
    }

    load();
    return () => {
      ignore = true;
    };
  }, []);

  if (loading) return <p>Chargement…</p>;
  if (error) return <p style={{ color: "crimson" }}>Erreur: {error.message}</p>;

  return (
    <ul>
      {posts.slice(0, 10).map((p) => (
        <li key={p.id}>{p.title}</li>
      ))}
    </ul>
  );
}
```

### Points clés

- `fetch()` renvoie une promesse résolue même si HTTP=404/500 → il faut tester `res.ok`.
- `useEffect([])` = appel au montage.
- Le flag `ignore` évite de faire un `setState` après un unmount (cas fréquent quand on navigue vite).

## 3.2 POST (création)

```js
async function createPost(payload) {
  const res = await fetch("https://jsonplaceholder.typicode.com/posts", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
    },
    body: JSON.stringify(payload),
  });

  if (!res.ok) throw new Error(`HTTP ${res.status}`);
  return res.json();
}
```

Usage dans un composant (soumission de formulaire) :

```jsx
import { useState } from "react";

export function CreatePostFetch() {
  const [title, setTitle] = useState("");
  const [status, setStatus] = useState("idle"); // idle | loading | success | error
  const [error, setError] = useState(null);

  async function handleSubmit(e) {
    e.preventDefault();
    setStatus("loading");
    setError(null);

    try {
      await createPost({ title, body: "Hello", userId: 1 });
      setStatus("success");
      setTitle("");
    } catch (e) {
      setStatus("error");
      setError(e);
    }
  }

  return (
    <form onSubmit={handleSubmit}>
      <input value={title} onChange={(e) => setTitle(e.target.value)} />
      <button disabled={status === "loading"}>Créer</button>

      {status === "loading" && <p>Envoi…</p>}
      {status === "success" && <p>Créé !</p>}
      {status === "error" && <p style={{ color: "crimson" }}>{error.message}</p>}
    </form>
  );
}
```

---

# 4) Consommer une API avec `axios`

`axios` est une librairie très utilisée. Avantages :

- Intercepteurs (auth, logs)
- Transformations JSON automatiques
- Gestion plus simple de certaines erreurs
- Timeouts plus faciles

Installation :

```bash
npm i axios
```

## 4.1 GET avec axios

```jsx
import { useEffect, useState } from "react";
import axios from "axios";

export function PostsAxios() {
  const [posts, setPosts] = useState([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);

  useEffect(() => {
    const controller = new AbortController();

    async function load() {
      setLoading(true);
      setError(null);

      try {
        const res = await axios.get(
          "https://jsonplaceholder.typicode.com/posts",
          { signal: controller.signal }
        );
        setPosts(res.data);
      } catch (e) {
        // axios rejette la promesse sur 4xx/5xx (comportement souvent attendu)
        setError(e);
      } finally {
        setLoading(false);
      }
    }

    load();
    return () => controller.abort();
  }, []);

  if (loading) return <p>Chargement…</p>;
  if (error) return <p style={{ color: "crimson" }}>Erreur: {error.message}</p>;

  return (
    <ul>
      {posts.slice(0, 10).map((p) => (
        <li key={p.id}>{p.title}</li>
      ))}
    </ul>
  );
}
```

## 4.2 POST avec axios

```js
import axios from "axios";

export async function createPostAxios(payload) {
  const res = await axios.post(
    "https://jsonplaceholder.typicode.com/posts",
    payload,
    {
      headers: { "Content-Type": "application/json" },
      timeout: 10_000,
    }
  );
  return res.data;
}
```

---

# 5) Gérer l’état : loading / error / data

## 5.1 Pourquoi c’est indispensable

Sans gestion claire :

- l’utilisateur ne sait pas ce qui se passe,
- l’application peut afficher des données incohérentes,
- vous risquez des bugs (rendu sur `null`, erreurs silencieuses).

## 5.2 Pattern recommandé : `status`

Au lieu de 2–3 states séparés, vous pouvez centraliser :

```js
const [state, setState] = useState({
  status: "idle", // idle | loading | success | error
  data: null,
  error: null,
});
```

Mise à jour :

```js
setState({ status: "loading", data: null, error: null });
// ...
setState({ status: "success", data: json, error: null });
// ...
setState({ status: "error", data: null, error: e });
```

## 5.3 Affichage conditionnel

```jsx
if (state.status === "idle") return <p>Prêt.</p>;
if (state.status === "loading") return <p>Chargement…</p>;
if (state.status === "error") return <p>Erreur: {state.error.message}</p>;
return <pre>{JSON.stringify(state.data, null, 2)}</pre>;
```

---

# 6) Structurer son code : services, client HTTP, configuration

Quand l’application grandit, il faut éviter :

- des URLs dupliquées,
- une logique réseau copiée/collée,
- des headers/auth gérés partout.

## 6.1 Variables d’environnement (base URL)

Avec Vite (exemple) :

```env
VITE_API_BASE_URL=https://api.example.com
```

```js
const API_BASE_URL = import.meta.env.VITE_API_BASE_URL;
```

## 6.2 Créer un client axios

`src/api/httpClient.js` :

```js
import axios from "axios";

export const http = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL,
  timeout: 10_000,
});

// Exemple: ajouter un token automatiquement
http.interceptors.request.use((config) => {
  const token = localStorage.getItem("token");
  if (token) config.headers.Authorization = `Bearer ${token}`;
  return config;
});

// Exemple: normaliser les erreurs
http.interceptors.response.use(
  (response) => response,
  (error) => {
    // Vous pouvez mapper les erreurs ici
    return Promise.reject(error);
  }
);
```

## 6.3 Créer un service par domaine

`src/api/postsApi.js` :

```js
import { http } from "./httpClient";

export async function listPosts(params) {
  const res = await http.get("/posts", { params });
  return res.data;
}

export async function getPost(id) {
  const res = await http.get(`/posts/${id}`);
  return res.data;
}

export async function createPost(payload) {
  const res = await http.post("/posts", payload);
  return res.data;
}
```

Avantages :

- composants plus lisibles,
- tests facilités,
- évolution de l’API centralisée.

---

# 7) Cas pratiques fréquents

## 7.1 Pagination

Deux stratégies :

- Pagination côté serveur : `GET /posts?page=2&limit=20`
- Pagination côté client : charger tout puis `slice` (souvent inefficace)

Exemple minimal (pagination serveur) :

```jsx
import { useEffect, useState } from "react";
import axios from "axios";

export function PostsPaginated() {
  const [page, setPage] = useState(1);
  const [posts, setPosts] = useState([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);

  useEffect(() => {
    const controller = new AbortController();

    async function load() {
      setLoading(true);
      setError(null);

      try {
        const res = await axios.get(
          "https://jsonplaceholder.typicode.com/posts",
          {
            params: { _page: page, _limit: 10 },
            signal: controller.signal,
          }
        );
        setPosts(res.data);
      } catch (e) {
        setError(e);
      } finally {
        setLoading(false);
      }
    }

    load();
    return () => controller.abort();
  }, [page]);

  return (
    <section>
      <header style={{ display: "flex", gap: 8, alignItems: "center" }}>
        <button onClick={() => setPage((p) => Math.max(1, p - 1))}>
          Précédent
        </button>
        <strong>Page {page}</strong>
        <button onClick={() => setPage((p) => p + 1)}>Suivant</button>
      </header>

      {loading && <p>Chargement…</p>}
      {error && <p style={{ color: "crimson" }}>{error.message}</p>}

      <ul>
        {posts.map((p) => (
          <li key={p.id}>{p.title}</li>
        ))}
      </ul>
    </section>
  );
}
```

## 7.2 Recherche + debounce

Objectif : éviter un appel API à chaque frappe.

Principe : attendre X ms après la dernière frappe.

```jsx
import { useEffect, useState } from "react";

function useDebouncedValue(value, delayMs) {
  const [debounced, setDebounced] = useState(value);

  useEffect(() => {
    const id = setTimeout(() => setDebounced(value), delayMs);
    return () => clearTimeout(id);
  }, [value, delayMs]);

  return debounced;
}

export function SearchPosts() {
  const [q, setQ] = useState("");
  const debouncedQ = useDebouncedValue(q, 400);

  const [posts, setPosts] = useState([]);
  const [status, setStatus] = useState("idle");
  const [error, setError] = useState(null);

  useEffect(() => {
    if (!debouncedQ) {
      setPosts([]);
      setStatus("idle");
      return;
    }

    const controller = new AbortController();

    async function load() {
      setStatus("loading");
      setError(null);

      try {
        const res = await fetch(
          `https://jsonplaceholder.typicode.com/posts?q=${encodeURIComponent(
            debouncedQ
          )}`,
          { signal: controller.signal }
        );
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        const json = await res.json();
        setPosts(json);
        setStatus("success");
      } catch (e) {
        if (e.name === "AbortError") return;
        setError(e);
        setStatus("error");
      }
    }

    load();
    return () => controller.abort();
  }, [debouncedQ]);

  return (
    <div>
      <input
        value={q}
        onChange={(e) => setQ(e.target.value)}
        placeholder="Rechercher…"
      />

      {status === "idle" && <p>Saisissez une recherche.</p>}
      {status === "loading" && <p>Chargement…</p>}
      {status === "error" && <p style={{ color: "crimson" }}>{error.message}</p>}

      <ul>
        {posts.slice(0, 10).map((p) => (
          <li key={p.id}>{p.title}</li>
        ))}
      </ul>
    </div>
  );
}
```

---

# 8) Erreurs, timeouts, annulation, retries

## 8.1 Différences `fetch` vs `axios` sur les erreurs

- `fetch` :
  - rejette surtout sur erreurs réseau (pas sur 404/500)
  - nécessite `if (!res.ok) throw ...`
- `axios` :
  - rejette sur 4xx/5xx
  - fournit une structure riche `error.response`, `error.request`

## 8.2 Timeout

- `axios` : option `timeout`.
- `fetch` : via `AbortController` + `setTimeout`.

Exemple `fetch` avec timeout :

```js
export async function fetchWithTimeout(url, { timeoutMs = 8000, ...options } = {}) {
  const controller = new AbortController();
  const id = setTimeout(() => controller.abort(), timeoutMs);

  try {
    const res = await fetch(url, { ...options, signal: controller.signal });
    return res;
  } finally {
    clearTimeout(id);
  }
}
```

## 8.3 Retry (simple)

Attention : ne pas retry aveuglément (risque de surcharge). Réservez aux erreurs réseau/`429`/`503`.

Pseudo-code :

```js
async function retry(fn, { retries = 2, delayMs = 500 } = {}) {
  let last;
  for (let i = 0; i <= retries; i++) {
    try {
      return await fn();
    } catch (e) {
      last = e;
      if (i === retries) break;
      await new Promise((r) => setTimeout(r, delayMs));
    }
  }
  throw last;
}
```

---

# 9) Optimisations et bonnes pratiques

## 9.1 Attention au double appel en dev (React 18 StrictMode)

En développement, `StrictMode` peut monter/démonter/remonter certains composants pour détecter des effets non idempotents. Résultat : **vous voyez parfois deux appels API**.

Bonnes pratiques :

- rendre l’effet idempotent,
- gérer l’annulation (AbortController),
- éviter des effets non maîtrisés.

## 9.2 Cache et librairies spécialisées

Pour des projets réels, considérez :

- **TanStack Query (React Query)** : cache, refetch, retries, pagination.
- **SWR** : approche « stale-while-revalidate ».

Même si cette formation se concentre sur `fetch/axios + state`, ces librairies sont utiles pour la production.

## 9.3 Normalisation / transformation des données

Ne polluez pas vos composants avec des transformations complexes.

- faites-les dans le service API
- ou dans une fonction utilitaire `mapPostDtoToModel(dto)`

---

# 10) Sécurité & contraintes

## 10.1 CORS

Si votre front est sur `http://localhost:5173` et l’API sur un autre domaine, l’API doit autoriser CORS.

- Ce n’est pas un bug React.
- C’est une règle navigateur.

## 10.2 Auth : tokens

Approches courantes :

- Token en **mémoire** (plus sûr vis-à-vis XSS, mais perdu au refresh)
- Token en **localStorage** (plus simple, mais exposé en cas de XSS)
- Cookie **HttpOnly** (souvent recommandé côté web, nécessite une config CORS/CSRF)

## 10.3 Ne pas exposer de secrets

- Les variables d’environnement front (`VITE_*`) sont **publiques** une fois build.
- Un secret API doit rester côté serveur.

---

# 11) Mise en pratique guidée — Mini-projet

## Énoncé

Créer une mini application `Posts` :

- afficher une liste paginée de posts
- afficher un détail au clic
- ajouter un post (formulaire)
- gérer loading/error partout

API de démo possible : `jsonplaceholder.typicode.com`.

## Étapes

1. **Créer les composants** :
   - `PostsList`
   - `PostDetails`
   - `CreatePostForm`

2. **Créer le service API** :
   - `listPosts({ page, limit })`
   - `getPost(id)`
   - `createPost(payload)`

3. **Relier au state** :
   - `useEffect` pour list/détail
   - `onSubmit` pour create

4. **Gérer l’annulation** :
   - `AbortController` sur les effets

5. **UX** :
   - désactiver les boutons en `loading`
   - afficher les messages d’erreur

---

# 12) Pièges courants (et solutions)

- **Oublier `res.ok` avec `fetch`** → vérifiez systématiquement.
- **Mettre `async` directement dans `useEffect`** → créez une fonction interne `load()`.
- **Boucles infinies** → surveillez les dépendances de `useEffect`.
- **SetState après unmount** → annulation (AbortController) ou flag `ignore`.
- **Rendu sur `null/undefined`** → valeur initiale cohérente + garde.
- **Coupler trop l’UI à l’API** → services + mapping.

---

# Annexes

## A) Snippet : wrapper `fetchJson`

```js
export async function fetchJson(url, options) {
  const res = await fetch(url, {
    headers: { "Content-Type": "application/json", ...(options?.headers || {}) },
    ...options,
  });

  if (!res.ok) {
    let details = "";
    try {
      details = await res.text();
    } catch {
      // ignore
    }
    throw new Error(`HTTP ${res.status} ${res.statusText} ${details}`);
  }

  // Attention: 204 No Content
  if (res.status === 204) return null;
  return res.json();
}
```

## B) Exemples de structures de dossiers

```txt
src/
  api/
    httpClient.js
    postsApi.js
  components/
    PostsList.jsx
    PostDetails.jsx
    CreatePostForm.jsx
  pages/
    PostsPage.jsx
```

## C) Exercices

1. Ajouter un filtre par `userId` côté serveur.
2. Ajouter un tri (ex: `sort=title` si l’API le permet).
3. Implémenter un timeout `fetch` et afficher un message dédié.
4. Ajouter une gestion `retry` uniquement sur erreurs réseau.
5. Remplacer le state local par TanStack Query (bonus).

---

## Conclusion

La consommation d’API dans React repose sur un triptyque simple :

- **déclencher** l’appel (souvent via `useEffect` ou un événement utilisateur),
- **stocker** le résultat dans l’état,
- **afficher** une UI robuste (loading/error/success).

Ensuite, la qualité du code vient de la **factorisation** (services, client HTTP), de la **gestion des erreurs** et de l’**expérience utilisateur**.