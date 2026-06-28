# Building Apps with OOP (C# / .NET)

> **Public cible** : développeurs C#/.NET (débutant à intermédiaire) souhaitant solidifier leur pratique de la POO et apprendre à concevoir des applications maintenables.
>
> **Pré-requis** : bases C# (types, conditions, boucles, méthodes), notions de collections et LINQ utiles.
>
> **Durée suggérée** : 1 journée (6–7h) ou 2 demi-journées.
>
> **Objectifs**
>
> - Comprendre et appliquer les piliers de la POO : encapsulation, abstraction, héritage, polymorphisme.
> - Savoir choisir entre **class** et **record**.
> - Maîtriser **properties**, **constructors**, membres **static**, et bonnes pratiques de conception.
> - Construire un mini-domaine (ex: commande / livraison) en code propre, testable et extensible.

---

## Plan de formation

1. **Introduction à la POO dans .NET**
   - Pourquoi la POO ? Maintenabilité, extensibilité, modélisation du domaine
   - Valeur vs identité, comportements vs données
   - Survol : classes, records, propriétés, membres statiques

2. **Classes**
   - Définition, champs, méthodes
   - Modéliser un concept métier (ex: `Customer`, `Order`)
   - Identité, invariants, responsabilité unique

3. **Records**
   - Records : données immuables, égalité structurelle
   - `record` vs `class`
   - `with` expressions, déconstruction, init-only

4. **Properties**
   - Auto-properties, propriétés calculées
   - `get; set;`, `init;`, access modifiers
   - Validation et invariants (setter privé, méthodes de modification)

5. **Encapsulation**
   - Champs privés, API publique minimale
   - Invariants métier
   - Encapsulation via méthodes : éviter le « modèle anémique »

6. **Abstraction**
   - Interfaces, classes abstraites
   - Contrats, séparation des responsabilités
   - Réduction du couplage

7. **Constructors**
   - Constructeurs primaires (si applicable), surcharge, chaînage
   - Guard clauses
   - Injection de dépendances (principe, sans framework dans l’exemple)

8. **Héritage**
   - Quand l’utiliser (et quand éviter)
   - `virtual`/`override`, `sealed`
   - Composition vs héritage

9. **Polymorphisme**
   - Polymorphisme par interface / héritage
   - Pattern matching
   - Open-Closed Principle (OCP)

10. **Static members**
    - Membres statiques : constantes, factories, état partagé
    - Pièges (couplage, testabilité)
    - Patterns : `static factory`, `singleton` (avec prudence)

11. **Atelier fil rouge : Mini application "Order Processing"**
    - Modélisation du domaine
    - Implémentation progressive des concepts
    - Extensions (nouveau type de discount, nouveau mode de shipping)

12. **Récapitulatif & bonnes pratiques**
    - Check-list de conception
    - Erreurs courantes
    - Pistes d’approfondissement

---

## 1) Introduction : pourquoi la POO ?

La Programmation Orientée Objet vise à **modéliser** un domaine via des **objets** qui encapsulent :

- **Données (état)** : ex. un prix, une liste d’articles.
- **Comportements** : ex. calculer un total, appliquer une remise.

Objectifs principaux :

- **Encapsulation** : protéger les invariants.
- **Abstraction** : cacher les détails pour simplifier l’usage.
- **Héritage & polymorphisme** : permettre l’extension sans casser l’existant.

Dans .NET, la POO est omniprésente : `string`, `DateTime`, `HttpClient`, etc.

---

## 2) Classes

### 2.1 Définir une classe

```csharp
public class Customer
{
    public Guid Id { get; }
    public string Name { get; private set; }

    public Customer(Guid id, string name)
    {
        if (id == Guid.Empty) throw new ArgumentException("Id is required", nameof(id));
        if (string.IsNullOrWhiteSpace(name)) throw new ArgumentException("Name is required", nameof(name));

        Id = id;
        Name = name;
    }

    public void Rename(string newName)
    {
        if (string.IsNullOrWhiteSpace(newName))
            throw new ArgumentException("Name is required", nameof(newName));

        Name = newName;
    }
}
```

Points clés :

- **API publique minimale** : `Rename` au lieu d’un `set` public.
- **Invariants** : `Id` jamais vide, `Name` jamais null/whitespace.

