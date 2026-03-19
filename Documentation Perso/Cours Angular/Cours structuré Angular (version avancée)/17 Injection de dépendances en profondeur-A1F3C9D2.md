# Formation Angular — Injection de dépendances en profondeur

> **Public** : développeurs Angular (intermédiaire à avancé) qui souhaitent maîtriser la DI (Dependency Injection) dans ses subtilités : hiérarchie des injecteurs, portée réelle des dépendances, tokens, multi-providers, factories, patterns avancés.

---

## 0. Objectifs pédagogiques

À la fin de cette formation, vous serez capable de :

- Expliquer **la hiérarchie des injecteurs** Angular (EnvironmentInjector, ElementInjector) et l’algorithme de résolution.
- Choisir la **bonne portée** (root / feature / composant / directive / lazy-loaded) et anticiper les effets (singleton vs instance par sous-arbre).
- Utiliser correctement les **tokens** (`InjectionToken`, tokens existants comme `APP_INITIALIZER`, etc.).
- Mettre en œuvre des **multi-providers** (ex : chaînes de handlers, plugins, interceptors).
- Écrire des **factories** et configurer des providers avancés (`useFactory`, `deps`, `useExisting`, `useClass`, `useValue`).
- Déboguer et résoudre des problèmes concrets : dépendances circulaires, providers dupliqués, DI dans les modules lazy, scope inattendu, couplage fort, etc.

---

## 1. Prérequis et setup

### 1.1 Prérequis

- Connaître TypeScript et les bases d’Angular (components, services, modules/standalone).
- Avoir déjà utilisé `@Injectable({ providedIn: 'root' })`.
- Connaître les bases de RxJS est un plus.

### 1.2 Version et contexte

Cette formation s’appuie sur les concepts Angular modernes (Angular 14+ / 15+ / 16+ / 17+). Les explications restent pertinentes même si l’API évolue, car la majorité des mécanismes DI sont stables.

### 1.3 Projet de démonstration (suggestion)

Structure recommandée :

- `core/` : services singletons, config globale, interceptors
- `features/` : features lazy-loaded
- `shared/` : composants réutilisables
- `app.config.ts` (si app standalone) ou `app.module.ts`

---

## 2. Rappels utiles : qu’est-ce que la DI Angular ?

### 2.1 Pourquoi la DI ?

La DI permet de :

- **Découpler** le code (un service ne construit pas lui-même ses dépendances).
- Centraliser le **cycle de vie** et la **configuration**.
- Faciliter les **tests** (remplacer un provider par un mock).

### 2.2 Les 3 éléments clés

- **Consumer (consommateur)** : une classe (component/service/directive) qui demande une dépendance.
- **Token** : la “clé” utilisée pour retrouver une dépendance (souvent une classe, mais pas toujours).
- **Provider** : règle qui dit à Angular *comment créer* ou *fournir* la dépendance.

Exemple minimal :

```ts
@Injectable({ providedIn: 'root' })
export class LoggerService {
  log(msg: string) { console.log(msg); }
}

@Component({
  selector: 'app-demo',
  template: `...`,
})
export class DemoComponent {
  constructor(private logger: LoggerService) {
    this.logger.log('Hello');
  }
}
```

---

## 3. La hiérarchie des injecteurs (le cœur du sujet)

### 3.1 Deux grandes familles d’injecteurs

Angular utilise une hiérarchie composée principalement de :

1. **EnvironmentInjector** (injecteur “environnement”) :
   - associée à l’application, au bootstrap et aux boundaries type lazy-load.
   - correspond historiquement aux injecteurs de modules (NgModuleInjector).

2. **ElementInjector** (injecteur “élément”) :
   - attaché à chaque nœud du rendu (composants/directives).
   - permet la résolution au niveau du composant/directive.

> Intuition : **EnvironmentInjector** = niveau “application/feature”. **ElementInjector** = niveau “arbre de composants”.

### 3.2 Algorithme de résolution (simplifié mais fidèle)

Quand Angular doit injecter un token `T` dans un composant/directive :

1. Cherche dans l’**injecteur local** (ElementInjector du nœud courant).
2. Remonte **dans les injecteurs parents** (ElementInjector parents).
3. En absence, bascule vers l’**EnvironmentInjector** associé au contexte (souvent root ou lazy feature).
4. Si toujours absent : erreur `NullInjectorError: No provider for T!`.

