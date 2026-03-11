# Formation : Tests (Node.js & React)

> **Objectif** : maîtriser les tests pour vérifier le bon fonctionnement du code, en mettant en place des tests unitaires et d’intégration avec des frameworks tels que **Jest** et **Mocha**.

---

## 1) Informations générales

- **Public** : développeurs Node.js / React (débutant à intermédiaire en tests)
- **Pré-requis** : JavaScript/TypeScript, npm/yarn/pnpm, bases Node.js, bases React
- **Durée suggérée** : 1 à 2 jours (7h à 14h)
- **Format** : théorie + démonstrations + ateliers guidés + exercices

### Compétences visées

À la fin, le participant saura :

- Expliquer le rôle des tests et les différents niveaux (unitaire, intégration, E2E)
- Mettre en place un environnement de test sur un projet Node.js et/ou React
- Écrire des tests unitaires robustes avec **Jest** (et comprendre **Mocha**)
- Tester le code asynchrone (promises, async/await, timers)
- Utiliser des **mocks**, **spies**, **stubs** et des fixtures
- Tester une API HTTP (express/fastify) via des tests d’intégration
- Tester des composants React (React Testing Library)
- Mesurer et interpréter la **couverture de code**
- Intégrer l’exécution des tests dans une CI

---

## 2) Plan de formation

1. **Introduction aux tests**
   - Pourquoi tester ?
   - Types de tests : unitaires, intégration, E2E
   - Notions : AAA (Arrange-Act-Assert), test pyramid, déterminisme
2. **Outils et écosystème**
   - Jest : principe, atouts, configuration
   - Mocha + Chai + Sinon : philosophie et usage
   - Organisation des fichiers, conventions de nommage
3. **Tests unitaires Node.js avec Jest**
   - Assertions, matchers, snapshots (prudence)
   - Tests asynchrones
   - Paramétrisation et tests de table
4. **Isolation : mocks, spies, stubs**
   - Mock de modules, fonctions, dates, timers
   - Bonnes pratiques pour éviter les tests fragiles
5. **Tests d’intégration Node.js**
   - API HTTP : supertest, lancement/arrêt serveur
   - DB : stratégies (mocks vs conteneurs vs sqlite in-memory)
   - Gestion des fixtures et nettoyage
6. **Tests React**
   - Principes (tester le comportement, pas l’implémentation)
   - React Testing Library : render, queries, events
   - Mock des appels réseau
7. **Qualité et industrialisation**
   - Couverture (lines/branches/functions)
   - Linting, formatage, pré-commit
   - Intégration CI (GitHub Actions / GitLab CI)
8. **Ateliers et exercices de synthèse**

---

## 3) Introduction : pourquoi tester ?

Les tests permettent de **vérifier le bon fonctionnement du code** et d’éviter les régressions lors de l’évolution d’un projet.

### Bénéfices concrets

- **Régression** : détecter rapidement qu’une modification casse un comportement existant
- **Confiance** : livrer plus souvent avec moins de risques
- **Documentation vivante** : un test décrit le comportement attendu
- **Design** : un code testable pousse à mieux séparer les responsabilités

### Niveaux de tests

- **Tests unitaires** : valident une petite unité (fonction, module) de manière isolée
- **Tests d’intégration** : valident plusieurs composants ensemble (API + DB, routing + services)
- **E2E (end-to-end)** : valident un parcours utilisateur de bout en bout (souvent via navigateur)

> **Idée clé** : plus un test est haut niveau, plus il est coûteux et fragile. D’où l’intérêt de la **pyramide de tests** (beaucoup d’unitaires, moins d’intégration, peu d’E2E).

### AAA : Arrange – Act – Assert

Structure simple d’un test :

1. **Arrange** : préparer les données / mocks
2. **Act** : exécuter la fonction / déclencher l’action
3. **Assert** : vérifier le résultat

---

## 4) Outils : Jest et Mocha

### 4.1 Jest (recommandé pour démarrer)

**Jest** est un framework tout-en-un : runner, assertions, mocking, snapshots, couverture.

Points forts :

- Configuration minimale
- Très bon support TypeScript et React
- Mocking intégré et simple

### 4.2 Mocha (avec Chai et Sinon)

**Mocha** est un test runner flexible. Souvent combiné avec :

- **Chai** : assertions (*expect/should*)
- **Sinon** : spies, stubs, mocks

Mocha est souvent utilisé dans des projets existants, ou lorsqu’on veut un stack très modulaire.

---

## 5) Mise en place d’un projet de test (Node.js)

### 5.1 Installation minimale Jest

```bash
npm init -y
npm i -D jest
```

Dans `package.json` :

```json
{
  "scripts": {
    "test": "jest",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage"
  }
}
```

### 5.2 Convention de fichiers

- `src/` : code
- `tests/` ou tests colocalisés : `*.test.js` / `*.spec.js`

