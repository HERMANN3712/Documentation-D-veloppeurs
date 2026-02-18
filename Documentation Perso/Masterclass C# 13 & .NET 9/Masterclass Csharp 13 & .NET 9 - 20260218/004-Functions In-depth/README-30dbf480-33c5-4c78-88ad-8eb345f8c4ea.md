# Masterclass C# 13 & .NET 9 — 004. Functions In-depth

> Public visé : développeurs .NET/C# (débutant avancé → intermédiaire)
>
> Prérequis : bases du langage C# (types, variables, conditions, boucles), Visual Studio / Rider, .NET SDK

---

## Objectifs pédagogiques

À la fin de cette formation, vous saurez :

- Déclarer et concevoir des méthodes idiomatiques en C# (niveaux d’accès, `static`, surcharge, `ref`/`in`/`out`, membres d’instance).
- Maîtriser les paramètres (optionnels, nommés, `params`, tuples, `ref readonly`, etc.).
- Utiliser des **fonctions lambda** (closures, inférence, `Func`/`Action`, delegates, expressions).
- Écrire du code asynchrone robuste avec `async`/`await` (Task, exceptions, annulation, concurrence).
- Déboguer efficacement (breakpoints conditionnels, watch, call stack, exceptions, async debugging).
- Mettre en place des **tests unitaires** avec **NUnit** et **xUnit** (arrange/act/assert, data-driven tests, assertions).
- Comprendre la **récursion** (cas d’arrêt, stack, optimisation, alternatives itératives).
- Choisir les **types de retour** (void, value types, refs, Task/ValueTask, `IEnumerable`, tuples, résultats structurés).

---

## Format suggéré

- **Durée** : 1 jour (7h) ou 2 demi-journées
- **Pédagogie** : alternance théorie + démos + exercices guidés
- **Livrables** : supports + code d’exemple + exercices (solutions)

---

# Plan de formation