### 3.3 Conséquences pratiques

- Un provider déclaré au niveau d’un **composant** “masque” (shadow) celui défini au **root**.
- Dans un module/feature lazy-loaded, un provider peut être **scopé** à la feature (donc pas global).
- Les directives peuvent aussi déclarer des providers et influencer l’injection.

---

## 4. Portées (scopes) et patterns de fourniture

### 4.1 `providedIn: 'root'` — singleton applicatif

```ts
@Injectable({ providedIn: 'root' })
export class AuthService {}
```

- Une seule instance pour toute l’app (dans le root EnvironmentInjector).
- Idéal pour : auth, configuration globale, caches partagés, services “core”.

### 4.2 `providedIn: 'any'` — par injecteur (attention)

```ts
@Injectable({ providedIn: 'any' })
export class FeatureScopedService {}
```

- Crée une instance **par injecteur** qui le résout.
- Souvent observé avec des lazy-loaded : chaque feature peut avoir sa propre instance.
- Peut surprendre (n’est pas un singleton global).

### 4.3 Provider au niveau Module / Feature

- Avec modules (NgModule) : `providers: []`
- Avec standalone : via `bootstrapApplication(..., { providers: [...] })` ou `provideRouter(...)` etc.

Usage : configurer une feature entière, isoler des services.

### 4.4 Provider au niveau Component

```ts
@Component({
  selector: 'app-a',
  template: `<app-b/>`,
  providers: [LoggerService]
})
export class AComponent {}
```

- **Nouvelle instance** pour `AComponent` et tout son sous-arbre (B, C, etc.).
- Très utile pour des états locaux : wizard, table state, cache local, façade par composant, etc.

### 4.5 Provider au niveau Directive

```ts
@Directive({
  selector: '[appLocal] ',
  providers: [LocalStateService]
})
export class LocalDirective {}
```

- Scopé au nœud DOM qui porte la directive (et son sous-arbre selon le besoin).
- Pattern intéressant pour attacher un service à un “comportement”.

---

## 5. Tokens : au-delà des classes

### 5.1 Quand utiliser un `InjectionToken` ?

Cas typiques :

- Injecter une **configuration** (objet, string, number).
- Fournir **plusieurs implémentations** (stratégies, plugins).
- Éviter les collisions/incohérences de tokens.

### 5.2 Exemple : injection de configuration

```ts
export interface ApiConfig {
  baseUrl: string;
  timeoutMs: number;
}

export const API_CONFIG = new InjectionToken<ApiConfig>('API_CONFIG');

export const appProviders = [
  {
    provide: API_CONFIG,
    useValue: { baseUrl: 'https://api.example.com', timeoutMs: 8000 }
  }
];

@Injectable({ providedIn: 'root' })
export class ApiClient {
  constructor(@Inject(API_CONFIG) private cfg: ApiConfig) {}
}
```

### 5.3 Tokens Angular utiles (panorama)

- `APP_INITIALIZER` (multi) : exécuter du code avant bootstrap.
- `HTTP_INTERCEPTORS` (multi) : chaîne d’intercepteurs HTTP.
- `ErrorHandler` : gestion globale des erreurs.
- `LOCALE_ID` : i18n.

---

## 6. Providers : `useClass`, `useValue`, `useExisting`, `useFactory`

### 6.1 `useClass` — fournir une implémentation

```ts
export abstract class Clock {
  abstract now(): Date;
}

@Injectable()
export class SystemClock implements Clock {
  now() { return new Date(); }
}

providers: [{ provide: Clock, useClass: SystemClock }]
```

### 6.2 `useValue` — valeur constante

```ts
providers: [{ provide: 'APP_NAME', useValue: 'MyApp' }]
```

> Préférez `InjectionToken` à une string pour éviter collisions.

### 6.3 `useExisting` — alias vers un provider existant

```ts
providers: [
  LoggerService,
  { provide: AuditLogger, useExisting: LoggerService }
]
```

- Deux tokens, **même instance**.
- Utile pour compatibilité ou pour exposer une API différente.

### 6.4 `useFactory` — construction dynamique

