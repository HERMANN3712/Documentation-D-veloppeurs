# Mastering Interfaces and Inheriting Classes (C# / .NET)

> Formation avancée orientée pratique pour développeurs et formateurs .NET/C#.

## Objectifs pédagogiques

À l’issue de cette formation, vous saurez :

- Concevoir des **interfaces** robustes (API de contrats) et les faire évoluer.
- Exploiter le **polymorphisme via interfaces** pour découpler et tester.
- Choisir entre **interfaces**, **classes abstraites** et **composition**.
- Maîtriser l’héritage : `abstract`, `virtual`, `override`, `new`, `sealed`.
- Comprendre les implications **mémoire/performance** (dispatch virtuel, boxing, allocations, GC) et écrire du code efficace.

## Pré-requis

- Très bonne maîtrise de C# (LINQ, generics, async/await, exceptions).
- Compréhension de base de l’OOP (encapsulation, héritage, polymorphisme).
- Environnement : .NET 8/9, IDE (VS/VS Code/Rider).

## Public

- Développeurs .NET confirmés.
- Tech leads, architectes applicatifs.
- Formateurs souhaitant structurer un module avancé sur les contrats et la hiérarchie de types.

## Format & durée

- Durée recommandée : **1 journée (6–7h)** ou **2 demi-journées**.
- Alternance : théorie → démo → exercices → correction.

---

# Plan de la formation

1. **Pourquoi les interfaces ?** (contrats, découplage, testabilité, architecture)
2. **Interfaces en C# :** conception, bonnes pratiques, pièges, évolution
3. **Polymorphisme via interfaces :** DI, stratégies, décorateurs, adaptateurs
4. **Classes abstraites :** quand et pourquoi, template method, état partagé
5. **Héritage et résolution des membres :** `virtual`/`override`/`new`, `sealed`
6. **Gestion mémoire & performance :** vtable/dispatch, boxing, allocations, GC, patterns efficaces
7. **Atelier de synthèse :** refactoring d’un mini-projet vers une architecture orientée contrats

---

# 1) Pourquoi les interfaces ?

## 1.1 Contrat vs implémentation

Une **interface** définit *ce que* fait un composant ; une classe définit *comment*.

- Contrat stable = moins de couplage.
- Implémentation interchangeable = testabilité, extensibilité.

### Example

```csharp
public interface IClock
{
    DateTimeOffset Now { get; }
}

public sealed class SystemClock : IClock
{
    public DateTimeOffset Now => DateTimeOffset.Now;
}
```

Ici, votre code métier ne dépend pas de `DateTimeOffset.Now` directement.

## 1.2 Cas d’usage typiques

- **Infrastructure** : horloge, accès fichiers, HTTP, DB.
- **Port/Adapter (Hexagonal)** : ports = interfaces, adapters = implémentations.
- **Politiques** (strategies) : tarification, scoring, validation.
- **Tests** : mocks/fakes pour isoler.

## 1.3 Quand éviter une interface ?

- Si le type est **final** et ne doit pas varier.
- Si l’abstraction n’a qu’une seule implémentation *et* ne présente pas d’intérêt d’injection/simulation.
- Si vous ne savez pas encore ce qui varie : une interface prématurée peut devenir une dette.

> Heuristique : **abstraire le “volatil”** (ce qui change) et garder le reste simple.

---

# 2) Interfaces en C# : conception, bonnes pratiques, évolution

## 2.1 Nommer et scoper

- Préfixe `I` : `IPaymentGateway`.
- Interfaces petites et cohérentes : **ISP** (Interface Segregation Principle).

### Exemple : éviter la “god interface”

```csharp
public interface IRepository<T>
{
    Task<T?> GetByIdAsync(Guid id, CancellationToken ct);
    Task AddAsync(T entity, CancellationToken ct);
    Task SaveAsync(CancellationToken ct);
}
```

Si `SaveAsync` ne concerne pas tous les cas, séparez : `IUnitOfWork`.

## 2.2 Membres d’interface en C# (rappel)

- Signatures de méthodes
- Propriétés, indexeurs
- Événements
- Implémentations explicites

### Implémentation explicite : maîtriser l’API publique

```csharp
public interface IResettable
{
    void Reset();
}

public sealed class Counter : IResettable
{
    public int Value { get; private set; }

    void IResettable.Reset() => Value = 0; // pas visible via Counter.Reset()
}
```

Usage :

