# Masterclass C# 13 & .NET 9 — 002. C# Fundamentals

> Public cible : développeurs débutants à intermédiaires en C#/.NET, ou développeurs d’autres langages souhaitant maîtriser rapidement les bases.
>
> Durée indicative : 1 journée (6–7h) ou 2 demi-journées.
>
> Pré-requis : bases de programmation (variables, conditions, boucles). Outils : .NET 9 SDK, un IDE (Visual Studio / Rider / VS Code), Git optionnel.

---

## Objectifs pédagogiques

À l’issue de cette formation, vous serez capable de :

- Écrire du code C# idiomatique en comprenant la syntaxe essentielle.
- Déclarer et utiliser les principaux **types de données**.
- Manipuler les **opérateurs** (arithmétiques, logiques, comparaison, affectation, nullability, etc.).
- Comprendre et organiser le code avec **namespaces**.
- Distinguer clairement **value types** et **reference types** et leurs implications mémoire/comportement.
- Appliquer les **conversions de types** (implicites, explicites, `Parse/TryParse`, `Convert`).
- Utiliser correctement **identifiers** et connaître les **keywords** et mots réservés.

---

## Plan de la formation

1. Introduction & conventions
2. **C# syntax basics**
   - Structure d’un programme
   - Instructions, blocs, commentaires
   - Variables, `var`, `const`
   - Contrôle de flux : `if`, `switch`, boucles
3. **Data types**
   - Types numériques, `bool`, `char`, `string`
   - `decimal` vs `double`
   - `DateTime`, `TimeSpan`
   - `enum`
   - Types nullables
4. **Operators**
   - Arithmétiques, comparaison, logiques
   - Affectation, incrémentation
   - Opérateurs de null (`??`, `?.`, `??=`)
   - Opérateurs de type (`is`, `as`)
   - Opérateurs bitwise (bases)
5. **Namespaces**
   - Déclaration, `using`, alias
   - Global usings (vue d’ensemble)
   - Bonnes pratiques d’organisation
6. **Value and reference types**
   - Concepts, copies, assignations
   - `struct` vs `class`
   - `string` (référence mais immuable)
   - `record` (aperçu) et égalité
   - Boxing/unboxing
7. **Type conversion**
   - Conversions implicites/explicites
   - `checked`/`unchecked`
   - `Parse`/`TryParse`
   - `Convert` et culture
8. **Identifiers and keywords**
   - Règles de nommage
   - Mots-clés, mots contextuels
   - `@` (verbatim identifier)
9. Exercices & mini-lab (fil rouge)
10. Quiz de validation & synthèse

---

# 1) Introduction & conventions

## 1.1 Créer un projet de travail

En ligne de commande :

```bash
dotnet new console -n CsharpFundamentals
cd CsharpFundamentals
dotnet run
```

Structure typique :

- `Program.cs` : point d’entrée
- `*.csproj` : configuration du projet

## 1.2 Conventions de style (recommandations)

- **PascalCase** : types, méthodes, propriétés (`CustomerService`, `CalculateTotal()`)
- **camelCase** : variables locales, paramètres (`totalPrice`, `customerId`)
- **_camelCase** : champs privés (`_logger`)
- Fichiers : 1 type principal par fichier (souvent)

---

# 2) C# syntax basics

## 2.1 Structure d’un programme

Depuis C# 9, on peut écrire un « top-level program » (sans la classe `Program` explicite) :

```csharp
using System;

Console.WriteLine("Hello C#");
```

Equivalent « explicite » :

```csharp
using System;

namespace CsharpFundamentals;

public class Program
{
    public static void Main(string[] args)
    {
        Console.WriteLine("Hello C#");
    }
}
```

### Points importants

- Une **instruction** se termine par `;`
- Les **blocs** utilisent `{ ... }`
- Le code est **sensible à la casse** (`Console` ≠ `console`)

## 2.2 Commentaires

```csharp
// Commentaire sur une ligne

/*
 Commentaire
 multi-lignes
*/

/// <summary>
/// Commentaire XML (pour documentation)
/// </summary>
```

## 2.3 Déclaration de variables

### Typage explicite

```csharp
int count = 10;
string name = "Ada";
bool isActive = true;
```

### Inférence de type avec `var`

