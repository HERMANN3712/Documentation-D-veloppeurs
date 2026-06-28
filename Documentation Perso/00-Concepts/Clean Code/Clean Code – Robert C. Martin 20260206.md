## Résumé condensé et actionnable de Clean Code – Robert C. Martin, pensé pour un dev (Java / C# / .NET) comme toi 👇

## 🧠 Philosophie générale

> **Du code est lu bien plus souvent qu’il n’est écrit.**</br>
> Le but du clean code est donc **la lisibilité**, la **maintenabilité** et la **robustesse**, pas l’astuce ou la performance prématurée.

---

## 🧩 1. Nommage (Naming)

- Un nom doit révéler l’intention

- Pas d’abréviations obscures

- Éviter la désinformation (list, array, map si ce n’est pas vrai)

- Même concept → même vocabulaire

- Différents concepts → mots différents

✅ Bon :
```csharp
getUserById()
```

❌ Mauvais :
```csharp
getData()
```
---

## 🧱 2. Fonctions

- Petites (souvent < 20 lignes)

- Une seule responsabilité

- Peu ou pas de paramètres (0–2 idéalement)

- Pas d’effets de bord cachés

- Les noms décrivent ce que ça fait, pas comment

💡 Une fonction doit raconter une histoire lisible de haut en bas.

---

## 🔀 3. Commentaires

- Le meilleur commentaire est celui dont on n’a pas besoin

- Éviter les commentaires redondants

- Les commentaires compensent souvent un mauvais code

- Autorisés pour :

    . Intentions complexes

    . Contraintes métier

    . TODO temporaires

❌
```csharp
// incrémente i
i++;
```
---

## 🧪 4. Tests

- Les tests sont du code de production

- Lisibles, rapides, indépendants

- AAA : Arrange / Act / Assert

- Un test = une seule raison d’échouer

💡 Un code difficile à tester est souvent mal conçu.

---

## 🚨 5. Gestion des erreurs

- Utiliser les exceptions, pas les codes de retour

- Ne pas retourner null → préférer Optional, Result, Null Object

- Les messages d’erreur doivent être clairs et exploitables

---

## 🧼 6. Formatage

- La mise en forme est une communication

- Code aéré, indentations cohérentes

-  Grouper les éléments liés

- Les fichiers doivent être courts

---

## 🧠 7. Objets & Structures de données

- Objets → exposent des comportements

- Structures → exposent des données

- Ne pas mélanger les deux

- Respecter l’encapsulation

---

## 🔁 8. Duplication

- DRY (Don’t Repeat Yourself)

- La duplication est la racine du mal

- Factoriser dès que le sens est clair

- Mais ne pas abstraire trop tôt

---

## 🏗️ 9. Classes

- Classes petites

- Une seule responsabilité (SRP)

- Forte cohésion, faible couplage

- Les changements doivent toucher une seule classe

---

## 🧯 10. Code smells (odeurs de code)

- À traquer en permanence :

    . Méthodes trop longues

    . Classes “God Object”

    . Trop de paramètres

    . Conditions complexes

    . Noms vagues

    . Commentaires excessifs

---

## 🧰 11. Refactoring

- Le clean code est un processus continu

- Refactorer avant d’ajouter une feature

- Les tests sont le filet de sécurité

- Pas de refactoring “big bang”



🎯 Message clé du livre

Le clean code est un acte de respect
Pour :

    - Tes collègues

    - Ton futur toi

    - Le produit


---



</br></br></br>

### 👍 Test techniqe et un export Github-ready


### 1️⃣ un exemple typique orienté test technique

### 2️⃣ un README.md “GitHub-ready” que tu peux copier/coller tel quel dans ton repo.

---

### 1️⃣ Exemple orienté test technique C#

👉 Contexte classique :

> “Voici une méthode, améliorez-la (clean code, testable, maintenable)”

❌ Code de départ (volontairement mauvais)
```csharp
public double Compute(int type, int days)
{
    if (type == 1)
    {
        if (days > 10)
            return days * 1.1;
        else
            return days;
    }
    else if (type == 2)
    {
        return days * 0.9;
    }
    return days;
}
```

🔴 Problèmes détectés par un recruteur / Sonar :

- Magic numbers (1, 2)

- Forte complexité cognitive

- Aucun nom métier

- Impossible à tester proprement

- Violation OCP

✅ Version attendue en test technique

```csharp
public interface IRatePolicy
{
    double Apply(int days);
}

public sealed class StandardRatePolicy : IRatePolicy
{
    public double Apply(int days) =>
        days > 10 ? days * 1.1 : days;
}

public sealed class DiscountRatePolicy : IRatePolicy
{
    public double Apply(int days) =>
        days * 0.9;
}
```
```csharp
public sealed class RateCalculator
{
    public double Compute(int days, IRatePolicy policy)
    {
        ArgumentNullException.ThrowIfNull(policy);
        return policy.Apply(days);
    }
}
```

✅ Ce que tu peux expliquer à l’oral :

- Réduction de la complexité

- Polymorphisme au lieu de if/else

- Code extensible

- Testabilité maximale

- Sonar OK

🧪 Test associé (AAA, clair, propre)

```csharp
[Fact]
public void Should_apply_discount_rate()
{
    // Arrange
    var policy = new DiscountRatePolicy();
    var calculator = new RateCalculator();

    // Act
    var result = calculator.Compute(10, policy);

    // Assert
    result.Should().Be(9);
}
```

💡 Très apprécié en entretien :

> “Chaque règle métier a son propre test”


### 2️⃣ README.md GitHub-ready (Test Technique)

- 👉 À mettre à la racine du repo
```csharp
md


# C# Clean Code – Technical Test

This repository demonstrates clean, maintainable, and testable C# code
aligned with **Microsoft C# Coding Conventions** and **SonarQube rules**.

---

## 🎯 Goals
- Readable code
- Low cognitive complexity
- Clear responsibilities
- Full testability
- SonarQube compliant

---

## ✅ Clean Code Principles Applied

### Naming
- PascalCase for public members
- _camelCase for private fields
- Explicit and meaningful names

### Classes
- Single Responsibility Principle
- Small, focused classes
- No God Objects

### Methods
- One responsibility
- Few parameters
- Early return
- No side effects

### Error Handling
- Exceptions instead of return codes
- No empty catch blocks
- Explicit error messages

---

## 🧪 Testing Strategy
- AAA pattern (Arrange / Act / Assert)
- One behavior per test
- No logic in tests
- Domain rules fully covered

---

## 🚦 SonarQube Expectations
- Bugs: 0
- Vulnerabilities: 0
- Code smells: justified
- Cognitive complexity minimized

---

## 📁 Project Structure

src/
├─ Domain/
├─ Application/
├─ Infrastructure/
└─ Tests/

---

## 🧠 Interview Talking Points
- Why the code is easy to read
- How complexity was reduced
- Why polymorphism was chosen
- How tests support refactoring

---

## 📚 References
- Microsoft C# Coding Conventions
- SonarQube Rules for C#
- Clean Code – Robert C. Martin

```

🎯 Conseil final (recruteur mindset)

- Un bon test technique est un code que tu peux défendre à l’oral.
- S’il est clair sans explication → excellent
- S’il est clair avec explication → validé
- S’il faut s’excuser → rejeté 😄