```csharp
var c = new Counter();
((IResettable)c).Reset();
```

**Avantages** :
- Évite les collisions de noms.
- Délimite ce qui fait partie du contrat.

## 2.3 Interfaces + generics

### Constrains et expressivité

```csharp
public interface ISerializer<T>
{
    string Serialize(T value);
    T Deserialize(string data);
}
```

### Variance : `out` et `in`

- `out` (covariant) : producteur.
- `in` (contravariant) : consommateur.

```csharp
public interface IProducer<out T>
{
    T Create();
}

public interface IConsumer<in T>
{
    void Consume(T item);
}
```

> Utile pour APIs de plugins, pipelines, médiateurs.

## 2.4 Évolution d’une interface (compatibilité)

Changer une interface casse les implémentations.

Stratégies :

1. **Ajouter une nouvelle interface** (versionnée)

```csharp
public interface IClockV2 : IClock
{
    TimeZoneInfo TimeZone { get; }
}
```

2. Ajouter des méthodes d’extension (n’ajoute pas au contrat, mais enrichit l’usage)

```csharp
public static class ClockExtensions
{
    public static DateOnly Today(this IClock clock) => DateOnly.FromDateTime(clock.Now.DateTime);
}
```

3. Fournir des implémentations via une classe de base abstraite (voir section 4)

> Règle : **évitez de faire évoluer brutalement** une interface publique (lib/SDK).

---

# 3) Polymorphisme via interfaces

## 3.1 Le polymorphisme “à la demande”

Vous codez contre l’interface : l’implémentation est injectée.

### Exemple : stratégie de tarification

```csharp
public interface IPricingStrategy
{
    decimal ComputePrice(Order order);
}

public sealed class RegularPricing : IPricingStrategy
{
    public decimal ComputePrice(Order order) => order.Subtotal;
}

public sealed class BlackFridayPricing : IPricingStrategy
{
    public decimal ComputePrice(Order order) => order.Subtotal * 0.7m;
}

public sealed class PricingService
{
    private readonly IPricingStrategy _strategy;

    public PricingService(IPricingStrategy strategy) => _strategy = strategy;

    public decimal Price(Order order) => _strategy.ComputePrice(order);
}

public sealed record Order(decimal Subtotal);
```

Votre code métier ne dépend plus d’un `switch` sur un type.

## 3.2 DI (Dependency Injection) et composition

Dans une application .NET moderne, l’interface s’associe naturellement à DI.

```csharp
services.AddSingleton<IClock, SystemClock>();
services.AddScoped<IPricingStrategy, RegularPricing>();
```

### Attention

- Trop d’interfaces = complexité cognitive.
- Construire un graphe d’objets démesuré peut coûter en perf et lisibilité.

## 3.3 Decorator pattern (sans héritage)

Pour enrichir sans modifier (OCP).

```csharp
public sealed class CachingClock : IClock
{
    private readonly IClock _inner;
    private DateTimeOffset _cached;
    private DateTimeOffset _cachedAt;

    public CachingClock(IClock inner) => _inner = inner;

    public DateTimeOffset Now
    {
        get
        {
            var now = _inner.Now;
            if ((now - _cachedAt) > TimeSpan.FromSeconds(1))
            {
                _cachedAt = now;
                _cached = now;
            }
            return _cached;
        }
    }
}
```

## 3.4 Adapter pattern

```csharp
public interface IEmailSender
{
    Task SendAsync(string to, string subject, string body, CancellationToken ct);
}

public sealed class SmtpEmailSenderAdapter : IEmailSender
{
    private readonly SmtpClient _client;
    public SmtpEmailSenderAdapter(SmtpClient client) => _client = client;

    public Task SendAsync(string to, string subject, string body, CancellationToken ct)
    {
        // adapter vers votre lib SMTP historique
        return _client.SendMailAsync("noreply@site", to, subject, body);
    }
}
```

---

# 4) Classes abstraites

## 4.1 Différences clés interface vs abstract class

| Aspect | Interface | Classe abstraite |
|---|---|---|
| Héritage multiple | Oui (plusieurs interfaces) | Non (une seule classe) |
| État (fields) | Non (pas de champs d’instance) | Oui |
| Implémentation partagée | Via méthodes d’extension / défaut / helpers | Naturel |
| Contrat pur | Excellent | Moins strict |
| Breaking changes | Ajouter membre casse | Ajouter abstract casse, ajouter virtual peut être OK |

