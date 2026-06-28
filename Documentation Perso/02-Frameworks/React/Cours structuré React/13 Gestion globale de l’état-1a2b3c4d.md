# Formation React – Gestion globale de l’état

> **Public** : développeurs React (niveau débutant → intermédiaire)
>
> **Pré-requis** : JavaScript ES6+, bases de React (composants, props, state, hooks)
>
> **Objectif** : savoir **partager et synchroniser des données** entre plusieurs composants, en choisissant la bonne approche : **prop drilling**, **Context API**, ou une **librairie de state management** (Redux / Zustand).

---

## Plan de la formation

1. **Comprendre le problème** : état local vs état partagé
2. **Stratégies natives**
   1. Remonter l’état (lifting state up)
   2. Limites du prop drilling
3. **Context API**
   1. Quand utiliser Context
   2. Création d’un Context + Provider
   3. Consommation via `useContext`
   4. Structuration en “Domain contexts”
   5. Performance : re-render, séparation des contexts, memoization
   6. Context + `useReducer`
4. **Redux (Toolkit)**
   1. Pourquoi Redux sur des apps complexes
   2. Concepts : store, slice, actions, reducers, selectors
   3. Mise en place avec Redux Toolkit
   4. Bonnes pratiques d’architecture
5. **Zustand**
   1. Positionnement vs Redux
   2. Store minimaliste + hooks
   3. Selectors et performance
6. **Choisir la bonne solution** (matrice de décision)
7. **Atelier / exercices**
8. **Checklist** : erreurs fréquentes et bonnes pratiques

---

# 1) Comprendre le problème : état local vs état partagé

## 1.1 État local
Un état est dit **local** quand il ne concerne qu’un composant (ex. : ouverture d’un menu, valeur d’un input non réutilisée ailleurs, UI state).

Exemples :
- `isModalOpen`
- `searchText` dans une barre de recherche isolée
- `isLoading` d’un bouton

## 1.2 État partagé / global
Un état est **partagé** lorsqu’il est utilisé par **plusieurs composants** (souvent situés à des niveaux différents de l’arbre).

Exemples récurrents :
- utilisateur authentifié (`currentUser`)
- panier e-commerce (`cartItems`)
- thème (dark/light)
- préférences, feature flags

### Symptôme classic
Vous vous retrouvez à passer des props à travers de nombreux composants qui ne les utilisent pas… juste pour “transporter” l’information.

---

# 2) Stratégies natives

## 2.1 Lifting state up (remonter l’état)
Si **deux composants frères** ont besoin du même état, le réflexe standard en React est de **remonter l’état** dans leur parent commun.

```jsx
function Parent() {
  const [count, setCount] = React.useState(0);

  return (
    <>
      <CounterDisplay count={count} />
      <CounterButtons onInc={() => setCount(c => c + 1)} />
    </>
  );
}

function CounterDisplay({ count }) {
  return <p>Count: {count}</p>;
}

function CounterButtons({ onInc }) {
  return <button onClick={onInc}>+1</button>;
}
```

### Avantages
- Simple, idiomatique
- Facile à traquer
- Pas d’outil supplémentaire

### Limites
- Fonctionne bien tant que l’arbre n’est pas trop profond
- Devient lourd quand l’état doit être accessible “partout”

## 2.2 Prop drilling (et ses limites)
Le **prop drilling** : passer une prop à travers plusieurs niveaux.

```jsx
function App() {
  const [theme, setTheme] = React.useState("dark");
  return <Layout theme={theme} setTheme={setTheme} />;
}

function Layout({ theme, setTheme }) {
  return <Header theme={theme} setTheme={setTheme} />;
}

function Header({ theme, setTheme }) {
  return <ThemeToggle theme={theme} setTheme={setTheme} />;
}
```

### Problèmes
- Couplage entre composants
- API de composants “polluée”
- Refactor coûteux (le moindre changement remonte/descend)

---

# 3) Context API

## 3.1 Quand utiliser Context
Le Context est pertinent pour :
- des données **globales** ou **semi-globales**
- des données lues par **beaucoup de composants**
- des valeurs plutôt **stables** (ou mises à jour raisonnablement)

Exemples :
- `ThemeContext`
- `AuthContext`
- `I18nContext` (langue)

> À éviter pour un état qui change **très fréquemment** dans de gros arbres (risque de re-render massif), sauf si vous appliquez des techniques de séparation/selector.

## 3.2 Créer un Context + Provider

### 3.2.1 Version minimale
```jsx
import React from "react";

const ThemeContext = React.createContext(null);

export function ThemeProvider({ children }) {
  const [theme, setTheme] = React.useState("dark");

  const value = React.useMemo(() => ({ theme, setTheme }), [theme]);

  return (
    <ThemeContext.Provider value={value}>
      {children}
    </ThemeContext.Provider>
  );
}

export function useTheme() {
  const ctx = React.useContext(ThemeContext);
  if (!ctx) {
    throw new Error("useTheme must be used within ThemeProvider");
  }
  return ctx;
}
```

