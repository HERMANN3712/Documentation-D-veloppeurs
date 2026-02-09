Guide exhaustif du développement moderne, de la gestion des données et du déploiement.
1. Introduction à C# 13 et .NET 9
L'écosystème .NET

.NET est une plateforme de développement open-source et cross-platform.

    Le Runtime (CLR) : Exécute le code.

    Les Bibliothèques : Ensembles de fonctions prêtes à l'emploi (Base Class Library).

    Le Compilateur (Roslyn) : Transforme le code C# en code intermédiaire (IL).

Quoi de neuf dans C# 13 ?

    Params Collections : Le mot-clé params accepte désormais IEnumerable<T>, Span<T>, etc., et non plus seulement des tableaux.

    Nouvelle séquence d'échappement : \e pour le caractère ESC.

    Types "Ref" améliorés : Plus de flexibilité dans l'utilisation des structures ref.

Types d'applications
Type	Usage
Console	Outils CLI, automatisation.
ASP.NET Core	APIs Web et sites dynamiques.
Blazor	Interfaces Web interactives en C#.
.NET MAUI	Applications natives iOS, Android, macOS, Windows.
2. Fondamentaux du langage C#
Syntaxe et Variables

C# est un langage fortement typé.

    Value Types (Stack) : int, double, bool, char, struct.

    Reference Types (Heap) : string, class, interface, delegate.

C#

int age = 25; // Entier
string nom = "Gemini"; // Chaîne
var temperature = 21.5; // Typage implicite (double)
const double PI = 3.14159; // Constante

Opérateurs et Conversion

    Arithmétiques : +, -, *, /, %.

    Null-conditional : objet?.Propriete (évite la NullReferenceException).

    Casting : ```csharp double d = 9.8; int i = (int)d; // Conversion explicite


3. Maîtrise du flux de code
Structures de sélection

L'instruction switch moderne est extrêmement puissante avec le Pattern Matching :
C#

string resultat = temperature switch {
    < 0 => "Il gèle",
    0 => "Point de congélation",
    <= 20 => "Frais",
    _ => "Chaud" // Valeur par défaut
};

Itérations

    foreach : Idéal pour parcourir les collections sans index.

    yield return : Permet de créer un itérateur personnalisé sans construire de liste intermédiaire.

Gestion des Nullables

Dans C# moderne, la protection contre les null est activée par défaut.
C#

string? nomPeutEtreNull = null; // Autorisé
string nomObligatoire = "Test"; // Ne peut pas être null

4. Fonctions en profondeur
Lambda Expressions

Une expression lambda est une fonction anonyme utilisée pour créer des délégués.
C#

// Syntaxe : (paramètres) => expression
Func<int, int, int> addition = (a, b) => a + b;
Console.WriteLine(addition(5, 3)); // 8

Débogage et Tests (xUnit)

Le test unitaire assure que chaque fonction ("unité") fonctionne isolément.
C#

[Fact]
public void TestAddition() {
    var resultat = Calculateur.Ajouter(2, 2);
    Assert.Equal(4, resultat);
}

5. Programmation Orientée Objet (POO)
Classes vs Records

    Class : Identité par référence (utilisée pour la logique métier).

    Record (C# 9+) : Identité par valeur. Idéal pour les DTO (Data Transfer Objects).
    C#

    public record User(string Email, string Role);

Encapsulation et Propriétés

Utilisez des propriétés pour protéger vos champs (fields).
C#

public class CompteBancaire {
    private decimal _solde; // Champ privé
    public decimal Solde => _solde; // Propriété en lecture seule
}

6. Interfaces et Héritage

L'interface définit un contrat. Une classe qui implémente l'interface s'engage à fournir les méthodes définies.
C#

public interface IVehicule {
    void Demarrer();
}

public class Voiture : IVehicule {
    public void Demarrer() => Console.WriteLine("Vroum !");
}

7. .NET Toolbox

    StringBuilder : Utilisez-le pour concaténer des chaînes dans une boucle (plus performant).

    Date & Time : .NET 9 utilise DateTime pour les dates complètes, mais privilégiez DateOnly si l'heure n'est pas nécessaire.

    GUID : Guid.NewGuid() génère un identifiant unique universel.

8. Data in Motion (E/S et Sérialisation)
Sérialisation JSON (System.Text.Json)

C'est la méthode standard pour transformer un objet en texte (pour une API).
C#

var json = JsonSerializer.Serialize(monObjet);
var objet = JsonSerializer.Deserialize<MonType>(json);

9. Entity Framework Core (EF Core)

EF Core est un ORM (Object-Relational Mapper). Il permet de manipuler une base de données avec des objets C#.
CRUD avec EF Core

    Create : db.Users.Add(newUser);

    Read : db.Users.Where(u => u.IsActive).ToList();

    Update : user.Name = "Nouveau Nom";

    Delete : db.Users.Remove(user); N'oubliez pas db.SaveChanges(); pour valider.

10. LINQ Unleashed

LINQ (Language Integrated Query) permet de filtrer et transformer des données de manière déclarative.
C#

var experts = employes
    .Where(e => e.Anciennete > 5)
    .OrderBy(e => e.Nom)
    .Select(e => e.Email);

11-13. ASP.NET Core & MVC

    Middleware : Composants logiciels assemblés dans un pipeline pour gérer les requêtes et réponses.

    MVC : * Model : Vos données/classes.

        View : Vos fichiers HTML/Razor.

        Controller : Reçoit la requête, interagit avec le modèle et renvoie la vue.

14. Web Services (API REST)

Une API moderne renvoie du JSON.
C#

[HttpGet]
public IActionResult GetProducts() {
    return Ok(_db.Products.ToList());
}

Swagger : Outil intégré qui génère une page web permettant de tester vos points d'entrée (endpoints) sans interface client.
15. Blazor : UI en C#

Blazor permet de créer des interfaces web sans JavaScript.

    Blazor Server : Le code s'exécute sur le serveur.

    Blazor WebAssembly (WASM) : Le code est téléchargé et s'exécute dans le navigateur de l'utilisateur.

16. Déploiement
Native AOT (Ahead-Of-Time)

Nouveauté majeure de .NET 9 : compile l'application en code machine natif.

    Avantage : Pas besoin d'installer le runtime .NET sur le serveur, démarrage quasi instantané, consommation mémoire réduite.

    Commande : dotnet publish -c Release -r win-x64 --self-contained.
