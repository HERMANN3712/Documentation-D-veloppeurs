# Masterclass C# 13 & .NET 9 — 003 · Harnessing the Code

> **Objectif** : maîtriser le *contrôle d’exécution* (control flow), les boucles, les types nullable, les opérateurs `checked/unchecked`, la gestion des exceptions et les bonnes pratiques .NET modernes.

---

## Public visé
- Développeurs .NET / C# ayant déjà les bases du langage (types, classes, méthodes).
- Formateurs et développeurs souhaitant clarifier les pièges courants et les patterns recommandés.

## Pré-requis
- C# (niveau débutant+ à intermédiaire)
- Visual Studio / Rider + .NET SDK 9
- Notions de base sur les collections et LINQ (utile mais non indispensable)

## Durée suggérée
- 1 journée (6–7h) **ou** 2 demi-journées.

## Résultats attendus
À l’issue, vous saurez :
- Choisir le bon mécanisme de contrôle de flux (`if`, `switch`, expressions, stratégies)
- Écrire des boucles sûres, lisibles et performantes
- Utiliser correctement les *nullable reference types* et éviter les `NullReferenceException`
- Comprendre `checked/unchecked` et gérer les débordements numériques
- Concevoir une stratégie d’exceptions robuste (throw, catch, finally, filtres)
- Appliquer les bonnes pratiques modernes (guard clauses, early return, validation, logs)

---

# Plan de la formation

1. **Control flow (contrôle d’exécution)**
   - `if/else`, opérateur ternaire
   - `switch` statement vs `switch` expression
   - Pattern matching (type, property, relational, list patterns)
   - Guard clauses, early returns
   - Choix de design : table de dispatch, stratégie

2. **Boucles et itération**
   - `for`, `foreach`, `while`, `do/while`
   - `break`, `continue`, `return`
   - `yield return` et itérateurs
   - Pièges : modification de collection, closures, performances
   - Alternatives : LINQ vs boucles

3. **Nullable types**
   - Nullable value types (`int?`)
   - Nullable reference types (`string?`) et annotations
   - Opérateurs : `?.`, `??`, `??=`, `!`
   - Attributs et analyse statique (NRT)
   - Best practices publiques (API, DTO, domain)

4. **`checked` / `unchecked`**
   - Débordement (overflow) en C#
   - Contexte `checked` / `unchecked`
   - `checked` expression vs statement
   - Recommandations pour finance, compteur, sécurité

5. **Exception handling**
   - Hiérarchie d’exceptions .NET
   - `try/catch/finally` + filtres `when`
   - `using` / `await using` (disposal fiable)
   - Bonnes pratiques : quand *throw*, quand *return Result*
   - Logging, wrapping, `InnerException`

6. **Best practices transverses**
   - Lisibilité, invariants, validation
   - Robustesse : timeouts, cancellation, input validation
   - Erreurs : exceptions vs codes de retour vs résultats
   - Conventions (naming, messages d’erreur)
   - Checklist de revue de code

---

# 1) Control flow — contrôler le chemin d’exécution

## 1.1 `if/else` — la base, mais avec discipline

```csharp
if (customer is null)
    throw new ArgumentNullException(nameof(customer));

if (customer.IsVip)
{
    ApplyVipDiscount(order);
}
else
{
    ApplyStandardPricing(order);
}
```

### Bonnes pratiques
- **Préférer les *guard clauses*** pour éviter l’imbrication :

```csharp
public decimal ComputeTotal(Order order)
{
    if (order is null) throw new ArgumentNullException(nameof(order));
    if (order.Lines.Count == 0) return 0m;

    // flux principal “happy path”
    return order.Lines.Sum(l => l.Quantity * l.UnitPrice);
}
```

- **Éviter les conditions complexes** à rallonge : extraire en fonctions nommées.

```csharp
if (ShouldApplyPromo(customer, order, now))
    ApplyPromo(order);

static bool ShouldApplyPromo(Customer c, Order o, DateTimeOffset now)
    => c.IsVip && o.Total >= 100 && now.DayOfWeek is DayOfWeek.Friday;
```

