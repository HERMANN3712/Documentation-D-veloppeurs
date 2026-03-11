# 14 — Routing avec React Router

> **Public cible** : développeurs React (débutant → intermédiaire) souhaitant structurer la navigation d’une SPA.
>
> **Prérequis** : JavaScript ES6, React (composants, props, state), notions de hooks.
>
> **Objectifs pédagogiques**
> - Comprendre le rôle du routing côté client (SPA) et les principes de React Router.
> - Mettre en place une navigation multi-pages avec `react-router-dom`.
> - Utiliser routes imbriquées, layouts, paramètres d’URL, query string et redirections.
> - Protéger des routes (auth), gérer les erreurs 404 et les pages de fallback.
> - Optimiser l’expérience (lazy-loading, scroll restoration) et tester le routing.

---

## Plan de la formation

1. **Introduction au routing en SPA**
   - Pourquoi router côté client ?
   - Comment React Router associe URL → Composant
   - Principes : `Router`, `Routes`, `Route`, navigation sans rechargement

2. **Installation & démarrage rapide**
   - Installation de `react-router-dom`
   - Structure minimale d’une application routée

3. **Routes de base**
   - Définir des routes
   - Lien de navigation : `Link` et `NavLink`
   - Route index, route 404

4. **Layouts et routes imbriquées**
   - `Outlet`
   - Layout global (header/nav/footer)
   - Layouts par section (ex: `/admin`)

5. **Navigation programmatique et redirections**
   - `useNavigate`
   - `Navigate` (redirection déclarative)
   - Gestion des retours (`-1`) et state de navigation

6. **Paramètres d’URL & query string**
   - `useParams`
   - `useSearchParams`
   - Bonnes pratiques (validation, fallback)

7. **Routes protégées (auth/roles)**
   - Composant `RequireAuth`
   - Conservation de la destination (`location.state.from`)

8. **Chargement de données et erreurs (data routers)**
   - `createBrowserRouter`, `RouterProvider`
   - `loader`, `action`, `errorElement`
   - `useLoaderData`

9. **Optimisations & UX**
   - Code splitting avec `lazy`/`Suspense`
   - Scroll restoration
   - États de navigation

10. **Tests & dépannage**
   - Tester avec `MemoryRouter`
   - Erreurs fréquentes

11. **Exercices corrigés**
   - Mini app “Catalogue”
   - Ajout d’un espace admin protégé

---

# 1) Introduction au routing en SPA

Dans une **SPA (Single Page Application)**, le navigateur charge une seule page HTML, puis React met à jour l’interface en fonction de l’état et de l’URL.

**React Router** (v6+) fournit :
- une **correspondance** entre **chemins d’URL** (`/`, `/products/:id`, etc.) et **composants** à afficher ;
- des **liens** qui changent l’URL **sans rechargement** de la page ;
- une gestion de **routes imbriquées**, de **layouts**, de **redirections**, etc.

> Idée clé : **URL ↔ état de navigation**. On peut partager une URL et retrouver le même écran.

---

# 2) Installation & démarrage rapide

## 2.1 Installation

```bash
npm i react-router-dom
```

> Pour une appli CRA/Vite/Next (côté client), on utilise généralement `react-router-dom`.

## 2.2 Exemple minimal

### `main.jsx` / `index.jsx`

```jsx
import React from "react";
import ReactDOM from "react-dom/client";
import { BrowserRouter } from "react-router-dom";
import App from "./App";

ReactDOM.createRoot(document.getElementById("root")).render(
  <React.StrictMode>
    <BrowserRouter>
      <App />
    </BrowserRouter>
  </React.StrictMode>
);
```

### `App.jsx`

```jsx
import { Routes, Route } from "react-router-dom";
import Home from "./pages/Home";
import About from "./pages/About";

export default function App() {
  return (
    <Routes>
      <Route path="/" element={<Home />} />
      <Route path="/about" element={<About />} />
    </Routes>
  );
}
```

---

# 3) Routes de base

## 3.1 Définir des routes

- `Routes` : conteneur de correspondance.
- `Route` : associe `path` à `element`.

Exemples :

```jsx
<Routes>
  <Route path="/" element={<Home />} />
  <Route path="/contact" element={<Contact />} />
  <Route path="/pricing" element={<Pricing />} />
</Routes>
```

## 3.2 Navigation avec `Link`

`Link` rend une balise similaire à `<a>`, mais utilise l’historique du navigateur sans rechargement.