### 2.2 Responsabilité

Une classe devrait avoir une responsabilité principale (SRP) :

- `Order` gère la commande.
- `PricingService` calcule un prix total si le calcul est complexe.

---

## 3) Records

### 3.1 Pourquoi des records ?

Les records sont adaptés aux **objets de données** (DTO, messages, résultats) :

- **Égalité structurelle** (par valeur)
- Syntaxe concise
- Immutabilité encouragée

```csharp
public record Money(string Currency, decimal Amount);
```

```csharp
var a = new Money("EUR", 10m);
var b = new Money("EUR", 10m);
Console.WriteLine(a == b); // True (égalité structurelle)
```

### 3.2 `with` expression

```csharp
var m1 = new Money("EUR", 10m);
var m2 = m1 with { Amount = 12m };
```

### 3.3 Records vs classes (règle pratique)

- Utiliser un **record** pour :
  - DTO / messages / résultats
  - Identité non importante
  - Données immuables
- Utiliser une **class** pour :
  - Entités domaine avec identité et cycle de vie
  - Comportements riches
  - Mutation contrôlée

---

## 4) Properties

### 4.1 Auto-properties

```csharp
public class Product
{
    public string Sku { get; }
    public string Name { get; }
    public decimal UnitPrice { get; }

    public Product(string sku, string name, decimal unitPrice)
    {
        Sku = string.IsNullOrWhiteSpace(sku) ? throw new ArgumentException("SKU required") : sku;
        Name = string.IsNullOrWhiteSpace(name) ? throw new ArgumentException("Name required") : name;
        UnitPrice = unitPrice < 0 ? throw new ArgumentOutOfRangeException(nameof(unitPrice)) : unitPrice;
    }
}
```

### 4.2 Propriétés calculées

```csharp
public class OrderLine
{
    public Product Product { get; }
    public int Quantity { get; private set; }

    public decimal LineTotal => Product.UnitPrice * Quantity;

    public OrderLine(Product product, int quantity)
    {
        Product = product ?? throw new ArgumentNullException(nameof(product));
        ChangeQuantity(quantity);
    }

    public void ChangeQuantity(int quantity)
    {
        if (quantity <= 0) throw new ArgumentOutOfRangeException(nameof(quantity), "Quantity must be > 0");
        Quantity = quantity;
    }
}
```

### 4.3 `init` pour immutabilité

```csharp
public record Address
{
    public required string Street { get; init; }
    public required string City { get; init; }
}

var addr = new Address { Street = "1 Main St", City = "Paris" };
```

---

## 5) Encapsulation

Encapsuler = **protéger l’état** et exposer des **opérations** sûres.

### Anti-pattern : setters publics partout

```csharp
public class BadOrder
{
    public decimal Total { get; set; } // peut être modifié n'importe comment
}
```

### Approche encapsulée

```csharp
public class Order
{
    private readonly List<OrderLine> _lines = new();

    public Guid Id { get; }
    public IReadOnlyList<OrderLine> Lines => _lines;

    public Order(Guid id)
    {
        Id = id == Guid.Empty ? throw new ArgumentException("Id required") : id;
    }

    public void AddLine(Product product, int quantity)
    {
        _lines.Add(new OrderLine(product, quantity));
    }

    public decimal Total() => _lines.Sum(l => l.LineTotal);
}
```

- La liste interne est privée.
- L’extérieur ne peut lire que `IReadOnlyList`.
- Les modifications passent par des méthodes.

---

## 6) Abstraction

### 6.1 Interfaces : contrat

On veut pouvoir changer la stratégie de remise sans modifier `Order`.

```csharp
public interface IDiscountPolicy
{
    decimal ComputeDiscount(decimal subtotal);
}

public sealed class NoDiscountPolicy : IDiscountPolicy
{
    public decimal ComputeDiscount(decimal subtotal) => 0m;
}

public sealed class PercentageDiscountPolicy : IDiscountPolicy
{
    private readonly decimal _rate; // ex: 0.10 = -10%

    public PercentageDiscountPolicy(decimal rate)
    {
        if (rate < 0 || rate > 1) throw new ArgumentOutOfRangeException(nameof(rate));
        _rate = rate;
    }

    public decimal ComputeDiscount(decimal subtotal) => subtotal * _rate;
}
```

