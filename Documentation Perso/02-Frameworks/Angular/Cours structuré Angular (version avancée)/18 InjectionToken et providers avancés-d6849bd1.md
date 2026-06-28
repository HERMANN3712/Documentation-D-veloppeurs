# Formation Angular — InjectionToken et providers avancés

> **Thème** : `InjectionToken` & providers avancés (`useClass`, `useValue`, `useExisting`, `useFactory`)  
> **Public** : développeurs Angular (intermédiaire → avancé)  
> **Objectif** : construire des architectures plus souples, testables et extensibles grâce à l’injection de dépendances (DI).

---

## 1) Objectifs pédagogiques

À l’issue de cette formation, vous saurez :

- Expliquer le rôle du **DI container** Angular et le fonctionnement de la résolution des dépendances.
- Utiliser **`InjectionToken`** pour injecter des valeurs **non liées à une classe** (config, stratégie, feature flags, URL, IDs…).
- Manipuler les providers avancés :
  - `useClass` (substitution / polymorphisme)
  - `useValue` (valeurs constantes, mocks)
  - `useExisting` (alias)
  - `useFactory` (construction dynamique)
- Mettre en place des patterns : **configuration globale**, **stratégies**, **multi-providers**, **environnements**, **tests**.
- Diagnostiquer des erreurs DI courantes (`NullInjectorError`, cycles, providers dupliqués, scopes).

---

## 2) Prérequis

- Angular récent (v15+) recommandé.
- Connaissances : Components, Services, Modules (ou architecture standalone), RxJS basique.
- Notions TypeScript : interfaces, generics, types.

---

## 3) Plan de la formation

1. **Rappels DI Angular** (providers, injectors, scopes)
2. **Pourquoi `InjectionToken` ?** (interfaces vs classes, valeurs primitives)
3. **Créer et utiliser un `InjectionToken`**
4. **Providers avancés**
   - `useValue`
   - `useClass`
   - `useExisting`
   - `useFactory`
5. **Multi-providers** et composition de comportements
6. **Cas d’usage architecturaux**
   - Configuration d’app
   - Stratégies / abstractions
   - Feature flags
   - Logging extensible
7. **Testabilité** (override providers, mocks)
8. **Pièges & bonnes pratiques**
9. **Atelier fil rouge** : architecture configurable et extensible

---

## 4) Rappels : comment fonctionne l’injection de dépendances Angular

### 4.1 Concepts

- Un **provider** dit à Angular **comment fournir** une dépendance (valeur, classe à instancier, factory…)
- Un **injector** est un conteneur qui :
  - enregistre des providers
  - construit des instances
  - met en cache (singleton à l’échelle de l’injector)

### 4.2 Scopes / hiérarchie d’injecteurs

Selon votre architecture :

- **Root injector** : providers globaux (souvent `providedIn: 'root'`)
- **Environment injector** (standalone) / modules : scope applicatif/feature
- **Component injector** : providers locaux à un composant et ses enfants

✅ **Règle clé** : l’injecteur le plus proche “gagne” (shadowing).

### 4.3 Résolution

Lorsqu’une classe A dépend de B :

- Angular lit le type (ou token) de B
- Cherche un provider correspondant dans l’injecteur courant
- Sinon remonte la hiérarchie
- Si introuvable : `NullInjectorError: No provider for ...`

---

## 5) Pourquoi `InjectionToken` ?

### 5.1 Interfaces TypeScript : non disponibles au runtime

En Angular, l’injection se fait au runtime. Or :

- `interface` n’existe pas à l’exécution (effacé à la compilation)
- vous ne pouvez pas écrire :

```ts
constructor(private cfg: AppConfig) {}
```

…si `AppConfig` est une interface.

### 5.2 Valeurs non liées à une classe

Vous voulez injecter :

- une **configuration** (objet littéral)
- un **endpoint** (string)
- une **stratégie** (fonction)
- des **flags** (booleans)
- un **adapter** abstrait

➡️ `InjectionToken<T>` sert exactement à cela.

---

