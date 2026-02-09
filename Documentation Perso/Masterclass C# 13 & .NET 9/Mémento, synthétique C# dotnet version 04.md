Architecture, Développement Applicatif et Stratégies de Déploiement
1. Introduction à l'écosystème .NET 9
Structure et Objectifs

L'objectif est de maîtriser une plateforme unifiée capable de s'exécuter partout : du cloud (ASP.NET) au mobile (.NET MAUI) en passant par le bureau (WPF/WinForms).
Choisir les bons outils

    IDE : Visual Studio 2022 (Windows), VS Code (Cross-platform) avec le C# Dev Kit.

    Compilateur : Roslyn (C# vers IL).

    Source Control : Intégration native de Git.

    Unit Testing : MSTest, nUnit ou xUnit.

Types d'applications et Architectures

    Console : Idéal pour les outils système.

    Web : ASP.NET Core (APIs, Razor Pages, MVC).

    Desktop : WinForms (simplicité), WPF (graphismes riches via XAML).

    Mobile : .NET MAUI (successeur de Xamarin).

    Jeux : Unity (C# est le langage standard).

    Architectures modernes : MVVM (Model-View-ViewModel) pour WPF/MAUI et MVU (Model-View-Update) pour les interfaces réactives.

Déploiement Cross-Platform

Le passage de .NET Framework (Windows uniquement) à .NET Core/.NET 5-9 permet le déploiement sur Linux et macOS. L'utilisation de conteneurs (Docker) et de pipelines CI/CD est désormais la norme.
2. Fondamentaux de C#
Syntaxe et Variables

    Types Valeurs : int, bool, struct, enum (Stack).

    Types Références : class, interface, string, dynamic (Heap).

    Opérateurs : Arithmétiques, logiques et les opérateurs de nullité (??, ?.).

Mots-clés et Identifiants

    Mots-clés : Réservés par le langage (using, namespace, static).

    Conventions : PascalCase pour les classes/méthodes, camelCase pour les variables locales.

3. Maîtrise du flux (Harnessing the Code)
Instructions de sélection et itération

    Selection : if, else if, switch (avec les switch expressions de C# 13).

    Boucles : for, while, do-while et foreach.

    Instructions spéciales : yield return pour la génération de séquences paresseuses, goto (à éviter), et break/continue.

Nullabilité et Sécurité

    Types Nullables : int? a = null;.

    Checked/Unchecked : Pour contrôler le dépassement de capacité (overflow) arithmétique.

    Exception Handling : try-catch-finally. Utilisation de filtres when pour attraper des erreurs spécifiques.

4. Fonctions In-depth
Déclarations et Paramètres

    Paramètres : ref (référence), out (sortie obligatoire), params (nombre variable d'arguments - amélioré en C# 13).

    Surcharge : Plusieurs méthodes avec le même nom mais des signatures différentes.

Lambda et Programmation Fonctionnelle

    Expressions Lambda : (x => x * x).

    Délégués intégrés : Action<> (ne retourne rien) et Func<> (retourne une valeur).

    Anonymes : Fonctions créées à la volée sans nom.

Débogage et Tests

    Outils : "Attach to Process", SOS Debugging (pour les problèmes de mémoire).

    Unit Testing : Utilisation de Stubs et Mocks pour isoler le code testé.

    Récursion : Comprendre la pile d'appels (Call Stack) et la "Tail Recursion".

5 & 6. Programmation Orientée Objet (POO) & Interfaces
Classes, Objets et Records

    Records : public record Person(string Name);. Introduits pour l'immuabilité et la comparaison par valeur.

    Fields vs Properties : Utilisation des auto-properties { get; set; } pour l'encapsulation.

Piliers de la POO

    Encapsulation : Cacher l'état interne (modificateurs private, protected).

    Abstraction : abstract class (classe incomplète) vs interface (contrat total).

    Héritage : Réutilisation via : BaseClass.

    Polymorphisme : Utilisation de virtual et override.

Concepts Avancés

    Méthodes d'extension : Ajouter des fonctionnalités à une classe existante sans la modifier.

    Static vs Instance : Les membres static appartiennent à la classe, pas à l'objet.

7 & 8. .NET Toolbox & Data in Motion
Manipulation de données

    System.String : Utilisation de StringBuilder pour la performance.

    Dates : DateTime, DateOnly, TimeOnly.

    Collections : List<T>, Dictionary<TKey, TValue>, Stack, Queue.

Fichiers et Flux (I/O)

    System.IO : File, Directory, Path.

    Streams : FileStream, MemoryStream, StreamReader/Writer.

    Sérialisation : Transformation d'objets en JSON (via System.Text.Json) ou XML pour le stockage ou l'échange réseau.

9 & 10. Entity Framework Core & LINQ
EF Core (ORM)

    Modélisation : DbContext et DbSet.

    Migrations : Gestion des versions du schéma de base de données.

    CRUD : Méthodes Add, Update, Remove.

    Performance : Lazy loading vs Eager loading (Include).

LINQ (Language Integrated Query)

    Permet d'interroger n'importe quelle source de données (SQL, XML, Objets).

    Opérateurs : Where, Select, OrderBy, GroupBy, Join.

    Deferred Execution : La requête n'est exécutée que lorsqu'on énumère les résultats.

11 à 14. ASP.NET Core, MVC et Web Services
Architecture Web

    Middleware : Pipeline de traitement des requêtes (Authentification -> Routage -> Réponse).

    Dependency Injection (DI) : Injection des services via le constructeur.

    Razor Pages vs MVC : Razor Pages est orienté "page", MVC est orienté "action/contrôleur".

APIs et Services

    RESTful APIs : Utilisation de ControllerBase et des attributs [HttpGet], [HttpPost].

    gRPC : Pour une communication inter-services ultra-rapide (binaire).

    Pagination & Versioning : Essentiels pour la scalabilité des APIs.

15. Blazor pour l'UI moderne

    Composants : Architecture basée sur des composants réutilisables (.razor).

    Hosting Models :

        Blazor WebAssembly : S'exécute dans le navigateur (client-side).

        Blazor Server : S'exécute sur le serveur via SignalR.

    JS Interop : Permet d'appeler des bibliothèques JavaScript depuis C#.

16. Packaging et Déploiement

    NuGet : Créer et consommer des paquets de code.

    Assembly Versioning : Gérer les versions Major.Minor.Build.Revision.

    Native AOT (.NET 9) : Compilation directe en code machine pour supprimer la dépendance au runtime et accélérer le démarrage (Cold Start).

    Conteneurisation : Publication d'images Docker pour le Cloud.

Mémento rapide des commandes CLI

    dotnet new console : Créer un projet.

    dotnet build : Compiler.

    dotnet publish -c Release : Préparer pour la production.

    dotnet ef migrations add Name : Créer une migration DB.

