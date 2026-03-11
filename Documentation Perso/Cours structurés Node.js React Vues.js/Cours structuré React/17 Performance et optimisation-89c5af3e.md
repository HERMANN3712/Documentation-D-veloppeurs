# 17 — Performance et optimisation (React)

> **Public** : développeurs React (intermédiaire) et formateurs souhaitant structurer un cours sur l’optimisation.
>
> **Objectif** : comprendre *pourquoi* une app React peut devenir lente, *comment* mesurer, puis *quoi* optimiser via **React.memo**, **useMemo**, **useCallback** et le **lazy loading**.

---

## Table des matières

1. [Objectifs pédagogiques](#1-objectifs-pédagogiques)
2. [Pré-requis](#2-pré-requis)
3. [Notions clés : d’où viennent les lenteurs ?](#3-notions-clés--doù-viennent-les-lenteurs-)
4. [Mesurer avant d’optimiser : les outils](#4-mesurer-avant-doptimiser--les-outils)
5. [Optimiser les re-renders avec `React.memo`](#5-optimiser-les-re-renders-avec-reactmemo)
6. [Mémoïser des calculs coûteux avec `useMemo`](#6-mémoïser-des-calculs-coûteux-avec-usememo)
7. [Stabiliser des callbacks avec `useCallback`](#7-stabiliser-des-callbacks-avec-usecallback)
8. [Lazy loading (code splitting) avec `React.lazy` + `Suspense`](#8-lazy-loading-code-splitting-avec-reactlazy--suspense)
9. [Atelier guidé : refactor pas à pas](#9-atelier-guidé--refactor-pas-à-pas)
10. [Anti-patterns & pièges fréquents](#10-anti-patterns--pièges-fréquents)
11. [Checklist de performance (à emporter)](#11-checklist-de-performance-à-emporter)
12. [Quiz de validation](#12-quiz-de-validation)

---

## 1. Objectifs pédagogiques

À la fin de cette formation, vous serez capable de :

- Expliquer les causes principales de re-renders inutiles dans React.
- Repérer les zones coûteuses avec **React DevTools Profiler**.
- Utiliser **`React.memo`** pour éviter des re-renders de composants purs.
- Utiliser **`useMemo`** pour mémoriser un résultat de calcul coûteux.
- Utiliser **`useCallback`** pour stabiliser une fonction passée en props.
- Implémenter le **lazy loading** de composants (code splitting) avec **`React.lazy`** et **`Suspense`**.
- Éviter les optimisations inutiles et savoir quand s’arrêter.

---

## 2. Pré-requis

- Bon niveau JavaScript/TypeScript.
- Compréhension du rendu React, state, props, hooks.
- Savoir lire et écrire des composants fonctionnels.

Environnement recommandé :
- React 18+
- React DevTools (extension navigateur)
- Un bundler moderne (Vite / CRA / Next.js)

---

## 3. Notions clés : d’où viennent les lenteurs ?

### 3.1 Le re-render n’est pas toujours un problème
React est conçu pour re-render souvent. Un re-render :

- **recalcule** le JSX (phase de render),
- puis **réconcilie** (diff) avec l’arbre précédent,
- puis applique les changements au DOM (commit) si nécessaire.

Un re-render peut être **très rapide** si le composant est simple.

### 3.2 Les principales causes de lenteur

1. **Renders trop fréquents**
   - state mis à jour trop souvent
   - propagation du state trop haut dans l’arbre

2. **Renders trop coûteux**
   - gros tableaux, filtrage/tri, calculs lourds
   - composants complexes (charts, éditeurs, tables)

3. **Renders inutiles**
   - un composant re-render alors que ses entrées (props) n’ont pas changé *fonctionnellement*
   - props instables : `onClick={() => ...}` recréé à chaque render, objets/arrays littéraux recréés

4. **Taille de bundle élevée**
   - trop de code téléchargé au démarrage
   - dépendances lourdes non découpées

### 3.3 Règle d’or

> **Mesurer → identifier un goulot → optimiser → re-mesurer.**

---

## 4. Mesurer avant d’optimiser : les outils

### 4.1 React DevTools Profiler

- Permet d’enregistrer une interaction et de voir :
  - quels composants re-render
  - combien de temps ils prennent
  - pourquoi ils re-render (souvent visible via les props/state)

**Process** :
1. Ouvrir React DevTools → onglet **Profiler**
2. Démarrer un enregistrement
3. Interagir avec l’app
4. Stop → analyser les "flame charts" et "ranked"

### 4.2 Logs pédagogiques (pour apprendre)

- `console.log('render')` dans un composant aide à comprendre la fréquence.
- Utiliser avec modération (le log perturbe aussi les perfs).

### 4.3 Métriques simples

- Temps d’ouverture d’écran (TTI ressenti)
- Latence au clic/scroll
- Réactivité lors de saisies (input)

---

## 5. Optimiser les re-renders avec `React.memo`

### 5.1 Concept

`React.memo` mémorise le rendu d’un composant **en fonction de ses props**.

- Si les props **ne changent pas** (comparaison par défaut : **shallow**), React réutilise le rendu précédent.
- Utile pour les composants "purs" (UI) dont le rendu est stable.

### 5.2 Exemple sans `memo`

```jsx
import { useState } from "react";

function Child({ label }) {
  console.log("Child render");
  return <button>{label}</button>;
}

export function Parent() {
  const [count, setCount] = useState(0);

  return (
    <div>
      <p>count: {count}</p>
      <button onClick={() => setCount((c) => c + 1)}>+1</button>
      <Child label="OK" />
    </div>
  );
}
```

À chaque incrément, `Parent` re-render → `Child` re-render aussi, même si `label` n’a pas changé.

### 5.3 Avec `React.memo`

```jsx
import { memo, useState } from "react";

const Child = memo(function Child({ label }) {
  console.log("Child render");
  return <button>{label}</button>;
});

export function Parent() {
  const [count, setCount] = useState(0);

  return (
    <div>
      <p>count: {count}</p>
      <button onClick={() => setCount((c) => c + 1)}>+1</button>
      <Child label="OK" />
    </div>
  );
}
```

Résultat : `Child` ne re-render plus lors de `count++`.

### 5.4 Props instables : pourquoi `memo` “ne marche pas” parfois

Même avec `memo`, si vous passez :

- une fonction inline : `onClick={() => ...}`
- un objet recréé : `{ theme: 'dark' }`
- un tableau recréé : `[1,2,3]`

…alors la comparaison shallow voit une nouvelle référence → re-render.

**Solution** : stabiliser via `useCallback` / `useMemo` ou remonter la création dans le composant mémorisé (selon le cas).

### 5.5 Comparateur personnalisé (cas avancé)

```jsx
const Row = memo(RowBase, (prev, next) => {
  // retourner true si on considère les props équivalentes
  return prev.item.id === next.item.id && prev.item.status === next.item.status;
});
```

À utiliser avec prudence :
- risque de bugs (rendu stale)
- coût du comparateur lui-même

### 5.6 Quand utiliser `React.memo`

✅ Bon pour :
- composants de liste (rows) rendus souvent
- composants lourds (chart, table)
- UI pure réutilisable

❌ Moins utile si :
- composant léger
- props changent tout le temps
- vous n’avez pas de problème mesuré

---

## 6. Mémoïser des calculs coûteux avec `useMemo`

### 6.1 Concept

`useMemo` mémorise **une valeur calculée** entre deux renders.

- Le calcul n’est refait **que si** une dépendance change.
- Objectif : éviter des calculs coûteux lors de renders fréquents.

> `useMemo` n’est pas une garantie de perf dans tous les cas : c’est un *hint* et un compromis (mémoire + complexité).

### 6.2 Exemple : filtrage/tri coûteux

```jsx
import { useMemo, useState } from "react";

function expensiveSort(items) {
  // Simulation d’un traitement coûteux
  return [...items].sort((a, b) => a.name.localeCompare(b.name));
}

export function Directory({ items }) {
  const [query, setQuery] = useState("");

  const visibleItems = useMemo(() => {
    const q = query.toLowerCase();
    const filtered = items.filter((x) => x.name.toLowerCase().includes(q));
    return expensiveSort(filtered);
  }, [items, query]);

  return (
    <section>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      <ul>
        {visibleItems.map((x) => (
          <li key={x.id}>{x.name}</li>
        ))}
      </ul>
    </section>
  );
}
```

Sans `useMemo`, chaque frappe dans l’input déclenche un re-render et recalcul complet non maîtrisé.

### 6.3 Dangers courants

- **Dépendances incorrectes** → valeur désynchronisée.
- `useMemo(() => ({a: 1}), [])` utilisé uniquement pour “stabiliser une prop”, sans besoin réel.

### 6.4 Quand `useMemo` est pertinent

✅ Oui si :
- calcul CPU significatif (tri, regroupement, mapping complexe)
- rendu fréquent (animations, input)
- données volumineuses

❌ Non si :
- le calcul est trivial
- vous comptez dessus pour "corriger" une architecture (state trop haut, etc.)

---

## 7. Stabiliser des callbacks avec `useCallback`

### 7.1 Concept

`useCallback(fn, deps)` mémorise **la référence de la fonction**.

- Utile surtout quand vous passez des callbacks à des enfants mémorisés (`React.memo`).
- Utile aussi comme dépendance stable pour d’autres hooks (ex: `useEffect`).

### 7.2 Exemple : `memo` + callback instable

```jsx
import { memo, useState } from "react";

const Child = memo(function Child({ onIncrement }) {
  console.log("Child render");
  return <button onClick={onIncrement}>Increment</button>;
});

export function Parent() {
  const [count, setCount] = useState(0);
  const [theme, setTheme] = useState("light");

  // ⚠️ nouvelle fonction à chaque render
  const handleIncrement = () => setCount((c) => c + 1);

  return (
    <div>
      <p>{count}</p>
      <button onClick={() => setTheme((t) => (t === "light" ? "dark" : "light"))}>
        Toggle theme
      </button>
      <Child onIncrement={handleIncrement} />
    </div>
  );
}
```

Même si `Child` est `memo`, il re-render à chaque changement de `theme`, car `handleIncrement` change de référence.

### 7.3 Correction avec `useCallback`

```jsx
import { memo, useCallback, useState } from "react";

const Child = memo(function Child({ onIncrement }) {
  console.log("Child render");
  return <button onClick={onIncrement}>Increment</button>;
});

export function Parent() {
  const [count, setCount] = useState(0);
  const [theme, setTheme] = useState("light");

  const handleIncrement = useCallback(() => {
    setCount((c) => c + 1);
  }, []);

  return (
    <div>
      <p>{count}</p>
      <button onClick={() => setTheme((t) => (t === "light" ? "dark" : "light"))}>
        Toggle theme
      </button>
      <Child onIncrement={handleIncrement} />
    </div>
  );
}
```

Résultat : `Child` ne re-render plus lors du changement de `theme`.

### 7.4 Bien gérer les dépendances

- Si le callback utilise une valeur `x` du scope, elle doit être dans `deps`.
- Préférer les **updates fonctionnels** (`setState(prev => ...)`) pour réduire les dépendances.

Exemple :

```jsx
const handleAdd = useCallback(() => {
  setCount((c) => c + step);
}, [step]);
```

### 7.5 Quand `useCallback` est pertinent

✅ Oui si :
- vous passez le callback à un composant `memo`
- vous observez des re-renders inutiles

❌ Non si :
- le callback n’est pas passé en props (usage local)
- vous n’avez pas de problème mesuré

---

## 8. Lazy loading (code splitting) avec `React.lazy` + `Suspense`

### 8.1 Objectif

Réduire le JS chargé au démarrage en **découpant** le code en chunks :
- l’écran initial charge vite
- les écrans secondaires se chargent à la demande

### 8.2 Implémentation simple

```jsx
import { lazy, Suspense } from "react";

const SettingsPage = lazy(() => import("./pages/SettingsPage"));

export function App() {
  return (
    <Suspense fallback={<div>Chargement…</div>}>
      <SettingsPage />
    </Suspense>
  );
}
```

### 8.3 Avec un routeur (pattern courant)

Avec React Router (conceptuellement) :

```jsx
import { lazy, Suspense } from "react";
import { Routes, Route } from "react-router-dom";

const Home = lazy(() => import("./pages/Home"));
const Dashboard = lazy(() => import("./pages/Dashboard"));

export function AppRoutes() {
  return (
    <Suspense fallback={<div>Chargement de la page…</div>}>
      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/dashboard" element={<Dashboard />} />
      </Routes>
    </Suspense>
  );
}
```

### 8.4 Bonnes pratiques

- Fallback adapté : skeleton, spinner, layout stable.
- Précharger dans certains cas (préfetch) selon le bundler :
  - ex: sur hover d’un lien, démarrer l’import.
- Découper par *routes* ou grosses features, pas par petits composants.

### 8.5 Pièges

- Lazy loading excessif : trop de chunks → overhead réseau.
- UX dégradée si fallback trop fréquent.

---

## 9. Atelier guidé : refactor pas à pas

### 9.1 Point de départ (problème)

Objectif : une page avec liste d’items et un panneau latéral. Modifier le thème ne doit pas re-render toute la liste.

```jsx
import { useState } from "react";

function Row({ item, onSelect }) {
  console.log("Row render", item.id);
  return (
    <li>
      <button onClick={() => onSelect(item.id)}>{item.name}</button>
    </li>
  );
}

export function Screen({ items }) {
  const [theme, setTheme] = useState("light");
  const [selectedId, setSelectedId] = useState(null);

  const onSelect = (id) => setSelectedId(id);

  return (
    <div data-theme={theme}>
      <aside>
        <button onClick={() => setTheme((t) => (t === "light" ? "dark" : "light"))}>
          Toggle theme
        </button>
        <p>Selected: {selectedId ?? "none"}</p>
      </aside>

      <ul>
        {items.map((item) => (
          <Row key={item.id} item={item} onSelect={onSelect} />
        ))}
      </ul>
    </div>
  );
}
```

**Constat** : `Toggle theme` fait re-render toutes les `Row`.

### 9.2 Étape 1 — `React.memo` sur la ligne

```jsx
import { memo } from "react";

const Row = memo(function Row({ item, onSelect }) {
  console.log("Row render", item.id);
  return (
    <li>
      <button onClick={() => onSelect(item.id)}>{item.name}</button>
    </li>
  );
});
```

**Problème** : ça re-render toujours car `onSelect` change de référence à chaque render.

### 9.3 Étape 2 — `useCallback` pour stabiliser `onSelect`

```jsx
import { useCallback, useState } from "react";

export function Screen({ items }) {
  const [theme, setTheme] = useState("light");
  const [selectedId, setSelectedId] = useState(null);

  const onSelect = useCallback((id) => {
    setSelectedId(id);
  }, []);

  return (
    <div data-theme={theme}>
      {/* ... */}
      <ul>
        {items.map((item) => (
          <Row key={item.id} item={item} onSelect={onSelect} />
        ))}
      </ul>
    </div>
  );
}
```

**Résultat** : les `Row` ne re-render plus quand `theme` change.

### 9.4 Étape 3 — `useMemo` pour éviter un tri/filtre coûteux

Si `items` est filtré/trié dans le render :

```jsx
const visibleItems = useMemo(() => {
  // filtre + tri potentiellement coûteux
  return items
    .filter((x) => x.enabled)
    .sort((a, b) => a.name.localeCompare(b.name));
}, [items]);

return (
  <ul>
    {visibleItems.map((item) => (
      <Row key={item.id} item={item} onSelect={onSelect} />
    ))}
  </ul>
);
```

### 9.5 Étape 4 — Lazy load d’un panneau avancé

Si un panneau (ex: analytics) est rarement ouvert :

```jsx
import { lazy, Suspense, useState } from "react";

const AnalyticsPanel = lazy(() => import("./AnalyticsPanel"));

export function Screen({ items }) {
  const [showAnalytics, setShowAnalytics] = useState(false);

  return (
    <div>
      <button onClick={() => setShowAnalytics(true)}>Open analytics</button>

      {showAnalytics && (
        <Suspense fallback={<div>Chargement analytics…</div>}>
          <AnalyticsPanel />
        </Suspense>
      )}

      {/* ... */}
    </div>
  );
}
```

---

## 10. Anti-patterns & pièges fréquents

1. **Tout mémoïser “par principe”**
   - Complexifie le code
   - Peut empirer les perfs (comparaisons + mémoire)

2. **Dépendances incorrectes**
   - `useMemo`/`useCallback` avec deps manquantes → bugs subtils

3. **Comparer profondément dans `memo`**
   - Un deep compare peut coûter plus cher qu’un re-render.

4. **Références recréées**
   - `style={{...}}`, `options={{...}}`, `onClick={() => ...}`

5. **Optimiser le symptôme au lieu de la cause**
   - Parfois il faut :
     - déplacer du state plus bas
     - splitter le composant
     - virtualiser une liste (hors scope de ce cours, mais à connaître)

---

## 11. Checklist de performance (à emporter)

### Mesure
- [ ] J’ai utilisé le **Profiler** et identifié le composant lent.
- [ ] J’ai une interaction réelle qui déclenche le ralentissement.

### Re-renders
- [ ] Les composants de liste importants sont candidats à `React.memo`.
- [ ] Les callbacks passés en props sont stabilisés avec `useCallback` si nécessaire.
- [ ] Les objets/tableaux passés en props sont stabilisés (`useMemo`) ou reconstruits côté enfant.

### Calcul
- [ ] Les tris/filtrages coûteux sont dans `useMemo`.
- [ ] Les dépendances sont correctes.

### Bundle
- [ ] Les pages/écrans lourds sont chargés en lazy.
- [ ] Le fallback `Suspense` ne dégrade pas l’UX.

### Validation
- [ ] J’ai re-mesuré après optimisation.
- [ ] Le code reste compréhensible.

---

## 12. Quiz de validation

1. **Vrai/Faux** : un re-render React met toujours à jour le DOM.
2. Pourquoi `React.memo` peut ne pas empêcher un re-render si on passe un callback inline ?
3. Quelle est la différence entre `useMemo` et `useCallback` ?
4. Donnez un exemple où `useMemo` est inutile.
5. Quel est l’objectif principal du lazy loading ?

**Réponses attendues (synthèse)** :
1. Faux (le DOM n’est modifié que si nécessaire au commit).
2. Car la prop fonction change de référence à chaque render.
3. `useMemo` mémorise une valeur, `useCallback` mémorise une fonction.
4. Calcul trivial ou composant peu rendu.
5. Réduire le JS initial (code splitting) et améliorer le chargement perçu.

---

## Annexes (optionnelles pour le formateur)

### A. Grille de déroulé (suggestion)

- 10 min : rappels re-render / mesure
- 20 min : React.memo + exercices
- 20 min : useCallback (avec memo)
- 20 min : useMemo (calculs)
- 15 min : lazy loading
- 30 min : atelier refactor
- 5 min : quiz + checklist

### B. Exercice rapide (mini)

- Prenez un composant de liste.
- Ajoutez `console.log` pour observer les renders.
- Ajoutez `React.memo` puis stabilisez les props.
- Mesurez avec Profiler.