`var` demande au compilateur d’inférer le type à la compilation.

```csharp
var age = 42;          // int
var price = 9.99m;     // decimal (suffixe m)
var title = "C#";      // string
```

Bonnes pratiques :

- Utiliser `var` quand le type est **évident** ou améliore la lisibilité.
- Éviter `var` si cela rend le code ambigu.

### Constantes : `const`

```csharp
const int MaxItems = 100;
```

`const` : valeur connue **à la compilation**.
Pour des constantes calculées à l’exécution : utiliser `static readonly` (hors scope ici, mais à connaître).

## 2.4 Blocs et portée (scope)

```csharp
int x = 1;

if (true)
{
    int y = 2;
    Console.WriteLine(x + y);
}

// y n'existe plus ici
```

## 2.5 Contrôle de flux

### `if` / `else`

```csharp
int score = 75;

if (score >= 90)
    Console.WriteLine("A");
else if (score >= 80)
    Console.WriteLine("B");
else
    Console.WriteLine("C");
```

### `switch` (expression)

```csharp
int day = 3;

string label = day switch
{
    1 => "Mon",
    2 => "Tue",
    3 => "Wed",
    _ => "Unknown"
};

Console.WriteLine(label);
```

### Boucles

#### `for`

```csharp
for (int i = 0; i < 3; i++)
{
    Console.WriteLine(i);
}
```

#### `while`

```csharp
int n = 3;
while (n > 0)
{
    Console.WriteLine(n);
    n--;
}
```

#### `do ... while`

```csharp
int value;
do
{
    value = 1; // ici vous liriez une entrée utilisateur
}
while (value <= 0);
```

#### `foreach`

```csharp
string[] names = ["Ada", "Grace", "Linus"]; // syntaxe tableau moderne

foreach (var n2 in names)
{
    Console.WriteLine(n2);
}
```

---

# 3) Data types

## 3.1 Vue d’ensemble

C# est un langage **fortement typé**.

- **Value types** : stockent directement la valeur (ex: `int`, `bool`, `struct`)
- **Reference types** : stockent une référence vers un objet (ex: `string`, `class`, `array`)

## 3.2 Types numériques

### Entiers signés

- `sbyte` (8-bit), `short` (16-bit), `int` (32-bit), `long` (64-bit)

```csharp
int a = 2_000_000_000;
long b = 9_000_000_000L; // suffixe L
```

### Entiers non signés

- `byte`, `ushort`, `uint`, `ulong`

Utile pour représentation binaire, interop, protocoles.

### Flottants

- `float` (32-bit), `double` (64-bit)

```csharp
float f = 1.23f;  // suffixe f
double d = 1.23;  // double par défaut
```

### Décimal (`decimal`)

Préférer `decimal` pour les montants financiers (moins d’erreurs d’arrondi à base binaire).

```csharp
decimal price = 19.99m;
```

### Séparateurs et lisibilité

```csharp
int million = 1_000_000;
```

## 3.3 `bool`

```csharp
bool ok = true;
bool isAdult = age >= 18;
```

## 3.4 `char` et `string`

- `char` : un caractère Unicode
- `string` : texte, **immutable**

```csharp
char c = 'A';
string s = "Hello";
```

### Interpolation de chaînes

```csharp
string firstName = "Ada";
int years = 5;

string msg = $"{firstName} a {years} ans d'expérience";
```

### Verbatim strings

```csharp
string path = @"C:\Temp\file.txt";
```

## 3.5 `DateTime` et `TimeSpan` (bases)

```csharp
DateTime now = DateTime.UtcNow;
TimeSpan duration = TimeSpan.FromMinutes(90);
```

Bonnes pratiques rapides :

- Favoriser `UtcNow` côté serveur.
- Bien réfléchir aux conversions de fuseaux (hors scope, mais à anticiper).

## 3.6 `enum`

Un `enum` représente un ensemble de constantes nommées.

```csharp
enum OrderStatus
{
    Draft = 0,
    Paid = 1,
    Shipped = 2,
    Cancelled = 3
}

var status = OrderStatus.Paid;
```

## 3.7 Types nullables

### Références nullables (concept)

`string?` signifie : “peut être null”.

```csharp
string? middleName = null;
```

### Types valeur nullables : `T?`

