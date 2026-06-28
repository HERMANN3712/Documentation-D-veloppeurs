# LINQ Unleashed — Masterclass (.NET 9 / C# 13)

> Formation complète (plan + contenu détaillé) sur LINQ : syntaxe, groupements, agrégations, exécution différée, LINQ to XML/JSON et opérateurs avancés.

---

## Table des matières

1. [Objectifs pédagogiques](#objectifs-pédagogiques)
2. [Pré-requis](#pré-requis)
3. [Public cible](#public-cible)
4. [Format, durée et modalités](#format-durée-et-modalités)
5. [Environnement & setup](#environnement--setup)
6. [Programme détaillé (chapitres)](#programme-détaillé-chapitres)
    1. [Chapitre 1 — LINQ Mental Model](#chapitre-1--linq-mental-model)
    2. [Chapitre 2 — Syntaxe LINQ (Method vs Query)](#chapitre-2--syntaxe-linq-method-vs-query)
    3. [Chapitre 3 — Filtrage, projection, tri](#chapitre-3--filtrage-projection-tri)
    4. [Chapitre 4 — Grouping & partitionnement](#chapitre-4--grouping--partitionnement)
    5. [Chapitre 5 — Agrégation & statistiques](#chapitre-5--agrégation--statistiques)
    6. [Chapitre 6 — Deferred execution, streaming, pièges](#chapitre-6--deferred-execution-streaming-pièges)
    7. [Chapitre 7 — Opérateurs de jointure & set](#chapitre-7--opérateurs-de-jointure--set)
    8. [Chapitre 8 — Opérateurs avancés](#chapitre-8--opérateurs-avancés)
    9. [Chapitre 9 — LINQ to XML (XDocument, XElement)](#chapitre-9--linq-to-xml-xdocument-xelement)
    10. [Chapitre 10 — LINQ to JSON (System.Text.Json)](#chapitre-10--linq-to-json-systemtextjson)
    11. [Chapitre 11 — Performance, allocation, bonnes pratiques](#chapitre-11--performance-allocation-bonnes-pratiques)
    12. [Chapitre 12 — Atelier final “LINQ Unleashed”](#chapitre-12--atelier-final-linq-unleashed)
7. [Katas & exercices guidés (corrigés)](#katas--exercices-guidés-corrigés)
8. [Références & mémo opérateurs](#références--mémo-opérateurs)

---

## Objectifs pédagogiques

À l’issue de la formation, le participant saura :

- Expliquer le **modèle d’exécution** de LINQ (IEnumerable vs IQueryable, streaming vs buffering).
- Écrire des requêtes **en syntaxe méthode et en syntaxe query**.
- Utiliser efficacement **grouping**, **agrégation**, **jointures** et **opérateurs avancés**.
- Comprendre et maîtriser **deferred execution** et les effets de bords (sources mutables, multi-enumération, exceptions tardives).
- Manipuler des données semi-structurées via **LINQ to XML** et des équivalents **LINQ-like** sur **JSON**.
- Produire du code **lisible**, **testable** et **performant**.

---

## Pré-requis

- C# (niveau intermédiaire) : classes, generics, lambdas, LINQ basique éventuellement.
- Connaissances sur collections .NET (List, Dictionary) et notions d’immutabilité.

---

## Public cible

- Développeurs .NET / C#.
- Formateurs et référents techniques.
- Équipes travaillant avec des pipelines de données, APIs, ETL applicatifs.

---

## Format, durée et modalités

- **Durée recommandée** : 1 à 2 jours.
- Alternance : explications → démos → exercices → revue.
- Tous les exemples sont en **.NET 9 / C# 13** (compatibles globalement dès .NET 6+).

---

## Environnement & setup

- SDK : **.NET 9**
- IDE : Visual Studio / Rider / VS Code

Créez un projet console :

```bash
dotnet new console -n LinqUnleashed
cd LinqUnleashed
```

Ajoutez un fichier `Models.cs` avec les modèles utilisés dans les exemples.

---

## Programme détaillé (chapitres)

### Dataset de référence (utilisé dans la formation)

> On utilise un dataset simple, réutilisable dans tous les chapitres.

```csharp
public sealed record Customer(int Id, string Name, string Country);
public sealed record Order(int Id, int CustomerId, DateTime Date, decimal Amount, string Currency, string[] Tags);

public static class SampleData
{
    public static readonly Customer[] Customers =
    [
        new(1, "Ada", "FR"),
        new(2, "Bo", "FR"),
        new(3, "Chloé", "BE"),
        new(4, "Dinesh", "IN"),
        new(5, "Eve", "US"),
    ];

    public static readonly Order[] Orders =
    [
        new(101, 1, new DateTime(2024, 01, 12), 120m, "EUR", ["hardware", "priority"]),
        new(102, 1, new DateTime(2024, 02, 08),  80m, "EUR", ["software"]),
        new(103, 2, new DateTime(2024, 02, 20),  35m, "EUR", ["software", "trial"]),
        new(104, 3, new DateTime(2024, 03, 02), 220m, "EUR", ["hardware"]),
        new(105, 3, new DateTime(2024, 03, 18),  10m, "EUR", ["support"]),
        new(106, 4, new DateTime(2024, 01, 05),  60m, "INR", ["support"]),
        new(107, 5, new DateTime(2024, 04, 10), 500m, "USD", ["hardware", "vip"]),
    ];
}
```

---

## Chapitre 1 — LINQ Mental Model

### 1.1 Qu’est-ce que LINQ ?

LINQ = **Language Integrated Query**. Concrètement :

- Une **API d’extension methods** sur `IEnumerable<T>` (LINQ to Objects) : `Where`, `Select`, `GroupBy`, etc.
- Une **syntaxe query** (sucre syntaxique) du langage C#.
- Des **providers** LINQ : `IQueryable<T>` (Entity Framework, etc.) où une expression est traduite (SQL, etc.).

### 1.2 IEnumerable vs IQueryable

- `IEnumerable<T>` :
  - Exécution dans le **runtime .NET**.
  - Les delegates (lambdas) s’exécutent en mémoire.
  - Très flexible, mais coûts dépendants de l’énumération.

- `IQueryable<T>` :
  - Les lambdas deviennent des **Expression Trees**.
  - Un provider traduit la requête.
  - Attention aux opérateurs non traduisibles.

> Cette formation se concentre sur **LINQ to Objects**, mais garde l’esprit “provider-based” pour éviter les pièges en EF.

### 1.3 Pipeline mental

Pensez à LINQ comme à un **pipeline** immuable :

- une source → des transformations (Select/Where) → un terminal (ToList/Sum/etc.).
- les transformations sont généralement **lazy** (voir chapitre 6).

---

## Chapitre 2 — Syntaxe LINQ (Method vs Query)

### 2.1 Syntaxe méthode (fluent)

```csharp
var frCustomers = SampleData.Customers
    .Where(c => c.Country == "FR")
    .Select(c => c.Name)
    .OrderBy(name => name)
    .ToList();
```

Points importants :

- Lecture de gauche à droite.
- Les étapes du pipeline sont explicites.
- Très complet : tous les opérateurs existent en méthode.

### 2.2 Syntaxe query

```csharp
var frCustomers =
    (from c in SampleData.Customers
     where c.Country == "FR"
     orderby c.Name
     select c.Name)
    .ToList();
```

Rappels :

- `from/where/select/orderby/group/join`.
- Traduction compilateur vers les methods `Where`, `Select`, `OrderBy`, `GroupBy`, `Join`, etc.

### 2.3 Quand utiliser quoi ?

- **Méthode** : standard équipe, composable, facile avec `SelectMany`, `Aggregate`, opérateurs avancés.
- **Query** : très lisible pour `join`, `group`, certains scénarios SQL-like.

> Bonne pratique : savoir lire *et* écrire les deux.

---

## Chapitre 3 — Filtrage, projection, tri

### 3.1 Where (filtrage)

```csharp
var bigOrders = SampleData.Orders.Where(o => o.Amount >= 100m);
```

Piège :

- `Where` ne s’exécute pas tout de suite (exécution différée).

### 3.2 Select (projection)

```csharp
var orderSummaries = SampleData.Orders
    .Select(o => new
    {
        o.Id,
        o.Amount,
        Year = o.Date.Year
    });
```

### 3.3 SelectMany (aplatissement)

Objectif : transformer **une séquence de séquences** en une séquence.

```csharp
var allTags = SampleData.Orders
    .SelectMany(o => o.Tags)
    .Distinct()
    .OrderBy(t => t)
    .ToList();
```

Modèle mental : `SelectMany` = projection + flatten.

### 3.4 OrderBy / ThenBy

```csharp
var sorted = SampleData.Orders
    .OrderBy(o => o.Currency)
    .ThenByDescending(o => o.Amount)
    .ToList();
```

### 3.5 Take/Skip et pagination

```csharp
int page = 2;
int pageSize = 3;

var page2 = SampleData.Orders
    .OrderBy(o => o.Id)
    .Skip((page - 1) * pageSize)
    .Take(pageSize)
    .ToList();
```

---

## Chapitre 4 — Grouping & partitionnement

### 4.1 GroupBy — principes

`GroupBy` produit des groupes `IGrouping<TKey, TElement>`.

```csharp
var byCurrency = SampleData.Orders
    .GroupBy(o => o.Currency);

foreach (var g in byCurrency)
{
    Console.WriteLine($"Currency={g.Key}, Count={g.Count()}");
}
```

> Attention : compter dans la boucle déclenche des énumérations (potentiellement multiples). Voir streaming/buffering.

### 4.2 GroupBy + projection immédiate

Souvent on veut **résumer** chaque groupe.

```csharp
var currencyStats = SampleData.Orders
    .GroupBy(o => o.Currency)
    .Select(g => new
    {
        Currency = g.Key,
        Count = g.Count(),
        Total = g.Sum(x => x.Amount),
        Max = g.Max(x => x.Amount)
    })
    .OrderByDescending(x => x.Total)
    .ToList();
```

### 4.3 GroupBy composite key

```csharp
var byCountryYear = SampleData.Orders
    .Join(SampleData.Customers,
        o => o.CustomerId,
        c => c.Id,
        (o, c) => new { o, c })
    .GroupBy(x => new { x.c.Country, Year = x.o.Date.Year })
    .Select(g => new
    {
        g.Key.Country,
        g.Key.Year,
        Total = g.Sum(v => v.o.Amount)
    })
    .ToList();
```

### 4.4 Partitionnement vs GroupBy

- `GroupBy` : regroupe par clé (hash-based).
- Partitionnement : `Take/Skip`, `Chunk`, `TakeWhile/SkipWhile`.

Exemple `Chunk` (.NET 6+) :

```csharp
var chunks = SampleData.Orders
    .OrderBy(o => o.Date)
    .Chunk(2)
    .ToList();
```

---

## Chapitre 5 — Agrégation & statistiques

### 5.1 Aggregate vs agrégateurs dédiés

Agrégateurs courants : `Count`, `Sum`, `Min`, `Max`, `Average`.

```csharp
var totalEur = SampleData.Orders
    .Where(o => o.Currency == "EUR")
    .Sum(o => o.Amount);
```

### 5.2 Count vs LongCount

- `Count()` retourne `int`.
- `LongCount()` retourne `long` (utilisable si très gros volumes).

### 5.3 Aggregate (réduction générale)

Exemple : concaténer des tags uniques triés en CSV.

```csharp
var csv = SampleData.Orders
    .SelectMany(o => o.Tags)
    .Distinct()
    .OrderBy(t => t)
    .Aggregate(seed: "", (acc, t) => acc.Length == 0 ? t : acc + "," + t);
```

> Pour de grandes chaînes : préférez `string.Join`.

### 5.4 Statistiques “par groupe”

```csharp
var avgPerCustomer = SampleData.Orders
    .GroupBy(o => o.CustomerId)
    .Select(g => new
    {
        CustomerId = g.Key,
        Avg = g.Average(o => o.Amount),
        Total = g.Sum(o => o.Amount)
    })
    .ToList();
```

### 5.5 Scan / cumul (rolling)

LINQ standard n’a pas `Scan`, mais on peut le coder pour certains besoins.

```csharp
public static IEnumerable<TAcc> Scan<TSource, TAcc>(
    this IEnumerable<TSource> source,
    TAcc seed,
    Func<TAcc, TSource, TAcc> accumulator)
{
    var acc = seed;
    foreach (var item in source)
    {
        acc = accumulator(acc, item);
        yield return acc;
    }
}

var rollingTotals = SampleData.Orders
    .OrderBy(o => o.Date)
    .Select(o => o.Amount)
    .Scan(0m, (acc, amount) => acc + amount)
    .ToList();
```

---

## Chapitre 6 — Deferred execution, streaming, pièges

### 6.1 Exécution différée (deferred execution)

Beaucoup d’opérateurs LINQ sont **lazy** : `Where`, `Select`, `GroupBy` (partiellement selon implémentation), etc.

```csharp
var query = SampleData.Orders.Where(o => o.Amount > 50);
// Rien n'est exécuté ici.

foreach (var o in query)
{
    // Exécution ici.
}
```

### 6.2 Opérateurs terminal / materialization

- `ToList()`, `ToArray()` : matérialisation.
- `Count()`, `Sum()` : terminal.
- `First()`, `Single()` : terminal.

### 6.3 Streaming vs buffering

- **Streaming** : production élément par élément (`Where`, `Select`, `Take`).
- **Buffering** : a besoin de tout lire (`OrderBy`, `GroupBy` en pratique, `ToList`).

Exemple : `OrderBy` bufferise.

```csharp
var q = SampleData.Orders.OrderBy(o => o.Amount);
// Nécessite de trier => lecture complète à l’énumération.
```

### 6.4 Multi-enumération

```csharp
var q = SampleData.Orders.Where(o => o.Currency == "EUR");

var c1 = q.Count(); // énumère
var c2 = q.Count(); // ré-énumère
```

Solution : matérialiser quand nécessaire.

```csharp
var eur = q.ToList();
var c1 = eur.Count;
var c2 = eur.Count;
```

### 6.5 Captures et sources mutables

Si la source change entre la définition et l’exécution, le résultat change.

```csharp
var list = new List<int> { 1, 2, 3 };
var q = list.Where(x => x > 1);

list.Add(4);
var result = q.ToList(); // 2,3,4
```

### 6.6 Exceptions tardives

```csharp
var q = SampleData.Orders.Select(o => 100m / o.Amount);
// exception potentielle à l’énumération, pas à la définition.
```

---

## Chapitre 7 — Opérateurs de jointure & set

### 7.1 Join (inner join)

```csharp
var ordersWithCustomer = SampleData.Orders
    .Join(SampleData.Customers,
        o => o.CustomerId,
        c => c.Id,
        (o, c) => new { o.Id, Customer = c.Name, o.Amount, o.Currency })
    .ToList();
```

### 7.2 GroupJoin / left join

`GroupJoin` permet un left join en combinant avec `DefaultIfEmpty`.

```csharp
var leftJoin =
    from c in SampleData.Customers
    join o in SampleData.Orders on c.Id equals o.CustomerId into orders
    from o in orders.DefaultIfEmpty()
    select new
    {
        Customer = c.Name,
        OrderId = o?.Id,
        Amount = o?.Amount
    };
```

### 7.3 Opérateurs d’ensemble

- `Distinct`, `Union`, `Intersect`, `Except`.

Exemple : comparer les pays clients vs pays livrables.

```csharp
var customerCountries = SampleData.Customers.Select(c => c.Country).Distinct();
var supported = new[] { "FR", "BE", "DE" };

var unsupported = customerCountries.Except(supported).ToList();
```

> Pour des types complexes : utilisez un `IEqualityComparer<T>`.

---

## Chapitre 8 — Opérateurs avancés

### 8.1 Quantifiers : Any, All

```csharp
bool anyVip = SampleData.Orders.Any(o => o.Tags.Contains("vip"));
bool allEur = SampleData.Orders.All(o => o.Currency == "EUR");
```

### 8.2 Element : First/Single/Last et variantes OrDefault

- `First` : au moins un élément.
- `Single` : exactement un élément.
- `Last` : nécessite souvent de parcourir toute la séquence (sauf `IList`).

```csharp
var firstEur = SampleData.Orders.First(o => o.Currency == "EUR");
var maybe = SampleData.Orders.FirstOrDefault(o => o.Currency == "GBP");
```

### 8.3 Chunk, Append, Prepend

```csharp
var extended = SampleData.Customers
    .Append(new Customer(99, "Zed", "FR"))
    .Prepend(new Customer(0, "Root", "FR"));
```

### 8.4 Zip

Combine deux séquences.

```csharp
var a = new[] { 1, 2, 3 };
var b = new[] { 10, 20, 30, 40 };

var zipped = a.Zip(b, (x, y) => x + y).ToList(); // 11,22,33
```

### 8.5 TakeWhile / SkipWhile

```csharp
var untilBig = SampleData.Orders
    .OrderBy(o => o.Date)
    .TakeWhile(o => o.Amount < 200m)
    .ToList();
```

### 8.6 GroupBy vs ToLookup

- `ToLookup` matérialise et offre un accès dictionnaire-like.

```csharp
var lookup = SampleData.Orders.ToLookup(o => o.Currency);
var eurOrders = lookup["EUR"].ToList();
```

### 8.7 DistinctBy, ExceptBy, IntersectBy, UnionBy (NET 6+)

```csharp
var distinctCustomersByCountry = SampleData.Customers
    .DistinctBy(c => c.Country)
    .ToList();
```

### 8.8 OrderBy with custom comparer

```csharp
var byNameLength = SampleData.Customers
    .OrderBy(c => c.Name.Length)
    .ThenBy(c => c.Name)
    .ToList();
```

### 8.9 MinBy/MaxBy

```csharp
var biggest = SampleData.Orders.MaxBy(o => o.Amount);
```

### 8.10 Collections : ToDictionary / ToHashSet

```csharp
var customersById = SampleData.Customers.ToDictionary(c => c.Id);
var tagSet = SampleData.Orders.SelectMany(o => o.Tags).ToHashSet();
```

> Bonus lisibilité : `TryGetValue` plutôt que `First`.

---

## Chapitre 9 — LINQ to XML (XDocument, XElement)

### 9.1 Pourquoi LINQ to XML

- API XML moderne, orientée objets.
- Requêtage via `Descendants`, `Elements`, etc.

### 9.2 Exemple : créer un document XML

```csharp
using System.Xml.Linq;

var doc = new XDocument(
    new XElement("orders",
        SampleData.Orders.Select(o =>
            new XElement("order",
                new XAttribute("id", o.Id),
                new XAttribute("customerId", o.CustomerId),
                new XAttribute("date", o.Date.ToString("yyyy-MM-dd")),
                new XElement("amount",
                    new XAttribute("currency", o.Currency),
                    o.Amount),
                new XElement("tags", o.Tags.Select(t => new XElement("tag", t)))
            ))));

Console.WriteLine(doc);
```

### 9.3 Query XML : extraire et agréger

```csharp
var totalsByCurrency = doc
    .Root!
    .Elements("order")
    .GroupBy(x => (string)x.Element("amount")!.Attribute("currency")!)
    .Select(g => new
    {
        Currency = g.Key,
        Total = g.Sum(o => (decimal)o.Element("amount")!)
    })
    .ToList();
```

### 9.4 Bonnes pratiques XML

- Utiliser des casts LINQ to XML : `(string)`, `(int)`, `(decimal)`.
- Attention aux `null` : `Element(...)` peut être `null`.
- Normaliser namespaces quand nécessaire.

---

## Chapitre 10 — LINQ to JSON (System.Text.Json)

> Il n’existe pas “LINQ to JSON” *natif* dans `System.Text.Json` au sens `JToken` (Newtonsoft). Mais on peut écrire un style LINQ-like avec `JsonDocument`/`JsonNode`.

### 10.1 Parsing JSON avec JsonNode

```csharp
using System.Text.Json;
using System.Text.Json.Nodes;

var json = """
{
  "orders": [
    { "id": 101, "customerId": 1, "amount": 120, "currency": "EUR", "tags": ["hardware","priority"] },
    { "id": 107, "customerId": 5, "amount": 500, "currency": "USD", "tags": ["hardware","vip"] }
  ]
}
""";

var node = JsonNode.Parse(json)!;
var orders = node["orders"]!.AsArray();

var eurIds = orders
    .Where(o => (string)o!["currency"]! == "EUR")
    .Select(o => (int)o!["id"]!)
    .ToList();
```

### 10.2 Agrégation sur JSON

```csharp
var totalByCurrency = orders
    .GroupBy(o => (string)o!["currency"]!)
    .Select(g => new
    {
        Currency = g.Key,
        Total = g.Sum(x => (decimal)x!["amount"]!)
    })
    .ToList();
```

### 10.3 Projection vers modèles typés

Bon pattern : désérialiser vers des records, puis LINQ classique.

```csharp
public sealed record OrderDto(int Id, int CustomerId, decimal Amount, string Currency, string[] Tags);
public sealed record RootDto(OrderDto[] Orders);

var root = JsonSerializer.Deserialize<RootDto>(json,
    new JsonSerializerOptions { PropertyNameCaseInsensitive = true })!;

var vipOrders = root.Orders
    .Where(o => o.Tags.Contains("vip"))
    .ToList();
```

### 10.4 Quand préférer Newtonsoft.Json

- Besoin d’un DOM JSON très LINQ-friendly (`JObject`, `JToken`), d’outils de transformation avancée.
- Écosystème existant.

---

## Chapitre 11 — Performance, allocation, bonnes pratiques

### 11.1 Lisibilité avant tout

- Nommer les variables selon l’intention (`ordersByCurrency`, `highValueCustomers`).
- Éviter les chaines trop longues : extraire des étapes.

```csharp
var eurOrders = SampleData.Orders.Where(o => o.Currency == "EUR");
var highValueEur = eurOrders.Where(o => o.Amount >= 100m);
var result = highValueEur.Select(o => o.Id).ToList();
```

### 11.2 Eviter les re-tris inutiles

- `OrderBy` est coûteux.
- Trier une fois, puis appliquer `Take`, `Skip`.

### 11.3 Matérialisation volontaire

- Matérialiser quand :
  - multi-énumération,
  - source externe instable,
  - besoin de snapshot.

### 11.4 Choisir les bonnes structures

- Pour lookup fréquents : `ToDictionary`, `ToLookup`, `HashSet`.
- Pour membership : `HashSet.Contains` au lieu de `Any`.

```csharp
var vipCustomerIds = new HashSet<int> { 5 };
var vipOrders = SampleData.Orders.Where(o => vipCustomerIds.Contains(o.CustomerId));
```

### 11.5 Complexity hints

- `Where/Select` : O(n)
- `OrderBy` : O(n log n)
- `GroupBy` : O(n) average (hash)
- `Join` : construit un lookup, puis itère.

### 11.6 Allocation & itérateurs

- Chaque opérateur LINQ crée des itérateurs.
- Grande perf : parfois une boucle `foreach` est plus rapide et plus lisible.
- Bon compromis : LINQ pour l’expressivité, boucle pour hot paths.

---

## Chapitre 12 — Atelier final “LINQ Unleashed”

### Objectif

À partir du dataset (customers/orders), produire un tableau de bord.

### Exigences

1. Liste des clients triée par **CA total** décroissant.
2. Pour chaque client :
   - `TotalAmount`, `OrderCount`, `AvgBasket`, `LastOrderDate`.
3. Filtrer sur un pays (ex: `FR`) et un minimum de CA.
4. Extraire le top 3 des tags les plus utilisés.
5. Exporter le résultat en XML (LINQ to XML).
6. Bonus : produire le même export en JSON.

### Correction (une solution possible)

```csharp
var customers = SampleData.Customers;
var orders = SampleData.Orders;

string countryFilter = "FR";
decimal minRevenue = 50m;

var customerKpis = orders
    .Join(customers,
        o => o.CustomerId,
        c => c.Id,
        (o, c) => new { o, c })
    .Where(x => x.c.Country == countryFilter)
    .GroupBy(x => x.c)
    .Select(g => new
    {
        CustomerId = g.Key.Id,
        CustomerName = g.Key.Name,
        Country = g.Key.Country,
        OrderCount = g.Count(),
        TotalAmount = g.Sum(x => x.o.Amount),
        AvgBasket = g.Average(x => x.o.Amount),
        LastOrderDate = g.Max(x => x.o.Date)
    })
    .Where(x => x.TotalAmount >= minRevenue)
    .OrderByDescending(x => x.TotalAmount)
    .ToList();

var topTags = orders
    .SelectMany(o => o.Tags)
    .GroupBy(t => t)
    .Select(g => new { Tag = g.Key, Count = g.Count() })
    .OrderByDescending(x => x.Count)
    .ThenBy(x => x.Tag)
    .Take(3)
    .ToList();

// XML export
using System.Xml.Linq;

var xml = new XDocument(
    new XElement("dashboard",
        new XElement("customers",
            customerKpis.Select(c =>
                new XElement("customer",
                    new XAttribute("id", c.CustomerId),
                    new XAttribute("name", c.CustomerName),
                    new XAttribute("country", c.Country),
                    new XElement("orderCount", c.OrderCount),
                    new XElement("totalAmount", c.TotalAmount),
                    new XElement("avgBasket", c.AvgBasket),
                    new XElement("lastOrderDate", c.LastOrderDate.ToString("yyyy-MM-dd"))
                ))),
        new XElement("topTags",
            topTags.Select(t =>
                new XElement("tag",
                    new XAttribute("name", t.Tag),
                    new XAttribute("count", t.Count))))));

Console.WriteLine(xml);
```

---

## Katas & exercices guidés (corrigés)

### Kata 1 — Détecter les clients “VIP”

**Énoncé** : un client est VIP s’il a au moins une commande avec le tag `vip` OU si son total en USD dépasse 300.

**Solution** :

```csharp
var vipCustomerIds = SampleData.Orders
    .GroupBy(o => o.CustomerId)
    .Where(g =>
        g.Any(o => o.Tags.Contains("vip")) ||
        g.Where(o => o.Currency == "USD").Sum(o => o.Amount) > 300m)
    .Select(g => g.Key)
    .ToHashSet();

var vipCustomers = SampleData.Customers
    .Where(c => vipCustomerIds.Contains(c.Id))
    .ToList();
```

### Kata 2 — Normaliser les devises

**Énoncé** : convertir toutes les commandes en EUR via un taux fictif.

```csharp
decimal ConvertToEur(decimal amount, string currency) => currency switch
{
    "EUR" => amount,
    "USD" => amount * 0.92m,
    "INR" => amount * 0.011m,
    _ => throw new NotSupportedException(currency)
};

var eurAmounts = SampleData.Orders
    .Select(o => new { o.Id, AmountEur = ConvertToEur(o.Amount, o.Currency) })
    .ToList();
```

### Kata 3 — Anti-pattern : Where + First

**Énoncé** : repérer pourquoi c’est mauvais et corriger.

```csharp
// Mauvais : double parcours et exception si vide.
var c = SampleData.Customers.Where(c => c.Id == 3).First();

// Bien :
var c2 = SampleData.Customers.First(c => c.Id == 3);
```

---

## Références & mémo opérateurs

### Opérateurs incontournables

- Projection/filtre : `Select`, `Where`, `SelectMany`
- Tri : `OrderBy`, `ThenBy`
- Grouping : `GroupBy`, `ToLookup`
- Agrégation : `Count`, `Sum`, `Average`, `Min/Max`, `Aggregate`
- Sets : `Distinct`, `Union`, `Intersect`, `Except`, versions `*By`
- Quantifiers : `Any`, `All`
- Elements : `First/Single/Last` + `OrDefault`
- Conversion : `ToList`, `ToArray`, `ToDictionary`, `ToHashSet`

### Ressources

- Docs Microsoft : https://learn.microsoft.com/dotnet/csharp/programming-guide/concepts/linq/
- `System.Linq` reference : https://learn.microsoft.com/dotnet/api/system.linq.enumerable
- LINQ to XML : https://learn.microsoft.com/dotnet/standard/linq/linq-xml-overview
- System.Text.Json : https://learn.microsoft.com/dotnet/standard/serialization/system-text-json/overview

---

## Annexes

### A. Conseils de formateur

- Commencer par la **lecture** de requêtes LINQ existantes.
- Faire verbaliser *“source → transformation → terminal”*.
- Insister sur :
  - exécution différée,
  - `SelectMany`,
  - `GroupBy` + projection,
  - `Join` et `GroupJoin`.

### B. Checklist code review LINQ

- [ ] La requête est-elle énumérée plusieurs fois ?
- [ ] Des appels `Count()/Any()` sont-ils répétés ?
- [ ] Y a-t-il un `OrderBy` inutile ?
- [ ] Le `GroupBy` est-il suivi d’une projection adaptée ?
- [ ] Les opérateurs `First/Single` sont-ils justifiés ?
- [ ] Les structures (`HashSet`, `Dictionary`) sont-elles appropriées ?