## 4.2 Quand préférer une classe abstraite ?

- Vous avez un **comportement commun** + **état**.
- Vous voulez fournir une **implémentation par défaut**.
- Vous utilisez le pattern **Template Method**.

## 4.3 Template Method

```csharp
public abstract class FileImporter
{
    public async Task ImportAsync(string path, CancellationToken ct)
    {
        var content = await ReadAsync(path, ct);
        var items = Parse(content);
        await PersistAsync(items, ct);
    }

    protected virtual Task<string> ReadAsync(string path, CancellationToken ct)
        => File.ReadAllTextAsync(path, ct);

    protected abstract IEnumerable<string> Parse(string content);

    protected abstract Task PersistAsync(IEnumerable<string> items, CancellationToken ct);
}

public sealed class CsvImporter : FileImporter
{
    protected override IEnumerable<string> Parse(string content)
        => content.Split('\n', StringSplitOptions.RemoveEmptyEntries);

    protected override Task PersistAsync(IEnumerable<string> items, CancellationToken ct)
        => Task.CompletedTask;
}
```

- `ImportAsync` fixe le squelette.
- `Parse` et `PersistAsync` sont obligatoires.
- `ReadAsync` est personnalisable (ex: stockage cloud).

## 4.4 Coupler plusieurs abstractions

Approche classique : interface + classe abstraite

- Interface = contrat public
- Abstract class = base réutilisable

```csharp
public interface IImporter
{
    Task ImportAsync(string path, CancellationToken ct);
}

public abstract class ImporterBase : IImporter
{
    public abstract Task ImportAsync(string path, CancellationToken ct);
}
```

---

# 5) `virtual` / `override` / `new` (et `sealed`)

## 5.1 Dispatch statique vs dynamique

- **Non-virtuel** : résolution à la compilation.
- **Virtuel** : résolution au runtime selon le type réel.

## 5.2 `virtual` et `override`

```csharp
public class Animal
{
    public virtual string Speak() => "...";
}

public class Dog : Animal
{
    public override string Speak() => "Woof";
}

Animal a = new Dog();
Console.WriteLine(a.Speak()); // Woof
```

- `virtual` ouvre la porte.
- `override` remplace le comportement.

## 5.3 `new` : masquage (hiding) et pièges

`new` ne remplace pas, il **cache**.

```csharp
public class Base
{
    public virtual string Name() => "Base";
}

public class Derived : Base
{
    public new string Name() => "Derived";
}

Base b = new Derived();
Derived d = new Derived();

Console.WriteLine(b.Name()); // Base (virtuel de Base, pas override)
Console.WriteLine(d.Name()); // Derived (appel résolu sur le type statique Derived)
```

### Quand utiliser `new` ?

- Rare. Plutôt pour compatibilité ascendante ou API legacy.
- Préférez `override` (si possible) ou renommez.

## 5.4 `sealed` : fermer une hiérarchie ou une surcharge

- `sealed class` empêche l’héritage.
- `sealed override` empêche une surcharge ultérieure.

```csharp
public class SecureStream : Stream
{
    public sealed override bool CanRead => true;
    // ...
}
```

### Pourquoi ?

- Conception : garantir des invariants.
- Performance : parfois de meilleures optimisations (devirtualization) possibles.

## 5.5 `abstract` : imposer un contrat d’implémentation

```csharp
public abstract class Shape
{
    public abstract double Area { get; }
}

public sealed class Circle : Shape
{
    public double Radius { get; }
    public Circle(double radius) => Radius = radius;

    public override double Area => Math.PI * Radius * Radius;
}
```

---

# 6) Gestion mémoire & performance (interfaces + héritage)

Cette section relie conception (abstraction) et implications runtime.

## 6.1 Allocations : types référence vs types valeur

- `class` = référence sur le heap → GC.
- `struct` = valeur (souvent sur la stack / inline) → pas de GC *direct*.

Mais :
- Captures, boxing, closures peuvent allouer.
- Un `struct` copié souvent peut coûter cher.

## 6.2 Boxing via interface

Si un `struct` est utilisé comme interface, il peut être **boxé** (allocation) :

```csharp
public interface IIntProvider { int Get(); }

public readonly struct Five : IIntProvider
{
    public int Get() => 5;
}

IIntProvider p = new Five(); // boxing probable
Console.WriteLine(p.Get());
```

### Éviter

- Utiliser des generics :