### 3.2.2 Utilisation
```jsx
function ThemeToggle() {
  const { theme, setTheme } = useTheme();

  return (
    <button
      onClick={() => setTheme(t => (t === "dark" ? "light" : "dark"))}
    >
      Current: {theme}
    </button>
  );
}

function App() {
  return (
    <ThemeProvider>
      <ThemeToggle />
    </ThemeProvider>
  );
}
```

## 3.3 Structurer par domaines
Évitez un seul “AppContext” géant. Préférez plusieurs contexts :
- `AuthProvider`
- `ThemeProvider`
- `CartProvider`

Cela :
- limite les re-renders
- clarifie les responsabilités
- facilite les tests

## 3.4 Performance et re-render
Le Context déclenche des re-renders des composants consommateurs lorsque la valeur (`value`) change.

### Bonnes pratiques
1. **Mémoïser** la valeur fournie (`useMemo`) si elle contient des objets/fonctions.
2. **Séparer** les contexts (ex. `ThemeValueContext` et `ThemeActionsContext`) quand les mises à jour sont fréquentes.
3. Découper : **un context par domaine**.

### Exemple : séparer “state” et “actions”
```jsx
const ThemeStateContext = React.createContext(null);
const ThemeActionsContext = React.createContext(null);

export function ThemeProvider({ children }) {
  const [theme, setTheme] = React.useState("dark");

  const actions = React.useMemo(
    () => ({
      toggle: () => setTheme(t => (t === "dark" ? "light" : "dark")),
      setDark: () => setTheme("dark"),
      setLight: () => setTheme("light"),
    }),
    []
  );

  return (
    <ThemeStateContext.Provider value={theme}>
      <ThemeActionsContext.Provider value={actions}>
        {children}
      </ThemeActionsContext.Provider>
    </ThemeStateContext.Provider>
  );
}

export const useThemeValue = () => {
  const v = React.useContext(ThemeStateContext);
  if (v == null) throw new Error("useThemeValue must be used within ThemeProvider");
  return v;
};

export const useThemeActions = () => {
  const a = React.useContext(ThemeActionsContext);
  if (!a) throw new Error("useThemeActions must be used within ThemeProvider");
  return a;
};
```

## 3.5 Context + useReducer (approche “mini-Redux”)
Quand la logique de mise à jour devient plus riche (actions, transitions), `useReducer` est populaire.

```jsx
const CartContext = React.createContext(null);

function cartReducer(state, action) {
  switch (action.type) {
    case "add": {
      const { id, name } = action.item;
      const existing = state.items[id];
      const qty = (existing?.qty ?? 0) + 1;
      return {
        ...state,
        items: { ...state.items, [id]: { id, name, qty } },
      };
    }
    case "remove": {
      const next = { ...state.items };
      delete next[action.id];
      return { ...state, items: next };
    }
    default:
      return state;
  }
}

export function CartProvider({ children }) {
  const [state, dispatch] = React.useReducer(cartReducer, { items: {} });

  const value = React.useMemo(() => ({ state, dispatch }), [state]);

  return <CartContext.Provider value={value}>{children}</CartContext.Provider>;
}

export function useCart() {
  const ctx = React.useContext(CartContext);
  if (!ctx) throw new Error("useCart must be used within CartProvider");
  return ctx;
}
```

**Avantage** : transitions explicites via `action.type`.

**Limites** :
- boilerplate (actions, reducer)
- performance/re-renders à surveiller

---

# 4) Redux (Redux Toolkit)

## 4.1 Pourquoi Redux ?
Redux devient intéressant quand :
- beaucoup de features partagent un état central
- besoin d’outils (DevTools, time travel/debug)
- logique métier complexe, actions nombreuses
- état normalisé, cache de données, etc.

> Aujourd’hui, on recommande **Redux Toolkit (RTK)** plutôt que Redux “vanilla”.

## 4.2 Concepts clés
- **Store** : source unique de vérité
- **Slice** : regroupement d’un sous-état + reducers + actions
- **Reducer** : fonction pure qui calcule le prochain état
- **Action** : événement décrivant “ce qui s’est passé”
- **Selector** : fonction de lecture (dérivation)

## 4.3 Installation
```bash
npm i @reduxjs/toolkit react-redux
```

## 4.4 Mise en place (exemple : auth)

### store.js
```js
import { configureStore } from "@reduxjs/toolkit";
import authReducer from "./authSlice";

export const store = configureStore({
  reducer: {
    auth: authReducer,
  },
});
```

### authSlice.js
```js
import { createSlice } from "@reduxjs/toolkit";

const authSlice = createSlice({
  name: "auth",
  initialState: {
    user: null,
    status: "anonymous", // anonymous | authenticated
  },
  reducers: {
    loginSuccess(state, action) {
      state.user = action.payload;
      state.status = "authenticated";
    },
    logout(state) {
      state.user = null;
      state.status = "anonymous";
    },
  },
});

export const { loginSuccess, logout } = authSlice.actions;
export default authSlice.reducer;
```