## 1.2 Opérateur ternaire `?:`

Utile pour des expressions courtes et lisibles.

```csharp
var label = isEnabled ? "Enabled" : "Disabled";
```

À éviter si la logique devient opaque :

```csharp
// Trop complexe => préférer if/else
var price = isVip
    ? (isHoliday ? vipHolidayPrice : vipPrice)
    : (isHoliday ? holidayPrice : basePrice);
```

## 1.3 `switch` statement vs `switch` expression

### `switch` statement

```csharp
switch (status)
{
    case OrderStatus.Draft:
        HandleDraft(order);
        break;

    case OrderStatus.Paid:
        HandlePaid(order);
        break;

    default:
        throw new NotSupportedException($"Unknown status: {status}");
}
```

### `switch` expression — plus concise, plus “expression-oriented”

```csharp
var next = status switch
{
    OrderStatus.Draft => OrderStatus.Submitted,
    OrderStatus.Submitted => OrderStatus.Paid,
    _ => throw new NotSupportedException($"Unknown status: {status}")
};
```

## 1.4 Pattern matching — écrire du code plus sûr

### a) Type patterns

```csharp
string Describe(object value) => value switch
{
    null => "<null>",
    int i => $"int: {i}",
    decimal d => $"decimal: {d}",
    string s => $"string: '{s}'",
    _ => value.ToString() ?? "<no text>"
};
```

### b) Property patterns

```csharp
bool CanShip(Order order) => order is
{
    Status: OrderStatus.Paid,
    ShippingAddress: not null
};
```

### c) Relational patterns

```csharp
static string Bucket(int age) => age switch
{
    < 0 => throw new ArgumentOutOfRangeException(nameof(age)),
    < 18 => "minor",
    < 65 => "adult",
    _ => "senior"
};
```

### d) List patterns (C# 11+)

```csharp
static bool LooksLikeSemVer(int[] parts) => parts is [> 0, >= 0, >= 0];
```

## 1.5 Design : éviter les `switch` gigantesques

Quand les cas deviennent nombreux et évolutifs :
- **Table de dispatch** (dictionnaire de handlers)
- **Strategy pattern** (par type / interface)

### Exemple — table de dispatch

```csharp
var handlers = new Dictionary<OrderStatus, Action<Order>>
{
    [OrderStatus.Draft] = HandleDraft,
    [OrderStatus.Paid] = HandlePaid,
    [OrderStatus.Cancelled] = HandleCancelled
};

if (!handlers.TryGetValue(order.Status, out var handler))
    throw new NotSupportedException($"Unknown status: {order.Status}");

handler(order);
```

---

# 2) Boucles et itération

## 2.1 `for` — index et performance

```csharp
for (int i = 0; i < list.Count; i++)
{
    var item = list[i];
    // ...
}
```

### Quand utiliser `for`
- Accès indexé nécessaire
- Optimisation micro (rare mais utile sur hot paths)
- Besoin de parcourir par pas (`i += 2`)

## 2.2 `foreach` — lisibilité et sécurité

```csharp
foreach (var line in order.Lines)
{
    total += line.Quantity * line.UnitPrice;
}
```

### Piège : modification de collection

```csharp
// 🚫 Peut lever InvalidOperationException
foreach (var x in list)
    if (x.IsInvalid) list.Remove(x);

// ✅ Alternatives
list.RemoveAll(x => x.IsInvalid);
// ou
for (int i = list.Count - 1; i >= 0; i--)
    if (list[i].IsInvalid) list.RemoveAt(i);
```

## 2.3 `while` / `do-while`

- `while` : 0..n itérations
- `do/while` : 1..n itérations

```csharp
while (!token.IsCancellationRequested)
{
    await PollAsync(token);
    await Task.Delay(500, token);
}
```

## 2.4 `break`, `continue`, `return`