```csharp
public static int Compute<T>(T provider) where T : struct, IIntProvider
    => provider.Get(); // évite boxing
```

- Conserver le type valeur hors interface si hot path.

## 6.3 Dispatch virtuel / appel d’interface

- Appel virtuel/interface = indirection.
- JIT peut parfois **devirtualiser** si le type concret est connu.

Bonnes pratiques :
- Dans des boucles critiques, minimiser l’abstraction *si* nécessaire.
- Mesurer avant d’optimiser (BenchmarkDotNet).

## 6.4 Vtables, object header, overhead

Notions utiles :

- Un objet sur le heap contient un **header** (sync block, type handle).
- Les classes avec virtuals ont une table de méthodes (conceptuellement vtable).
- Interface dispatch est plus complexe (lookup via tables), mais optimisée par le JIT.

> Pour la plupart des applications business, l’impact est négligeable ; il devient important en code intensif (parsing, traitements massifs, low-latency).

## 6.5 Finalizers et IDisposable

### Finalizer

- `~Type()` (finalizer) retarde la libération (2 passages GC).
- À éviter sauf si vous encapsulez des ressources non managées.

### IDisposable

- Modèle standard pour ressources managées/non-managées.
- Composition > héritage pour la gestion de ressources.

```csharp
public sealed class ResourceHolder : IDisposable
{
    private readonly Stream _stream;
    public ResourceHolder(Stream stream) => _stream = stream;

    public void Dispose() => _stream.Dispose();
}
```

### Pattern héritage : Dispose(bool)

Utile si une classe de base gère des ressources et autorise l’héritage.

```csharp
public abstract class DisposableBase : IDisposable
{
    private bool _disposed;

    public void Dispose()
    {
        Dispose(disposing: true);
        GC.SuppressFinalize(this);
    }

    protected void ThrowIfDisposed()
    {
        if (_disposed) throw new ObjectDisposedException(GetType().Name);
    }

    protected virtual void Dispose(bool disposing)
    {
        if (_disposed) return;
        if (disposing)
        {
            // free managed
        }
        // free unmanaged
        _disposed = true;
    }

    ~DisposableBase() => Dispose(disposing: false);
}
```

> Conseil : n’introduisez un finalizer que si vous avez vraiment des ressources non managées.

## 6.6 Éviter les fuites “logiques”

- Abonnements événements non désabonnés → objets retenus.
- Caches statiques non bornés.
- Captures de closures stockées.

### Ex : événements

```csharp
publisher.Changed += handler; // si handler réfère à un objet long-lived, fuite possible
```

Solution :
- `-=` au bon moment
- patterns weak event (selon besoin)
- utiliser `IDisposable` pour gérer l’abonnement

---

# 7) Atelier de synthèse (projet fil rouge)

## 7.1 Énoncé

Vous avez un service qui calcule une facture en appelant des dépendances concrètes :

- Lecture de l’heure
- Règle de remise
- Persistance

Objectifs :

1. Introduire des **interfaces** pour isoler les dépendances.
2. Implémenter une stratégie de remise via polymorphisme.
3. Ajouter un décorateur de logging/caching.
4. Utiliser une **classe abstraite** pour factoriser un pipeline commun.
5. Vérifier les implications mémoire : allocations évitables, boxing, etc.

## 7.2 Point de départ (code volontairement couplé)

```csharp
public sealed class BillingService
{
    public decimal Compute(decimal subtotal)
    {
        // dépendances concrètes + règles inline
        var now = DateTimeOffset.Now;

        var discount = now.Month == 11 ? 0.3m : 0.0m;
        var total = subtotal * (1 - discount);

        File.AppendAllText("billing.log", $"{now}: {subtotal} -> {total}\n");
        return total;
    }
}
```

## 7.3 Refactoring attendu (suggestion)

- `IClock`
- `IDiscountPolicy`
- `IBillingLogger`