### Fournir le store
```jsx
import { Provider } from "react-redux";
import { store } from "./store";

function App() {
  return (
    <Provider store={store}>
      <Routes />
    </Provider>
  );
}
```

### Lire / écrire depuis un composant
```jsx
import { useDispatch, useSelector } from "react-redux";
import { loginSuccess, logout } from "./authSlice";

export function Profile() {
  const dispatch = useDispatch();
  const user = useSelector(state => state.auth.user);
  const status = useSelector(state => state.auth.status);

  if (status !== "authenticated") {
    return (
      <button
        onClick={() => dispatch(loginSuccess({ id: 1, name: "Ada" }))}
      >
        Login
      </button>
    );
  }

  return (
    <div>
      <p>Hello {user.name}</p>
      <button onClick={() => dispatch(logout())}>Logout</button>
    </div>
  );
}
```

## 4.5 Bonnes pratiques Redux
- utiliser **RTK** (`createSlice`, `configureStore`)
- factoriser des **selectors**
- éviter de mettre dans Redux du state purement UI local (modales, champs temporaires) sauf besoin transversal
- normaliser les données (ex. dictionnaire par id) si gros volumes

---

# 5) Zustand

## 5.1 Pourquoi Zustand ?
Zustand est un state manager :
- minimaliste
- très simple à introduire
- basé sur des hooks
- avec selectors pour limiter les re-renders

Installation :
```bash
npm i zustand
```

## 5.2 Store simple
```js
import { create } from "zustand";

export const useAuthStore = create(set => ({
  user: null,
  login: user => set({ user }),
  logout: () => set({ user: null }),
}));
```

Utilisation :
```jsx
import { useAuthStore } from "./useAuthStore";

function Profile() {
  const user = useAuthStore(s => s.user);
  const login = useAuthStore(s => s.login);
  const logout = useAuthStore(s => s.logout);

  if (!user) {
    return <button onClick={() => login({ id: 1, name: "Ada" })}>Login</button>;
  }

  return (
    <div>
      <p>Hello {user.name}</p>
      <button onClick={logout}>Logout</button>
    </div>
  );
}
```

## 5.3 Selectors et performance
Le pattern `useStore(selector)` permet de ne re-render que lorsque *la partie sélectionnée* change.

Exemple :
```jsx
const cartCount = useCartStore(s => s.items.length);
```

---

# 6) Choisir la bonne solution (matrice de décision)

| Besoin | Option recommandée |
|---|---|
| 2-3 composants frères partagent un state | Lifting state up |
| Donnée globale “stable” (theme, auth, i18n) | Context API |
| Logique de transitions complexe mais scope limité | Context + useReducer |
| App complexe, grandes équipes, besoin DevTools & conventions | Redux Toolkit |
| Besoin d’un store global simple, rapide à intégrer, peu de boilerplate | Zustand |

Règle pratique :
1. commencer simple (state local + lifting)
2. ajouter Context quand prop drilling devient pénible
3. passer à une librairie quand : complexité/échelle/outillage le justifient

---

# 7) Atelier (exercices)

## Exercice 1 — Theme (Context)
**Objectif** : créer `ThemeProvider`, `useThemeValue`, `useThemeActions` et afficher un bouton toggle dans le header.

Critères :
- pas de prop drilling
- valeur mémorisée

## Exercice 2 — Auth (Redux Toolkit)
**Objectif** : créer un `authSlice` avec `loginSuccess` et `logout`, puis afficher un `Profile`.

Critères :
- selectors dédiés (`selectUser`, `selectAuthStatus`)

## Exercice 3 — Cart (Zustand)
**Objectif** : gérer une liste d’items avec `addItem`, `removeItem`, `clear`.

Critères :
- utiliser des selectors
- calculer `totalItems`

---

# 8) Checklist & erreurs fréquentes

## Checklist
- [ ] L’état UI local reste local autant que possible
- [ ] Les contexts sont séparés par domaines
- [ ] `value` du Provider est stable (`useMemo`)
- [ ] Les hooks custom `useX()` vérifient la présence du Provider
- [ ] Les librairies (Redux/Zustand) sont introduites pour une vraie raison (échelle, complexité, outillage)

## Erreurs fréquentes
- Mettre **tout** dans un seul Context, et déclencher des re-renders partout
- Passer des objets/fonctions non mémoïsés dans `value`
- Utiliser Redux pour gérer des états locaux temporaires sans besoin transversal

---

## Annexes – mini glossaire
- **Prop drilling** : faire transiter des props à travers des composants intermédiaires.
- **Provider** : composant fournissant une valeur de context.
- **Selector** : fonction qui extrait une partie de l’état.
- **Reducer** : fonction pure décrivant la transition d’état.