## 6) Créer et utiliser un InjectionToken

### 6.1 Définition d’un token typé

```ts
import { InjectionToken } from '@angular/core';

export interface AppConfig {
  apiBaseUrl: string;
  enableDebug: boolean;
}

export const APP_CONFIG = new InjectionToken<AppConfig>('APP_CONFIG');
```

Bonnes pratiques :
- Donner un nom explicite (`'APP_CONFIG'`) utile en debug.
- Utiliser un type générique `InjectionToken<AppConfig>`.

### 6.2 Fournir une valeur via `useValue`

#### Avec application standalone

```ts
import { bootstrapApplication } from '@angular/platform-browser';
import { APP_CONFIG } from './app-config.token';
import { AppComponent } from './app.component';

bootstrapApplication(AppComponent, {
  providers: [
    {
      provide: APP_CONFIG,
      useValue: {
        apiBaseUrl: 'https://api.example.com',
        enableDebug: false,
      },
    },
  ],
});
```

#### Avec modules (si applicable)

```ts
@NgModule({
  providers: [
    { provide: APP_CONFIG, useValue: { apiBaseUrl: '...', enableDebug: true } }
  ]
})
export class AppModule {}
```

### 6.3 Injecter le token

Dans une classe/service (constructor injection) :

```ts
import { Inject, Injectable } from '@angular/core';
import { APP_CONFIG, AppConfig } from './app-config.token';

@Injectable({ providedIn: 'root' })
export class ApiClient {
  constructor(@Inject(APP_CONFIG) private cfg: AppConfig) {}

  get baseUrl() {
    return this.cfg.apiBaseUrl;
  }
}
```

Dans un contexte fonctionnel (Angular moderne) :

```ts
import { inject } from '@angular/core';
import { APP_CONFIG } from './app-config.token';

export function makeUrl(path: string) {
  const cfg = inject(APP_CONFIG);
  return `${cfg.apiBaseUrl}/${path}`;
}
```

---

## 7) Providers avancés : panorama

Un provider prend la forme générale :

```ts
{ provide: TOKEN, useXxx: ... }
```

Où `TOKEN` peut être :
- une **classe** (`ApiClient`)
- un **InjectionToken** (`APP_CONFIG`)

Les stratégies avancées permettent :
- substitution d’implémentations
- configuration dynamique
- aliasing
- composition multi-implémentations

---

## 8) `useValue` — injecter une valeur (constante, config, mock)

### 8.1 Usage typique : config

```ts
{ provide: APP_CONFIG, useValue: { apiBaseUrl: '...', enableDebug: true } }
```

### 8.2 Usage typique : mock en test

```ts
TestBed.configureTestingModule({
  providers: [
    { provide: APP_CONFIG, useValue: { apiBaseUrl: 'http://fake', enableDebug: true } }
  ]
});
```

### 8.3 Attention

- `useValue` **ne** crée **pas** de nouvelle instance via DI : c’est la valeur telle quelle.
- Si c’est un objet mutable, il est partagé dans le scope de l’injector ➜ privilégier immutabilité.

---

## 9) `useClass` — substituer une implémentation

### 9.1 Motivation : abstraction + implémentations multiples

Imaginons une abstraction “logger”. On veut pouvoir brancher différentes implémentations selon l’environnement.

#### Token basé sur une abstraction

On peut utiliser :
- une classe abstraite
- ou un `InjectionToken<Logger>`

Ici, token par interface via `InjectionToken` (moderne et flexible) :

```ts
export interface Logger {
  log(message: string, context?: Record<string, unknown>): void;
}

export const LOGGER = new InjectionToken<Logger>('LOGGER');
```

#### Implémentations

```ts
import { Injectable } from '@angular/core';

@Injectable()
export class ConsoleLogger implements Logger {
  log(message: string, context: Record<string, unknown> = {}) {
    console.log(message, context);
  }
}

@Injectable()
export class SilentLogger implements Logger {
  log(_: string): void {}
}
```

### 9.2 Provider `useClass`