- `break` : sortir de la boucle
- `continue` : passer à l’itération suivante
- `return` : sortir de la méthode

```csharp
foreach (var item in items)
{
    if (!item.IsEnabled) continue;
    if (item.Id == targetId) return item;
}

return null;
```

### Bonnes pratiques
- `continue` peut améliorer la lisibilité (réduit l’imbrication)
- Trop de `break/continue` peut rendre le flux difficile à suivre

## 2.5 Itérateurs : `yield return`

Permet de produire une séquence **lazy**.

```csharp
public static IEnumerable<int> RangeWithStep(int start, int count, int step)
{
    for (int i = 0, v = start; i < count; i++, v += step)
        yield return v;
}
```

### Pièges `yield`
- La logique s’exécute **au moment de l’énumération**, pas à l’appel.
- Les exceptions peuvent être levées plus tard.

## 2.6 LINQ vs boucles

- **LINQ** : expressif, composable
- **Boucles** : parfois plus simples / plus performantes / plus contrôlables

Exemple : validation avec early exit (souvent mieux en boucle)

```csharp
bool HasInvalidLine(List<OrderLine> lines)
{
    foreach (var l in lines)
        if (l.Quantity <= 0 || l.UnitPrice < 0) return true;
    return false;
}
```

---

# 3) Nullable types — éviter le `null` accidentel

## 3.1 Nullable value types (`int?`, `DateTime?`)

```csharp
int? pageSize = TryGetPageSize();
int effective = pageSize ?? 50;
```

### Accès à la valeur

```csharp
if (pageSize.HasValue)
{
    int size = pageSize.Value;
}
```

Préférer le pattern matching :

```csharp
if (pageSize is int size)
{
    // size est non-null ici
}
```

## 3.2 Nullable reference types (NRT)

Activer (recommandé) :

```xml
<PropertyGroup>
  <Nullable>enable</Nullable>
</PropertyGroup>
```

Avec NRT :
- `string` signifie **non-null** (contrat)
- `string?` signifie **peut être null**

```csharp
string GetName(User user) => user.Name;      // Name devrait être non-null
string? TryGetNickname(User user) => user.NickName; // ok si optionnel
```

## 3.3 Opérateurs clés

### a) Null-conditional `?.`

```csharp
int? len = user.NickName?.Length;
```

### b) Null-coalescing `??`

```csharp
var name = user.NickName ?? user.Name;
```

### c) Null-coalescing assignment `??=`

```csharp
cache[key] ??= LoadFromDb(key);
```

### d) Null-forgiving `!`

```csharp
// J’affirme au compilateur : “ce n’est pas null”
var name = user.Name!;
```

⚠️ À utiliser avec parcimonie : cela masque un problème potentiel.

## 3.4 Paramètres, retours et invariants

### Guard clauses

```csharp
public User(string name)
{
    Name = name ?? throw new ArgumentNullException(nameof(name));
}

public string Name { get; }
```

### API publiques : être explicite
- Si une valeur peut manquer : `T?` + doc claire
- Sinon : refuser `null` à l’entrée et garantir non-null en sortie

## 3.5 Interop legacy / JSON / EF