### 6.2 Classes abstraites

Utile en présence de code partagé + contrat partiel :

```csharp
public abstract class ShippingMethod
{
    public string Name { get; }

    protected ShippingMethod(string name)
        => Name = string.IsNullOrWhiteSpace(name) ? throw new ArgumentException("Name required") : name;

    public abstract decimal ComputeFee(decimal orderTotal);
}

public sealed class StandardShipping : ShippingMethod
{
    public StandardShipping() : base("Standard") { }

    public override decimal ComputeFee(decimal orderTotal) => orderTotal >= 50m ? 0m : 5m;
}

public sealed class ExpressShipping : ShippingMethod
{
    public ExpressShipping() : base("Express") { }

    public override decimal ComputeFee(decimal orderTotal) => 12m;
}
```

---

## 7) Constructeurs

### 7.1 Rôle : garantir un objet valide

Un objet doit sortir du constructeur dans un état cohérent.

- Guard clauses : `ArgumentNullException`, `ArgumentOutOfRangeException`
- Dépendances obligatoires via paramètres

### 7.2 Surcharge et chaînage

```csharp
public class InvoiceNumberGenerator
{
    private readonly string _prefix;

    public InvoiceNumberGenerator() : this("INV") { }

    public InvoiceNumberGenerator(string prefix)
    {
        _prefix = string.IsNullOrWhiteSpace(prefix) ? throw new ArgumentException("Prefix required") : prefix;
    }

    public string Next(int sequence) => $"{_prefix}-{sequence:000000}";
}
```

### 7.3 Injection (sans framework)

```csharp
public sealed class CheckoutService
{
    private readonly IDiscountPolicy _discountPolicy;

    public CheckoutService(IDiscountPolicy discountPolicy)
        => _discountPolicy = discountPolicy ?? throw new ArgumentNullException(nameof(discountPolicy));

    public decimal ComputeTotal(Order order)
    {
        var sub = order.Total();
        var discount = _discountPolicy.ComputeDiscount(sub);
        return sub - discount;
    }
}
```

---

## 8) Héritage

### 8.1 Quand l’utiliser

- Modéliser une relation **"est-un"** stable.
- Partage de comportement commun.

À éviter si :

- Le modèle change souvent.
- Les sous-classes forcent des overrides incohérents.
- On veut juste réutiliser du code → préférer la **composition**.

### 8.2 Exemple

```csharp
public abstract class PaymentMethod
{
    public abstract string Kind { get; }
    public abstract void Validate();
}

public sealed class CardPayment : PaymentMethod
{
    public override string Kind => "Card";
    public string Last4 { get; }

    public CardPayment(string last4)
        => Last4 = last4?.Length == 4 ? last4 : throw new ArgumentException("Last4 must be 4 chars");

    public override void Validate() { /* validation spécifique */ }
}

public sealed class BankTransferPayment : PaymentMethod
{
    public override string Kind => "BankTransfer";
    public string Iban { get; }

    public BankTransferPayment(string iban)
        => Iban = string.IsNullOrWhiteSpace(iban) ? throw new ArgumentException("IBAN required") : iban;

    public override void Validate() { /* validation spécifique */ }
}
```

---

## 9) Polymorphisme

Le polymorphisme permet d’écrire du code qui manipule une abstraction sans connaître l’implémentation.

### 9.1 Via interface

```csharp
public sealed class PricingService
{
    private readonly IDiscountPolicy _discount;

    public PricingService(IDiscountPolicy discount) => _discount = discount;

    public decimal FinalTotal(Order order)
    {
        var subtotal = order.Total();
        return subtotal - _discount.ComputeDiscount(subtotal);
    }
}
```

### 9.2 Via héritage + `virtual/override`

```csharp
public abstract class Discount
{
    public abstract decimal Apply(decimal subtotal);
}

public sealed class FixedDiscount : Discount
{
    private readonly decimal _amount;
    public FixedDiscount(decimal amount) => _amount = amount;

    public override decimal Apply(decimal subtotal) => Math.Min(_amount, subtotal);
}
```

### 9.3 Pattern matching (complément)