```ts
{
  provide: LOGGER,
  useClass: ConsoleLogger,
}
```

En environnement prod, on peut substituer :

```ts
{
  provide: LOGGER,
  useClass: SilentLogger,
}
```

### 9.3 Points à connaître

- `useClass` laisse Angular gérer les dépendances du constructeur de la classe fournie.
- L’instance créée est “cachée” sous le token `LOGGER` (et non sous `ConsoleLogger` sauf si elle est aussi fournie comme elle-même).

---

## 10) `useExisting` — aliaser un provider existant

### 10.1 Idée

`useExisting` crée un **alias** : deux tokens pointent vers la **même instance**.

### 10.2 Exemple

On a une implémentation unique, mais on veut exposer deux tokens (par compat / migration).

```ts
export const NEW_LOGGER = new InjectionToken<Logger>('NEW_LOGGER');
export const LEGACY_LOGGER = new InjectionToken<Logger>('LEGACY_LOGGER');

providers: [
  { provide: NEW_LOGGER, useClass: ConsoleLogger },
  { provide: LEGACY_LOGGER, useExisting: NEW_LOGGER },
]
```

✅ `LEGACY_LOGGER` et `NEW_LOGGER` renvoient **exactement la même instance**.

### 10.3 Différence vs `useClass`

- `useClass` : crée **une instance** (par token)
- `useExisting` : **réutilise** l’instance associée à un autre token

---

## 11) `useFactory` — création dynamique, dépendante d’autres providers

### 11.1 Idée

`useFactory` permet de calculer la dépendance à partir d’une fonction :

- selon l’environnement
- selon une configuration
- en combinant plusieurs services
- en initialisant une lib externe

### 11.2 Exemple : choisir un logger selon `APP_CONFIG`

```ts
import { APP_CONFIG } from './app-config.token';

export function loggerFactory(cfg: AppConfig): Logger {
  return cfg.enableDebug ? new ConsoleLogger() : new SilentLogger();
}

providers: [
  {
    provide: LOGGER,
    useFactory: loggerFactory,
    deps: [APP_CONFIG],
  },
]
```

Ici :
- Angular injecte `APP_CONFIG`
- appelle `loggerFactory(cfg)`
- enregistre le résultat comme valeur pour `LOGGER`

### 11.3 Variante : factory qui s’appuie sur DI pour construire des classes

Si vous voulez construire une classe ayant des dépendances, évitez `new SomeClass(...)` “à la main” et préférez injecter ses dépendances via `deps`.

Exemple :

```ts
@Injectable()
export class HttpLogger implements Logger {
  constructor(private http: HttpClient) {}
  log(message: string) {
    this.http.post('/logs', { message }).subscribe();
  }
}

export function httpOrConsoleLoggerFactory(cfg: AppConfig, http: HttpClient): Logger {
  return cfg.enableDebug ? new ConsoleLogger() : new HttpLogger(http);
}

providers: [
  {
    provide: LOGGER,
    useFactory: httpOrConsoleLoggerFactory,
    deps: [APP_CONFIG, HttpClient],
  }
]
```

### 11.4 Bonnes pratiques

- Garder les factories **pures et testables**.
- Éviter une logique trop complexe dans un provider : déplacer dans un service dédié si nécessaire.

---

## 12) Multi-providers — plusieurs valeurs pour un même token

### 12.1 Pourquoi

Vous voulez enregistrer une liste extensible de contributeurs :

- interceptors
- validateurs
- stratégies de transformation
- plugins

### 12.2 Définition

```ts
export type TextTransform = (input: string) => string;
export const TEXT_TRANSFORMS = new InjectionToken<TextTransform[]>('TEXT_TRANSFORMS');
```

### 12.3 Fournir plusieurs valeurs

```ts
export const trimTransform: TextTransform = (s) => s.trim();
export const upperTransform: TextTransform = (s) => s.toUpperCase();

providers: [
  { provide: TEXT_TRANSFORMS, useValue: trimTransform, multi: true },
  { provide: TEXT_TRANSFORMS, useValue: upperTransform, multi: true },
]
```