```ts
export const API_BASE_URL = new InjectionToken<string>('API_BASE_URL');

export function apiBaseUrlFactory(location: Location): string {
  return location.hostname.includes('localhost')
    ? 'http://localhost:3000'
    : 'https://api.prod.com';
}

providers: [
  {
    provide: API_BASE_URL,
    useFactory: apiBaseUrlFactory,
    deps: [Location],
  }
]
```

- Permet de décider au runtime.
- `deps` explicite les dépendances de la factory.

---

## 7. Multi-providers : construire des pipelines extensibles

### 7.1 Principe

Un **multi-provider** associe un token à **une liste de valeurs** (ou services), au lieu d’un seul.

Exemple incontournable : `HTTP_INTERCEPTORS`.

### 7.2 Créer son propre token multi

```ts
export interface Exporter {
  name: string;
  export(data: unknown): string;
}

export const EXPORTERS = new InjectionToken<Exporter[]>('EXPORTERS');

providers: [
  { provide: EXPORTERS, useValue: { name: 'json', export: JSON.stringify }, multi: true },
  { provide: EXPORTERS, useValue: { name: 'text', export: (d) => String(d) }, multi: true },
]

@Injectable({ providedIn: 'root' })
export class ExportService {
  constructor(@Inject(EXPORTERS) private exporters: Exporter[]) {}

  exportAs(name: string, data: unknown) {
    const exp = this.exporters.find(e => e.name === name);
    if (!exp) throw new Error('Exporter not found');
    return exp.export(data);
  }
}
```

### 7.3 Pièges fréquents

- Oublier `multi: true` : le dernier provider écrase les précédents.
- Annuler involontairement une liste multi en ré-déclarant le token sans `multi`.

---

## 8. Résolution avancée : décorateurs d’injection

### 8.1 `@Optional()`

- Injecte `null` (ou `undefined` selon cas) si le provider n’existe pas.

```ts
constructor(@Optional() private parent: ParentService | null) {}
```

### 8.2 `@Self()`

- Cherche **uniquement** dans l’injecteur local.
- Utile pour forcer une dépendance à être fournie au niveau composant/directive.

```ts
constructor(@Self() private localState: LocalStateService) {}
```

### 8.3 `@SkipSelf()`

- Ignore l’injecteur local, remonte au parent.
- Utile pour éviter l’auto-référence avec des providers au même niveau.

### 8.4 `@Host()`

- Limite la résolution au “host” (frontière du composant hôte).
- Surtout pertinent avec content projection et directives.

---

## 9. DI et lazy-loading : scopes réels et surprises

### 9.1 Rappel : lazy = injecteur d’environnement distinct

Une feature lazy-loaded crée typiquement un injecteur distinct. Un service fourni dans cette feature :

- est **singleton dans la feature**,
- mais **pas** singleton global.

### 9.2 Symptôme : « j’ai 2 instances alors que je pensais singleton »

Causes fréquentes :

- Service déclaré `providedIn: 'any'`.
- Service fourni dans un module lazy (ou dans `providers` d’un route provider pour un segment lazy).
- Service fourni au niveau component alors qu’on souhaite global.

### 9.3 Bonnes pratiques

- `core` : `providedIn: 'root'`.
- `feature state` : provider au niveau composant/route/feature.
- Documenter explicitement la portée attendue de chaque service.

---

## 10. Patterns avancés

### 10.1 Pattern “service scoped par composant” (stateful façade)

Objectif : un service d’état qui vit et meurt avec un composant.

```ts
@Injectable()
export class TableState {
  page = 1;
  pageSize = 20;
}

@Component({
  selector: 'app-table',
  templateUrl: './table.html',
  providers: [TableState]
})
export class TableComponent {
  constructor(public state: TableState) {}
}
```

### 10.2 Plugin architecture via multi-provider

- Token `PLUGINS` multi.
- Chaque feature ajoute ses plugins.
- Un orchestrateur les consomme.

### 10.3 Stratégies interchangeables

```ts
export interface PricingStrategy {
  compute(amount: number): number;
}

export const PRICING_STRATEGY = new InjectionToken<PricingStrategy>('PRICING_STRATEGY');

@Injectable()
export class DefaultPricing implements PricingStrategy {
  compute(a: number) { return a; }
}

@Injectable({ providedIn: 'root' })
export class CheckoutService {
  constructor(@Inject(PRICING_STRATEGY) private strategy: PricingStrategy) {}
}
```