```csharp
int? maybeCount = null;
maybeCount = 10;
```

Pourquoi ? Représenter l’absence de valeur (ex: champ optionnel).

---

# 4) Operators

## 4.1 Arithmétiques

- `+`, `-`, `*`, `/`, `%`

```csharp
int x = 10;
int y = 3;

Console.WriteLine(x / y); // 3 (division entière)
Console.WriteLine(x % y); // 1
```

Avec des flottants :

```csharp
double dx = 10;
double dy = 3;
Console.WriteLine(dx / dy); // 3.333...
```

## 4.2 Incrémentation / décrémentation

```csharp
int i = 0;

i++;   // post-incrément
++i;   // pré-incrément

i--;   // post-décrément
--i;   // pré-décrément
```

Différence :

```csharp
int a = 1;
int b = a++; // b=1, a=2

int c = 1;
int d = ++c; // d=2, c=2
```

## 4.3 Comparaison

- `==`, `!=`, `<`, `<=`, `>`, `>=`

```csharp
bool isEqual = (5 == 5);
```

## 4.4 Logiques (booléens)

- `&&` (ET court-circuit)
- `||` (OU court-circuit)
- `!` (NON)

```csharp
bool canAccess = isActive && isAdult;
```

Court-circuit : si la première condition suffit, la seconde n’est pas évaluée.

## 4.5 Affectation

- `=`, `+=`, `-=`, `*=`, `/=`, `%=`

```csharp
int total = 10;
total += 5; // 15
```

## 4.6 Opérateurs de null

### Coalescence : `??`

```csharp
string? input = null;
string value = input ?? "default";
```

### Affectation coalescente : `??=`

```csharp
string? name = null;
name ??= "Anonymous"; // si null, assigner
```

### Navigation conditionnelle : `?.`

```csharp
string? maybeText = null;
int? len = maybeText?.Length; // null si maybeText est null
```

## 4.7 Opérateurs de type : `is` et `as`

### `is`

```csharp
object obj = "hello";

if (obj is string text)
{
    Console.WriteLine(text.ToUpper());
}
```

### `as` (cast qui retourne null si incompatible)

```csharp
object obj2 = "hello";
string? text2 = obj2 as string;
```

## 4.8 Bitwise (bases)

- `&`, `|`, `^`, `~`, `<<`, `>>`

Exemple :

```csharp
int mask = 0b_0001;
int value2 = 0b_0101;

bool hasFlag = (value2 & mask) != 0;
```

---

# 5) Namespaces

## 5.1 Pourquoi des namespaces ?

- Éviter les collisions de noms (`Logger`, `File`, `Task`, etc.).
- Organiser le code par domaine fonctionnel.

## 5.2 Déclarer un namespace

Syntaxe “file-scoped” (recommandée si un seul namespace par fichier) :

```csharp
namespace MyCompany.Trainings;

public class Calculator
{
    public int Add(int a, int b) => a + b;
}
```

Syntaxe “block-scoped” :

```csharp
namespace MyCompany.Trainings
{
    public class Calculator
    {
        public int Add(int a, int b) => a + b;
    }
}
```

## 5.3 Importer avec `using`

```csharp
using System;
using MyCompany.Trainings;

var calc = new Calculator();
Console.WriteLine(calc.Add(1, 2));
```

## 5.4 Alias avec `using`

Utile en cas de conflit :

```csharp
using Text = System.String;

Text title = "Hello";
```

## 5.5 `global using` (aperçu)

Vous pouvez déclarer des `global using` (souvent dans `GlobalUsings.cs`) pour éviter de répéter.

```csharp
global using System;
```

---

# 6) Value and reference types

## 6.1 Rappel : value vs reference

- **Value type** : l’assignation **copie la valeur**.
- **Reference type** : l’assignation copie la **référence** (les deux variables pointent vers le même objet).

## 6.2 Exemple avec type valeur

```csharp
int a = 10;
int b = a;

b++;

Console.WriteLine(a); // 10
Console.WriteLine(b); // 11
```

## 6.3 Exemple avec type référence

```csharp
var list1 = new List<int> { 1, 2, 3 };
var list2 = list1;

list2.Add(4);

Console.WriteLine(list1.Count); // 4 (même instance)
```