Exemples :

- `src/math.js`
- `src/math.test.js`

---

## 6) Tests unitaires avec Jest (Node.js)

### 6.1 Exemple simple

`src/math.js`

```js
export function sum(a, b) {
  return a + b;
}
```

`src/math.test.js`

```js
import { sum } from "./math.js";

test("sum additionne deux nombres", () => {
  // Arrange
  const a = 2;
  const b = 3;

  // Act
  const result = sum(a, b);

  // Assert
  expect(result).toBe(5);
});
```

### 6.2 Matchers utiles

- `toBe()` : égalité stricte (primitives)
- `toEqual()` : égalité profonde (objets)
- `toHaveLength()` : taille
- `toThrow()` : exceptions
- `toMatch()` : regex

Exemple :

```js
test("valide une structure", () => {
  const user = { id: 1, roles: ["admin"] };
  expect(user).toEqual({ id: 1, roles: ["admin"] });
  expect(user.roles).toHaveLength(1);
});
```

### 6.3 Tester les erreurs

```js
export function divide(a, b) {
  if (b === 0) throw new Error("Division by zero");
  return a / b;
}
```

```js
import { divide } from "./divide.js";

test("divide lève une erreur si b=0", () => {
  expect(() => divide(10, 0)).toThrow("Division by zero");
});
```

---

## 7) Tests asynchrones (promises, async/await, callbacks)

### 7.1 Promises

```js
export function fetchUser() {
  return Promise.resolve({ id: 1, name: "Ada" });
}
```

```js
import { fetchUser } from "./fetchUser.js";

test("fetchUser retourne un user", () => {
  return fetchUser().then((user) => {
    expect(user.name).toBe("Ada");
  });
});
```

### 7.2 async/await

```js
test("fetchUser retourne un user (async/await)", async () => {
  const user = await fetchUser();
  expect(user).toEqual({ id: 1, name: "Ada" });
});
```

### 7.3 Timers

Jest permet de contrôler le temps :

```js
test("timer", () => {
  jest.useFakeTimers();

  const fn = jest.fn();
  setTimeout(fn, 1000);

  expect(fn).not.toHaveBeenCalled();

  jest.advanceTimersByTime(1000);
  expect(fn).toHaveBeenCalledTimes(1);

  jest.useRealTimers();
});
```

---

## 8) Paramétrisation et tests de table

Pratique pour tester de nombreuses entrées/sorties.

```js
import { sum } from "./math.js";

describe("sum", () => {
  test.each([
    [1, 1, 2],
    [2, 3, 5],
    [-1, 1, 0]
  ])("sum(%i, %i) = %i", (a, b, expected) => {
    expect(sum(a, b)).toBe(expected);
  });
});
```

---

## 9) Isolation : mocks, spies, stubs

### 9.1 Spy : observer un appel

```js
const calculator = {
  add(a, b) {
    return a + b;
  }
};

test("spy sur add", () => {
  const spy = jest.spyOn(calculator, "add");
  calculator.add(1, 2);

  expect(spy).toHaveBeenCalledWith(1, 2);
  spy.mockRestore();
});
```

### 9.2 Mock : remplacer une implémentation

```js
test("mock de fonction", () => {
  const fn = jest.fn().mockReturnValue(42);
  expect(fn("x")).toBe(42);
  expect(fn).toHaveBeenCalledWith("x");
});
```

### 9.3 Mock d’un module (exemple)

`src/config.js`

```js
export function getEnv() {
  return process.env.NODE_ENV || "development";
}
```

`src/service.js`

```js
import { getEnv } from "./config.js";

export function getBaseUrl() {
  return getEnv() === "production" ? "https://api.prod" : "http://localhost:3000";
}
```

`src/service.test.js`

```js
jest.mock("./config.js", () => ({
  getEnv: () => "production"
}));

import { getBaseUrl } from "./service.js";

test("getBaseUrl utilise l'env mockée", () => {
  expect(getBaseUrl()).toBe("https://api.prod");
});
```

### Bonnes pratiques de mocking

- Ne mockez que ce qui est **lent**, **aléatoire** ou **externe** (réseau, temps, filesystem)
- Préférez tester des **résultats** (output) plutôt que des détails d’implémentation
- Évitez les mocks cascades qui rendent le test illisible

---

## 10) Tests d’intégration Node.js (API HTTP)

### 10.1 Exemple API Express

`src/app.js`

```js
import express from "express";

export function createApp() {
  const app = express();
  app.use(express.json());

  app.get("/health", (req, res) => {
    res.json({ ok: true });
  });

  app.post("/sum", (req, res) => {
    const { a, b } = req.body;
    res.json({ result: a + b });
  });

  return app;
}
```

Test d’intégration :

```bash
npm i -D supertest express
```

`src/app.test.js`

