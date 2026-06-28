
# Concepts clés (avec exemples C#)

> Exemples principalement orientés **ORM / EF Core** et bonnes pratiques d’architecture.

---

## 1) LAZY vs EAGER Loading

### Lazy Loading (chargement à la demande)
Les données liées ne sont chargées **que lorsqu’on accède** à la navigation.

**EF Core (Lazy Loading Proxies)**
```csharp
// 1) Installer: Microsoft.EntityFrameworkCore.Proxies
// 2) Activer:
services.AddDbContext<AppDbContext>(opt =>
    opt.UseSqlServer(connString)
       .UseLazyLoadingProxies());

// 3) Navigations virtual pour permettre le proxy:
public class Order
{
    public int Id { get; set; }
    public virtual Customer Customer { get; set; } = default!;
}
```

**Exemple d’utilisation**
```csharp
var orders = await db.Orders.ToListAsync();

// Ici EF déclenche une requête supplémentaire par accès si Customer n'est pas déjà chargé
foreach (var o in orders)
{
    Console.WriteLine(o.Customer.Name); // Lazy load -> requête(s) au runtime
}
```

✅ Avantages : code simple, charge uniquement ce qui est utilisé.  
⚠️ Risques : *N+1*, requêtes inattendues, problèmes en sérialisation (JSON), perf difficiles à maîtriser.

---

### Eager Loading (chargement immédiat)
Les données liées sont chargées **en une seule intention de lecture** via `Include`.

**Exemple EF Core**
```csharp
var orders = await db.Orders
    .Include(o => o.Customer)
    .ToListAsync();

foreach (var o in orders)
{
    Console.WriteLine(o.Customer.Name); // pas de requête supplémentaire
}
```

✅ Avantages : requêtes maîtrisées, évite souvent le N+1.  
⚠️ Risques : peut sur-charger la requête si on inclut “tout et n’importe quoi”.

---

## 2) N+1 Problem

### Pourquoi ça arrive ?
On fait **1 requête** pour récupérer N entités, puis **N requêtes** supplémentaires pour récupérer une relation (souvent via Lazy Loading ou accès navigation non préchargée).

**Exemple (N+1)**
```csharp
var orders = await db.Orders.ToListAsync(); // 1 requête

foreach (var o in orders)                   // N itérations
{
    // Si Customer n'est pas déjà en mémoire: 1 requête par order
    Console.WriteLine(o.Customer.Name);     // N requêtes -> total = N + 1
}
```

### Corrections courantes

**A) Eager loading avec Include**
```csharp
var orders = await db.Orders
    .Include(o => o.Customer)
    .ToListAsync();
```

**B) Projection (souvent le meilleur en lecture)**
```csharp
var orderDtos = await db.Orders
    .Select(o => new
    {
        o.Id,
        CustomerName = o.Customer.Name
    })
    .ToListAsync();
```

**C) Split queries (utile si grosse explosion de jointures)**
```csharp
var orders = await db.Orders
    .Include(o => o.Customer)
    .AsSplitQuery() // évite certains cartésiens, au prix de plusieurs requêtes contrôlées
    .ToListAsync();
```

---

# Méthodes & Architectures (≤ 2 phrases + mini-exemple C#)

## SOLID
Cinq principes OO pour rendre le code **maintenable, testable et évolutif** (responsabilités claires + dépendances maîtrisées). On y gagne surtout via l’injection de dépendances et l’abstraction.

**Exemple (DIP)**
```csharp
public interface IClock { DateTime UtcNow { get; } }
public sealed class SystemClock : IClock { public DateTime UtcNow => DateTime.UtcNow; }

public sealed class TokenService
{
    private readonly IClock _clock;
    public TokenService(IClock clock) => _clock = clock;

    public bool IsExpired(DateTime expiryUtc) => _clock.UtcNow >= expiryUtc;
}
```

## Architecture hexagonale
Le domaine (cœur métier) est indépendant des technologies : les I/O passent par des **ports** (interfaces) et des **adaptateurs** (implémentations). Cela facilite tests, remplacements (DB, HTTP, bus), et évolution.

**Exemple (Port + Adapter)**
```csharp
// Port
public interface IOrderRepository { Task<Order?> GetByIdAsync(int id); }

// Adapter (EF Core)
public sealed class EfOrderRepository : IOrderRepository
{
    private readonly AppDbContext _db;
    public EfOrderRepository(AppDbContext db) => _db = db;

    public Task<Order?> GetByIdAsync(int id) => _db.Orders.FindAsync(id).AsTask();
}
```

## CQRS
On sépare **écriture** (Command) et **lecture** (Query) pour clarifier le modèle et scaler différemment selon les besoins. Les lectures retournent souvent des DTO optimisés.

**Exemple (Command / Query)**
```csharp
public sealed record CreateOrderCommand(int CustomerId);
public sealed record GetOrderQuery(int OrderId);

public sealed class CreateOrderHandler
{
    public Task<int> Handle(CreateOrderCommand cmd) { /* write model */ return Task.FromResult(1); }
}

public sealed class GetOrderHandler
{
    public Task<OrderDto> Handle(GetOrderQuery q) { /* read model */ return Task.FromResult(new OrderDto(q.OrderId)); }
}

public sealed record OrderDto(int Id);
```

## Microservices
L’application est découpée en services autonomes, **déployables indépendamment**, communiquant via HTTP/Events. On gagne en autonomie d’équipes/scalabilité, mais on paie en complexité distribuée (observabilité, réseau, cohérence).

**Exemple (contrat API minimal)**
```csharp
app.MapGet("/orders/{id:int}", async (int id, AppDbContext db) =>
    await db.Orders.FindAsync(id) is { } o ? Results.Ok(o) : Results.NotFound());
```

## Clean Architecture
Des couches concentriques où le domaine et les cas d’usage ne dépendent pas des frameworks (UI/DB/HTTP sont des détails). Les dépendances pointent **vers l’intérieur**.

**Exemple (Use Case + Interface)**
```csharp
public interface IEmailSender { Task SendAsync(string to, string subject, string body); }

public sealed class RegisterUserUseCase
{
    private readonly IEmailSender _email;
    public RegisterUserUseCase(IEmailSender email) => _email = email;

    public Task ExecuteAsync(string email) =>
        _email.SendAsync(email, "Welcome", "Hello!");
}
```

## DDD (Domain-Driven Design)
On modélise le logiciel autour du **domaine métier** : entités, value objects, agrégats, invariants, et un langage commun (Ubiquitous Language). L’objectif est une logique métier riche et cohérente, plutôt qu’un modèle anémique.

**Exemple (Value Object + invariant)**
```csharp
public readonly record struct Money(decimal Amount, string Currency)
{
    public static Money Eur(decimal amount)
        => amount < 0 ? throw new ArgumentOutOfRangeException(nameof(amount))
                      : new Money(amount, "EUR");
}

public sealed class Invoice
{
    public int Id { get; init; }
    public Money Total { get; private set; } = Money.Eur(0);

    public void AddLine(Money lineAmount)
    {
        if (lineAmount.Currency != Total.Currency) throw new InvalidOperationException("Currency mismatch");
        Total = Money.Eur(Total.Amount + lineAmount.Amount);
    }
}
```

---

## Mini check-list (pratique)
- Si tu vois des boucles qui touchent des navigations EF -> suspecte **N+1**.
- En lecture, préfère **projection** (`Select(...)`) et DTO.
- Utilise `Include` de façon ciblée, et `AsSplitQuery()` si ta requête explose en jointures.