```csharp
public interface IDiscountPolicy
{
    decimal DiscountRate(DateTimeOffset now, decimal subtotal);
}

public sealed class BlackFridayPolicy : IDiscountPolicy
{
    public decimal DiscountRate(DateTimeOffset now, decimal subtotal)
        => now.Month == 11 ? 0.3m : 0.0m;
}

public interface IBillingLogger
{
    void Log(DateTimeOffset now, decimal subtotal, decimal total);
}

public sealed class FileBillingLogger : IBillingLogger
{
    private readonly string _path;
    public FileBillingLogger(string path) => _path = path;

    public void Log(DateTimeOffset now, decimal subtotal, decimal total)
        => File.AppendAllText(_path, $"{now}: {subtotal} -> {total}\n");
}

public sealed class BillingService
{
    private readonly IClock _clock;
    private readonly IDiscountPolicy _policy;
    private readonly IBillingLogger _logger;

    public BillingService(IClock clock, IDiscountPolicy policy, IBillingLogger logger)
        => (_clock, _policy, _logger) = (clock, policy, logger);

    public decimal Compute(decimal subtotal)
    {
        var now = _clock.Now;
        var rate = _policy.DiscountRate(now, subtotal);
        var total = subtotal * (1 - rate);
        _logger.Log(now, subtotal, total);
        return total;
    }
}
```

## 7.4 Tests (exemple)

```csharp
public sealed class FakeClock : IClock
{
    public DateTimeOffset Now { get; set; }
}

public sealed class InMemoryLogger : IBillingLogger
{
    public List<string> Lines { get; } = [];
    public void Log(DateTimeOffset now, decimal subtotal, decimal total)
        => Lines.Add($"{now:o}: {subtotal} -> {total}");
}

// Exemple de test (xUnit)
[Fact]
public void Compute_AppliesBlackFridayDiscount()
{
    var clock = new FakeClock { Now = new DateTimeOffset(2026, 11, 10, 12, 0, 0, TimeSpan.Zero) };
    var logger = new InMemoryLogger();

    var sut = new BillingService(clock, new BlackFridayPolicy(), logger);

    var total = sut.Compute(100m);

    Assert.Equal(70m, total);
    Assert.Single(logger.Lines);
}
```

---

# Exercices (avec objectifs)

## Exercice 1 — Segmentation d’interface (ISP)

**But** : refactor une interface trop large en 2–3 interfaces cohérentes.

- Repérez les responsabilités.
- Évitez d’imposer des méthodes inutiles aux implémentations.

## Exercice 2 — `new` vs `override`

**But** : démontrer un bug lié à `new`.

- Créez une base avec méthode `virtual`.
- Dérivez en utilisant `new` volontairement.
- Observez le comportement selon le type statique.

## Exercice 3 — Boxing dans un hot path

**But** : écrire deux versions d’un traitement :

1. `IProcessor` + struct implémentant l’interface (risque boxing)
2. version générique `where T : struct, IProcessor`

Mesurez allocations/temps avec BenchmarkDotNet.

## Exercice 4 — Pipeline via abstract class

**But** : implémenter un import “JSON” en héritant d’une base.

- `ReadAsync` virtual
- `Parse` abstract
- `PersistAsync` abstract

---

# Anti-patterns et recommandations

## Anti-patterns

- “Une interface par classe” sans besoin réel.
- Interfaces géantes (violations ISP).
- Hiérarchies profondes (fragilité, effet domino).
- Utiliser `new` pour masquer accidentellement.
- Ajouter un finalizer “au cas où”.

## Recommandations

- Favorisez **composition** et **polymorphisme via interface**.
- Limitez l’héritage à des cas où il apporte un **vrai** gain.
- Considérez l’évolution : versionner les contrats.
- Mesurez les impacts performance uniquement dans les zones critiques.

---

# Checklist de conception (mémo)

- [ ] Quelle partie du système est volatile (change souvent) ?
- [ ] Ai-je un contrat clair et minimal ?
- [ ] L’interface est-elle testable et mockable ?
- [ ] Ai-je besoin d’état/implémentation partagée (classe abstraite) ?
- [ ] Ai-je introduit du `virtual` sans raison ? (risque d’API fragile)
- [ ] Risque de boxing si structs via interface ?
- [ ] Ai-je des ressources : `IDisposable` correctement géré ?

---

# Annexes

## Glossaire

- **Polymorphisme** : possibilité d’utiliser une abstraction (interface/base) pour manipuler des objets de types concrets différents via le même contrat.
- **Dispatch virtuel** : résolution d’un appel de méthode basée sur le type runtime.
- **Boxing** : conversion d’un type valeur vers `object` ou une interface (allocation heap).
- **Devirtualization** : optimisation JIT transformant un appel virtuel en appel direct.

## Références conseillées

- Documentation Microsoft : Interfaces, Héritage, `IDisposable`.
- BenchmarkDotNet pour mesurer allocations et performance.

---

*Fin de la formation — Mastering Interfaces and Inheriting Classes.*