```js
import request from "supertest";
import { createApp } from "./app.js";

describe("API", () => {
  test("GET /health", async () => {
    const app = createApp();
    const res = await request(app).get("/health");

    expect(res.status).toBe(200);
    expect(res.body).toEqual({ ok: true });
  });

  test("POST /sum", async () => {
    const app = createApp();
    const res = await request(app).post("/sum").send({ a: 2, b: 3 });

    expect(res.status).toBe(200);
    expect(res.body).toEqual({ result: 5 });
  });
});
```

### 10.2 Stratégies d’intégration DB

- **Mock repository** : rapide, mais moins réaliste
- **DB in-memory** (ex: sqlite) : bon compromis
- **Containers** (Docker/Testcontainers) : proche prod, plus lent

> Choix guidé par le contexte : CI, temps d’exécution, criticité.

---

## 11) Tests React (composants)

### 11.1 Principes

- Tester **ce que voit et fait** l’utilisateur : texte, boutons, interactions
- Éviter de tester directement l’état interne ou la structure DOM fragile

### 11.2 Stack recommandée

- **Jest** (runner)
- **React Testing Library** (RTL)
- **@testing-library/jest-dom** (matchers DOM)

Installation (exemple) :

```bash
npm i -D @testing-library/react @testing-library/jest-dom
```

### 11.3 Exemple : composant simple

`src/Counter.jsx`

```jsx
import { useState } from "react";

export function Counter() {
  const [count, setCount] = useState(0);

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount((c) => c + 1)}>Increment</button>
    </div>
  );
}
```

`src/Counter.test.jsx`

```jsx
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { Counter } from "./Counter";

test("incrémente le compteur", async () => {
  const user = userEvent.setup();
  render(<Counter />);

  expect(screen.getByText(/Count: 0/i)).toBeInTheDocument();

  await user.click(screen.getByRole("button", { name: /increment/i }));
  expect(screen.getByText(/Count: 1/i)).toBeInTheDocument();
});
```

### 11.4 Mock des appels réseau

Approches :

- Mock de `fetch` (simple)
- MSW (Mock Service Worker) (souvent plus réaliste)

Exemple simple avec `global.fetch` :

```js
beforeEach(() => {
  global.fetch = jest.fn();
});

afterEach(() => {
  jest.resetAllMocks();
});

test("affiche des données chargées", async () => {
  fetch.mockResolvedValue({
    ok: true,
    json: async () => ({ title: "Hello" })
  });

  // ...render component that calls fetch
});
```

---

## 12) Couverture de code

Exécuter :

```bash
npm run test:coverage
```

Notions :

- **Lines** : lignes exécutées
- **Statements** : instructions
- **Functions** : fonctions appelées
- **Branches** : chemins conditionnels

Bonnes pratiques :

- Un taux élevé n’est pas une garantie de qualité
- Visez plutôt une couverture utile sur les zones critiques (business logic)

---

## 13) Qualité et CI

### 13.1 Exécution en CI (exemple GitHub Actions)

`.github/workflows/test.yml`

```yml
name: test
on:
  push:
  pull_request:

jobs:
  unit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - run: npm ci
      - run: npm test
```

### 13.2 Bonnes pratiques d’équipe

- Lancer `npm test` localement avant push
- Ajouter un job `test` obligatoire dans la branche principale
- Revue de PR incluant la qualité des tests (pas seulement le code)

---

## 14) Ateliers (guidés)

### Atelier 1 — Convertir une logique métier en module testable

**But** : extraire une fonction pure depuis un handler/contrôleur.

- Étape 1 : écrire un test qui décrit le comportement
- Étape 2 : implémenter la fonction
- Étape 3 : refactorer sans casser les tests

### Atelier 2 — API express : tests d’intégration

- Tester `GET /health`
- Tester `POST /sum`
- Ajouter la validation d’entrée et tester les erreurs (400)

### Atelier 3 — React : tester un formulaire

- Rendu initial
- Saisie utilisateur
- Soumission et affichage d’un message de succès

---

## 15) Pièges fréquents

- Tests trop couplés à l’implémentation (fragiles)
- Mocks excessifs (tests inutiles)
- Données non déterministes (date, random) non contrôlées
- Tests lents = pipeline lent = contournement par l’équipe

---

## 16) Synthèse

- Les tests servent à **vérifier le fonctionnement du code** et limiter les régressions.
- **Jest** est excellent pour unitaires + intégration simples, et très adapté à React.
- **Mocha** reste une option solide et flexible, souvent couplée à Chai/Sinon.
- Visez des tests **lisibles**, **déterministes**, et alignés sur le comportement attendu.

---

## Annexes : équivalent rapide avec Mocha (aperçu)

Installation :

```bash
npm i -D mocha chai sinon
```

Exemple :

```js
import { expect } from "chai";

describe("sum", () => {
  it("additionne", () => {
    expect(2 + 3).to.equal(5);
  });
});
```
