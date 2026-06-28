# 18 — Tests en React (Jest + React Testing Library)

> **Public** : développeurs React (débutant → intermédiaire)
>
> **Objectif** : apprendre à écrire des tests fiables et maintenables pour des composants React avec **Jest** et **React Testing Library (RTL)**, afin de vérifier le comportement, prévenir les régressions et améliorer la qualité du code.

---

## Plan de la formation

1. [Pourquoi tester ?](#1-pourquoi-tester-)
2. [Panorama des types de tests](#2-panorama-des-types-de-tests)
3. [Outillage : Jest & React Testing Library](#3-outillage--jest--react-testing-library)
4. [Démarrer : configuration et conventions](#4-démarrer--configuration-et-conventions)
5. [Fondamentaux React Testing Library](#5-fondamentaux-react-testing-library)
6. [Écrire des tests de composants (render + assertions)](#6-écrire-des-tests-de-composants-render--assertions)
7. [Tester les interactions utilisateur](#7-tester-les-interactions-utilisateur)
8. [Tester l’asynchrone : API, délais, états de chargement](#8-tester-lasynchrone--api-délais-états-de-chargement)
9. [Mocks : modules, fonctions, fetch/axios, timers](#9-mocks--modules-fonctions-fetchaxios-timers)
10. [Tester les formulaires](#10-tester-les-formulaires)
11. [Tester le routage (React Router)](#11-tester-le-routage-react-router)
12. [Tester le state management (Context / Redux)](#12-tester-le-state-management-context--redux)
13. [Organisation, lisibilité, anti-patterns](#13-organisation-lisibilité-anti-patterns)
14. [Couverture, CI et bonnes pratiques](#14-couverture-ci-et-bonnes-pratiques)
15. [Atelier final guidé](#15-atelier-final-guidé)
16. [Annexes : snippets & checklists](#16-annexes--snippets--checklists)

---

## 1. Pourquoi tester ?

Les tests servent à :

- **Vérifier le comportement** réel d’un composant (ce que l’utilisateur voit et fait).
- **Prévenir les régressions** en sécurisant les évolutions.
- **Documenter le code** : un test clair explique l’usage attendu.
- **Faciliter le refactoring** en gardant une “ceinture de sécurité”.
- **Améliorer la qualité** : moins de bugs en production, plus de confiance.

> En React, on cherche généralement à tester **le comportement observable** plutôt que des détails d’implémentation.

---

## 2. Panorama des types de tests

### 2.1 Tests unitaires
- Ciblent une fonction/utilitaire.
- Rapides, nombreux.

### 2.2 Tests de composants (intégration “légère”)
- Rendent un composant et vérifient ce qui s’affiche.
- Simulent des interactions utilisateur.
- Typiquement réalisés avec **RTL**.

### 2.3 Tests End-to-End (E2E)
- Testent l’application dans un navigateur (Cypress/Playwright).
- Plus lents, mais très représentatifs.

**Dans ce cours** : focus sur **Jest + RTL**, c’est-à-dire tests unitaires & tests de composants.

---

## 3. Outillage : Jest & React Testing Library

### 3.1 Jest
- Runner de tests (exécute les tests)
- Assertions et mocks
- Snapshot (à utiliser avec prudence)

### 3.2 React Testing Library
- API centrée sur l’utilisateur : queries `getByRole`, `getByLabelText`, etc.
- Encouragement des bonnes pratiques d’accessibilité
- Évite de tester les détails internes (state, méthodes privées)

### 3.3 user-event
- Simulation d’interactions plus réalistes que `fireEvent`

---

## 4. Démarrer : configuration et conventions

> Selon votre stack, l’installation varie. Exemples ci-dessous.

### 4.1 Créer un projet (au choix)
- **Vite** (souvent privilégié) : `npm create vite@latest`
- **Create React App** (legacy mais encore rencontré)
- **Next.js** (configuration spécifique)

### 4.2 Installation (exemple Vite + React)
```bash
npm i -D jest @testing-library/react @testing-library/jest-dom @testing-library/user-event
```

### 4.3 Ajouter jest-dom
Dans un fichier de setup (ex : `src/setupTests.ts` ou `jest.setup.js`) :
```ts
import '@testing-library/jest-dom';
```

### 4.4 Conventions de nommage
- Fichiers : `Component.test.tsx` ou `Component.spec.tsx`
- Tests regroupés par `describe`
- Utiliser le pattern **Arrange / Act / Assert**

Exemple :
```ts
describe('Button', () => {
  it('affiche le libellé', () => {
    // Arrange
    // Act
    // Assert
  });
});
```

---

## 5. Fondamentaux React Testing Library

### 5.1 render
```ts
import { render, screen } from '@testing-library/react';

render(<MyComponent />);
```

### 5.2 screen
`screen` représente le “point d’entrée” pour interroger le DOM rendu.

### 5.3 Les queries (ordre recommandé)
RTL recommande d’interroger le DOM comme un utilisateur :

1. `getByRole` (souvent le meilleur)
2. `getByLabelText` (formulaires)
3. `getByPlaceholderText`
4. `getByText`
5. `getByDisplayValue`
6. `getByAltText`
7. `getByTitle`
8. `getByTestId` (à éviter si possible)

### 5.4 getBy / queryBy / findBy
- `getBy...` : lève une erreur si non trouvé (synchrone)
- `queryBy...` : retourne `null` si non trouvé (utile pour absence)
- `findBy...` : async (attend l’apparition, basée sur `waitFor`)

Exemples :
```ts
screen.getByRole('button', { name: /envoyer/i });
expect(screen.queryByText(/erreur/i)).not.toBeInTheDocument();
await screen.findByText(/chargé/i);
```

---

## 6. Écrire des tests de composants (render + assertions)

### 6.1 Exemple simple : composant d’affichage
**Composant** :
```tsx
// Greeting.tsx
export function Greeting({ name }: { name: string }) {
  return <h1>Bonjour {name}</h1>;
}
```

**Test** :
```tsx
import { render, screen } from '@testing-library/react';
import { Greeting } from './Greeting';

describe('Greeting', () => {
  it('affiche le prénom', () => {
    render(<Greeting name="Ada" />);
    expect(screen.getByRole('heading', { name: 'Bonjour Ada' })).toBeInTheDocument();
  });
});
```

### 6.2 Tester une condition d’affichage
**Composant** :
```tsx
export function Message({ isAdmin }: { isAdmin: boolean }) {
  return (
    <div>
      {isAdmin ? <strong>Accès admin</strong> : <span>Accès utilisateur</span>}
    </div>
  );
}
```

**Test** :
```tsx
import { render, screen } from '@testing-library/react';
import { Message } from './Message';

it('affiche le message admin si isAdmin=true', () => {
  render(<Message isAdmin={true} />);
  expect(screen.getByText(/accès admin/i)).toBeInTheDocument();
});

it('affiche le message utilisateur si isAdmin=false', () => {
  render(<Message isAdmin={false} />);
  expect(screen.getByText(/accès utilisateur/i)).toBeInTheDocument();
});
```

---

## 7. Tester les interactions utilisateur

### 7.1 user-event
`user-event` simule des actions proches du navigateur : clic, saisie, tabulation…

**Composant** :
```tsx
import { useState } from 'react';

export function Counter() {
  const [count, setCount] = useState(0);

  return (
    <div>
      <p>Compteur : {count}</p>
      <button onClick={() => setCount((c) => c + 1)}>Incrémenter</button>
    </div>
  );
}
```

**Test** :
```tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { Counter } from './Counter';

it('incrémente au clic', async () => {
  const user = userEvent.setup();
  render(<Counter />);

  await user.click(screen.getByRole('button', { name: /incrémenter/i }));

  expect(screen.getByText(/compteur : 1/i)).toBeInTheDocument();
});
```

### 7.2 Tester le focus / navigation clavier
Exemple :
```ts
await user.tab();
expect(screen.getByRole('button', { name: /incrémenter/i })).toHaveFocus();
```

---

## 8. Tester l’asynchrone : API, délais, états de chargement

### 8.1 Exemple : chargement de données
**Composant** :
```tsx
import { useEffect, useState } from 'react';

type User = { id: number; name: string };

export function UsersList() {
  const [users, setUsers] = useState<User[] | null>(null);

  useEffect(() => {
    let alive = true;
    fetch('/api/users')
      .then((r) => r.json())
      .then((data) => {
        if (alive) setUsers(data);
      });
    return () => {
      alive = false;
    };
  }, []);

  if (!users) return <p>Chargement…</p>;

  return (
    <ul>
      {users.map((u) => (
        <li key={u.id}>{u.name}</li>
      ))}
    </ul>
  );
}
```

### 8.2 Test avec `findBy...`
On mock `fetch` et on attend l’apparition des éléments.
```tsx
import { render, screen } from '@testing-library/react';
import { UsersList } from './UsersList';

beforeEach(() => {
  // @ts-expect-error - on remplace fetch global dans le test
  global.fetch = jest.fn();
});

afterEach(() => {
  jest.resetAllMocks();
});

it('affiche les utilisateurs après chargement', async () => {
  (global.fetch as jest.Mock).mockResolvedValue({
    json: async () => [
      { id: 1, name: 'Ada' },
      { id: 2, name: 'Linus' },
    ],
  });

  render(<UsersList />);

  expect(screen.getByText(/chargement/i)).toBeInTheDocument();

  expect(await screen.findByText('Ada')).toBeInTheDocument();
  expect(screen.getByText('Linus')).toBeInTheDocument();
});
```

### 8.3 `waitFor` (quand nécessaire)
Utiliser `waitFor` pour attendre une condition plus complexe.
```ts
import { waitFor } from '@testing-library/react';

await waitFor(() => {
  expect(screen.queryByText(/chargement/i)).not.toBeInTheDocument();
});
```

---

## 9. Mocks : modules, fonctions, fetch/axios, timers

### 9.1 Mock d’un module
Si votre composant dépend d’un module utilitaire :

```ts
// math.ts
export const sum = (a: number, b: number) => a + b;
```

```ts
jest.mock('./math', () => ({
  sum: jest.fn(() => 42),
}));
```

### 9.2 Spy sur une fonction
```ts
import * as math from './math';

const spy = jest.spyOn(math, 'sum').mockReturnValue(10);
// ... test
spy.mockRestore();
```

### 9.3 Mock des timers
Utile si le composant utilise `setTimeout` / `setInterval`.
```ts
jest.useFakeTimers();

// ... déclencher l’action
jest.advanceTimersByTime(1000);

jest.useRealTimers();
```

> Préférez cependant l’attente basée sur le comportement utilisateur quand c’est possible.

---

## 10. Tester les formulaires

### 10.1 Exemple : login simple
**Composant** :
```tsx
import { useState } from 'react';

export function LoginForm({ onSubmit }: { onSubmit: (email: string) => void }) {
  const [email, setEmail] = useState('');

  return (
    <form
      onSubmit={(e) => {
        e.preventDefault();
        onSubmit(email);
      }}
    >
      <label htmlFor="email">Email</label>
      <input
        id="email"
        type="email"
        value={email}
        onChange={(e) => setEmail(e.target.value)}
      />
      <button type="submit">Se connecter</button>
    </form>
  );
}
```

**Test** :
```tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { LoginForm } from './LoginForm';

it('soumet l’email saisi', async () => {
  const user = userEvent.setup();
  const onSubmit = jest.fn();

  render(<LoginForm onSubmit={onSubmit} />);

  await user.type(screen.getByLabelText(/email/i), 'dev@site.com');
  await user.click(screen.getByRole('button', { name: /se connecter/i }));

  expect(onSubmit).toHaveBeenCalledWith('dev@site.com');
  expect(onSubmit).toHaveBeenCalledTimes(1);
});
```

### 10.2 Tester les erreurs de validation
Stratégie :
- Vérifier le message d’erreur rendu
- Vérifier que `onSubmit` n’est pas appelé

---

## 11. Tester le routage (React Router)

### 11.1 Rendre avec un Router en test
```tsx
import { MemoryRouter, Routes, Route } from 'react-router-dom';
import { render, screen } from '@testing-library/react';

function AppRoutes() {
  return (
    <Routes>
      <Route path="/" element={<h1>Home</h1>} />
      <Route path="/about" element={<h1>About</h1>} />
    </Routes>
  );
}

it('affiche About sur /about', () => {
  render(
    <MemoryRouter initialEntries={["/about"]}>
      <AppRoutes />
    </MemoryRouter>
  );

  expect(screen.getByRole('heading', { name: 'About' })).toBeInTheDocument();
});
```

### 11.2 Tester la navigation
- Clic sur un lien
- Vérifier que la page attendue s’affiche

---

## 12. Tester le state management (Context / Redux)

### 12.1 Avec Context
Créer un helper `renderWithProviders`.

**Exemple (simplifié)** :
```tsx
import { createContext, useContext } from 'react';
import { render } from '@testing-library/react';

const ThemeContext = createContext<'light' | 'dark'>('light');

export function useTheme() {
  return useContext(ThemeContext);
}

export function ThemedTitle() {
  const theme = useTheme();
  return <h1>Theme: {theme}</h1>;
}

export function renderWithTheme(ui: React.ReactNode, theme: 'light' | 'dark') {
  return render(<ThemeContext.Provider value={theme}>{ui}</ThemeContext.Provider>);
}
```

**Test** :
```tsx
import { screen } from '@testing-library/react';
import { ThemedTitle, renderWithTheme } from './ThemedTitle';

it('affiche le thème fourni', () => {
  renderWithTheme(<ThemedTitle />, 'dark');
  expect(screen.getByRole('heading', { name: 'Theme: dark' })).toBeInTheDocument();
});
```

### 12.2 Avec Redux (principe)
- Utiliser `Provider`
- Injecter un store de test
- Optionnel : utiliser des factories/fixtures pour préparer l’état initial

---

## 13. Organisation, lisibilité, anti-patterns

### 13.1 Bonnes pratiques
- Nommer les tests comme des comportements : *"affiche …", "permet de …"*
- Préférer `getByRole` + `name` pour robustesse et accessibilité
- Utiliser `userEvent` plutôt que `fireEvent`
- Tester ce qui est **observable**

### 13.2 Anti-patterns fréquents
- Tester l’état interne (`useState`) plutôt que le rendu
- Utiliser `data-testid` partout (à réserver aux cas difficiles)
- Trop de snapshots (fragiles)
- Tests trop couplés à la structure DOM (ex: `.children[0]`)

### 13.3 Structure recommandée
- `__tests__/` ou co-location : `Component.tsx` + `Component.test.tsx`
- Helpers de rendu : `test/utils.tsx` (providers, router, etc.)

---

## 14. Couverture, CI et bonnes pratiques

### 14.1 Couverture
- La couverture aide à repérer des zones non testées, mais ne garantit pas la qualité.
- Objectif réaliste : couvrir les comportements critiques.

Commande typique :
```bash
npm test -- --coverage
```

### 14.2 Intégration CI
- Lancer les tests sur chaque PR
- Activer `--runInBand` en CI si nécessaire (selon ressources)
- Échouer le build si tests KO

### 14.3 Règles d’or
- Un test doit être **fiable** (pas flaky)
- Un test doit être **rapide**
- Un test doit être **lisible**

---

## 15. Atelier final guidé

### Objectif
Tester un composant “réaliste” combinant :
- chargement asynchrone,
- gestion d’erreur,
- interaction utilisateur.

### Composant : ProductSearch
**Spécifications** :
1. Affiche un champ de recherche et un bouton.
2. Au clic sur “Rechercher”, appelle `fetch('/api/products?q=...')`.
3. Affiche “Chargement…” pendant l’appel.
4. Affiche la liste des produits si succès.
5. Affiche un message d’erreur si échec.

**Implémentation possible** (exemple) :
```tsx
import { useState } from 'react';

type Product = { id: number; name: string };

export function ProductSearch() {
  const [q, setQ] = useState('');
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);
  const [items, setItems] = useState<Product[]>([]);

  async function onSearch() {
    setLoading(true);
    setError(null);
    try {
      const res = await fetch(`/api/products?q=${encodeURIComponent(q)}`);
      const data = await res.json();
      setItems(data);
    } catch (e) {
      setError('Une erreur est survenue');
    } finally {
      setLoading(false);
    }
  }

  return (
    <div>
      <label htmlFor="q">Recherche</label>
      <input id="q" value={q} onChange={(e) => setQ(e.target.value)} />
      <button onClick={onSearch}>Rechercher</button>

      {loading && <p>Chargement…</p>}
      {error && <p role="alert">{error}</p>}

      <ul>
        {items.map((p) => (
          <li key={p.id}>{p.name}</li>
        ))}
      </ul>
    </div>
  );
}
```

### Tests attendus
1. **Saisie + appel** : tape une recherche, clique, vérifie que `fetch` est appelé avec la bonne URL.
2. **Loading** : vérifie que “Chargement…” apparaît puis disparaît.
3. **Succès** : vérifie que les produits sont affichés.
4. **Erreur** : vérifie que le `role="alert"` apparaît.

**Exemple de test (succès)** :
```tsx
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { ProductSearch } from './ProductSearch';

beforeEach(() => {
  // @ts-expect-error
  global.fetch = jest.fn();
});

afterEach(() => {
  jest.resetAllMocks();
});

it('affiche les produits après recherche', async () => {
  const user = userEvent.setup();

  (global.fetch as jest.Mock).mockResolvedValue({
    json: async () => [
      { id: 1, name: 'Clavier' },
      { id: 2, name: 'Souris' },
    ],
  });

  render(<ProductSearch />);

  await user.type(screen.getByLabelText(/recherche/i), 'pc');
  await user.click(screen.getByRole('button', { name: /rechercher/i }));

  expect(screen.getByText(/chargement/i)).toBeInTheDocument();

  expect(await screen.findByText('Clavier')).toBeInTheDocument();
  expect(screen.getByText('Souris')).toBeInTheDocument();

  await waitFor(() => {
    expect(screen.queryByText(/chargement/i)).not.toBeInTheDocument();
  });

  expect(global.fetch).toHaveBeenCalledWith('/api/products?q=pc');
});
```

---

## 16. Annexes : snippets & checklists

### 16.1 Checklist : écrire un bon test RTL
- [ ] Est-ce que je teste un **comportement utilisateur** ?
- [ ] Est-ce que j’utilise une query robuste (`getByRole`) ?
- [ ] Est-ce que le test est lisible (AAA) ?
- [ ] Est-ce que le test est stable (pas de timing fragile) ?
- [ ] Est-ce que j’ai mocké seulement ce qui est nécessaire ?

### 16.2 Snippet : render avec Router
```tsx
import { MemoryRouter } from 'react-router-dom';
import { render } from '@testing-library/react';

export function renderWithRouter(ui: React.ReactNode, { route = '/' } = {}) {
  return render(<MemoryRouter initialEntries={[route]}>{ui}</MemoryRouter>);
}
```

### 16.3 Snippet : render avec Providers
```tsx
import { render } from '@testing-library/react';

export function renderWithProviders(ui: React.ReactNode) {
  // Ajouter ici Provider Redux, QueryClientProvider, i18n, etc.
  return render(<>{ui}</>);
}
```

---

## Résumé

- **Jest** exécute vos tests, fournit mocks/spies/timers.
- **React Testing Library** vous pousse à tester **ce que l’utilisateur voit et fait**.
- Les bons tests React sont centrés sur : rendu, accessibilité, interactions, asynchrone.
- Une suite de tests solide améliore la qualité du code et sécurise les évolutions.
