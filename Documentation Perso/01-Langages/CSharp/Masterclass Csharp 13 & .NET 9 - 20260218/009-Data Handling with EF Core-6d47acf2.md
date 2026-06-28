# Masterclass — Data Handling with EF Core (.NET 9 / C# 13)

> Formation destinée à des développeurs .NET/C# souhaitant maîtriser la persistance des données avec **Entity Framework Core** : **DbContext**, **DbSet**, **migrations**, **CRUD**, **requêtes LINQ**, **optimisation des performances** et **configuration base de données**.

---

## Sommaire

1. [Objectifs & prérequis](#1-objectifs--prérequis)
2. [Vue d’ensemble : EF Core & architecture](#2-vue-densemble--ef-core--architecture)
3. [Démarrage du projet & packages](#3-démarrage-du-projet--packages)
4. [Modélisation du domaine : entités, clés, relations](#4-modélisation-du-domaine--entités-clés-relations)
5. [DbContext : rôle, cycle de vie, configuration](#5-dbcontext--rôle-cycle-de-vie-configuration)
6. [DbSet : accès aux agrégats, requêtes, tracking](#6-dbset--accès-aux-agrégats-requêtes-tracking)
7. [Migrations : création, application, gestion des environnements](#7-migrations--création-application-gestion-des-environnements)
8. [CRUD complet : Create / Read / Update / Delete](#8-crud-complet--create--read--update--delete)
9. [LINQ avec EF Core : traductions SQL & pièges](#9-linq-avec-ef-core--traductions-sql--pièges)
10. [Optimisation des performances](#10-optimisation-des-performances)
11. [Configuration base de données : providers, options, conventions](#11-configuration-base-de-données--providers-options-conventions)
12. [Atelier fil rouge : mini API + bonnes pratiques](#12-atelier-fil-rouge--mini-api--bonnes-pratiques)
13. [Checklist récap & ressources](#13-checklist-récap--ressources)

---

## 1. Objectifs & prérequis

### Objectifs pédagogiques
À la fin de cette formation, vous serez capable de :

- Structurer une couche data avec **EF Core** de manière propre et testable.
- Concevoir un modèle relationnel via **entités**, relations, contraintes, index.
- Maîtriser **DbContext** (configuration, cycle de vie, tracking, transactions).
- Utiliser **DbSet** pour requêter et manipuler les données.
- Mettre en place des **migrations** fiables (dev/CI/CD/prod).
- Écrire du **CRUD** robuste et performant.
- Écrire des **requêtes LINQ** correctement traduites en SQL.
- Optimiser : **AsNoTracking**, projections, split queries, indexation, compiled queries, batching.
- Configurer le provider et les **options** EF Core selon les besoins.

### Prérequis
- C# (classes, records, async/await), LINQ de base.
- Connaissances SQL de base (SELECT/WHERE/JOIN/INDEX).

### Format conseillé
- Durée indicative : **1 jour (7h)** ou **2 x 3h30**.
- Alternance : concepts → démo → exercices.

---

## 2. Vue d’ensemble : EF Core & architecture

### EF Core, c’est quoi ?
Entity Framework Core est un ORM (Object-Relational Mapper) :

- Permet de manipuler des **objets .NET** (entités) tout en persistant dans une **base relationnelle**.
- Gère la traduction **LINQ → SQL**.
- Gère le **change tracking** (suivi des modifications) et la génération de commandes SQL.

### Concepts clés
- **Entité** : objet persisté.
- **Model** : description du schéma (tables, relations, types).
- **DbContext** : unité de travail, session avec la base.
- **DbSet<TEntity>** : porte d’entrée vers un type d’entité.
- **Provider** : SQL Server, PostgreSQL, SQLite, etc.
- **Migrations** : versionnement du schéma.

### Patterns associés
- **Unit of Work** : `DbContext`.
- **Repository** : optionnel. Souvent inutile si on expose `DbContext` proprement. Peut être utile pour encapsuler des requêtes complexes.

---

## 3. Démarrage du projet & packages

### Choix du scénario
Nous utiliserons un mini domaine « e-commerce » :

- `Customer` (client)
- `Order` (commande)
- `OrderLine` (lignes de commande)
- `Product`

### Création d’un projet (exemple)
- API : `dotnet new webapi`
- Ou console : `dotnet new console`

### Packages utiles
Pour SQL Server (exemple) :

```bash
dotnet add package Microsoft.EntityFrameworkCore
dotnet add package Microsoft.EntityFrameworkCore.SqlServer
dotnet add package Microsoft.EntityFrameworkCore.Design
```

> Pour PostgreSQL : `Npgsql.EntityFrameworkCore.PostgreSQL`.

### Outils CLI EF
Installer (global ou local tool) :

```bash
dotnet tool install --global dotnet-ef
# ou en local
# dotnet new tool-manifest
# dotnet tool install dotnet-ef
```

---

## 4. Modélisation du domaine : entités, clés, relations

### Entités (exemple)

```csharp
public sealed class Customer
{
    public int Id { get; set; }
    public required string Email { get; set; }
    public string? DisplayName { get; set; }

    public List<Order> Orders { get; set; } = new();
}

public sealed class Order
{
    public int Id { get; set; }
    public DateTime CreatedUtc { get; set; } = DateTime.UtcNow;

    public int CustomerId { get; set; }
    public Customer Customer { get; set; } = default!;

    public List<OrderLine> Lines { get; set; } = new();
}

public sealed class OrderLine
{
    public int Id { get; set; }

    public int OrderId { get; set; }
    public Order Order { get; set; } = default!;

    public int ProductId { get; set; }
    public Product Product { get; set; } = default!;

    public int Quantity { get; set; }
    public decimal UnitPrice { get; set; }
}

public sealed class Product
{
    public int Id { get; set; }
    public required string Sku { get; set; }
    public required string Name { get; set; }
    public decimal Price { get; set; }
}
```

### Conventions vs Fluent API vs Data Annotations
EF Core fonctionne par conventions (Id → clé primaire, `CustomerId` → FK, etc.).

- **Conventions** : rapide, suffisant pour des modèles simples.
- **Data annotations** : utiles pour quelques contraintes (`[MaxLength]`, `[Required]`).
- **Fluent API (OnModelCreating)** : recommandé dès que le modèle se complexifie.

### Configuration du modèle (Fluent API)

```csharp
using Microsoft.EntityFrameworkCore;

public sealed class AppDbContext : DbContext
{
    public DbSet<Customer> Customers => Set<Customer>();
    public DbSet<Order> Orders => Set<Order>();
    public DbSet<OrderLine> OrderLines => Set<OrderLine>();
    public DbSet<Product> Products => Set<Product>();

    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Customer>(e =>
        {
            e.HasIndex(x => x.Email).IsUnique();
            e.Property(x => x.Email).HasMaxLength(320);
        });

        modelBuilder.Entity<Product>(e =>
        {
            e.HasIndex(x => x.Sku).IsUnique();
            e.Property(x => x.Sku).HasMaxLength(64);
            e.Property(x => x.Name).HasMaxLength(200);
            e.Property(x => x.Price).HasPrecision(18, 2);
        });

        modelBuilder.Entity<OrderLine>(e =>
        {
            e.Property(x => x.UnitPrice).HasPrecision(18, 2);
            e.HasIndex(x => new { x.OrderId, x.ProductId });
        });
    }
}
```

---

## 5. DbContext : rôle, cycle de vie, configuration

### Rôle
`DbContext` est :

- une session de travail avec la base
- un **change tracker** (suit les entités chargées)
- un orchestrateur pour `SaveChanges()` / `SaveChangesAsync()`
- une fabrique de transactions et de connexions

### Cycle de vie (ASP.NET Core)
En général : `Scoped` (un context par requête HTTP).

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
{
    options.UseSqlServer(builder.Configuration.GetConnectionString("AppDb"));
});
```

#### Bonnes pratiques
- **Ne pas** réutiliser un même `DbContext` sur une longue durée (risque mémoire + tracking).
- **Ne pas** partager un `DbContext` entre threads.
- Préférer `await using` si vous créez le contexte manuellement.

### Change tracking : les états
- `Added`, `Unchanged`, `Modified`, `Deleted`, `Detached`

Inspection utile :

```csharp
var entries = context.ChangeTracker.Entries();
```

### Transactions
`SaveChanges` est transactionnel par défaut pour un lot de commandes.
Pour un scénario multi-étapes :

```csharp
await using var tx = await context.Database.BeginTransactionAsync();

// ... opérations
await context.SaveChangesAsync();
await tx.CommitAsync();
```

---

## 6. DbSet : accès aux agrégats, requêtes, tracking

### DbSet : la porte d’entrée
- Ajout : `Add`, `AddAsync`
- Suppression : `Remove`
- Requêtes : LINQ (`Where`, `Select`, `Include`…)

### Exemple : ajout

```csharp
var customer = new Customer { Email = "alice@demo.com", DisplayName = "Alice" };
context.Customers.Add(customer);
await context.SaveChangesAsync();
```

### Find vs Single/First
- `Find` utilise le cache du context (tracking) puis la base si nécessaire.

```csharp
var customer = await context.Customers.FindAsync(customerId);
```

### Include / ThenInclude
Pour charger un graphe :

```csharp
var order = await context.Orders
    .Include(o => o.Lines)
    .ThenInclude(l => l.Product)
    .SingleAsync(o => o.Id == orderId);
```

> Attention : le `Include` peut générer des jointures lourdes ou du « cartesian explosion ».

---

## 7. Migrations : création, application, gestion des environnements

### Pourquoi des migrations ?
- Versionner le schéma au même rythme que le code.
- Déployer des évolutions de manière contrôlée.

### Créer une migration

```bash
dotnet ef migrations add InitialCreate
```

### Appliquer à la base

```bash
dotnet ef database update
```

### Stratégies d’utilisation
- Dev : migrations fréquentes.
- CI/CD : appliquer au démarrage (selon politique) ou via pipeline.

### Bonnes pratiques
- Nommer clairement les migrations.
- Relire les scripts générés (surtout opérations détruisant des données).
- Préférer des migrations petites et cohérentes.

### Script SQL de migration (utile en prod)

```bash
dotnet ef migrations script --idempotent -o migration.sql
```

---

## 8. CRUD complet : Create / Read / Update / Delete

### Create (avec relations)

```csharp
var order = new Order
{
    CustomerId = customerId,
    Lines =
    {
        new OrderLine { ProductId = 1, Quantity = 2, UnitPrice = 10m },
        new OrderLine { ProductId = 2, Quantity = 1, UnitPrice = 20m }
    }
};

context.Orders.Add(order);
await context.SaveChangesAsync();
```

### Read (projection DTO)

```csharp
var orders = await context.Orders
    .Where(o => o.CustomerId == customerId)
    .OrderByDescending(o => o.CreatedUtc)
    .Select(o => new
    {
        o.Id,
        o.CreatedUtc,
        LinesCount = o.Lines.Count,
        Total = o.Lines.Sum(l => l.UnitPrice * l.Quantity)
    })
    .ToListAsync();
```

### Update (tracked)

```csharp
var product = await context.Products.SingleAsync(p => p.Id == id);
product.Price = newPrice;
await context.SaveChangesAsync();
```

### Update (detached)
Cas fréquent : API reçoit un DTO.

```csharp
var product = new Product { Id = id, Price = newPrice, Name = "X", Sku = "Y" };
context.Products.Attach(product);
context.Entry(product).Property(p => p.Price).IsModified = true;
await context.SaveChangesAsync();
```

> Ne marquez pas toute l’entité en `Modified` sans raison : risque d’écraser des champs.

### Delete

```csharp
var customer = await context.Customers.FindAsync(id);
if (customer is null) return;

context.Customers.Remove(customer);
await context.SaveChangesAsync();
```

### Concurrency (optimistic)
Ajoutez un token (SQL Server `rowversion`) :

```csharp
public sealed class Product
{
    public int Id { get; set; }
    public required string Sku { get; set; }
    public required string Name { get; set; }
    public decimal Price { get; set; }

    public byte[] RowVersion { get; set; } = default!;
}
```

Config Fluent :

```csharp
modelBuilder.Entity<Product>()
    .Property(p => p.RowVersion)
    .IsRowVersion();
```

Gérer l’exception : `DbUpdateConcurrencyException`.

---

## 9. LINQ avec EF Core : traductions SQL & pièges

### Règle d’or
**Tout LINQ n’est pas traduisible** en SQL. EF Core traduit une expression LINQ en SQL, sinon :
- soit il lève une exception,
- soit il exécute certaines parties côté client (de moins en moins autorisé).

### Exemples de requêtes idiomatiques

#### Filtrer, trier
```csharp
var products = await context.Products
    .Where(p => p.Price >= 10m)
    .OrderBy(p => p.Name)
    .ToListAsync();
```

#### Agrégats
```csharp
var avg = await context.Products.AverageAsync(p => p.Price);
```

#### GroupBy (attention)
Le `GroupBy` peut être coûteux. Préférez projeter ce qui est nécessaire.

```csharp
var salesByProduct = await context.OrderLines
    .GroupBy(l => l.ProductId)
    .Select(g => new { ProductId = g.Key, Qty = g.Sum(x => x.Quantity) })
    .ToListAsync();
```

### Pièges courants

- **N+1 queries** : boucle + requête par item.
- `Include` excessifs : charge des données inutiles.
- `ToList()` trop tôt : force l’exécution et bascule en mémoire.
- Méthodes .NET non traduisibles (certaines regex, fonctions custom…).

### Debug : voir le SQL

```csharp
var sql = context.Orders
    .Where(o => o.CustomerId == customerId)
    .ToQueryString();
```

---

## 10. Optimisation des performances

### 10.1 AsNoTracking / Tracking
Pour des requêtes read-only :

```csharp
var list = await context.Products
    .AsNoTracking()
    .Where(p => p.Price > 0)
    .ToListAsync();
```

- `AsNoTracking()` supprime le coût du change tracker.
- `AsNoTrackingWithIdentityResolution()` limite les duplications d’instances (utile avec `Include`).

### 10.2 Projections (DTO) plutôt que Include
Si l’objectif est un écran / API DTO, projetez :

```csharp
var dto = await context.Orders
  .AsNoTracking()
  .Where(o => o.Id == orderId)
  .Select(o => new OrderDetailsDto(
      o.Id,
      o.CreatedUtc,
      o.Customer.Email,
      o.Lines.Select(l => new OrderLineDto(l.Product.Sku, l.Quantity, l.UnitPrice)).ToList()
  ))
  .SingleAsync();
```

### 10.3 Split queries vs Single query
Les `Include` multi-collections peuvent exploser en nombre de lignes.

```csharp
var orders = await context.Orders
    .AsSplitQuery()
    .Include(o => o.Lines)
    .ToListAsync();
```

### 10.4 Compiled queries
Pour des requêtes très fréquentes :

```csharp
static readonly Func<AppDbContext, int, Task<Product?>> GetProductById =
    EF.CompileAsyncQuery((AppDbContext ctx, int id) =>
        ctx.Products.AsNoTracking().SingleOrDefault(p => p.Id == id));

var p = await GetProductById(context, id);
```

### 10.5 Batching & SaveChanges
EF Core regroupe souvent les commandes, mais :
- faites des `SaveChanges` à des points cohérents,
- évitez 1 `SaveChanges` par entité dans une boucle.

### 10.6 Indexation & schéma
- Index sur colonnes filtrées/triées fréquemment (`Email`, `Sku`, `CreatedUtc`…)
- Contrainte d’unicité pour la cohérence.

### 10.7 Pooling de DbContext
Dans certains workloads, activez le pooling :

```csharp
builder.Services.AddDbContextPool<AppDbContext>(options =>
{
    options.UseSqlServer(builder.Configuration.GetConnectionString("AppDb"));
});
```

> Assurez-vous que votre configuration par requête est compatible (pas d’état mutable dans le contexte).

### 10.8 Logging & diagnostics

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
{
    options
        .UseSqlServer(cs)
        .EnableSensitiveDataLogging(false)
        .EnableDetailedErrors();
});
```

- Activez `SensitiveDataLogging` uniquement en dev.

---

## 11. Configuration base de données : providers, options, conventions

### Connection string
Exemple `appsettings.json` :

```json
{
  "ConnectionStrings": {
    "AppDb": "Server=localhost;Database=EfCoreTraining;Trusted_Connection=True;TrustServerCertificate=True"
  }
}
```

### Provider SQL Server

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
{
    options.UseSqlServer(
        builder.Configuration.GetConnectionString("AppDb"),
        sql =>
        {
            sql.EnableRetryOnFailure(5);
            sql.CommandTimeout(30);
        });
});
```

### Conventions globales
Par exemple fixer une précision par défaut pour les `decimal` :

```csharp
protected override void ConfigureConventions(ModelConfigurationBuilder builder)
{
    builder.Properties<decimal>().HavePrecision(18, 2);
}
```

### Value conversions
Mapper un type domaine vers un type stocké.

```csharp
public readonly record struct Sku(string Value);

modelBuilder.Entity<Product>()
    .Property<string>("Sku")
    .HasConversion(
        v => v,
        v => v
    );
```

> Souvent on fait : `Property(p => p.Sku).HasConversion(s => s.Value, v => new Sku(v))`.

### Interceptors (avancé)
- Log des commandes
- Audit
- Multi-tenant
- Soft delete

---

## 12. Atelier fil rouge : mini API + bonnes pratiques

### Objectif atelier
Créer une API minimaliste :

- `POST /customers`
- `POST /orders`
- `GET /orders/{id}`
- `GET /products?minPrice=...`

### DTOs (exemple)

```csharp
public sealed record CreateCustomerDto(string Email, string? DisplayName);
public sealed record CreateOrderDto(int CustomerId, List<CreateOrderLineDto> Lines);
public sealed record CreateOrderLineDto(int ProductId, int Quantity);

public sealed record OrderDetailsDto(
    int Id,
    DateTime CreatedUtc,
    string CustomerEmail,
    List<OrderLineDto> Lines);

public sealed record OrderLineDto(string Sku, int Quantity, decimal UnitPrice);
```

### Endpoint exemple (Minimal API)

```csharp
app.MapPost("/customers", async (CreateCustomerDto dto, AppDbContext db) =>
{
    var customer = new Customer { Email = dto.Email, DisplayName = dto.DisplayName };

    db.Customers.Add(customer);
    await db.SaveChangesAsync();

    return Results.Created($"/customers/{customer.Id}", new { customer.Id });
});
```

### Exercices proposés
1. Ajouter un index sur `Order.CreatedUtc` et mesurer une requête de listing.
2. Remplacer un `Include` par une projection DTO.
3. Corriger un N+1 (boucle) via requête unique.
4. Mettre en place une migration qui ajoute un champ `IsActive` à `Customer`.
5. Implémenter une mise à jour partielle (PATCH) en évitant l’écrasement de colonnes.

---

## 13. Checklist récap & ressources

### Checklist
- [ ] `DbContext` scoped, pas partagé, pas long-lived.
- [ ] Requêtes read-only en `AsNoTracking()`.
- [ ] DTO via `Select` (projection) quand possible.
- [ ] Limiter les `Include`, envisager `AsSplitQuery()`.
- [ ] Éviter `ToList()` prématuré et la logique côté client.
- [ ] Index sur colonnes fréquemment filtrées/triées.
- [ ] Migrations petites, revues, scriptées pour prod.
- [ ] Gérer la concurrence (token) si besoin.

### Ressources
- Docs EF Core : https://learn.microsoft.com/ef/core/
- Performance EF Core : https://learn.microsoft.com/ef/core/performance/
- Migrations : https://learn.microsoft.com/ef/core/managing-schemas/migrations/

---

## Annexes — Snippets utiles

### A. Seed simple (dev)

```csharp
if (app.Environment.IsDevelopment())
{
    using var scope = app.Services.CreateScope();
    var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();

    await db.Database.MigrateAsync();

    if (!await db.Products.AnyAsync())
    {
        db.Products.AddRange(
            new Product { Sku = "SKU-001", Name = "Keyboard", Price = 49.99m },
            new Product { Sku = "SKU-002", Name = "Mouse", Price = 19.99m }
        );
        await db.SaveChangesAsync();
    }
}
```

### B. Pagination

```csharp
int page = 1, pageSize = 20;

var items = await context.Products
    .AsNoTracking()
    .OrderBy(p => p.Id)
    .Skip((page - 1) * pageSize)
    .Take(pageSize)
    .ToListAsync();
```

### C. Exécuter du SQL brut (à utiliser avec prudence)

```csharp
var rows = await context.Database.ExecuteSqlInterpolatedAsync(
    $"UPDATE Products SET Price = Price * 1.05 WHERE Price < {threshold}");
```

> Préférez toujours les requêtes LINQ lorsque c’est possible.
