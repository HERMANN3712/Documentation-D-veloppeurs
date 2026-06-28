# 📑 Masterclass C# 13 & .NET 9 : Le Guide Complet</br>
</br>

# Architecture, Développement Applicatif, Gestion des Données et Déploiement
</br>

## 1. Introduction à l'écosystème .NET 9
### Structure et Évolution


- .NET 9 : Plateforme open-source unifiée et cross-platform (succédant à .NET 8). Focus sur la performance, l'IA et le Native AOT.

- Le Runtime (CLR) : Environnement qui exécute le code.
	
- Les Bibliothèques : Ensembles de fonctions prêtes à l'emploi (Base Class Library).
	
- C# 13 : Le langage phare. Apporte des améliorations de syntaxe comme les nouveaux paramètres params (supportant IEnumerable, Span), et des optimisations de mémoire.

- Le Compilateur (Roslyn) : Transforme le code C# en code intermédiaire (IL - Intermediate Language).

- Architectures modernes : Transition du monolithique vers les microservices. Utilisation de MVVM (Model-View-ViewModel) pour le desktop/mobile et MVU (Model-View-Update) pour les interfaces réactives (pour le web moderne).

### Choisir les bons outils

- IDE : Visual Studio 2022 (complet, Windows), VS Code (léger avec C# Dev Kit), JetBrains Rider, et le CLI .NET.

- Gestion de version : Intégration native de Git.
	
- CI/CD : Pipelines automatisés pour Windows (MSI/EXE), Web (Azure/AWS), Mobile (APK/IPA) et Mac (APP/PKG).

- CLI .NET : Commandes essentielles (dotnet new, dotnet build, dotnet run, dotnet watch).
	
- Tests : Frameworks MSTest, nUnit ou xUnit pour garantir la qualité. (SonarQube)
	
### Quoi de neuf dans C# 13 ?

- Params Collections : Le mot-clé params accepte désormais IEnumerable<T>, Span<T>, etc., et non plus seulement des tableaux.

- Nouvelle séquence d'échappement : \e pour le caractère ESC.

- Types "Ref" améliorés : Plus de flexibilité dans l'utilisation des structures ref.

### Types d'applications
|Type|Technologies|Usage|
|----|------------|-----|
|Console|System.Console|Outils CLI, automatisation, services système, outils système[]
|Web|ASP.NET Core|(MVC, Razor, APIs)Sites dynamiques et backends|
|Desktop|WinForms, WPF (XAML), WinUI 3|WinUI 3 est l'infrastructure d'interface utilisateur native moderne de Microsoft pour la création d'applications de bureau Windows|
|Mobile|.NET MAUI|Cross-platform (iOS, Android, macOS, Windows)|
|Jeux|Unity|C# est le langage standard de l'industrie|


## 2. Fondamentaux du Langage C#
### Syntaxe et Variables

C# est un langage fortement typé et statique.

* Value Types (Stack) : int, double, bool, char, struct, enum (stockés sur la pile/stack). Stockent directement la valeur.
* Reference Types (Heap) : string, object, class, interface, record, delegate. Stockent une adresse mémoire.
* Nouveauté C# 13 : Optimisation du type Span<T> et nouvelles séquences d'échappement comme \e pour ESC.

### Opérateurs et Conversion

* Opérateurs : Arithmétiques : +, -, *, /, %, logiques (&&, ||), et de nullité/Null-coalescing (??, ?., ??=). Null-conditional : objet?.Propriete (évite la NullReferenceException).
* Mots-clés : Réservés par le langage (using, namespace, static).
* Conversion (Casting) : Implicite (sûre) ou explicite (int)monDouble. Utilisation de int.TryParse() pour éviter les erreurs de format.
* Casting : ```csharp double d = 9.8; int i = (int)d; // Conversion explicite```
	
```C#
int age = 30; // Value type (Stack)
string nom = "Gemini"; // Reference type (Heap)
var temperature = 25.5; // Typage implicite
```

- Namespaces : Organisation logique (ex: namespace MonProjet.Services, namespace MonApp.Data).

- Conventions : PascalCase pour les méthodes et classes, camelCase pour les variables privées.

### Types et Conversions

- Value vs Reference : Les types valeurs contiennent la donnée, les types références contiennent l'adresse.
- Casting : Conversions implicites (sûres) ou explicites (int)monDouble.


## 3. Maîtrise du flux de code (Harnessing the Code)
### Sélection et Itération
- Selection : if, else if, switch (avec les switch expressions de C# 13).
    Switch Expression (C# 13) moderne : Utilisation des Switch Expressions avec Pattern Matching.
```C#
    string result = status switch { 200 => "OK", 404 => "Not Found", _ => "Unknown" };
	string resultat = temperature switch {
		< 0 => "Il gèle",
		0 => "Point de congélation",
		<= 20 => "Frais",
		_ => "Chaud" // Valeur par défaut
	};
```
- Boucles : for, while, do-while, et foreach (pour parcourir les collections). foreach : Idéal pour parcourir les collections sans index.
	
- Instructions spéciales : yield return pour la génération de séquences paresseuses, goto (à éviter), et break/continue.
    Yield : yield return crée des itérateurs personnalisés sans construire de listes massives en mémoire ou de liste intermédiaire.

### Nullabilité et Sécurité

- Types Nullables : int? score = null;. Protection contre les NullReferenceException activée par défaut.
```C#
	string? nomPeutEtreNull = null; // Autorisé
	string nomObligatoire = "Test"; // Ne peut pas être null
```
- Checked/Unchecked : Pour contrôler les dépassements de capacité (overflow) arithmétique.

### Gestion des Exceptions

- Blocs : try, catch, finally.
	
- Exception Handling : try { ... } catch (Exception ex) { ... } finally { ... }.

- Filtres : catch (Exception ex) when (ex.HResult == 404).
	

## 4. Fonctions en profondeur (Fonctions In-depth)
### Déclarations et Paramètres (méthode)

- Passage : ref (référence), out (sortie obligatoire / uniquement), in (lecture seule).</br>
	Passage par valeur, par référence (ref, out, in).

- Surcharge (Overloading) : Plusieurs méthodes avec le même nom mais des signatures différentes.

- C# 13 Params : Le mot-clé params accepte désormais IEnumerable<T>, Span<T>, etc. ... en plus des tableaux.

### Lambdas et Programmation Fonctionnelle

- Expressions Lambda : (x, y) => x + y. Une expression lambda est une fonction anonyme utilisée pour créer des délégués.
```C#
	// Syntaxe : (paramètres) => expression
	Func<int, int, int> addition = (a, b) => a + b;
	Console.WriteLine(addition(5, 3)); //
```

- Délégués intégrés : Action<> (ne retourne rien) et Func<> (retourne une valeur).

- Anonymes : Fonctions créées à la volée sans nom.    

- Lambdas : (x, y) => x + y;. Types Action<> (void) et Func<> (retourne une valeur).

### Débogage et Tests

- Débogage avancé : Points d'arrêt, fenêtre espion (Watch), "Attach to Process", et SOS Debugging pour la mémoire.

- Unit Testing : Frameworks xUnit ou NUnit. Utilisation de Mocks (simulations) et Stubs pour isoler le code. Le test unitaire assure que chaque fonction ("unité") fonctionne isolément.

```C#
	[Fact]
	public void TestAddition() {
		var resultat = Calculateur.Ajouter(2, 2);
		Assert.Equal(4, resultat);
	}
```
	
- Récursion : Comprendre la pile d'appels (Call Stack) et la récursion terminale/"Tail Recursion".

## 5 & 6. Programmation Orientée Objet (POO) & Interfaces
### Classes, Objets et Records

- Classes : Identité par référence.
	
- Membres : Champs (fields), Propriétés (get; set;), Méthodes, Constructeurs.

- Records : Identité par valeur. Idéal pour les DTO (Data Transfer Objects) et l'immuabilité.
```C#
public record User(string Id, string Name); //Immuabilité et comparaison par valeur
```

## Les Piliers de la POO

- Héritage : Réutilisation via : BaseClass. Mot-clé base pour appeler le parent.
	
- Encapsulation et Propriétés. Utilisation des propriétés `get; set;` avec modificateurs d'accès (private, internal, protected).</br>
Utilisez des propriétés pour protéger vos champs (fields) :
```C#
	public class CompteBancaire {
		private decimal _solde; // Champ privé
		public decimal Solde => _solde; // Propriété en lecture seule
	}
```

- Abstraction : interface (contrat pur / strict) vs abstract class (peut contenir de la logique). 
 Une classe peut implémenter plusieurs interfaces.</br>
 `abstract class` ne peut pas être instanciée.
	
- Polymorphisme : Mots-clés virtual (parent) et override (enfant).</br>
    Méthodes d'extension : Ajouter des fonctions à une classe existante sans la modifier.
	
## 6. Interfaces et Hiérarchie

- Interfaces : Définissent un contrat. Un membre peut implémenter plusieurs interfaces.

- Gestion de la mémoire : Distinction entre type valeur (Stack) et type référence (Heap).

- Mots-clés avancés : new (masquage de membre), sealed (empêche l'héritage).
	
### Concepts Avancés

- Méthodes d'extension : Ajouter des fonctionnalités à une classe existante sans la modifier.

- Static vs Instance : Les membres static appartiennent à la classe, pas à l'objet.
	

## 7 & 8. .NET Toolbox & Data in Motion
### Manipulation de données

- StringBuilder : Plus performant pour les concaténations dans les boucles.

- Dates : DateTime (complet), DateOnly et TimeOnly (plus précis pour certains usages). DateOnly et TimeOnly introduits pour simplifier les calculs sans fuseau horaire.

- Collections : List<T>, Dictionary<TKey, TValue>, Stack, Queue, Span<T> (performance mémoire).
	
- GUID : Guid.NewGuid() génère un identifiant unique universel.
	
- Regex : Puissant pour la validation de formats (Emails, téléphones).

### I/O et Sérialisation

- System.IO : (File, Directory, Path) Manipulation de fichiers (File), répertoires (Directory) et chemins (Path).

- Streams : FileStream, MemoryStream pour la lecture/écriture progressive. Lecture/écriture asynchrone pour ne pas bloquer l'application.

- JSON : System.Text.Json est le standard haute performance pour .NET 9.
	
## 8. Gestion des données et fichiers / Data in Motion (E/S et Sérialisation)

- System.IO : File.ReadAllText(), Directory.GetFiles().

- Streams : FileStream, MemoryStream pour la lecture/écriture progressive.

- Sérialisation : * JSON : System.Text.Json (très performant).
	Transformation d'objets en JSON (via System.Text.Json) ou XML pour le stockage ou l'échange réseau.

    `XML : XmlSerializer`.
```C#
	var json = JsonSerializer.Serialize(monObjet);
	var objet = JsonSerializer.Deserialize<MonType>(json);
```
		

##  9 & 10. Entity Framework Core (EF Core) & LINQ
### EF Core (ORM)
- Entity Framework Core (EF Core) : * Approche Code-First : On définit les classes, EF génère la base de données via les Migrations.
	
- EF Core est un ORM (Object-Relational Mapper). Il permet de manipuler une base de données avec des objets C#.
CRUD avec EF Core

    - Create : db.Users.Add(newUser);

    - Read : db.Users.Where(u => u.IsActive).ToList();

    - Update : user.Name = "Nouveau Nom";
    
    - Delete : db.Users.Remove(user); N'oubliez pas db.SaveChanges(); pour valider.


- Modèle : DbContext (pont DB) et DbSet (collections d'entités).

- Migrations : Gestion du versioning du schéma SQL (dotnet ef migrations add).
		
- Relations : One-to-one, One-to-many, Many-to-many.

- Chargement/Performance : Eager loading (Include) vs Lazy loading.

- Optimisation : AsNoTracking() pour les requêtes en lecture seule. Utiliser AsNoTracking() pour les lectures seules et le chargement immédiat (Include) pour éviter le problème du N+1.	

- LINQ (Language Integrated Query) (LINQ Unleashed déchaîner/libérer/détacher)

    - Syntaxe : Méthode (list.Where(...)) ou Requête (from x in list...). var result = items.Where(x => x.Prix > 10).OrderBy(x => x.Nom);.
	
    - Opérateurs : Select, Where, OrderBy, GroupBy, Join, Any, First.

    - Deferred Execution : La requête n'est exécutée qu'au moment de l'itération (foreach, .ToList()).
	
    - LINQ (Language Integrated Query) permet de filtrer et transformer des données de manière déclarative.

    - LINQ to Entities : Traduit les requêtes C# en SQL.
```C#
	var experts = employes
		.Where(e => e.Anciennete > 5)
		.OrderBy(e => e.Nom)
		.Select(e => e.Email);
```

## 11 & 12. ASP.NET Core et Razor Pages

- Middleware : Pipeline de traitement des requêtes (Authentification -> Routage -> Réponse). Pipeline où chaque étape (Auth, Log, Cache) traite la requête HTTP.

- Injection de Dépendances (Dependency Injection) (DI) : Transient (éphémère), Scoped (par requête), Singleton (unique). Injection des services via le constructeur.
Enregistrement des services en mode Transient (éphémère), Scoped (par requête) ou Singleton (unique).

- Patterns : MVC (Contrôleur/Vue)(Contrôleur gère la logique, Vue affiche l'UI) vs Razor Pages (orienté Page / Plus simple, chaque fichier .cshtml contient sa propre logique.).

- Razor Pages : Modèle basé sur les pages avec MVVM léger (PageModel). Approche centrée sur la page (plus simple que MVC pour les sites basés sur des formulaires).

- Tag Helpers : asp-for, asp-validation-for.

## 13. Le pattern MVC (Model-View-Controller)

### Séparation des préoccupations :

- Model : Données (POCO).

- View : Interface utilisateur (HTML/Razor).

- Controller : Logique métier et orchestration.  Reçoit la requête, interagit avec le modèle et renvoie la vue

- Routage : [Route("api/[controller]")].


## 14. Services Web (API & gRPC)

Une API moderne renvoie du JSON.
```C#
	[HttpGet]
	public IActionResult GetProducts() {
		return Ok(_db.Products.ToList());
	}
```

- RESTful APIs : Utilisation des verbes HTTP (GET, POST, PUT, DELETE) et codes de statut (200, 404, 500). Utilisation de ControllerBase et des attributs [HttpGet], [HttpPost].

- gRPC : Communication binaire haute performance (HTTP/2), idéal pour le micro-service à micro-service. Protocole haute performance basé sur HTTP/2 et Protocol Buffers.
	
- API Versioning : Gérer plusieurs versions d'une API.
- Pagination & Versioning : Essentiels pour la scalabilité des APIs.

- Swagger/OpenAPI : Documentation et test automatique des points d'entrée (endpoints). Documentation auto-générée de vos APIs.
	Outil intégré qui génère une page web permettant de tester vos points d'entrée (endpoints) sans interface client.	

### 15. Blazor : UI Moderne en C#
Blazor : Développement d'interfaces interactives en C# (remplace JavaScript).

- Composants : Fichiers .razor réutilisables.

- Modes de rendu (.NET 9) /  Hosting Models :

- Blazor Server (via SignalR) : Exécution côté serveur via SignalR.

- Blazor WebAssembly (WASM)(client-side) : Exécution côté client (téléchargement du runtime). Le code est téléchargé et s'exécute dans le navigateur de l'utilisateur. Approche centrée sur la page (plus simple que MVC pour les sites basés sur des formulaires).

- Auto Mode : Bascule intelligente entre les deux. // .NET 9 Hybrid : Mélange des modes de rendu.

- JS Interop : Communication bidirectionnelle avec JavaScript. Permet d'appeler des bibliothèques JavaScript depuis C#.

## 16. Packaging et Déploiement
### Stratégies et Publication

- NuGet : Gestionnaire de paquets pour distribuer des bibliothèques. Créer et consommer des paquets de code.
	
- Packaging : Création de bibliothèques via NuGet.
	
- Assembly Versioning : Gérer les versions Major.Minor.Build.Revision

- Native AOT (Ahead-Of-Time)(Nouveauté majeure de .NET 9) : Compilation directe en code machine. Démarrage instantané, pas de dépendance au runtime, idéal pour le Cloud/Lambda.
Compilation directe en code machine pour supprimer la dépendance au runtime et accélérer le démarrage (Cold Start).
    - Avantage : Pas besoin d'installer le runtime .NET sur le serveur, démarrage quasi instantané, consommation mémoire réduite.
    - Commande : dotnet publish -c Release -r win-x64 --self-contained.

- CI/CD : Automatisation via GitHub Actions ou Azure DevOps. GitHub Actions ou Azure DevOps pour automatiser les tests et le déploiement.
	
- Publication :

    - Framework-dependent : Nécessite le runtime .NET installé.

    - Self-contained : Inclut le runtime dans l'exécutable.

    - Native AOT : Compilation en code machine natif pour un démarrage instantané.
	
- Déploiement Mobile : Génération des fichiers .apk (Android) et .ipa (iOS) via .NET MAUI. .NET MAUI pour cibler iOS, Android, Windows et Mac à partir d'un seul code.

- Conteneurisation : Publication d'images Docker. Publication d'images Docker pour le Cloud.

### 🛠 Mémento Technique Rapide
| Objectif | Commande / Concept| Utilité|
|----------|--------------------|-------|
|Initialisation|dotnet new template|Créer un nouveau projet|
|Base de données|DbContext|Gérer la liaison SQL. Passerelle entre le code et la DB|
|Performance|Native AOT	Supprimer le JIT et accélérer le démarrage|
|Immuabilité (record)|record|Comparaison par valeur simplifiée|
|Comparaison (record)|record|Égalité basée sur les données, pas l'adresse mémoire|
|Interface|	|Définir un comportement obligatoire. Définit "ce que l'objet fait" (contrat)|
|Nullabilité|string?|Évite les NullReferenceException|
|Asynchronisme|async / await|Ne pas bloquer le thread principal|
|Performance|Span<T>|Manipulation de mémoire efficace|

#### Mémento rapide des commandes CLI

`dotnet new console` : Créer un projet.


   `dotnet build` : Compiler.


`dotnet publish -c Release` : Préparer pour la production.


`dotnet ef migrations add Name` : Créer une migration DB.