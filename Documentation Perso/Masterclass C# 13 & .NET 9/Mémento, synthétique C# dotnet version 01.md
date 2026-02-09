Mémento structuré et synthétique de votre cours sur C# 13 et .NET 9. Ce guide est conçu pour réviser les concepts clés, des bases du langage aux stratégies de déploiement modernes.
1. Fondamentaux et Environnement (.NET 9 & C# 13)

    L'écosystème : .NET 9 est une plateforme unifiée (Cross-platform). Le compilateur Roslyn transforme le code en IL (Intermediate Language), exécuté par le CLR (Common Language Runtime).

    Outils : Visual Studio 2022 (complet), VS Code (léger avec C# Dev Kit), et le CLI .NET (dotnet new, dotnet build, dotnet run).

    Nouveautés C# 13 : Amélioration des params (support des collections), nouvelles séquences d'échappement, et optimisation du type Span.

2. Syntaxe et Programmation Orientée Objet (POO)

    Types de données : Distinction entre Value Types (struct, enum - stockés sur la stack) et Reference Types (class, interface, record - stockés sur le heap).

    Structures de contrôle : if/else, switch (et expressions de switch), boucles for, foreach, while.

    POO :

        Encapsulation : Usage des modificateurs d'accès (public, private, protected, internal).

        Héritage : Une classe dérive d'une classe de base (mot-clé : base).

        Polymorphisme : Utilisation de virtual dans la classe mère et override dans la classe fille.

        Abstraction : interface (contrat pur) vs abstract class (peut contenir de la logique).

    Records : Types immuables par défaut, parfaits pour le transport de données (DTO), utilisant l'égalité par valeur.

3. Fonctions et Logique Avancée

    Méthodes : Support des paramètres optionnels, nommés, et du passage par référence (ref, out, in).

    Lambdas & LINQ : Expressions anonymes (x => x * x) utilisées avec LINQ pour requêter des collections (méthodes Where, Select, OrderBy, GroupBy).

    Gestion des erreurs : Blocs try { ... } catch (Exception ex) { ... } finally { ... }.

    Asynchronisme : Utilisation de async et await pour libérer le thread principal lors d'opérations d'E/S (I/O).

4. Gestion des Données (EF Core & IO)

    System.IO : Manipulation de fichiers (File, Directory) et de flux (StreamReader, StreamWriter).

    Entity Framework Core (EF Core) : * Approche Code-First : On définit les classes, EF génère la base de données via les Migrations.

        LINQ to Entities : Traduit les requêtes C# en SQL.

        Performance : Utiliser AsNoTracking() pour les lectures seules et le chargement immédiat (Include) pour éviter le problème du N+1.

5. Développement Web (ASP.NET Core & Blazor)

    Patterns de rendu :

        MVC : Séparation Modèle-Vue-Contrôleur. Idéal pour les applications complexes.

        Razor Pages : Approche centrée sur la page (plus simple que MVC pour les sites basés sur des formulaires).

        Blazor : Développement d'interfaces interactives en C# (remplace JavaScript).

            Blazor Server (via SignalR).

            Blazor WebAssembly (client-side).

            .NET 9 Hybrid : Mélange des modes de rendu.

    Middleware : Pipeline de composants traitant les requêtes HTTP (Auth, Logging, Routage).

    Injection de Dépendances (DI) : Enregistrement des services en mode Transient (éphémère), Scoped (par requête) ou Singleton (unique).

6. Services Web et API

    REST : Utilisation des verbes HTTP (GET, POST, PUT, DELETE) et codes de statut (200, 404, 500).

    gRPC : Protocole haute performance basé sur HTTP/2 et Protocol Buffers.

    Documentation : Intégration de Swagger/OpenAPI pour tester les API.

7. Déploiement et Stratégies

    Packaging : Création de bibliothèques via NuGet.

    Déploiement Mobile : .NET MAUI pour cibler iOS, Android, Windows et Mac à partir d'un seul code.

    Optimisation .NET 9 : * Native AOT (Ahead-Of-Time) : Compile le code directement en machine native pour un démarrage ultra-rapide et une empreinte mémoire réduite.

        CI/CD : Automatisation via GitHub Actions ou Azure DevOps.

Mémo technique rapide :
Concept	Mot-clé / Outil	Utilité
Nullabilité	string?	Évite les NullReferenceException.
Comparaison	record	Égalité basée sur les données, pas l'adresse mémoire.
Base de données	DbContext	Passerelle entre le code et la DB.
Performance	Span<T>	Manipulation de mémoire efficace.
Interface	interface	Définit "ce que l'objet fait" (contrat).