```jsx
import { Link } from "react-router-dom";

export default function Navbar() {
  return (
    <nav>
      <Link to="/">Accueil</Link>
      <Link to="/pricing">Tarifs</Link>
      <Link to="/contact">Contact</Link>
    </nav>
  );
}
```

## 3.3 `NavLink` pour les liens actifs

`NavLink` ajoute une logique “actif” quand l’URL correspond.

```jsx
import { NavLink } from "react-router-dom";

export default function Navbar() {
  return (
    <nav>
      <NavLink
        to="/"
        style={({ isActive }) => ({
          fontWeight: isActive ? "bold" : "normal",
        })}
        end
      >
        Accueil
      </NavLink>
      <NavLink to="/pricing">Tarifs</NavLink>
    </nav>
  );
}
```

> `end` évite que `/` soit actif pour toutes les routes.

## 3.4 Route index et route 404

### Route index
Dans des routes imbriquées, une route `index` sert de défaut.

### 404 (Not Found)
En v6, on fait souvent :

```jsx
<Routes>
  <Route path="/" element={<Home />} />
  <Route path="/about" element={<About />} />
  <Route path="*" element={<NotFound />} />
</Routes>
```

---

# 4) Layouts et routes imbriquées

Les **routes imbriquées** permettent de composer des écrans avec un **layout** commun.

## 4.1 Principe : `Outlet`

- Le parent fournit le layout.
- L’enfant se rend dans `<Outlet />`.

### Exemple : layout global

#### `layouts/RootLayout.jsx`

```jsx
import { Outlet } from "react-router-dom";
import Navbar from "../components/Navbar";

export default function RootLayout() {
  return (
    <div>
      <Navbar />
      <main style={{ padding: 16 }}>
        <Outlet />
      </main>
      <footer style={{ padding: 16, opacity: 0.7 }}>
        © Mon App
      </footer>
    </div>
  );
}
```

#### `App.jsx`

```jsx
import { Routes, Route } from "react-router-dom";
import RootLayout from "./layouts/RootLayout";
import Home from "./pages/Home";
import About from "./pages/About";
import NotFound from "./pages/NotFound";

export default function App() {
  return (
    <Routes>
      <Route path="/" element={<RootLayout />}>
        <Route index element={<Home />} />
        <Route path="about" element={<About />} />
        <Route path="*" element={<NotFound />} />
      </Route>
    </Routes>
  );
}
```

> Remarquez : sous un parent `path="/"`, les enfants ont souvent des `path` **relatifs** (`about` et non `/about`).

## 4.2 Layouts par section

Exemple : `/admin` avec un layout dédié.

```jsx
<Route path="/" element={<RootLayout />}>
  <Route index element={<Home />} />
  <Route path="products" element={<Products />} />

  <Route path="admin" element={<AdminLayout />}>
    <Route index element={<AdminHome />} />
    <Route path="users" element={<AdminUsers />} />
  </Route>
</Route>
```

---

# 5) Navigation programmatique et redirections

## 5.1 `useNavigate`

Utile après un submit de formulaire, une action de login, etc.

```jsx
import { useNavigate } from "react-router-dom";

export default function Contact() {
  const navigate = useNavigate();

  function handleSubmit(e) {
    e.preventDefault();
    // ... envoyer les données
    navigate("/", { replace: true });
  }

  return (
    <form onSubmit={handleSubmit}>
      <button>Envoyer</button>
    </form>
  );
}
```

- `replace: true` remplace l’entrée d’historique (évite retour sur la page de formulaire).

## 5.2 Aller/retour dans l’historique

```jsx
navigate(-1); // back
navigate(1);  // forward
```

## 5.3 Redirection déclarative : `Navigate`

```jsx
import { Navigate } from "react-router-dom";

function Legacy() {
  return <Navigate to="/new" replace />;
}
```

---

# 6) Paramètres d’URL & query string

## 6.1 Paramètres de route : `:id`

### Route

```jsx
<Route path="products/:id" element={<ProductDetails />} />
```

### Lecture : `useParams`

```jsx
import { useParams } from "react-router-dom";

export default function ProductDetails() {
  const { id } = useParams();

  return (
    <div>
      <h1>Détails produit</h1>
      <p>Produit id = {id}</p>
    </div>
  );
}
```

**Bonnes pratiques** :
- considérer `id` comme une **string** ; convertir si besoin (`Number(id)`), valider, gérer le cas absent.

## 6.2 Query string : `useSearchParams`

Exemple : `/products?category=books&page=2`