```csharp
decimal Fee(ShippingMethod shipping, decimal total) => shipping switch
{
    StandardShipping s => s.ComputeFee(total),
    ExpressShipping e => e.ComputeFee(total),
    _ => throw new NotSupportedException("Unknown shipping")
};
```

---

## 10) Membres `static`

### 10.1 Quand utiliser `static`

- Valeurs constantes / partagées : `const`, `static readonly`
- Méthodes utilitaires pures
- Factories

```csharp
public static class MoneyFactory
{
    public static Money Eur(decimal amount) => new("EUR", amount);
    public static Money Usd(decimal amount) => new("USD", amount);
}
```

### 10.2 Pièges du `static` (état global)

```csharp
public static class BadCounter
{
    public static int Value; // état global → couplage + non deterministic en tests
}
```

Bonnes pratiques :

- Éviter l’état global mutable.
- Favoriser des méthodes statiques **pures** (sans effets de bord).

---

## 11) Atelier fil rouge : Order Processing (mini domaine)

### 11.1 Objectif

Concevoir un mini modèle permettant :

- créer une commande
- ajouter des lignes
- calculer sous-total
- appliquer une politique de remise (extensible)
- calculer des frais de livraison selon méthode

### 11.2 Implémentation complète (version pédagogique)

> Structure proposée :
> - `Domain/` : entités, records
> - `Services/` : pricing/checkout

#### 11.2.1 Modèle

```csharp
public record Money(string Currency, decimal Amount)
{
    public static Money operator +(Money a, Money b)
    {
        if (a.Currency != b.Currency) throw new InvalidOperationException("Currency mismatch");
        return a with { Amount = a.Amount + b.Amount };
    }

    public static Money operator -(Money a, Money b)
    {
        if (a.Currency != b.Currency) throw new InvalidOperationException("Currency mismatch");
        return a with { Amount = a.Amount - b.Amount };
    }
}

public sealed class Product
{
    public string Sku { get; }
    public string Name { get; }
    public Money UnitPrice { get; }

    public Product(string sku, string name, Money unitPrice)
    {
        Sku = string.IsNullOrWhiteSpace(sku) ? throw new ArgumentException("SKU required") : sku;
        Name = string.IsNullOrWhiteSpace(name) ? throw new ArgumentException("Name required") : name;
        UnitPrice = unitPrice;
    }
}

public sealed class OrderLine
{
    public Product Product { get; }
    public int Quantity { get; private set; }

    public Money LineTotal => new(Product.UnitPrice.Currency, Product.UnitPrice.Amount * Quantity);

    public OrderLine(Product product, int quantity)
    {
        Product = product ?? throw new ArgumentNullException(nameof(product));
        ChangeQuantity(quantity);
    }

    public void ChangeQuantity(int quantity)
    {
        if (quantity <= 0) throw new ArgumentOutOfRangeException(nameof(quantity));
        Quantity = quantity;
    }
}

public sealed class Order
{
    private readonly List<OrderLine> _lines = new();

    public Guid Id { get; }
    public IReadOnlyList<OrderLine> Lines => _lines;

    public Order(Guid id)
        => Id = id == Guid.Empty ? throw new ArgumentException("Id required") : id;

    public void AddLine(Product product, int quantity) => _lines.Add(new OrderLine(product, quantity));

    public Money Subtotal()
    {
        if (_lines.Count == 0) return new Money("EUR", 0m);

        var currency = _lines[0].Product.UnitPrice.Currency;
        var total = new Money(currency, 0m);

        foreach (var line in _lines)
        {
            if (line.Product.UnitPrice.Currency != currency)
                throw new InvalidOperationException("All lines must have same currency");

            total += line.LineTotal;
        }

        return total;
    }
}
```

#### 11.2.2 Abstractions