## 6.4 `struct` vs `class`

- `struct` : value type (souvent petit, immuable, “value-like”)
- `class` : reference type (souvent plus riche, identité, cycle de vie)

Exemple minimal :

```csharp
public struct Point
{
    public int X { get; init; }
    public int Y { get; init; }
}

public class Person
{
    public string Name { get; set; } = string.Empty;
}
```

## 6.5 Cas spécial : `string`

- `string` est un **type référence**.
- Mais il est **immutable** : toute “modification” crée une nouvelle chaîne.

```csharp
string s1 = "Hi";
string s2 = s1;

s2 += "!"; // s2 devient une nouvelle instance

Console.WriteLine(s1); // "Hi"
Console.WriteLine(s2); // "Hi!"
```

## 6.6 Égalité : `==` et référence

Sur de nombreux types référence, `==` peut être surchargé.

- Pour `string`, `==` compare le contenu.
- Pour la plupart des classes, `==` compare la référence (sauf surcharge).

## 6.7 Boxing / Unboxing

**Boxing** : conversion d’un value type vers `object` (allocation et encapsulation).
**Unboxing** : récupérer le value type depuis `object`.

```csharp
int n = 42;
object boxed = n;      // boxing
int unboxed = (int)boxed; // unboxing
```

Impacts : allocations, performances. À limiter dans les boucles critiques.

---

# 7) Type conversion

## 7.1 Conversions implicites

Quand il n’y a pas de perte potentielle :

```csharp
int i = 10;
long l = i; // OK
```

## 7.2 Conversions explicites (cast)

Quand il peut y avoir perte :

```csharp
long big = 10_000_000_000;
int small = (int)big; // attention overflow possible
```

## 7.3 `checked` et `unchecked`

`checked` déclenche une exception en cas d’overflow (contexte vérifié).

```csharp
checked
{
    int max = int.MaxValue;
    // int overflow = max + 1; // OverflowException
}
```

`unchecked` ignore l’overflow (wrap-around) :

```csharp
unchecked
{
    int max = int.MaxValue;
    int overflow = max + 1;
    Console.WriteLine(overflow); // valeur négative
}
```

## 7.4 Convertir depuis une chaîne

### `int.Parse`

```csharp
int n1 = int.Parse("123");
```

Risques : exception si format invalide.

### `int.TryParse` (recommandé)

```csharp
if (int.TryParse("123", out int n2))
{
    Console.WriteLine(n2);
}
else
{
    Console.WriteLine("Entrée invalide");
}
```

## 7.5 `Convert`

`Convert` gère certains cas `null` et conversions plus générales.

```csharp
object? value = "42";
int n3 = Convert.ToInt32(value);
```

## 7.6 Culture et parsing

Le parsing de nombres/dates dépend de la culture si on ne précise rien.

Exemple à connaître (sans approfondir) : `CultureInfo.InvariantCulture`.

---

# 8) Identifiers and keywords

## 8.1 Identifiers (noms)

Un identifiant : nom de variable, méthode, classe, namespace…

Règles principales :

- Lettres Unicode, chiffres (pas en premier), underscores.
- Pas d’espaces.
- **Case-sensitive**.
- Éviter les mots réservés.

Exemples valides :

```csharp
int customerId = 1;
string _internalName = "x";
```

Exemples invalides :

- `int 1st = 0;` (commence par chiffre)
- `string first name = "";` (espace)

## 8.2 Keywords (mots-clés)

Exemples de mots-clés courants :

- Types : `int`, `string`, `bool`, `object`
- Contrôle : `if`, `else`, `switch`, `for`, `foreach`, `while`, `do`, `break`, `continue`, `return`
- Déclaration : `class`, `struct`, `enum`, `namespace`, `using`
- Modificateurs : `public`, `private`, `protected`, `internal`, `static`, `const`, `readonly`
- Nullabilité : `null`

## 8.3 Mots-clés contextuels

Certains mots ont un sens de mot-clé **selon le contexte** (ex: `var`, `record`, `async`, `await`, `init`).

## 8.4 Utiliser un mot réservé comme identifiant : `@`

```csharp
int @class = 10; // légal mais déconseillé
Console.WriteLine(@class);
```

Cas d’usage : interop, données externes, SQL-like, etc.