```jsx
import { useSearchParams } from "react-router-dom";

export default function Products() {
  const [searchParams, setSearchParams] = useSearchParams();

  const category = searchParams.get("category") ?? "all";
  const page = Number(searchParams.get("page") ?? 1);

  function setCategory(next) {
    setSearchParams((prev) => {
      prev.set("category", next);
      prev.set("page", "1");
      return prev;
    });
  }

  return (
    <section>
      <h1>Catalogue</h1>
      <p>Catégorie: {category} — Page: {page}</p>
      <button onClick={() => setCategory("books")}>Books</button>
      <button onClick={() => setCategory("games")}>Games</button>
    </section>
  );
}
```

---

# 7) Routes protégées (auth/roles)

Objectif : empêcher l’accès à certaines routes si l’utilisateur n’est pas authentifié.

## 7.1 Exemple de contexte d’auth simplifié

```jsx
import { createContext, useContext, useMemo, useState } from "react";

const AuthContext = createContext(null);

export function AuthProvider({ children }) {
  const [user, setUser] = useState(null); // { id, name, role }

  const value = useMemo(
    () => ({
      user,
      login: (u) => setUser(u),
      logout: () => setUser(null),
    }),
    [user]
  );

  return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>;
}

export function useAuth() {
  return useContext(AuthContext);
}
```

## 7.2 Composant `RequireAuth`

- Si non connecté → redirige vers `/login`
- On conserve la page cible via `location.state.from`

```jsx
import { Navigate, useLocation } from "react-router-dom";
import { useAuth } from "../auth/AuthContext";

export default function RequireAuth({ children }) {
  const { user } = useAuth();
  const location = useLocation();

  if (!user) {
    return <Navigate to="/login" replace state={{ from: location }} />;
  }

  return children;
}
```

## 7.3 Utilisation sur une route

```jsx
<Route
  path="admin"
  element={
    <RequireAuth>
      <AdminLayout />
    </RequireAuth>
  }
>
  <Route index element={<AdminHome />} />
</Route>
```

## 7.4 Redirection après login

```jsx
import { useLocation, useNavigate } from "react-router-dom";
import { useAuth } from "../auth/AuthContext";

export default function LoginPage() {
  const navigate = useNavigate();
  const location = useLocation();
  const { login } = useAuth();

  const from = location.state?.from?.pathname || "/";

  function handleLogin() {
    login({ id: 1, name: "Ada", role: "admin" });
    navigate(from, { replace: true });
  }

  return (
    <div>
      <h1>Login</h1>
      <button onClick={handleLogin}>Se connecter</button>
    </div>
  );
}
```

---

# 8) Chargement de données et erreurs (Data Routers)

Depuis React Router 6.4+, on peut définir des routes via un **router objet** et associer des fonctions de chargement.

## 8.1 Mise en place avec `createBrowserRouter`

```jsx
import React from "react";
import ReactDOM from "react-dom/client";
import {
  createBrowserRouter,
  RouterProvider,
} from "react-router-dom";

import RootLayout from "./layouts/RootLayout";
import Home from "./pages/Home";
import Products, { loader as productsLoader } from "./pages/Products";
import ProductDetails, { loader as productLoader } from "./pages/ProductDetails";
import NotFound from "./pages/NotFound";

const router = createBrowserRouter([
  {
    path: "/",
    element: <RootLayout />,
    errorElement: <NotFound />,
    children: [
      { index: true, element: <Home /> },
      {
        path: "products",
        element: <Products />,
        loader: productsLoader,
      },
      {
        path: "products/:id",
        element: <ProductDetails />,
        loader: productLoader,
      },
    ],
  },
]);

ReactDOM.createRoot(document.getElementById("root")).render(
  <React.StrictMode>
    <RouterProvider router={router} />
  </React.StrictMode>
);
```

## 8.2 `loader` + `useLoaderData`

### Exemple `Products` (loader)

```jsx
import { useLoaderData, Link } from "react-router-dom";

export async function loader() {
  // Exemple : fetch API
  const res = await fetch("https://fakestoreapi.com/products?limit=5");
  if (!res.ok) throw new Response("Erreur de chargement", { status: 500 });
  return res.json();
}

export default function Products() {
  const products = useLoaderData();

  return (
    <div>
      <h1>Produits</h1>
      <ul>
        {products.map((p) => (
          <li key={p.id}>
            <Link to={`/products/${p.id}`}>{p.title}</Link>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

### Exemple `ProductDetails` (loader + params)

```jsx
import { useLoaderData } from "react-router-dom";

