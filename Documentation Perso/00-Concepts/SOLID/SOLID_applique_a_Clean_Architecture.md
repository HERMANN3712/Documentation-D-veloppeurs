# SOLID appliqué à Clean Architecture

## Introduction

Clean Architecture (popularisée par Robert C. Martin) sépare une
application en couches :

-   Domain
-   Application
-   Infrastructure
-   Presentation

Les principes SOLID permettent de structurer proprement ces couches et
de garantir maintenabilité, évolutivité et testabilité.

------------------------------------------------------------------------

## 1. SRP --- Single Responsibility Principle

Une classe ne doit avoir qu'une seule responsabilité.

### Dans le Domain

Une entité contient uniquement la logique métier.

❌ Mauvais exemple :

``` csharp
class Order {
    public void Save() { } // logique base de données ❌
}
```

✅ Bon exemple :

``` csharp
class Order {
    public void MarkAsShipped() { } // règle métier ✔
}
```

La persistance appartient à Infrastructure.

------------------------------------------------------------------------

## 2. OCP --- Open/Closed Principle

Une classe doit être ouverte à l'extension mais fermée à la
modification.

Exemple avec stratégie :

``` csharp
interface IDiscountStrategy
{
    decimal Apply(Order order);
}
```

On ajoute de nouvelles stratégies sans modifier la classe Order.

------------------------------------------------------------------------

## 3. LSP --- Liskov Substitution Principle

Une implémentation doit pouvoir remplacer une autre sans casser le
comportement.

Exemple :

``` csharp
IOrderRepository
```

Implémentations possibles : - EfOrderRepository -
DapperOrderRepository - InMemoryOrderRepository

L'Application ne doit pas dépendre d'une implémentation concrète.

------------------------------------------------------------------------

## 4. ISP --- Interface Segregation Principle

Préférer plusieurs interfaces spécifiques plutôt qu'une interface
générale.

❌ Mauvais :

``` csharp
IRepository {
    Add();
    Update();
    Delete();
    GetAll();
}
```

✅ Mieux :

``` csharp
IOrderReader
IOrderWriter
```

Compatible avec CQRS.

------------------------------------------------------------------------

## 5. DIP --- Dependency Inversion Principle

Les modules de haut niveau ne doivent pas dépendre des modules de bas
niveau mais d'abstractions.

### Dans Clean Architecture

Le Domain définit :

``` csharp
public interface IOrderRepository;
```

Infrastructure implémente :

``` csharp
public class EfOrderRepository : IOrderRepository;
```

Le cœur métier ne dépend pas d'Entity Framework ni d'une base de
données.

------------------------------------------------------------------------

# Comment expliquer SOLID en entretien (court et efficace)

SOLID est un ensemble de 5 principes de conception orientée objet
permettant d'écrire un code maintenable, évolutif et testable.\
L'objectif principal est de réduire le couplage, favoriser les
abstractions et faciliter les évolutions métier sans casser l'existant.

Version impactante :\
\> SOLID aide à écrire un code qui change facilement quand le métier
évolue.

------------------------------------------------------------------------

# Pièges classiques sur SOLID

## 1. Sur‑engineering

Trop d'abstraction, trop d'interfaces, complexité inutile.

## 2. Mauvaise compréhension du SRP

SRP signifie une seule responsabilité métier, pas une seule méthode.

## 3. OCP mal appliqué

Utiliser uniquement l'héritage au lieu de privilégier la composition et
les stratégies.

## 4. Violation du LSP

Une classe enfant modifie le comportement attendu ou lance des
exceptions inattendues.

## 5. DIP mal compris

Injecter une interface ne suffit pas ; il faut dépendre d'abstractions
réellement stables.

------------------------------------------------------------------------

## Résumé

-   SOLID améliore la qualité locale des classes.
-   Clean Architecture organise la structure globale.
-   DIP est le lien principal entre SOLID et Clean Architecture.
