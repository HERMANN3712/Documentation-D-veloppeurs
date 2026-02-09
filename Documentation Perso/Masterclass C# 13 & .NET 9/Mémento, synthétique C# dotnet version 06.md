📑 Masterclass C# 13 & .NET 9 : Le Guide Complet

Architecture, Développement Applicatif, Gestion des Données et Déploiement
1. Introduction à l'écosystème .NET 9
Structure et Évolution

    .NET 9 : Plateforme open-source unifiée et cross-platform (succédant à .NET 8). Focus sur la performance, l'IA et le Native AOT.

    Le Runtime (CLR) : Environnement qui exécute le code.

    Le Compilateur (Roslyn) : Transforme le code C# en code intermédiaire (IL - Intermediate Language).

    Architectures modernes : Transition du monolithique vers les microservices. Utilisation de MVVM (Model-View-ViewModel) pour le desktop/mobile et MVU (Model-View-Update) pour les interfaces réactives.

Choisir les bons outils

    IDE : Visual Studio 2022 (complet, Windows), VS Code (léger avec C# Dev Kit), JetBrains Rider.

    Gestion de version : Intégration native de Git.

    CLI .NET : Commandes essentielles (dotnet new, dotnet build, dotnet run, dotnet watch).

Types d'applications
Type	Technologies	Usage
Console	System.Console	Outils CLI, automatisation, services système.
Web	ASP.NET Core (MVC, Razor, API)	Sites dynamiques et backends.
Desktop	WinForms, WPF (XAML), WinUI 3	Applications Windows natives.
Mobile	.NET MAUI	Cross-platform (iOS, Android, macOS, Windows).
Jeux	Unity	C# est le langage standard de l'industrie.
2. Fondamentaux du Langage C#
Syntaxe et Variables

C# est un langage fortement typé et statique.

    Value Types (Stack) : int, double, bool, char, struct, enum. Stockent directement la valeur.

    Reference Types (Heap) : string, class, interface, record, delegate. Stockent une adresse mémoire.

    Nouveauté C# 13 : Optimisation du type Span<T> et nouvelles séquences d'échappement comme \e pour ESC.

Opérateurs et Conversion

    Opérateurs : Arithmétiques, logiques (&&, ||), et de nullité (??, ?.).

    Conversion (Casting) : Implicite (sûre) ou explicite (int)monDouble. Utilisation de int.TryParse() pour éviter les erreurs de format.

3. Maîtrise du flux de code (Harnessing the Code)
Sélection et Itération

    Switch moderne : Utilisation des Switch Expressions avec Pattern Matching.
    C#

    string result = status switch { 200 => "OK", 404 => "Not Found", _ => "Unknown" };

    Boucles : for, while, do-while, et foreach (pour parcourir les collections).

    Yield : yield return crée des itérateurs personnalisés sans construire de listes massives en mémoire.

Nullabilité et Sécurité

    Types Nullables : int? score = null;. Protection contre les NullReferenceException activée par défaut.

    Checked/Unchecked : Pour contrôler les dépassements de capacité (overflow) arithmétique.

Gestion des Exceptions

    Blocs : try, catch, finally.

    Filtres : catch (Exception ex) when (ex.HResult == 404).

4. Fonctions en profondeur
Déclarations et Paramètres

    Passage : ref (référence), out (sortie obligatoire), in (lecture seule).

    C# 13 Params : Le mot-clé params accepte désormais IEnumerable<T>, Span<T>, etc.

Lambdas et Programmation Fonctionnelle

    Expressions Lambda : (x, y) => x + y.

    Délégués standards : Action<> (ne retourne rien), Func<> (retourne une valeur).

    Récursion : Comprendre la pile d'appels (Call Stack) et la récursion terminale.

Débogage et Tests

    Débogage avancé : Points d'arrêt, fenêtre espion (Watch), "Attach to Process", et SOS Debugging pour la mémoire.

    Unit Testing : Frameworks xUnit ou NUnit. Utilisation de Mocks (simulations) et Stubs pour isoler le code.

5 & 6. Programmation Orientée Objet (POO) & Interfaces
Classes, Objets et Records

    Classes : Identité par référence.

    Records : Identité par valeur. Idéal pour les DTO (Data Transfer Objects) et l'immuabilité.

    Encapsulation : Propriétés get; set; et modificateurs (public, private, protected, internal).

Les Piliers de la POO

    Héritage : Réutilisation via : BaseClass. Mot-clé base pour appeler le parent.

    Abstraction : interface (contrat pur) vs abstract class (peut contenir de la logique).

    Polymorphisme : Mots-clés virtual (parent) et override (enfant).

    Méthodes d'extension : Ajouter des fonctions à une classe existante sans la modifier.

7 & 8. .NET Toolbox & Data in Motion
Manipulation de données

    StringBuilder : Plus performant pour les concaténations dans les boucles.

    Dates : DateTime (complet), DateOnly et TimeOnly (plus précis pour certains usages).

    Collections : List<T>, Dictionary<TKey, TValue>, Stack, Queue, Span<T> (performance mémoire).

I/O et Sérialisation

    System.IO : Manipulation de fichiers (File), répertoires (Directory) et chemins (Path).

    Streams : FileStream, MemoryStream pour la lecture/écriture progressive.

    JSON : System.Text.Json est le standard haute performance pour .NET 9.

9 & 10. Entity Framework Core (EF Core) & LINQ
EF Core (ORM)

    Modèle : DbContext (pont DB) et DbSet (collections d'entités).

    Migrations : Gestion du versioning du schéma SQL (dotnet ef migrations add).

    Chargement : Eager loading (Include) vs Lazy loading.

    Optimisation : AsNoTracking() pour les requêtes en lecture seule.

LINQ (Language Integrated Query)

    Syntaxe : Méthode (list.Where(...)) ou Requête (from x in list...).

    Deferred Execution : La requête n'est exécutée qu'au moment de l'itération (foreach, .ToList()).

11 à 14. Développement Web et APIs
ASP.NET Core

    Middleware : Pipeline de traitement des requêtes (Authentification -> Routage -> Réponse).

    Injection de Dépendances (DI) : Transient (éphémère), Scoped (par requête), Singleton (unique).

    Patterns : MVC (Contrôleur/Vue) vs Razor Pages (orienté Page).

Web Services

    RESTful APIs : Utilisation des verbes HTTP (GET, POST, PUT, DELETE).

    gRPC : Communication binaire haute performance (HTTP/2).

    Swagger/OpenAPI : Documentation et test automatique des points d'entrée (endpoints).

15. Blazor : UI Moderne en C#

    Composants : Fichiers .razor réutilisables.

    Modes de rendu (.NET 9) :

        Blazor Server : Exécution côté serveur via SignalR.

        Blazor WebAssembly (WASM) : Exécution côté client (téléchargement du runtime).

        Auto Mode : Bascule intelligente entre les deux.

    JS Interop : Communication bidirectionnelle avec JavaScript.

16. Packaging et Déploiement
Stratégies et Publication

    NuGet : Gestionnaire de paquets pour distribuer des bibliothèques.

    Native AOT (.NET 9) : Compilation directe en code machine. Démarrage instantané, pas de dépendance au runtime, idéal pour le Cloud/Lambda.

    CI/CD : Automatisation via GitHub Actions ou Azure DevOps.

    Conteneurisation : Publication d'images Docker.

🛠 Mémento Technique Rapide
Objectif	Commande / Concept	Utilité
Initialisation	dotnet new <template>	Créer un nouveau projet.
Base de données	DbContext	Gérer la liaison SQL.
Performance	Native AOT	Supprimer le JIT et accélérer le démarrage.
Immuabilité	record	Comparaison par valeur simplifiée.
Contrat	interface	Définir un comportement obligatoire.
Asynchronisme	async / await	Ne pas bloquer le thread principal.