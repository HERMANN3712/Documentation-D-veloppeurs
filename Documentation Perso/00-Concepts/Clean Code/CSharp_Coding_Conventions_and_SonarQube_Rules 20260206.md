# Microsoft C# Coding Conventions & SonarQube Rules (C#)

## 🧩 Microsoft C# Coding Conventions

👉 Objectif : lisibilité, cohérence, maintenabilité code

👉 Portée : style, structure, bonnes pratiques de base

---

### 1️⃣ 1. Nommage
- **PascalCase** : classes, méthodes, propriétés, namespaces
- **camelCase** : variables locales, paramètres
- **_camelCase** : champs privés
- Interfaces préfixées par **I**
- Éviter les abréviations ambiguës / sauf très connues (Id, Xml)

```
public class UserService
{
    private readonly IRepository _repository;
}
```
🎯 Pourquoi : lecture immédiate + conventions universelles .NET

---

### 2️⃣ 2. Organisation du code
- Un fichier = une classe (sauf exceptions justifiées)
- Ordre recommandé :
  1. Champs
  2. Constructeurs
  3. Propriétés
  4. Méthodes publiques
  5. Méthodes privées

---

### 3️⃣ 3. Mise en forme
- Indentation : **4 espaces**
- Accolades toujours explicites
- Une instruction par ligne
- Code aéré et lisible

```
if (isValid)
{
    Process();
}
```
🎯 Le formatage est un **outil de communication**

---

### 4️⃣ 4. Types & langage
- `var` si le type est évident
- Types explicites si ambigu
- Favoriser l’immutabilité (`readonly`, `record`)
- Utiliser `using` pour la gestion des ressources

---

### 5️⃣ 5. Commentaires & documentation
- Éviter les commentaires évidents
- XML comments pour les API publiques
- Expliquer le **pourquoi**, pas le **comment**

---

</br></br>
## 🚦SonarQube – Règles C# (qualité & sécurité)

### 🎯 Objectif
Détecter bugs, dettes techniques et vulnérabilités via analyse statique.

### 🐞 1. Bugs
- NullReference potentiels
- Exceptions non gérées
- Conditions inutiles / Conditions toujours vraies/fausses
- Code mort

```
if (obj != null)
{
    obj.Do();
}
```
---

### 🔐 2. Sécurité
- Données sensibles en clair
- Exceptions trop génériques
- Failles d’injection
- Algorithmes cryptographiques faibles
🚨 Très surveillé en contexte entreprise / audit

---

### 🧼 3. Code Smells
- Méthodes trop longues
- Trop de paramètres
- Classes trop complexes
- Duplication de code
- Conditions imbriquées

```
// Sonar alerte souvent > 7 paramètres
void Process(a, b, c, d, e, f, g, h)
```
🎯 Indique où **refactorer**, pas forcément une erreur

---

### 🧠 4. Complexité
- Surveillance de la **Cognitive Complexity**
- Trop de `if/else/switch`
- Préférer polymorphisme et early return

---

### 🧪 5. Tests
- Tests sans assertion
- Tests ignorés
- Duplication de tests
- Couverture insuffisante

---

## 🧩 Microsoft vs SonarQube

| Microsoft | SonarQube |
|---------|-----------|
| Conventions humaines | Analyse automatisée |
| Lisibilité | Qualité & sécurité |
| Guide | Gardien |
| Comment écrire | Comment éviter les erreurs |

---

## 🎯 Bonnes pratiques en entreprise

> - Activer StyleCop / EditorConfig</br>
> - Brancher SonarQube dans le CI</br>
> - Zéro bug / zéro vulnérabilité avant merge</br>
> - Ne pas corriger tout : prioriser</br>

---

## 🧠 Conclusion
Clean Code = **Conventions humaines (Microsoft)** + **Contrôles automatisés (SonarQube)**.
