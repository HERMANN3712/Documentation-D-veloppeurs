Maîtriser le développement applicatif, la gestion des données et le déploiement.
1. Introduction à l'écosystème
Structure et Évolution

    .NET 9 : La version la plus récente offrant des performances accrues pour l'IA et le Cloud.

    C# 13 : Le langage phare, avec des optimisations sur la gestion de la mémoire.

    Architectures : Passage du monolithique aux microservices, et de MVVM (Model-View-ViewModel) pour le desktop à MVU (Model-View-Update) pour le web moderne.

Outils de développement

    IDE : Visual Studio 2022 (complet), VS Code (léger), et le CLI .NET.

    CI/CD : Pipelines automatisés pour Windows (MSI/EXE), Web (Azure/AWS), Mobile (APK/IPA) et Mac (APP/PKG).

    Tests : Frameworks xUnit et nUnit pour garantir la qualité.

2. Fondamentaux de C#
Syntaxe et Variables

Le typage en C# est fort et statique.
C#

int age = 30; // Value type (Stack)
string nom = "Gemini"; // Reference type (Heap)
var temperature = 25.5; // Typage implicite

    Namespaces : Organisation logique (namespace MonApp.Data).

    Conventions : PascalCase pour les méthodes et classes, camelCase pour les variables privées.

Types et Conversions

    Value vs Reference : Les types valeurs contiennent la donnée, les types références contiennent l'adresse.

    Casting : Conversions implicites (sûres) ou explicites (int)monDouble.

3. Maîtrise du flux de code
Sélection et Itération

    Instructions : if, else if, else.

    Switch Expression (C# 13) :
    C#

    string message = code switch { 200 => "OK", 404 => "Not Found", _ => "Erreur" };

    Boucles : for, while, do-while et foreach.

    Yield : yield return permet de retourner des éléments un par un sans créer de liste en mémoire.

Nullabilité et Exceptions

    Nullable Types : int? score = null;.

    Exception Handling : try { ... } catch (Exception ex) { ... } finally { ... }.

    Checked/Unchecked : Gère le comportement en cas de dépassement arithmétique.

4. Fonctions en profondeur
Déclaration et Méthodes

    Paramètres : ref (entrée/sortie), out (sortie uniquement), in (lecture seule).

    Params (C# 13) : On peut désormais passer des Span<T> ou IEnumerable<T> en plus des tableaux.

Lambdas et Programmation Fonctionnelle

    Expressions Lambda : Func<int, int> carrer = x => x * x;.

    Action & Func : Délégués standards pour simplifier le passage de méthodes en paramètres.

Débogage et Tests unitaires

    SOS Debugging : Pour inspecter la mémoire à bas niveau.

    Mocking : Utilisation de Stubs pour simuler des dépendances (ex: base de données) lors des tests.

5 & 6. Programmation Orientée Objet (POO) & Interfaces
Classes, Objets et Records

    Records : public record User(string Id, string Name); (Immuabilité et comparaison par valeur).

    Encapsulation : Utilisation des propriétés get; set; avec modificateurs d'accès (private, internal, protected).

Les Piliers de la POO

    Héritage : class Voiture : Vehicule.

    Abstraction : abstract class ne peut pas être instanciée.

    Polymorphisme : Utilisation de virtual et override.

    Interfaces : Contrat strict. Une classe peut implémenter plusieurs interfaces.

7 & 8. .NET Toolbox & Data in Motion
Manipulation et Strings

    StringBuilder : Indispensable pour la concaténation dans les boucles.

    Date & Time : DateOnly et TimeOnly introduits pour simplifier les calculs sans fuseau horaire.

    Regex : Puissant pour la validation de formats (Emails, téléphones).

Fichiers et Sérialisation

    System.IO : Manipulation des chemins (Path), répertoires et fichiers.

    JSON : System.Text.Json est le standard pour le transfert de données web.

    Streams : Lecture/écriture asynchrone pour ne pas bloquer l'application.

9 & 10. Entity Framework Core (EF Core) & LINQ
EF Core (ORM)

    DbContext : Votre pont vers la base de données.

    Migrations : Suivi des versions de votre schéma SQL.

    Chargement : Eager loading (.Include()) vs Lazy loading.

LINQ

    Requêtes : var result = items.Where(x => x.Prix > 10).OrderBy(x => x.Nom);.

    Deferred Execution : La requête n'est lancée qu'au moment du .ToList() ou du foreach.

11 à 14. Développement Web (ASP.NET Core & APIs)
Architecture MVC et Razor Pages

    MVC : Contrôleur gère la logique, Vue affiche l'UI.

    Razor Pages : Plus simple, chaque fichier .cshtml contient sa propre logique.

    Middleware : Pipeline où chaque étape (Auth, Log, Cache) traite la requête HTTP.

Web Services

    REST : Standard basé sur les verbes HTTP.

    gRPC : Haute performance, idéal pour le micro-service à micro-service.

    Swagger/OpenAPI : Documentation auto-générée de vos APIs.

15. Blazor : Le futur de l'UI

    Composants : Fichiers .razor combinant HTML et C#.

    Render Modes (.NET 9) :

        Server : UI fluide via SignalR.

        WebAssembly : Exécution 100% côté client.

        Static : Rendu rapide côté serveur sans interactivité complexe.

16. Packaging et Déploiement
Stratégies modernes

    NuGet : Distribuer votre code sous forme de bibliothèques.

    Native AOT : Compilation en code machine. Résultat : démarrage instantané et pas besoin de runtime installé.

    Déploiement Mobile : Génération des fichiers .apk (Android) et .ipa (iOS) via .NET MAUI.

Mémento Technique Rapide
Objectif	Commande / Concept
Créer un projet	dotnet new <template>
Mettre à jour DB	dotnet ef database update
Publier (AOT)	dotnet publish -c Release -r win-x64 /p:PublishAot=true
Immuabilité	record
Contrat	interface