- Les frameworks peuvent produire des `null` (désérialisation, DB)
- Techniques :
  - validation à l’entrée (DTO → domain)
  - `required` (C# 11) pour objets initialisés
  - `ArgumentNullException.ThrowIfNull()`

```csharp
public sealed class CreateUserRequest
{
    public required string Name { get; init; }
    public string? NickName { get; init; }
}

public User ToDomain(CreateUserRequest dto)
{
    ArgumentNullException.ThrowIfNull(dto);
    // dto.Name est required => non-nullable contractuel
    return new User(dto.Name) { NickName = dto.NickName };
}
```

---

# 4) `checked` / `unchecked` — maîtriser l’overflow

## 4.1 Overflow : que fait C# ?

Sur les entiers (`int`, `long`, etc.), un dépassement peut :
- **wrap-around** (comportement par défaut fréquent en release selon le contexte)
- **lever** `OverflowException` si le contexte est `checked`

Exemple :

```csharp
int x = int.MaxValue;
int y = x + 1; // peut déborder
```

## 4.2 Contexte `checked`

### `checked` statement

```csharp
checked
{
    int x = int.MaxValue;
    int y = x + 1; // OverflowException
}
```

### `checked` expression

```csharp
int y = checked(int.MaxValue + 1); // OverflowException
```

## 4.3 Contexte `unchecked`

```csharp
unchecked
{
    int y = int.MaxValue + 1; // wrap-around
}
```

## 4.4 Quand l’utiliser ?

- **Finance / billing / comptage** : privilégier `checked`.
- **Hashing / cryptographie / bit-twiddling** : `unchecked` peut être intentionnel.
- **Interop** : clarifier le choix (et le documenter).

### Recommandation
- Appliquer `checked` sur les opérations critiques plutôt que globalement.

```csharp
public int ComputeInvoiceId(int prefix, int number)
{
    // si overflow => bug / collision => on veut échouer
    return checked(prefix * 1_000_000 + number);
}
```

---

# 5) Exception handling — robuste, lisible, utile

## 5.1 Philosophie

Une exception en .NET :
- signale une **condition exceptionnelle** (erreur, état inattendu)
- porte une **stack trace** précieuse
- doit être **gérée** là où on peut agir, pas partout

## 5.2 `try/catch/finally`

```csharp
try
{
    await service.ProcessAsync(request, ct);
    return Results.Ok();
}
catch (ValidationException ex)
{
    return Results.BadRequest(new { ex.Message });
}
catch (Exception ex)
{
    logger.LogError(ex, "Unhandled error");
    return Results.Problem("An unexpected error occurred");
}
finally
{
    metrics.Increment("process_attempt");
}
```

### `finally`
- s’exécute même en cas d’exception
- idéal pour libérer des ressources *non gérées* si nécessaire

## 5.3 Filtres d’exception `when`

Permet de **filtrer** sans avaler des exceptions non désirées.

```csharp
try
{
    await client.CallAsync(ct);
}
catch (HttpRequestException ex) when (ex.StatusCode == System.Net.HttpStatusCode.NotFound)
{
    // cas “attendu”
    return null;
}
```

## 5.4 `throw` vs `throw ex`

- `throw;` **préserve** la stack trace.
- `throw ex;` **réécrit** la stack trace (à éviter).

```csharp
catch (Exception)
{
    // ✅
    throw;
}
```

## 5.5 Wrapping (exception de plus haut niveau)

Wrappez pour ajouter du contexte, en conservant l’exception originale dans `InnerException`.

```csharp
try
{
    return await repository.LoadAsync(id, ct);
}
catch (SqlException ex)
{
    throw new DataAccessException($"Failed to load entity {id}", ex);
}
```

## 5.6 Gestion des ressources : `using` et `await using`

### `using`

```csharp
using var stream = File.OpenRead(path);
// stream.Dispose() garanti
```

### `await using` (IAsyncDisposable)

```csharp
await using var conn = new NpgsqlConnection(cs);
await conn.OpenAsync(ct);
```

## 5.7 Exceptions vs Results (approche “fonctionnelle”)

Quand l’échec est **attendu** et fréquent (ex: validation, recherche optionnelle), retourner un résultat plutôt qu’une exception.

- `TryXxx(out ...)`
- `bool` + valeur
- `Result<T>` (type maison)

Exemple “Try” :

```csharp
public bool TryParseCustomerId(string input, out Guid id)
    => Guid.TryParse(input, out id);
```

Exemple `Result<T>` simple :

```csharp
public readonly record struct Result<T>(T? Value, string? Error)
{
    public bool IsSuccess => Error is null;
    public static Result<T> Ok(T value) => new(value, null);
    public static Result<T> Fail(string error) => new(default, error);
}
```

---

# 6) Best practices transverses — écrire du code “solide”

## 6.1 Guard clauses et validation

- À l’entrée des méthodes publiques
- Sur les invariants métier

```csharp
public void AddLine(OrderLine line)
{
    ArgumentNullException.ThrowIfNull(line);

    if (line.Quantity <= 0)
        throw new ArgumentOutOfRangeException(nameof(line.Quantity));

    _lines.Add(line);
}
```

## 6.2 Éviter les effets de bord dans les conditions

```csharp
// 🚫 Éviter : conditions avec assignations/effets
if (TryUpdate(out var x) && x > 0) { }

// ✅ Préférer la lisibilité
var updated = TryUpdate(out var x);
if (updated && x > 0) { }
```

## 6.3 Messages d’erreur utiles

- Inclure le *quoi* et le *pourquoi*
- Éviter d’exposer des secrets (connection strings, PII)

```csharp
throw new InvalidOperationException(
    $"Cannot ship order '{order.Id}' because status is '{order.Status}'.");
```

## 6.4 Logging : loguer au bon niveau

- Ne pas logger **et** rethrow partout (double logs)
- Logger à la frontière (API, worker, UI) ou au point de décision

## 6.5 Checklist de revue de code

### Control flow
- [ ] Pas de `if`/`else` profondément imbriqués
- [ ] `switch` exhaustif + `_ => throw` si nécessaire
- [ ] Conditions extraites en fonctions nommées si complexes

### Boucles
- [ ] Pas de modification de collection pendant un `foreach`
- [ ] Early exit si possible
- [ ] LINQ utilisé pour l’expression, pas pour masquer la logique

### Nullables
- [ ] NRT activé et warnings traités
- [ ] `!` justifié (rare)
- [ ] API publiques : contrats non-null explicites

### Overflow
- [ ] Opérations critiques en `checked`
- [ ] `unchecked` seulement intentionnel et commenté

### Exceptions
- [ ] `throw;` utilisé au lieu de `throw ex;`
- [ ] Exceptions wrapées avec `InnerException` quand besoin de contexte
- [ ] Pas d’exceptions pour le contrôle de flux “normal”

---

# Exercices (atelier)

## Exercice 1 — Refactor guard clauses
**But** : réduire l’imbrication.

1) On vous donne une méthode avec plusieurs `if` imbriqués.
2) Refactorez avec des guard clauses.
3) Ajoutez des exceptions précises (`ArgumentNullException`, `ArgumentOutOfRangeException`).