### 12.4 Injection

```ts
import { Inject } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class TextPipeline {
  constructor(@Inject(TEXT_TRANSFORMS) private transforms: TextTransform[]) {}

  run(input: string) {
    return this.transforms.reduce((acc, fn) => fn(acc), input);
  }
}
```

### 12.5 Notes importantes

- Sans `multi: true`, le dernier provider écrase les précédents.
- Avec `multi: true`, Angular agrège dans un tableau (ordre généralement celui d’enregistrement).

---

## 13) Cas d’usage architecturaux

### 13.1 Configuration d’application (environnements)

Token :

```ts
export interface AppConfig {
  apiBaseUrl: string;
  enableDebug: boolean;
  features: {
    billing: boolean;
    betaDashboard: boolean;
  };
}
export const APP_CONFIG = new InjectionToken<AppConfig>('APP_CONFIG');
```

Fourniture selon l’environnement :

```ts
import { environment } from '../environments/environment';

providers: [
  { provide: APP_CONFIG, useValue: environment.appConfig },
]
```

### 13.2 Stratégie de stockage (Strategy pattern)

```ts
export interface StorageStrategy {
  get(key: string): string | null;
  set(key: string, value: string): void;
}

export const STORAGE_STRATEGY = new InjectionToken<StorageStrategy>('STORAGE_STRATEGY');

@Injectable()
export class LocalStorageStrategy implements StorageStrategy {
  get(key: string) { return localStorage.getItem(key); }
  set(key: string, value: string) { localStorage.setItem(key, value); }
}

@Injectable()
export class MemoryStorageStrategy implements StorageStrategy {
  private map = new Map<string, string>();
  get(key: string) { return this.map.get(key) ?? null; }
  set(key: string, value: string) { this.map.set(key, value); }
}
```

Provider :

```ts
providers: [
  { provide: STORAGE_STRATEGY, useClass: LocalStorageStrategy }
]
```

Service consommateur :

```ts
@Injectable({ providedIn: 'root' })
export class PreferencesService {
  constructor(@Inject(STORAGE_STRATEGY) private storage: StorageStrategy) {}

  saveTheme(theme: string) {
    this.storage.set('theme', theme);
  }

  loadTheme() {
    return this.storage.get('theme');
  }
}
```

✅ En tests, remplacez `LocalStorageStrategy` par `MemoryStorageStrategy`.

### 13.3 Aliasing pour migration progressive

Si un ancien code injecte `LegacyPreferencesService`, mais vous migrez vers `PreferencesService` :

```ts
export const LEGACY_PREFS = new InjectionToken<PreferencesService>('LEGACY_PREFS');

providers: [
  PreferencesService,
  { provide: LEGACY_PREFS, useExisting: PreferencesService },
]
```

### 13.4 Factory pour brancher une lib externe

Ex : initialiser une SDK analytics.

```ts
export interface Analytics {
  track(event: string, props?: Record<string, unknown>): void;
}

export const ANALYTICS = new InjectionToken<Analytics>('ANALYTICS');

export function analyticsFactory(cfg: AppConfig): Analytics {
  if (!cfg.features.betaDashboard) {
    return { track: () => {} }; // noop
  }

  // pseudo-code : init d’une lib
  const sdk = createAnalyticsSdk({ baseUrl: cfg.apiBaseUrl });
  return {
    track: (event, props) => sdk.send(event, props),
  };
}

providers: [
  { provide: ANALYTICS, useFactory: analyticsFactory, deps: [APP_CONFIG] }
]
```

---

## 14) Testabilité : override providers

### 14.1 Override d’un token

```ts
TestBed.configureTestingModule({
  providers: [
    { provide: APP_CONFIG, useValue: { apiBaseUrl: 'http://test', enableDebug: true, features: { billing: false, betaDashboard: true } } },
  ],
});
```

### 14.2 Override d’une stratégie

```ts
TestBed.configureTestingModule({
  providers: [
    { provide: STORAGE_STRATEGY, useClass: MemoryStorageStrategy },
  ],
});
```