- Vous pouvez remplacer la stratégie par environnement, par feature ou même par composant.

---

## 11. Débogage et diagnostic

### 11.1 Lire `NullInjectorError`

Le message contient souvent une chaîne du type :

- `R3InjectorError(AppModule)[A -> B -> C]: NullInjectorError: No provider for C!`

Interprétation : A dépend de B dépend de C. C n’est pas fourni dans l’injecteur courant.

### 11.2 Problèmes courants

- **Dépendances circulaires** : ServiceA -> ServiceB -> ServiceA.
  - Solution : extraire une interface/token, inverser, utiliser `Injector` pour lazy-get, ou refactor.

- **Provider dupliqué** : service prétendument singleton, fourni deux fois.
  - Solution : mettre au root, supprimer `providers` au niveau module/feature.

- **Scope involontaire** via composant parent.
  - Solution : déplacer le provider ou utiliser `@SkipSelf()`.

### 11.3 Techniques utiles

- Injecter `Injector` pour investiguer :

```ts
constructor(private injector: Injector) {
  const logger = this.injector.get(LoggerService);
}
```

- Ajouter des logs dans constructors de services pour voir combien d’instances sont créées.

---

## 12. Exercices (avec énoncés et attentes)

### Exercice 1 — Comprendre le masquage (shadowing)

**Énoncé** :
- Déclarez `LoggerService` au root.
- Ajoutez `providers: [LoggerService]` sur un composant parent.
- Injectez-le dans parent et enfant et comparez les instances.

**Attendus** :
- Le parent et l’enfant partagent la même instance *du provider composant* (distincte du root).

### Exercice 2 — Token de config + useFactory

**Énoncé** :
- Créez `API_CONFIG` (InjectionToken) et un provider `useFactory` qui choisit l’URL selon `isDevMode()`.

**Attendus** :
- `ApiClient` reçoit une config différente selon mode.

### Exercice 3 — Multi-provider plugin

**Énoncé** :
- Créez un token `FORMATTERS` multi.
- Ajoutez 3 implémentations dans des fichiers/sections distinctes.
- Implémentez un `FormatService` qui les liste.

**Attendus** :
- `FORMATTERS` injecté en tableau, ordre cohérent, pas d’écrasement.

### Exercice 4 — Optional + Self

**Énoncé** :
- Dans une directive, injectez `@Optional() @Self() LocalStateService`.
- Observer le comportement selon que la directive déclare ou non le provider.

**Attendus** :
- Sans provider local : valeur null (pas d’erreur).

---

## 13. Checklist “DI avancée”

Avant de choisir une stratégie DI, posez-vous :

1. Cette dépendance doit-elle être **singleton global** ?
2. Doit-elle être **isolée par feature lazy** ?
3. Doit-elle être **isolée par composant** (multi-instances) ?
4. Est-ce une **implémentation** (classe) ou une **configuration** (token) ?
5. A-t-on besoin de **plusieurs valeurs** (multi-provider) ?
6. Risque-t-on des **providers dupliqués** ?
7. Le design facilite-t-il les tests (override providers) ?

---

## 14. Annexes — Références rapides (cheat sheet)

### 14.1 Provider syntaxe

```ts
providers: [
  MyService,
  { provide: TOKEN, useValue: ... },
  { provide: TOKEN, useClass: Impl },
  { provide: TOKEN, useExisting: OtherToken },
  { provide: TOKEN, useFactory: (a: A, b: B) => ..., deps: [A, B] },
  { provide: MULTI_TOKEN, useValue: ..., multi: true },
]
```

### 14.2 Décorateurs

- `@Inject(TOKEN)`
- `@Optional()`
- `@Self()`
- `@SkipSelf()`
- `@Host()`

---

## 15. Conclusion

La DI Angular est un système hiérarchique puissant : comprendre *où* un provider est déclaré, *quel injecteur* le détient, et *comment* Angular résout un token est indispensable pour maîtriser les architectures modernes (standalone, lazy loading, features modulaires, plugins).

La progression naturelle après cette formation :

- structurer une app en scopes explicites (core/feature/component)
- mettre en place des architectures extensibles (multi-providers)
- industrialiser tests et overrides de providers

---

*Fin de cours — Injection de dépendances en profondeur*.