```csharp
public interface IDiscountPolicy
{
    Money ComputeDiscount(Money subtotal);
}

public sealed class NoDiscount : IDiscountPolicy
{
    public Money ComputeDiscount(Money subtotal) => subtotal with { Amount = 0m };
}

public sealed class PercentageDiscount : IDiscountPolicy
{
    private readonly decimal _rate;

    public PercentageDiscount(decimal rate)
    {
        if (rate < 0m || rate > 1m) throw new ArgumentOutOfRangeException(nameof(rate));
        _rate = rate;
    }

    public Money ComputeDiscount(Money subtotal)
        => subtotal with { Amount = subtotal.Amount * _rate };
}

public abstract class ShippingMethod
{
    public abstract string Name { get; }
    public abstract Money ComputeFee(Money subtotal);
}

public sealed class StandardShipping : ShippingMethod
{
    public override string Name => "Standard";

    public override Money ComputeFee(Money subtotal)
        => subtotal.Amount >= 50m ? subtotal with { Amount = 0m } : subtotal with { Amount = 5m };
}

public sealed class ExpressShipping : ShippingMethod
{
    public override string Name => "Express";

    public override Money ComputeFee(Money subtotal)
        => subtotal with { Amount = 12m };
}
```

#### 11.2.3 Service de checkout

```csharp
public sealed class CheckoutService
{
    private readonly IDiscountPolicy _discount;
    private readonly ShippingMethod _shipping;

    public CheckoutService(IDiscountPolicy discount, ShippingMethod shipping)
    {
        _discount = discount ?? throw new ArgumentNullException(nameof(discount));
        _shipping = shipping ?? throw new ArgumentNullException(nameof(shipping));
    }

    public Money TotalToPay(Order order)
    {
        var subtotal = order.Subtotal();
        var discount = _discount.ComputeDiscount(subtotal);
        var shippingFee = _shipping.ComputeFee(subtotal);

        return subtotal - discount + shippingFee;
    }
}
```

#### 11.2.4 Démo d’utilisation

```csharp
var order = new Order(Guid.NewGuid());

var coffee = new Product("COF-001", "Coffee", new Money("EUR", 8m));
var mug = new Product("MUG-001", "Mug", new Money("EUR", 12m));

order.AddLine(coffee, 3);
order.AddLine(mug, 1);

IDiscountPolicy discount = new PercentageDiscount(0.10m);
ShippingMethod shipping = new StandardShipping();

var checkout = new CheckoutService(discount, shipping);
var total = checkout.TotalToPay(order);

Console.WriteLine($"Subtotal: {order.Subtotal().Amount} {order.Subtotal().Currency}");
Console.WriteLine($"Total: {total.Amount} {total.Currency}");
```

### 11.3 Exercices

1. **Encapsulation** : Interdire l’ajout de lignes dupliquées (même SKU) → créer une méthode `AddOrIncrease(Product p, int qty)`.
2. **Abstraction** : Ajouter `ThresholdDiscount` (ex: -5€ si subtotal >= 60€).
3. **Polymorphisme** : Introduire un `ShippingMethod` `PickupPointShipping`.
4. **Static** : Créer une factory `DiscountPolicies.BlackFriday()` sans état global.
5. **Records** : Ajouter un record `EmailAddress` (value object) avec validation.

---

## 12) Récapitulatif & check-list

### 12.1 Check-list de conception orientée objet

- L’objet est-il **toujours valide** après construction ?
- Les invariants sont-ils protégés (pas de setters publics inutiles) ?
- Une propriété est-elle vraiment un **état**, ou un comportement déguisé ?
- Utilise-t-on l’héritage uniquement pour un vrai **"est-un"** ?
- Les dépendances sont-elles exprimées via des **abstractions** ?
- Le `static` est-il justifié (pas d’état global mutable) ?

### 12.2 Erreurs fréquentes

- Confondre DTO (records) et entités domaine (classes riches).
- Exposer des collections modifiables (`List<T>`) directement.
- Utiliser l’héritage pour partager du code au lieu de composition.
- `static` partout → difficile à tester.

### 12.3 Pour aller plus loin

- SOLID (SRP, OCP, LSP, ISP, DIP)
- Domain-Driven Design (Entities, Value Objects, Aggregates)
- Tests unitaires (xUnit), mocking
- Patterns : Strategy, Factory, Decorator

---

## Annexes : mini glossaire

- **Encapsulation** : cacher l’état et exposer des opérations contrôlées.
- **Abstraction** : travailler avec un contrat, pas une implémentation.
- **Héritage** : spécialisation d’un type parent.
- **Polymorphisme** : le code appelle un contrat, et le bon comportement s’exécute selon le type concret.
- **Record** : type orienté données, égalité par valeur, immutabilité encouragée.