### 14.3 Fake / Spy Logger

```ts
class SpyLogger implements Logger {
  calls: string[] = [];
  log(message: string) { this.calls.push(message); }
}

TestBed.configureTestingModule({
  providers: [
    { provide: LOGGER, useClass: SpyLogger },
  ],
});

const logger = TestBed.inject(LOGGER) as SpyLogger;
expect(logger.calls).toContain('...');
```

---

## 15) Pièges courants & bonnes pratiques

### 15.1 `NullInjectorError`

Causes fréquentes :
- token non fourni
- provider déclaré dans un module non importé
- provider fourni à un scope inattendu

Diagnostic :
- vérifier l’endroit où le provider est déclaré
- vérifier le scope (component vs root)

### 15.2 Cycles de dépendances

Ex : A dépend de B, B dépend de A.  
Solutions :
- extraire une 3e dépendance commune
- introduire une factory
- revoir l’architecture

### 15.3 Ne pas abuser des tokens

- Garder un nombre de tokens raisonnable, structurés par domaine.
- Nommer clairement et centraliser (ex: `core/tokens/*`).

### 15.4 Préférer une config immuable

- Utiliser `as const` et éviter les modifications à l’exécution.

### 15.5 Séparer “API” et “implémentation”

- Token (contrat) dans un package/dossier stable
- Implémentations dans des features, remplaçables

---

## 16) Atelier fil rouge (guidé) — App configurable et extensible

### 16.1 Enoncé

Construire une mini-architecture :
- un `APP_CONFIG` injecté
- un `LOGGER` configurable (console vs silencieux)
- un pipeline de transformations extensible via multi-providers

### 16.2 Étapes

1. Créer `APP_CONFIG` et l’injecter dans `ApiClient`.
2. Créer `LOGGER` + `ConsoleLogger` + `SilentLogger`.
3. Brancher `LOGGER` via `useFactory` selon `enableDebug`.
4. Créer `TEXT_TRANSFORMS` en multi-provider et un `TextPipeline`.
5. Tester : override `APP_CONFIG` et remplacer `LOGGER` par un spy.

### 16.3 Critères qualité

- Typage strict (tokens génériques)
- Pas de `any`
- Providers déclarés dans le bon scope
- Tests simples et rapides (pas de dépendance navigateur pour le storage)

---

## 17) Récapitulatif

- **`InjectionToken`** : injecter des valeurs/abstractions non basées sur une classe (config, interface, stratégie…).
- `useValue` : valeur/objet constant, mocks.
- `useClass` : substitution d’implémentation (polymorphisme).
- `useExisting` : alias vers une instance existante.
- `useFactory` : construction dynamique avec `deps`.
- **Multi-providers** : extension “plugin-like”.

---

## 18) Annexes — snippets prêts à l’emploi

### 18.1 Token + config d’environnement

```ts
export const APP_CONFIG = new InjectionToken<AppConfig>('APP_CONFIG');

// environment.ts
export const environment = {
  production: false,
  appConfig: {
    apiBaseUrl: 'http://localhost:3000',
    enableDebug: true,
    features: { billing: false, betaDashboard: true },
  } satisfies AppConfig,
};
```

### 18.2 Provider “compat” via `useExisting`

```ts
export const HTTP_BASE_URL = new InjectionToken<string>('HTTP_BASE_URL');
export const API_BASE_URL = new InjectionToken<string>('API_BASE_URL');

providers: [
  { provide: API_BASE_URL, useValue: 'https://api.example.com' },
  { provide: HTTP_BASE_URL, useExisting: API_BASE_URL },
]
```

### 18.3 Multi-provider de “plugins”

```ts
export interface Plugin {
  name: string;
  init(): void;
}

export const PLUGINS = new InjectionToken<Plugin[]>('PLUGINS');

providers: [
  { provide: PLUGINS, useValue: { name: 'A', init: () => {} }, multi: true },
  { provide: PLUGINS, useValue: { name: 'B', init: () => {} }, multi: true },
]
```

---

### Fin de formation