---

# 9) Exercices & mini-lab (fil rouge)

## 9.1 Exercice 1 — Variables et opérateurs

1. Déclarer deux entiers `a` et `b`.
2. Calculer sum, diff, product, quotient (double), modulo.
3. Afficher les résultats via interpolation.

Solution (exemple) :

```csharp
int a = 10;
int b = 3;

int sum = a + b;
int diff = a - b;
int prod = a * b;
double quot = (double)a / b;
int mod = a % b;

Console.WriteLine($"sum={sum}, diff={diff}, prod={prod}, quot={quot:F2}, mod={mod}");
```

## 9.2 Exercice 2 — `switch` + `enum`

Créer un `enum` `UserRole` (`Guest`, `Member`, `Admin`) puis écrire une fonction qui retourne une phrase d’accueil.

Solution (exemple) :

```csharp
enum UserRole { Guest, Member, Admin }

static string Welcome(UserRole role) => role switch
{
    UserRole.Guest => "Bienvenue, invité.",
    UserRole.Member => "Ravi de vous revoir.",
    UserRole.Admin => "Accès administrateur.",
    _ => "Rôle inconnu"
};
```

## 9.3 Exercice 3 — Nullability

- Déclarer un `string?` pouvant être nul.
- Utiliser `??` pour fournir une valeur par défaut.
- Utiliser `?.` pour accéder à une propriété sans exception.

Solution (exemple) :

```csharp
string? input = null;
string normalized = input ?? "(empty)";
int? length = input?.Length;

Console.WriteLine(normalized);
Console.WriteLine(length?.ToString() ?? "no length");
```

## 9.4 Mini-lab — Analyse simple de commandes

Objectif : lire une commande (simulée) et appliquer une action.

- Entrée : `"ADD 10 20"`, `"MUL 3 4"`, `"DIV 10 0"`
- Sortie : résultat ou message d’erreur

Contraintes :

- Utiliser `TryParse`
- Utiliser `switch`
- Gérer division par zéro

Solution (exemple) :

```csharp
static string Execute(string command)
{
    // Exemple : "ADD 10 20"
    var parts = command.Split(' ', StringSplitOptions.RemoveEmptyEntries);
    if (parts.Length != 3) return "Invalid format";

    string op = parts[0].ToUpperInvariant();

    if (!int.TryParse(parts[1], out int left)) return "Invalid left operand";
    if (!int.TryParse(parts[2], out int right)) return "Invalid right operand";

    return op switch
    {
        "ADD" => (left + right).ToString(),
        "SUB" => (left - right).ToString(),
        "MUL" => (left * right).ToString(),
        "DIV" => right == 0 ? "Division by zero" : ((double)left / right).ToString("F2"),
        _ => "Unknown operation"
    };
}

Console.WriteLine(Execute("ADD 10 20"));
Console.WriteLine(Execute("DIV 10 0"));
```

---

# 10) Quiz de validation (rapide)

1. Quelle est la différence entre `var` et un type explicite ?
2. Que fait `??` ? Et `?.` ?
3. `int / int` produit quel type de résultat ?
4. Quelle différence entre value type et reference type lors d’une assignation ?
5. Pourquoi préférer `TryParse` à `Parse` ?

---

# Synthèse

- La syntaxe C# repose sur des blocs `{}` et des instructions `;`.
- Les types numériques ont des comportements différents (division entière, précision, overflow).
- Les opérateurs de null et de type aident à écrire du code robuste.
- Les namespaces structurent le code et évitent les collisions.
- Value types vs reference types expliquent la copie et les effets de bord.
- Les conversions doivent être maîtrisées pour éviter exceptions et bugs subtils.

---

## Annexes — Cheatsheet (récap)

### Types courants

- Entiers : `int`, `long`
- Flottants : `double`
- Financier : `decimal`
- Texte : `string`, `char`
- Booléen : `bool`
- Temps : `DateTime`, `TimeSpan`

### Opérateurs utiles

- Null : `??`, `??=`, `?.`
- Type : `is`, `as`
- Logiques : `&&`, `||`, `!`

### Conversion

- `(T)value` cast
- `T.Parse(string)`
- `T.TryParse(string, out var t)`
- `Convert.ToXxx(value)`