export async function loader({ params }) {
  const res = await fetch(`https://fakestoreapi.com/products/${params.id}`);
  if (!res.ok) throw new Response("Produit introuvable", { status: 404 });
  return res.json();
}

export default function ProductDetails() {
  const product = useLoaderData();

  return (
    <article>
      <h1>{product.title}</h1>
      <p>{product.description}</p>
    </article>
  );
}
```

## 8.3 Gestion des erreurs avec `errorElement`

- `throw new Response(...)` dans un loader déclenche `errorElement`.
- Utile pour centraliser une UI d’erreur.

---

# 9) Optimisations & UX

## 9.1 Lazy-loading des pages

Pour réduire le bundle initial :

```jsx
import { Suspense, lazy } from "react";
import { Routes, Route } from "react-router-dom";

const Home = lazy(() => import("./pages/Home"));
const About = lazy(() => import("./pages/About"));

export default function App() {
  return (
    <Suspense fallback={<p>Chargement…</p>}>
      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/about" element={<About />} />
      </Routes>
    </Suspense>
  );
}
```

## 9.2 Scroll restoration (approche simple)

React Router ne force pas le scroll en haut à chaque navigation. Vous pouvez ajouter un composant :

```jsx
import { useEffect } from "react";
import { useLocation } from "react-router-dom";

export default function ScrollToTop() {
  const { pathname } = useLocation();

  useEffect(() => {
    window.scrollTo(0, 0);
  }, [pathname]);

  return null;
}
```

Puis dans le layout :

```jsx
<>
  <ScrollToTop />
  <Outlet />
</>
```

## 9.3 États de navigation (data routers)

Avec les data routers, on peut utiliser `useNavigation()` pour connaître l’état (idle/loading/submitting) et afficher un indicateur.

---

# 10) Tests & dépannage

## 10.1 Tester avec `MemoryRouter`

Idéal pour les tests unitaires de composants routés.

```jsx
import { render, screen } from "@testing-library/react";
import { MemoryRouter, Routes, Route } from "react-router-dom";
import ProductDetails from "./ProductDetails";

test("affiche l'ID produit", () => {
  render(
    <MemoryRouter initialEntries={["/products/42"]}>
      <Routes>
        <Route path="/products/:id" element={<ProductDetails />} />
      </Routes>
    </MemoryRouter>
  );

  expect(screen.getByText(/42/)).toBeInTheDocument();
});
```

## 10.2 Erreurs fréquentes

- **Refresh sur une route profonde** (ex: `/about`) renvoie 404 côté serveur :
  - solution : config du serveur pour renvoyer `index.html` sur toutes les routes (fallback).
- Confusion entre `path="/about"` et `path="about"` dans les routes imbriquées.
- Oubli de `end` sur un `NavLink` de `/`.

---

# 11) Exercices corrigés

## Exercice 1 — Mini app “Catalogue”

### Énoncé
Créer les pages :
- `/` Accueil
- `/products` Liste produits
- `/products/:id` Détails
- `*` NotFound

Ajouter une navbar avec `NavLink`.

### Correction (extrait)

```jsx
<Routes>
  <Route path="/" element={<RootLayout />}>
    <Route index element={<Home />} />
    <Route path="products" element={<Products />} />
    <Route path="products/:id" element={<ProductDetails />} />
    <Route path="*" element={<NotFound />} />
  </Route>
</Routes>
```

## Exercice 2 — Espace admin protégé

### Énoncé
- Ajouter `/login`
- Protéger `/admin` via `RequireAuth`
- Après login, rediriger vers la page initialement demandée.

### Correction (extrait)

```jsx
<Route path="login" element={<LoginPage />} />
<Route
  path="admin"
  element={
    <RequireAuth>
      <AdminLayout />
    </RequireAuth>
  }
>
  <Route index element={<AdminHome />} />
</Route>
```

---

## Annexes — Cheat sheet

### Composants & hooks principaux
- `<BrowserRouter>` : router HTML5
- `<Routes>` / `<Route>` : mapping URL → composant
- `<Link to="...">` : navigation
- `<NavLink>` : lien actif
- `<Outlet>` : rendu enfant dans un layout
- `useParams()` : paramètres de route
- `useSearchParams()` : query string
- `useNavigate()` : navigation programmatique
- `Navigate` : redirection

### Bonnes pratiques
- Structurer par **pages** (`/pages`) et **layouts** (`/layouts`).
- Centraliser la navbar dans un layout.
- Une route `*` pour 404.
- Préférer les data routers pour projets avec chargements/erreurs/formulaires complexes.

---

*Fin de la formation — Routing avec React Router.*