1. [Modèle mental : qu’est-ce qu’une fonction en C# ?](#1-modèle-mental--quest-ce-quune-fonction-en-c)
2. [Déclaration de méthodes (Method declaration)](#2-déclaration-de-méthodes-method-declaration)
3. [Paramètres : design, contraintes et cas avancés](#3-paramètres--design-contraintes-et-cas-avancés)
4. [Lambdas, delegates et closures](#4-lambdas-delegates-et-closures)
5. [Async/Await en profondeur](#5-asyncawait-en-profondeur)
6. [Débogage des fonctions (debugging)](#6-débogage-des-fonctions-debugging)
7. [Tests unitaires avec NUnit et xUnit](#7-tests-unitaires-avec-nunit-et-xunit)
8. [Récursion](#8-récursion)
9. [Types de retour (Return types) : choix et patterns](#9-types-de-retour-return-types--choix-et-patterns)
10. [Atelier final : refactoring & robustes APIs](#10-atelier-final--refactoring--robustes-apis)

---

# 1. Modèle mental : qu’est-ce qu’une fonction en C# ?

En C#, on parle le plus souvent de :

- **Méthodes** : fonctions déclarées dans un type (classe/struct/record/interface). Exemple : `string.ToUpper()`.
- **Fonctions locales** : fonctions déclarées à l’intérieur d’une méthode.
- **Lambdas** : fonctions anonymes (souvent passées en paramètre).
- **Delegates** : types représentant une signature de fonction.

Une fonction = **entrée (paramètres)** + **sortie (type de retour)** + **contrat (exceptions/side effects)**.

### Qualités d’une bonne fonction

- **Nom clair** (intention) : `CalculateNetAmount`, `TryParse`, `IsValid`.
- **Faible complexité** : lisible, un seul niveau d’abstraction.
- **Testable** : dépendances injectées (ou paramétrées), effets de bord maîtrisés.
- **Déterministe si possible** : même entrée → même sortie (pur).

---

# 2. Déclaration de méthodes (Method declaration)

## 2.1 Syntaxe de base

```csharp
public int Add(int a, int b)
{
    return a + b;
}
```

Éléments :

- **Modificateurs** : `public`, `private`, `protected`, `internal`, `static`, `virtual`, `override`, `abstract`, `sealed`, `extern`, `unsafe`…
- **Type de retour** : `int` (ou `void`).
- **Nom** : `Add`.
- **Paramètres** : `int a, int b`.
- **Corps** : bloc `{ ... }` ou expression-bodied `=>`.

## 2.2 Expression-bodied members

```csharp
public int Add(int a, int b) => a + b;
```

À privilégier pour les fonctions très courtes et évidentes.

## 2.3 Méthodes d’instance vs statiques

```csharp
public sealed class Counter
{
    private int _value;

    public void Increment() => _value++;

    public int Value => _value;

    public static Counter Create() => new();
}
```

- Instance : opère sur l’état (`_value`).
- Static : utilitaire, factory, pure function.

## 2.4 Surcharge (overload)

```csharp
public int Sum(int a, int b) => a + b;
public int Sum(int a, int b, int c) => a + b + c;
```

Bonnes pratiques :

- Les overloads doivent **converger** vers une implémentation centrale.
- Attention aux ambiguïtés avec paramètres optionnels.

## 2.5 Méthodes génériques

```csharp
public static T Max<T>(T a, T b) where T : IComparable<T>
    => a.CompareTo(b) >= 0 ? a : b;
```

- Génériques : réutilisation + sécurité de type.
- Contraintes (`where`) : API plus claire.

## 2.6 Extensions methods

```csharp
public static class StringExtensions
{
    public static bool IsNullOrBlank(this string? s) => string.IsNullOrWhiteSpace(s);
}

// usage
if (name.IsNullOrBlank()) { ... }
```

- Domaine : utilitaires “naturels” sur un type.
- Attention : éviter les extensions qui masquent une mauvaise conception.

## 2.7 Fonctions locales

```csharp
public static int CountEven(IEnumerable<int> values)
{
    bool IsEven(int x) => x % 2 == 0;
    return values.Count(IsEven);
}
```

Intérêts :

- Encapsulation d’un détail d’implémentation.
- Capturer un contexte (comme une lambda) mais debuggable.

---

# 3. Paramètres : design, contraintes et cas avancés

## 3.1 Paramètres nommés

```csharp
public static string FormatMoney(decimal amount, string currency, bool useSymbol)
    => useSymbol ? $"{amount:0.00} {currency}" : $"{currency} {amount:0.00}";

var s = FormatMoney(amount: 10m, currency: "EUR", useSymbol: true);
```

- Améliore la lisibilité quand il y a plusieurs bools/valeurs semblables.

## 3.2 Paramètres optionnels

```csharp
public static string Greet(string name, string prefix = "Hello")
    => $"{prefix}, {name}!";
```

⚠️ Les paramètres optionnels sont **résolus à la compilation côté appelant**. Si vous changez la valeur par défaut dans une librairie, un appelant déjà compilé peut garder l’ancienne valeur.

## 3.3 `params`

```csharp
public static int Sum(params int[] values) => values.Sum();

var total = Sum(1, 2, 3, 4);
```

- Pratique pour API “variadique”.
- Attention : allocation du tableau (souvent acceptable, parfois non).

## 3.4 `ref`, `out`, `in`

### `out` (retour multiple / Try-pattern)

```csharp
public static bool TryParsePort(string s, out int port)
{
    if (int.TryParse(s, out var p) && p is >= 1 and <= 65535)
    {
        port = p;
        return true;
    }

    port = default;
    return false;
}
```

### `ref` (passage par référence)

```csharp
public static void Swap(ref int a, ref int b)
{
    (a, b) = (b, a);
}
```

- Puissant, mais complexifie l’API.
- À réserver aux scénarios perf/interop/algorithmique.

### `in` (référence en lecture seule)

```csharp
public readonly struct BigStruct
{
    public readonly long A, B, C, D;
    public BigStruct(long a, long b, long c, long d) => (A, B, C, D) = (a, b, c, d);
}

public static long Sum(in BigStruct s) => s.A + s.B + s.C + s.D;
```

- Évite la copie sur de gros structs.

## 3.5 Nullabilité et contrats

```csharp
public static string NormalizeName(string? name)
{
    if (string.IsNullOrWhiteSpace(name))
        throw new ArgumentException("Name is required", nameof(name));

    return name.Trim();
}
```

- Activez `#nullable enable` au niveau projet.
- Définissez clairement : accepte-t-on `null` ? quelle exception ?

## 3.6 Paramètres de type “résultat” vs “option”

Plutôt que :

```csharp
public void DoWork(bool dryRun, bool verbose, bool safeMode)
```

Préférez :

- Un **objet options** :

```csharp
public sealed record WorkOptions(bool DryRun = false, bool Verbose = false, bool SafeMode = true);
public void DoWork(WorkOptions options) { ... }
```

- Ou plusieurs méthodes explicites : `DoWork()` / `PreviewWork()`.

---

# 4. Lambdas, delegates et closures

## 4.1 Delegates : `Action` / `Func`

```csharp
Func<int, int, int> add = (a, b) => a + b;
Action<string> log = message => Console.WriteLine(message);
```

- `Func<...>` : dernier paramètre = type de retour.
- `Action<...>` : pas de retour.

## 4.2 Lambda : syntaxes

```csharp
x => x * 2
(x, y) => x + y
() => DateTimeOffset.UtcNow
```

Avec bloc :

```csharp
(int x) =>
{
    var y = x * 2;
    return y + 1;
}
```

## 4.3 Inférence de type et annotations

```csharp
var numbers = new[] { 1, 2, 3 };
var doubled = numbers.Select(x => x * 2);

// si ambigu, on peut typer :
var doubled2 = numbers.Select((int x) => x * 2);
```

## 4.4 Closures (captures)

```csharp
var factor = 10;
Func<int, int> multiply = x => x * factor;

factor = 20;
var result = multiply(2); // 40
```

- La lambda capture la **variable**, pas juste sa valeur.
- Attention aux pièges dans les boucles.

Piège classique :

```csharp
var actions = new List<Action>();
for (int i = 0; i < 3; i++)
{
    actions.Add(() => Console.WriteLine(i));
}
// imprime 3,3,3
```

Fix :

```csharp
for (int i = 0; i < 3; i++)
{
    int copy = i;
    actions.Add(() => Console.WriteLine(copy));
}
```

## 4.5 Méthode groupe (method group)

```csharp
static bool IsEven(int x) => x % 2 == 0;

var evens = numbers.Where(IsEven);
```

- Souvent plus lisible qu’une lambda.

## 4.6 `delegate` anonyme vs lambda

```csharp
Func<int, int> f1 = x => x + 1;
Func<int, int> f2 = delegate (int x) { return x + 1; };
```

Aujourd’hui : utilisez presque toujours les lambdas.

## 4.7 Expression trees (aperçu)

```csharp
using System.Linq.Expressions;

Expression<Func<int, bool>> expr = x => x > 10;
```

- Représentation d’un code sous forme d’arbre.
- Utilisé par LINQ providers (Entity Framework, etc.).

---

# 5. Async/Await en profondeur

## 5.1 Rappel : `Task` et `async`/`await`

```csharp
public static async Task<string> DownloadAsync(HttpClient http, string url)
{
    using var resp = await http.GetAsync(url);
    resp.EnsureSuccessStatusCode();
    return await resp.Content.ReadAsStringAsync();
}
```

- `await` libère le thread pendant l’attente I/O.
- Le code reste “linéaire”.

## 5.2 Signatures asynchrones

- `Task` : pas de valeur.
- `Task<T>` : valeur.
- `ValueTask` / `ValueTask<T>` : optimisation (scénarios avancés).

Évitez `async void` (sauf event handlers).

## 5.3 Gestion des exceptions

```csharp
try
{
    var content = await DownloadAsync(http, url);
}
catch (HttpRequestException ex)
{
    // log / retry / fallback
}
```

- Une exception dans une méthode async est stockée dans la `Task`.
- Elle est re-levée au moment du `await`.

## 5.4 Annulation avec `CancellationToken`

```csharp
public static async Task<string> DownloadAsync(HttpClient http, string url, CancellationToken ct)
{
    using var resp = await http.GetAsync(url, ct);
    resp.EnsureSuccessStatusCode();
    return await resp.Content.ReadAsStringAsync(ct);
}
```

Bonnes pratiques :

- Propager le `CancellationToken`.
- Ne pas “avaler” `OperationCanceledException` sans raison.

## 5.5 Concurrence : `Task.WhenAll` / `WhenAny`

```csharp
var tasks = urls.Select(u => DownloadAsync(http, u, ct));
var pages = await Task.WhenAll(tasks);
```

- `WhenAll` propage les exceptions (potentiellement agrégées).
- Contrôler le degré de parallélisme si nécessaire.

## 5.6 Deadlocks et contextes

- En applications UI/ASP.NET, le contexte de synchronisation peut jouer.
- Dans les libs, on voit souvent `ConfigureAwait(false)`.
- Sur ASP.NET Core moderne, le risque de deadlock est réduit, mais comprendre le sujet reste utile.

## 5.7 Async streams (aperçu)

```csharp
public static async IAsyncEnumerable<int> GenerateAsync([System.Runtime.CompilerServices.EnumeratorCancellation] CancellationToken ct = default)
{
    for (int i = 0; i < 5; i++)
    {
        ct.ThrowIfCancellationRequested();
        await Task.Delay(200, ct);
        yield return i;
    }
}
```

---

# 6. Débogage des fonctions (debugging)

## 6.1 Breakpoints utiles

- Breakpoint simple
- Breakpoint conditionnel (ex : `i == 42`)
- Hit count (déclencher au N-ième passage)
- Tracepoint (log sans stopper)

## 6.2 Inspection

- Variables locales
- **Watch** (expressions surveillées)
- **Autos** / **Locals**
- **Immediate Window** (évaluer du code)

## 6.3 Suivi du flux

- Step into / over / out
- **Call Stack** : qui a appelé qui ?
- Navigation vers le code appelant

## 6.4 Exceptions

- Configurer les “break on thrown” pour certaines exceptions.
- Lire la stack trace : repérer le premier frame métier.

## 6.5 Debug de l’async

- Task window / Parallel stacks (selon IDE)
- Comprendre les “async state machines” : stepping peut paraître étrange.

## 6.6 Debugging des lambdas et LINQ

- Convertir temporairement une query LINQ en boucle `foreach`.
- Stocker les étapes intermédiaires dans des variables.

Exemple :

```csharp
var q = customers
    .Where(c => c.IsActive)
    .Select(c => new { c.Id, Name = c.Name.Trim() })
    .ToList();
```

Astuce :

```csharp
var active = customers.Where(c => c.IsActive).ToList();
var shaped = active.Select(c => new { c.Id, Name = c.Name.Trim() }).ToList();
```

---

# 7. Tests unitaires avec NUnit et xUnit

Objectif : tester les fonctions en isolation, avec des spécifications claires.

## 7.1 Arrange / Act / Assert (AAA)

- Arrange : préparer les données.
- Act : exécuter.
- Assert : vérifier.

## 7.2 Exemple de code à tester

```csharp
public static class PriceCalculator
{
    public static decimal ComputeNet(decimal gross, decimal vatRate)
    {
        if (gross < 0) throw new ArgumentOutOfRangeException(nameof(gross));
        if (vatRate < 0) throw new ArgumentOutOfRangeException(nameof(vatRate));

        return gross / (1m + vatRate);
    }
}
```

## 7.3 NUnit

### Setup

- Package : `NUnit` + `NUnit3TestAdapter` + `Microsoft.NET.Test.Sdk`

### Test simple

```csharp
using NUnit.Framework;

[TestFixture]
public sealed class PriceCalculatorTests
{
    [Test]
    public void ComputeNet_WhenGrossIs120AndVatIs20Percent_Returns100()
    {
        // Arrange
        var gross = 120m;
        var vat = 0.20m;

        // Act
        var net = PriceCalculator.ComputeNet(gross, vat);

        // Assert
        Assert.That(net, Is.EqualTo(100m));
    }
}
```

### Data-driven tests

```csharp
[TestCase(120, 0.20, 100)]
[TestCase(110, 0.10, 100)]
public void ComputeNet_WorksForSeveralValues(decimal gross, decimal vat, decimal expected)
{
    var net = PriceCalculator.ComputeNet(gross, vat);
    Assert.That(net, Is.EqualTo(expected));
}
```

### Exceptions

```csharp
[Test]
public void ComputeNet_WhenGrossIsNegative_Throws()
{
    Assert.That(() => PriceCalculator.ComputeNet(-1m, 0.2m),
        Throws.TypeOf<ArgumentOutOfRangeException>());
}
```

## 7.4 xUnit

### Setup

- Package : `xunit` + `xunit.runner.visualstudio` + `Microsoft.NET.Test.Sdk`

### Test simple

```csharp
using Xunit;

public sealed class PriceCalculatorTests
{
    [Fact]
    public void ComputeNet_WhenGrossIs120AndVatIs20Percent_Returns100()
    {
        var net = PriceCalculator.ComputeNet(120m, 0.20m);
        Assert.Equal(100m, net);
    }
}
```

### Data-driven tests : `[Theory]` + `[InlineData]`

```csharp
[Theory]
[InlineData(120, 0.20, 100)]
[InlineData(110, 0.10, 100)]
public void ComputeNet_WorksForSeveralValues(decimal gross, decimal vat, decimal expected)
{
    var net = PriceCalculator.ComputeNet(gross, vat);
    Assert.Equal(expected, net);
}
```

### Exceptions

```csharp
[Fact]
public void ComputeNet_WhenGrossIsNegative_Throws()
{
    Assert.Throws<ArgumentOutOfRangeException>(() => PriceCalculator.ComputeNet(-1m, 0.2m));
}
```

## 7.5 Tester l’asynchrone

```csharp
[Fact]
public async Task DownloadAsync_ReturnsContent()
{
    // Ici, utiliser un HttpMessageHandler mocké est recommandé.
    // Le point clé : le test est async et attend la Task.

    var http = new HttpClient(new FakeHandler("hello"));

    var content = await Downloader.DownloadAsync(http, "https://example.test", CancellationToken.None);

    Assert.Equal("hello", content);
}
```

Bonnes pratiques :

- Éviter les `Thread.Sleep` dans les tests.
- Préférer des fakes/mocks/handlers contrôlés.

---

# 8. Récursion

La récursion appelle une fonction depuis elle-même.

## 8.1 Les règles

- **Cas d’arrêt** (base case) obligatoire.
- **Progression** vers ce cas d’arrêt.
- Attention à la **stack** (risque de `StackOverflowException`).

## 8.2 Exemple : factorielle

```csharp
public static long Factorial(int n)
{
    if (n < 0) throw new ArgumentOutOfRangeException(nameof(n));
    if (n is 0 or 1) return 1;

    return n * Factorial(n - 1);
}
```

## 8.3 Exemple : recherche dans un arbre

```csharp
public sealed class Node
{
    public required string Name { get; init; }
    public List<Node> Children { get; } = new();
}

public static Node? FindByName(Node root, string name)
{
    if (root.Name == name) return root;

    foreach (var child in root.Children)
    {
        var found = FindByName(child, name);
        if (found is not null) return found;
    }

    return null;
}
```

## 8.4 Alternatives itératives

- Utiliser une pile explicite (`Stack<T>`) pour éviter le risque stack overflow.
- Souvent plus verbeux mais plus robuste.

## 8.5 Tests unitaires de la récursion

Testez :

- Cas d’arrêt (ex : `n=0`)
- Valeurs typiques
- Valeurs limites (grandes valeurs, négatif)

---

# 9. Types de retour (Return types) : choix et patterns

## 9.1 `void` vs retour de valeur

- `void` : actions/commandes (effet de bord).
- Retour de valeur : fonctions pures/queries.

Souvent :

- Query → retourner une valeur.
- Command → `void` (ou `Task`).

## 9.2 Retourner `bool` : Try-pattern

```csharp
public static bool TryGetCustomer(int id, out Customer? customer)
{
    if (id <= 0)
    {
        customer = null;
        return false;
    }

    customer = new Customer(id);
    return true;
}
```

- Contract clair : pas d’exception pour contrôle de flux.

## 9.3 Tuples

```csharp
public static (decimal Net, decimal Vat) SplitVat(decimal gross, decimal vatRate)
{
    var net = gross / (1m + vatRate);
    var vat = gross - net;
    return (net, vat);
}

var (net, vat) = SplitVat(120m, 0.2m);
```

- Bien pour des retours “petits” et cohérents.
- Au-delà : préférer un type dédié (record/struct).

## 9.4 Retourner des séquences : `IEnumerable<T>`

```csharp
public static IEnumerable<int> GetEvenNumbers(IEnumerable<int> source)
{
    foreach (var x in source)
        if (x % 2 == 0)
            yield return x;
}
```

- Lazy evaluation.
- Attention : exceptions et exécution différée.

## 9.5 `Task<T>` et `ValueTask<T>`

- `Task<T>` : standard, simple.
- `ValueTask<T>` : micro-optimisation quand parfois synchrone, parfois async (ex : cache).

Exemple (cache) :

```csharp
public sealed class UserService
{
    private readonly Dictionary<int, User> _cache = new();

    public ValueTask<User> GetUserAsync(int id)
    {
        if (_cache.TryGetValue(id, out var user))
            return ValueTask.FromResult(user);

        return new ValueTask<User>(LoadAsync(id));

        async Task<User> LoadAsync(int userId)
        {
            await Task.Delay(50); // I/O
            var u = new User(userId);
            _cache[userId] = u;
            return u;
        }
    }
}
```

## 9.6 Résultats structurés (Result pattern)

Quand il faut renvoyer une valeur **ou** des erreurs métier sans exception.

```csharp
public readonly record struct Result<T>(bool IsSuccess, T? Value, string? Error)
{
    public static Result<T> Success(T value) => new(true, value, null);
    public static Result<T> Failure(string error) => new(false, default, error);
}

public static Result<int> ParsePositiveInt(string s)
{
    if (!int.TryParse(s, out var x)) return Result<int>.Failure("Not an int");
    if (x <= 0) return Result<int>.Failure("Must be positive");
    return Result<int>.Success(x);
}
```

## 9.7 Retour par référence (avancé)

- `ref return` / `ref readonly return` (scénarios perf/mémoire)
- À réserver à des besoins spécifiques.

---

# 10. Atelier final : refactoring & robustes APIs

## 10.1 Énoncé

Vous partez d’une API “utilitaire” mal conçue :

- Méthode longue et peu testable.
- Trop de bools/options.
- Exceptions utilisées comme contrôle de flux.
- Asynchrone mal géré.

Objectifs :

1. Découper en fonctions petites (single responsibility).
2. Concevoir des signatures propres (paramètres, retours).
3. Introduire un Try-pattern / Result pattern.
4. Écrire des tests NUnit ou xUnit.
5. Déboguer une régression (breakpoint conditionnel, watch).

## 10.2 Grille de validation

- Le code compile.
- Les tests couvrent : happy path + erreurs + limites.
- Les méthodes sont nommées clairement.
- Les types de retour sont appropriés.
- L’asynchrone propage correctement l’annulation.

---

# Annexes

## A. Checklist “signature de méthode”

- Nom = verbe + intention claire
- Paramètres : ordre logique (plus stables d’abord)
- Éviter : `bool` multiples (préférer options/type dédié)
- Choisir : exceptions vs Try/Result
- Documenter : nullabilité, range, unités
- Tester : cas d’erreur et edge cases

## B. Exercices (proposés)

1. **Refactor** : une méthode de 40 lignes en 4–6 méthodes cohérentes.
2. **Params** : transformer une signature avec 4 bools en `record Options`.
3. **Lambda** : écrire `Where`/`Select` lisibles (method groups vs lambdas).
4. **Async** : paralléliser des appels I/O via `Task.WhenAll` avec annulation.
5. **Recursion** : parcours d’arbre (récursif puis itératif).
6. **Testing** : écrire la suite de tests (NUnit ou xUnit) + dataset.

---

## Notes formateur

- Alterner démo + exercices courts.
- Faire verbaliser le contrat d’une fonction : entrées/retour/erreurs.
- Insister sur la testabilité : une bonne signature rend le test trivial.

