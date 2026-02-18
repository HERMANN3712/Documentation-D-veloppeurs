# Masterclass C# 13 & .NET 9 — 008. Data in Motion

> **Public** : développeurs .NET/C# (junior → confirmé)
>
> **Objectif** : maîtriser la lecture/écriture de données « en mouvement » (fichiers, flux, sérialisation), avec une attention particulière à la **performance**, la **robustesse** et la **sécurité**.
>
> **Technos** : .NET 9, C# 13, `System.IO`, `System.Text.Json`, XML (`System.Xml`), sérialisation binaire (bonnes pratiques), `async/await`.

---

## Table des matières

1. [Objectifs pédagogiques](#1-objectifs-pédagogiques)
2. [Prérequis & setup](#2-prérequis--setup)
3. [Vue d’ensemble : Data in Motion](#3-vue-densemble--data-in-motion)
4. [Module 1 — Fundamentals File I/O](#4-module-1--fundamentals-file-io)
5. [Module 2 — Streams : modèle unifié & patterns](#5-module-2--streams--modèle-unifié--patterns)
6. [Module 3 — Encodage, texte, buffers & performance](#6-module-3--encodage-texte-buffers--performance)
7. [Module 4 — Sérialisation JSON (System.Text.Json)](#7-module-4--sérialisation-json-systemtextjson)
8. [Module 5 — Sérialisation XML](#8-module-5--sérialisation-xml)
9. [Module 6 — « Binary » : options, risques & alternatives](#9-module-6--binary--options-risques--alternatives)
10. [Module 7 — Secure file handling](#10-module-7--secure-file-handling)
11. [Module 8 — Boîte à outils System.IO (pratique)](#11-module-8--boîte-à-outils-systemio-pratique)
12. [Ateliers & exercices](#12-ateliers--exercices)
13. [Cheat sheet](#13-cheat-sheet)
14. [Annexes : snippets & utilitaires](#14-annexes--snippets--utilitaires)

---

## 1) Objectifs pédagogiques

À la fin de ce module, vous saurez :

- Utiliser correctement `File`, `FileInfo`, `Directory`, `Path` et les flux (`Stream`, `FileStream`, `MemoryStream`, etc.).
- Choisir entre I/O **synchrone** et **asynchrone** selon le contexte.
- Lire/écrire efficacement des fichiers texte et binaires (buffers, encodage, `Span`/`Memory`).
- Sérialiser/désérialiser en **JSON** (options, compat, streaming, polymorphisme, perf).
- Manipuler des données **XML** (DOM, streaming avec `XmlReader/XmlWriter`, sérialisation).
- Éviter les pièges de la sérialisation binaire et adopter des alternatives modernes.
- Implémenter des pratiques de **sécurité** pour la gestion de fichiers (path traversal, ACL, validations, atomique, locking, chiffrement).

---

## 2) Prérequis & setup

### Prérequis techniques

- Bases solides en C# (classes, generics, exceptions, `async/await`).
- Notions de base sur JSON et XML.

### Environnement

- .NET SDK 9
- IDE : Visual Studio / Rider / VS Code

### Structure recommandée (repo d’exercices)

```
/DataInMotion
  /src
    DataInMotion.Demo/
    DataInMotion.Labs/
  /data
    input/
    output/
```

---

## 3) Vue d’ensemble : Data in Motion

### Définition

« Data in Motion » = données qui transitent entre :

- **Mémoire ↔ disque** (fichiers)
- **Mémoire ↔ mémoire** (streams)
- **Objets ↔ représentation** (JSON/XML/binaire)
- **Process ↔ process** (pipes), **app ↔ réseau** (streams applicables) — hors scope détaillé ici, mais mêmes patterns.

### Axe de décision

- **Format** : texte (JSON/XML/CSV) vs binaire
- **Taille** : petit (tout en mémoire) vs gros (streaming)
- **Contrat** : stable (versioning) vs ad hoc
- **Sécurité** : données sensibles, provenance non fiable

---

## 4) Module 1 — Fundamentals File I/O

### 4.1 `File` vs `FileInfo`

- `File` : API statique, simple, directe
- `FileInfo` : orientée objet, peut **réutiliser** des infos (ex. `Length`, `Exists`) et exprimer des opérations liées au fichier

Exemple : écrire/relire un fichier texte

```csharp
using System.Text;

var path = "data/output/hello.txt";
Directory.CreateDirectory(Path.GetDirectoryName(path)!);

await File.WriteAllTextAsync(path, "Hello .NET 9", Encoding.UTF8);
var text = await File.ReadAllTextAsync(path, Encoding.UTF8);
Console.WriteLine(text);
```

### 4.2 Lire/écrire en binaire

```csharp
var path = "data/output/payload.bin";
byte[] payload = [1, 2, 3, 4, 5];
await File.WriteAllBytesAsync(path, payload);

byte[] restored = await File.ReadAllBytesAsync(path);
Console.WriteLine(restored.Length);
```

### 4.3 `Directory` et exploration

Lister des fichiers (attention : peut être coûteux sur de gros répertoires)

```csharp
var root = "data";
foreach (var file in Directory.EnumerateFiles(root, "*.json", SearchOption.AllDirectories))
{
    Console.WriteLine(file);
}
```

> Préférez `EnumerateFiles` à `GetFiles` : **lazy**, évite de charger toute la liste.

### 4.4 `Path` : construire des chemins robustes

```csharp
var outputDir = Path.Combine("data", "output");
var fileName = "report.json";
var fullPath = Path.GetFullPath(Path.Combine(outputDir, fileName));
```

Bonnes pratiques :

- **Ne concaténez pas** des chemins à la main.
- Utilisez `Path.Combine`, `Path.Join`, `Path.GetFullPath`.
- Utilisez `Path.GetInvalidFileNameChars()` pour valider des noms.

### 4.5 Gestion d’erreurs

Erreurs fréquentes :

- `UnauthorizedAccessException`
- `DirectoryNotFoundException`
- `FileNotFoundException`
- `IOException` (fichier verrouillé, disque plein, etc.)

Pattern robuste :

```csharp
try
{
    using var stream = File.Open("data/input/missing.txt", FileMode.Open, FileAccess.Read, FileShare.Read);
}
catch (FileNotFoundException ex)
{
    Console.Error.WriteLine(ex.Message);
}
catch (UnauthorizedAccessException ex)
{
    Console.Error.WriteLine("Access denied: " + ex.Message);
}
```

---

## 5) Module 2 — Streams : modèle unifié & patterns

### 5.1 Concept de `Stream`

Un `Stream` représente une séquence d’octets, avec des capacités :

- `CanRead`, `CanWrite`, `CanSeek`
- `Position`, `Length` (si seekable)

Types courants :

- `FileStream` : fichier
- `MemoryStream` : mémoire
- `BufferedStream` : tampon
- `CryptoStream` : chiffrement/déchiffrement
- `GZipStream` : compression (System.IO.Compression)

### 5.2 `using` / `await using`

- `Stream` implémente `IDisposable`.
- `FileStream` (et certains streams) supportent `IAsyncDisposable`.

```csharp
await using var fs = new FileStream(
    "data/output/large.dat",
    FileMode.Create,
    FileAccess.Write,
    FileShare.None,
    bufferSize: 64 * 1024,
    options: FileOptions.Asynchronous);
```

### 5.3 Copier un stream vers un autre

```csharp
await using var input = File.OpenRead("data/input/big.bin");
await using var output = File.Create("data/output/big-copy.bin");

await input.CopyToAsync(output);
```

### 5.4 Reader/Writer : texte vs binaire

- `StreamReader`/`StreamWriter` : texte + encodage
- `BinaryReader`/`BinaryWriter` : primitives binaires

Exemple (texte) :

```csharp
await using var fs = File.Open("data/output/log.txt", FileMode.Create, FileAccess.Write, FileShare.Read);
await using var writer = new StreamWriter(fs, encoding: Encoding.UTF8);

await writer.WriteLineAsync("Line 1");
await writer.WriteLineAsync("Line 2");
```

Exemple (binaire) :

```csharp
await using var fs = File.Open("data/output/values.bin", FileMode.Create, FileAccess.Write, FileShare.None);
using var bw = new BinaryWriter(fs, Encoding.UTF8, leaveOpen: true);

bw.Write(42);
// Attention: Write(string) écrit une longueur encodée + bytes => format .NET spécifique.
bw.Write("hello");
```

### 5.5 Seek, Position, et pièges

- Tous les streams ne sont pas seekables (ex. réseau).
- `Position` peut être coûteux sur certains flux.

```csharp
using var ms = new MemoryStream();
ms.WriteByte(0x2A);
ms.Position = 0;
int b = ms.ReadByte();
```

---

## 6) Module 3 — Encodage, texte, buffers & performance

### 6.1 Encodage : UTF-8 par défaut (mais explicitez)

- UTF-8 est le standard moderne.
- Ne pas dépendre de l’encodage par défaut du système.

```csharp
var s = "éàç";
await File.WriteAllTextAsync("data/output/utf8.txt", s, Encoding.UTF8);
```

### 6.2 Streaming sur gros fichiers (éviter `ReadAllText`)

```csharp
var path = "data/input/huge.log";

using var fs = new FileStream(path, FileMode.Open, FileAccess.Read, FileShare.ReadWrite);
using var reader = new StreamReader(fs, Encoding.UTF8);

string? line;
while ((line = reader.ReadLine()) is not null)
{
    // traiter ligne par ligne
}
```

### 6.3 Buffers & `ArrayPool<T>`

Pour des opérations intensives, louez un buffer :

```csharp
using System.Buffers;

byte[] rented = ArrayPool<byte>.Shared.Rent(128 * 1024);
try
{
    // ... utiliser rented
}
finally
{
    ArrayPool<byte>.Shared.Return(rented, clearArray: true);
}
```

### 6.4 `Span<T>` / `Memory<T>` (rappels)

- `Span<T>` : vue sur mémoire **stack-only** (non async)
- `Memory<T>` : vue **heap-friendly** (peut traverser des `await`)

Utilisation fréquente en parsing/IO avancé.

---

## 7) Module 4 — Sérialisation JSON (System.Text.Json)

### 7.1 Modèle de données

```csharp
public sealed record Customer(
    Guid Id,
    string Name,
    int LoyaltyPoints,
    DateTimeOffset CreatedAt);
```

### 7.2 Sérialiser/désérialiser (simple)

```csharp
using System.Text.Json;

var customer = new Customer(Guid.NewGuid(), "Ada", 120, DateTimeOffset.UtcNow);

var json = JsonSerializer.Serialize(customer);
var restored = JsonSerializer.Deserialize<Customer>(json);
```

### 7.3 Options globales : camelCase, indented, enums, dates

```csharp
using System.Text.Json;
using System.Text.Json.Serialization;

var options = new JsonSerializerOptions
{
    PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
    WriteIndented = true,
    DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull,
    Converters =
    {
        new JsonStringEnumConverter(JsonNamingPolicy.CamelCase)
    }
};

var json = JsonSerializer.Serialize(customer, options);
```

### 7.4 Streaming JSON (gros volumes)

#### Écrire dans un flux

```csharp
using System.Text.Json;

var path = "data/output/customer.json";
await using var fs = File.Create(path);
await JsonSerializer.SerializeAsync(fs, customer);
```

#### Lire depuis un flux

```csharp
await using var fs = File.OpenRead("data/output/customer.json");
var c = await JsonSerializer.DeserializeAsync<Customer>(fs);
```

### 7.5 JSON Lines (NDJSON) : logs / pipelines

Format : 1 JSON par ligne.

```csharp
await using var fs = File.Create("data/output/customers.ndjson");
await using var sw = new StreamWriter(fs, Encoding.UTF8);

foreach (var c in customers)
{
    var line = JsonSerializer.Serialize(c);
    await sw.WriteLineAsync(line);
}
```

Lecture streaming :

```csharp
using var fs = File.OpenRead("data/output/customers.ndjson");
using var sr = new StreamReader(fs, Encoding.UTF8);

string? line;
while ((line = await sr.ReadLineAsync()) is not null)
{
    var c = JsonSerializer.Deserialize<Customer>(line);
    // traiter c
}
```

### 7.6 Versioning & compat

Bonnes pratiques :

- Ajouter des propriétés **optionnelles** (nullable / default) plutôt que casser le contrat.
- Utiliser `JsonExtensionData` pour capturer des propriétés inconnues.

```csharp
using System.Text.Json;
using System.Text.Json.Serialization;

public sealed record CustomerV2(
    Guid Id,
    string Name)
{
    [JsonExtensionData]
    public Dictionary<string, JsonElement>? Extra { get; init; }
}
```

### 7.7 Polymorphisme (attention) & sécurité

- Évitez d’accepter des types arbitraires.
- Préférez des discriminants explicites et des converters contrôlés.

---

## 8) Module 5 — Sérialisation XML

XML reste utile pour :

- systèmes legacy
- signatures/standards
- configuration

### 8.1 DOM : `XDocument` (LINQ to XML)

```csharp
using System.Xml.Linq;

var doc = new XDocument(
    new XElement("customer",
        new XAttribute("id", Guid.NewGuid()),
        new XElement("name", "Ada"),
        new XElement("points", 120)));

doc.Save("data/output/customer.xml");
```

Lecture :

```csharp
var loaded = XDocument.Load("data/output/customer.xml");
var name = (string?)loaded.Root?.Element("name");
```

### 8.2 Streaming : `XmlReader` / `XmlWriter`

Pour les gros fichiers XML, préférez le streaming.

Écriture :

```csharp
using System.Xml;

var settings = new XmlWriterSettings { Indent = true, Encoding = Encoding.UTF8 };
await using var fs = File.Create("data/output/customers.xml");
using var xw = XmlWriter.Create(fs, settings);

xw.WriteStartDocument();
xw.WriteStartElement("customers");

foreach (var c in customers)
{
    xw.WriteStartElement("customer");
    xw.WriteElementString("id", c.Id.ToString());
    xw.WriteElementString("name", c.Name);
    xw.WriteElementString("loyaltyPoints", c.LoyaltyPoints.ToString());
    xw.WriteEndElement();
}

xw.WriteEndElement();
xw.WriteEndDocument();
```

Lecture (pattern simplifié) :

```csharp
using System.Xml;

using var xr = XmlReader.Create("data/output/customers.xml");
while (xr.Read())
{
    if (xr.NodeType == XmlNodeType.Element && xr.Name == "customer")
    {
        // Lire sous-arbre ou naviguer selon votre schéma
    }
}
```

### 8.3 `XmlSerializer` : sérialisation objet ↔ XML

```csharp
using System.Xml.Serialization;

[XmlRoot("customer")]
public sealed class CustomerXml
{
    [XmlElement("id")] public string Id { get; set; } = "";
    [XmlElement("name")] public string Name { get; set; } = "";
}

var serializer = new XmlSerializer(typeof(CustomerXml));

var cxml = new CustomerXml { Id = Guid.NewGuid().ToString(), Name = "Ada" };

await using (var fs = File.Create("data/output/customer-serializer.xml"))
{
    serializer.Serialize(fs, cxml);
}
```

> Attention perf : `XmlSerializer` peut générer dynamiquement des assemblies; réutilisez l’instance.

### 8.4 Sécurité XML

- Désactivez DTD/entités externes si vous parsez des XML non fiables.
- Utilisez des settings adaptés (`XmlReaderSettings`).

---

## 9) Module 6 — « Binary » : options, risques & alternatives

### 9.1 Pourquoi la sérialisation binaire est dangereuse

Historique : `BinaryFormatter` est **obsolète** et **dangereux** (risque RCE). Ne pas l’utiliser.

### 9.2 Alternatives recommandées

- JSON : interop, versioning OK
- XML : standards/legacy
- Protobuf / MessagePack / Avro : binaires modernes (interop + perf)

> Dans le cadre standard library, vous pouvez utiliser :
>
>- `System.IO` + votre propre encodage stable
>- `System.Text.Json` en mode `Utf8JsonWriter` (binaire UTF-8)

### 9.3 Écriture binaire « contrôlée » (format maison)

Cas d’usage : format interne, hautes performances, contrat simple.

Exemple (contrat stable) :

- Magic header (4 bytes)
- Version (int)
- Length (int)
- Payload (bytes)

```csharp
public static async Task WritePacketAsync(string path, int version, byte[] payload, CancellationToken ct = default)
{
    await using var fs = new FileStream(path, FileMode.Create, FileAccess.Write, FileShare.None);

    // Écriture manuelle (endianness à définir !)
    // Ici on s'appuie sur BinaryWriter: format .NET, mais on limite aux primitives.
    using var bw = new BinaryWriter(fs, Encoding.UTF8, leaveOpen: true);

    bw.Write(new byte[] { (byte)'P', (byte)'K', (byte)'T', (byte)'1' }); // magic
    bw.Write(version);
    bw.Write(payload.Length);
    bw.Write(payload);

    await fs.FlushAsync(ct);
}

public static async Task<(int Version, byte[] Payload)> ReadPacketAsync(string path, CancellationToken ct = default)
{
    await using var fs = new FileStream(path, FileMode.Open, FileAccess.Read, FileShare.Read);
    using var br = new BinaryReader(fs, Encoding.UTF8, leaveOpen: true);

    var magic = br.ReadBytes(4);
    if (magic is not [ (byte)'P', (byte)'K', (byte)'T', (byte)'1' ])
        throw new InvalidDataException("Bad header");

    int version = br.ReadInt32();
    int length = br.ReadInt32();

    if (length < 0 || length > 100_000_000)
        throw new InvalidDataException("Invalid length");

    var payload = br.ReadBytes(length);
    if (payload.Length != length)
        throw new EndOfStreamException();

    await Task.CompletedTask;
    return (version, payload);
}
```

> Recommandation : si le format doit vivre longtemps / interopérer, utilisez Protobuf/MessagePack.

---

## 10) Module 7 — Secure file handling

### 10.1 Threat model : ce qui peut mal tourner

- **Path traversal** : `../../windows/system32` dans un nom de fichier
- Écrasement de fichier sensible
- Lecture de données non autorisées
- Fichier partiellement écrit (crash, arrêt, concurrence)
- TOCTOU (Time-of-check vs time-of-use)
- Liens symboliques / jonctions (selon OS)

### 10.2 Valider et confiner les chemins (anti-traversal)

Principe :

1. définir un **répertoire racine** autorisé
2. combiner + normaliser (`GetFullPath`)
3. vérifier que le chemin final est bien **dans** la racine

```csharp
public static string SafeCombine(string root, string userFileName)
{
    // 1) root absolu
    var rootFull = Path.GetFullPath(root);

    // 2) nettoyage minimal du nom (optionnel, selon besoin)
    foreach (var c in Path.GetInvalidFileNameChars())
        userFileName = userFileName.Replace(c, '_');

    // 3) chemin candidat
    var candidate = Path.GetFullPath(Path.Combine(rootFull, userFileName));

    // 4) confinement
    if (!candidate.StartsWith(rootFull + Path.DirectorySeparatorChar, StringComparison.OrdinalIgnoreCase)
        && !string.Equals(candidate, rootFull, StringComparison.OrdinalIgnoreCase))
    {
        throw new UnauthorizedAccessException("Path traversal detected");
    }

    return candidate;
}
```

### 10.3 Écriture atomique (anti-fichier partiel)

Pattern : écrire dans un fichier temporaire puis remplacer.

```csharp
public static async Task AtomicWriteAllTextAsync(string path, string content, Encoding encoding)
{
    var dir = Path.GetDirectoryName(path) ?? ".";
    Directory.CreateDirectory(dir);

    var tmp = Path.Combine(dir, $".{Path.GetFileName(path)}.{Guid.NewGuid():N}.tmp");

    await File.WriteAllTextAsync(tmp, content, encoding);

    // Replace est atomique sur Windows; sur Linux/macOS, Move/rename est atomique dans le même FS.
    if (OperatingSystem.IsWindows() && File.Exists(path))
    {
        File.Replace(tmp, path, destinationBackupFileName: null);
    }
    else
    {
        File.Move(tmp, path, overwrite: true);
    }
}
```

### 10.4 File locking & partage (`FileShare`)

- Lire un fichier en cours d’écriture : possible si le writer autorise `FileShare.Read`.
- Écriture exclusive : `FileShare.None`.

```csharp
using var fs = new FileStream(
    "data/output/locked.txt",
    FileMode.Create,
    FileAccess.Write,
    FileShare.None);
```

### 10.5 Droits d’accès (Windows/Unix)

- Sur Windows : ACL NTFS (gérées via `System.Security.AccessControl`).
- Sur Unix : permissions/umask.

> En formation : insistez sur le fait que les permissions sont *système*, et qu’appliquer le principe du moindre privilège (service account) est souvent plus important que « bricoler » des ACLs au runtime.

### 10.6 Ne jamais faire confiance au contenu

- Protéger contre `OutOfMemoryException` indirecte (fichiers trop gros)
- Valider les tailles (voir exemple binaire)
- Éviter les parseurs permissifs sur entrées externes

### 10.7 Données sensibles : chiffrement

Option simple : `Aes` + `CryptoStream` (clé gérée hors code).

```csharp
using System.Security.Cryptography;

public static async Task EncryptToFileAsync(string path, byte[] plaintext, byte[] key, byte[] iv)
{
    using var aes = Aes.Create();
    aes.Key = key;
    aes.IV = iv;

    await using var fs = File.Create(path);
    await using var crypto = new CryptoStream(fs, aes.CreateEncryptor(), CryptoStreamMode.Write);

    await crypto.WriteAsync(plaintext);
    await crypto.FlushAsync();
}
```

---

## 11) Module 8 — Boîte à outils System.IO (pratique)

### 11.1 `FileStream` options importantes

- `FileOptions.Asynchronous` : I/O async efficaces
- `FileOptions.SequentialScan` : hint OS (lecture séquentielle)
- `bufferSize` : 4K → 128K typiquement, à mesurer

```csharp
await using var fs = new FileStream(
    path,
    FileMode.Open,
    FileAccess.Read,
    FileShare.Read,
    bufferSize: 128 * 1024,
    options: FileOptions.Asynchronous | FileOptions.SequentialScan);
```

### 11.2 `MemoryStream` pour tests, pipelines, transformations

```csharp
byte[] data = Encoding.UTF8.GetBytes("hello");
using var ms = new MemoryStream(data);
using var sr = new StreamReader(ms, Encoding.UTF8);
Console.WriteLine(sr.ReadToEnd());
```

### 11.3 `StringReader` / `StringWriter`

Utile pour parser/générer du texte sans I/O disque.

### 11.4 `TemporaryFile` (pattern)

.NET n’a pas un type unique standard « temp file » avec auto-clean, mais vous pouvez faire :

```csharp
var tmp = Path.Combine(Path.GetTempPath(), $"dim-{Guid.NewGuid():N}.tmp");
try
{
    await File.WriteAllTextAsync(tmp, "temp", Encoding.UTF8);
    // ...
}
finally
{
    if (File.Exists(tmp)) File.Delete(tmp);
}
```

---

## 12) Ateliers & exercices

### Atelier 1 — Copier un répertoire en streaming

**Objectif** : copier N fichiers sans charger tout en mémoire.

- Parcourir avec `Directory.EnumerateFiles`
- Pour chaque fichier, `CopyToAsync`
- Créer les répertoires de destination

Critères de réussite :

- gère les erreurs fichier par fichier (rapport)
- utilise `async` correctement

### Atelier 2 — Convertisseur JSON ↔ XML

**Objectif** : lire une liste `customers.json` et produire `customers.xml`.

- Désérialisation JSON avec options `camelCase`
- Écriture XML via `XmlWriter` (streaming)

### Atelier 3 — Écriture atomique d’un fichier de configuration

**Objectif** : implémenter `AtomicWriteAllTextAsync` + tests.

- simuler crash : vérifier que le fichier final est toujours valide

### Atelier 4 — Anti-path traversal

**Objectif** : `SafeCombine(root, userInput)` et tests.

- cas `../` et chemins absolus
- valider confinement

### Atelier 5 — NDJSON pipeline

**Objectif** : écrire 100k lignes NDJSON puis relire en streaming.

- mesurer temps et mémoire (simple `Stopwatch`)
- comparer `ReadAllText` vs line-by-line

---

## 13) Cheat sheet

### API importantes

- `File.ReadAllTextAsync`, `File.WriteAllTextAsync`
- `File.ReadAllBytesAsync`, `File.WriteAllBytesAsync`
- `File.OpenRead`, `File.OpenWrite`, `FileStream`
- `Directory.EnumerateFiles`, `Directory.CreateDirectory`
- `Path.Combine`, `Path.GetFullPath`
- `Stream.CopyToAsync`
- `StreamReader/StreamWriter`
- `JsonSerializer.Serialize/Deserialize` et `SerializeAsync/DeserializeAsync`
- `XDocument`, `XmlReader/XmlWriter`, `XmlSerializer`

### Règles d’or

1. Gros fichiers → **streaming**
2. Toujours préciser l’**encodage**
3. Entrées utilisateur → **anti-traversal** + **validation**
4. Écriture critique → **atomique**
5. Sérialisation binaire généraliste → **à éviter**

---

## 14) Annexes : snippets & utilitaires

### 14.1 Lecture par blocs (buffer manuel)

```csharp
public static async Task<long> CountBytesAsync(string path, CancellationToken ct = default)
{
    const int BufferSize = 128 * 1024;
    byte[] buffer = new byte[BufferSize];

    await using var fs = new FileStream(path, FileMode.Open, FileAccess.Read, FileShare.Read,
        bufferSize: BufferSize, options: FileOptions.Asynchronous | FileOptions.SequentialScan);

    long total = 0;
    int read;
    while ((read = await fs.ReadAsync(buffer, ct)) > 0)
        total += read;

    return total;
}
```

### 14.2 Détection simple d’encodage (à manier avec prudence)

Sans lib externe, vous pouvez détecter un BOM pour UTF-8/UTF-16, mais souvent il faut imposer un contrat.

### 14.3 Exemple `JsonSerializerContext` (source generation) — performance

Pour des performances et AOT friendliness, utilisez la source generation.

```csharp
using System.Text.Json.Serialization;

[JsonSerializable(typeof(Customer))]
[JsonSerializable(typeof(List<Customer>))]
public partial class AppJsonContext : JsonSerializerContext;

// Utilisation :
// var json = JsonSerializer.Serialize(customer, AppJsonContext.Default.Customer);
```

---

## Conclusion

Vous disposez maintenant d’un ensemble cohérent de patterns et d’outils pour manipuler des données en mouvement :

- **Fichiers** pour la persistance simple
- **Streams** pour des pipelines robustes et scalables
- **JSON/XML** pour des contrats lisibles et versionnables
- **Binary contrôlé** ou formats modernes pour la performance
- **Sécurité** comme contrainte de conception (paths, atomique, validation, chiffrement)