## Exercice 2 — Switch expression + pattern matching
- Implémenter un tarif en fonction d’un type de client (Standard/Vip/Student)
- Utiliser `switch` expression
- Ajouter un `_ => throw` pour les cas non supportés

## Exercice 3 — Nullable NRT
- Activer `<Nullable>enable</Nullable>`
- Corriger les warnings sans utiliser `!` sauf si strictement nécessaire

## Exercice 4 — Overflow sécurisé
- Écrire une fonction qui calcule un *score* sur `int`
- Appliquer `checked` et couvrir par tests (cas overflow attendu)

## Exercice 5 — Exceptions & filtres
- Appeler une API HTTP
- Traiter `404` via `catch ... when`
- Logger les erreurs inattendues seulement

---

# Annexes

## A) Snippets utiles

### `ArgumentNullException.ThrowIfNull`

```csharp
public void Save(Customer customer)
{
    ArgumentNullException.ThrowIfNull(customer);
    // ...
}
```

### `TryGetValue` et pattern

```csharp
if (dict.TryGetValue(key, out var value))
{
    // value disponible
}
```

## B) Pistes de discussion avancées (optionnel)
- Exceptions en async/await et `AggregateException`
- `ValueTask` et patterns de performance
- `Span<T>` et boucles “hot path”

---

## Fin
Ce module “Harnessing the Code” sert de socle : en maîtrisant le flux d’exécution, la null-safety, la gestion des overflows et les exceptions, vous écrivez un C# plus **prévisible**, **maintenable** et **résilient**.
