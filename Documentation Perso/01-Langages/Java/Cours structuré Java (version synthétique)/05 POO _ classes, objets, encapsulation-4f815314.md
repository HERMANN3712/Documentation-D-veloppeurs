# 5) POO : classes, objets, encapsulation (Java)

> Formation destinée à des développeurs ayant déjà une pratique de la programmation (ici profil .NET/C#), souhaitant maîtriser la Programmation Orientée Objet en **Java** avec les bonnes pratiques.

---

## Objectifs pédagogiques

À l’issue de ce module, vous serez capable de :

- Expliquer les **concepts fondamentaux** de la POO : **encapsulation**, **héritage**, **polymorphisme**.
- Concevoir et implémenter des **classes** et **objets** en Java.
- Utiliser correctement les **visibilités** (`public`, `protected`, *package-private*, `private`).
- Différencier **membre d’instance** et **membre de classe** (`static`).
- Employer `this` et `super` à bon escient.
- Appliquer des **bonnes pratiques** : champs en `private`, invariants, immutabilité partielle, **composition plutôt qu’héritage** quand c’est pertinent.

---

## Pré-requis

- Savoir écrire des classes et méthodes simples.
- Notions de base de types, variables, méthodes, exceptions.

---

## Plan de la formation

1. [Rappels et vocabulaire : classe, objet, instance](#1-rappels-et-vocabulaire--classe-objet-instance)
2. [Encapsulation : protéger l’état, exposer un comportement](#2-encapsulation--protéger-létat-exposer-un-comportement)
3. [Visibilités (modificateurs d’accès)](#3-visibilités-modificateurs-daccès)
4. [`static` : membres de classe vs membres d’instance](#4-static--membres-de-classe-vs-membres-dinstance)
5. [`this` et `super`](#5-this-et-super)
6. [Héritage : quand et comment](#6-héritage--quand-et-comment)
7. [Polymorphisme : substitution, override, dispatch dynamique](#7-polymorphisme--substitution-override-dispatch-dynamique)
8. [Bonnes pratiques de conception : composition > héritage](#8-bonnes-pratiques-de-conception--composition--héritage)
9. [Atelier fil rouge (mini-cas)](#9-atelier-fil-rouge-mini-cas)
10. [Quiz / points de contrôle](#10-quiz--points-de-contrôle)

---

## 1) Rappels et vocabulaire : classe, objet, instance

### Classe
Une **classe** est un plan (template) décrivant :

- un **état** (champs / attributs)
- des **comportements** (méthodes)

```java
public class User {
    private String name;

    public void rename(String newName) {
        this.name = newName;
    }
}
```

### Objet / instance
Une **instance** (un objet) est un exemplaire concret créé à partir de la classe.

```java
User u1 = new User();
User u2 = new User();
```

- `u1` et `u2` sont deux objets distincts.
- chacun possède son propre état (ses propres champs).

### Référence
En Java, une variable d’objet stocke une **référence** (comme en C#). L’objet lui-même vit sur le tas (heap).

---

## 2) Encapsulation : protéger l’état, exposer un comportement

### Définition
**Encapsuler**, c’est :

1. **cacher** la représentation interne (champs)
2. **contrôler** l’accès/les modifications via une API (méthodes)
3. garantir des **invariants** (règles métier) durables

### Pourquoi ?
Sans encapsulation, n’importe quel code peut mettre l’objet dans un état invalide.

#### Mauvais exemple (données publiques)
```java
public class BankAccount {
    public double balance;
}

BankAccount acc = new BankAccount();
acc.balance = -10_000; // état potentiellement invalide
```

#### Bon exemple (état privé + méthodes)
```java
public class BankAccount {
    private double balance;

    public double getBalance() {
        return balance;
    }

    public void deposit(double amount) {
        if (amount <= 0) {
            throw new IllegalArgumentException("amount must be > 0");
        }
        balance += amount;
    }

    public void withdraw(double amount) {
        if (amount <= 0) {
            throw new IllegalArgumentException("amount must be > 0");
        }
        if (amount > balance) {
            throw new IllegalStateException("insufficient funds");
        }
        balance -= amount;
    }
}
```

### Invariants
Un **invariant** est une condition vraie avant/après chaque appel public.

Exemple : `balance >= 0`.

### Getters/Setters : pas systématique
Un piège courant : générer automatiquement getters/setters et perdre l’encapsulation.

- Préférez des méthodes métier : `rename(...)`, `changeAddress(...)`, `activate()`, etc.
- N’exposez un setter que si vous avez un vrai besoin.

### Encapsulation et immutabilité
Parfois, on choisit de rendre un objet **immutable** : son état ne change plus après construction.

```java
public final class Money {
    private final String currency;
    private final long cents;

    public Money(String currency, long cents) {
        if (cents < 0) throw new IllegalArgumentException();
        this.currency = currency;
        this.cents = cents;
    }

    public String currency() { return currency; }
    public long cents() { return cents; }
}
```

---

## 3) Visibilités (modificateurs d’accès)

Java propose 4 niveaux principaux :

| Modificateur | Même classe | Même package | Sous-classe | Partout |
|---|---:|---:|---:|---:|
| `private` | ✅ | ❌ | ❌ | ❌ |
| *(package-private)* (aucun mot-clé) | ✅ | ✅ | ❌* | ❌ |
| `protected` | ✅ | ✅ | ✅ | ❌ |
| `public` | ✅ | ✅ | ✅ | ✅ |

> 
> **Note** : `protected` donne accès aux sous-classes (même dans un autre package) **via l’héritage**. Le package-private ne donne pas automatiquement accès aux sous-classes hors package.

### 3.1 `private`
- Visibilité la plus restrictive.
- Utilisée pour **protéger l’état**.

```java
private int age;
```

### 3.2 *Package-private* (default)
- Aucun mot-clé.
- Visibilité limitée au **package**.
- Très utile pour *cacher* des classes/services internes à un module.

```java
class InternalParser {
    // visible uniquement dans le package
}
```

### 3.3 `protected`
- Accès dans le package **et** dans les sous-classes.
- Utile pour concevoir des classes destinées à être étendues.

```java
protected void onStarted() { }
```

### 3.4 `public`
- API exposée aux autres packages.
- Tout ce qui est `public` est un contrat : il faut stabiliser et documenter.

---

## 4) `static` : membres de classe vs membres d’instance

### 4.1 Membre d’instance
Un champ/méthode **non static** appartient à l’instance.

```java
public class Counter {
    private int value;

    public void increment() {
        value++;
    }
}
```

### 4.2 Membre de classe (`static`)
Un membre `static` appartient à la **classe** (unique, partagé), pas aux objets.

```java
public class Counter {
    private static int totalIncrements = 0;

    public static int getTotalIncrements() {
        return totalIncrements;
    }
}
```

### 4.3 Points d’attention
- `static` introduit un **état global** (souvent source de couplage).
- À utiliser pour :
  - constantes (`public static final`)
  - fonctions utilitaires stateless
  - factory methods (`of`, `from`, `create`)

#### Constantes
```java
public class HttpStatus {
    public static final int OK = 200;
}
```

#### Méthode statique de fabrique
```java
public final class UserId {
    private final String value;

    private UserId(String value) {
        this.value = value;
    }

    public static UserId of(String value) {
        if (value == null || value.isBlank()) throw new IllegalArgumentException();
        return new UserId(value);
    }
}
```

---

## 5) `this` et `super`

### 5.1 `this`
`this` est une référence vers **l’instance courante**.

Usages fréquents :

1) Désambiguïser un champ vs paramètre
```java
public class User {
    private String name;

    public void rename(String name) {
        this.name = name;
    }
}
```

2) Chaînage de constructeurs
```java
public class Point {
    private final int x;
    private final int y;

    public Point(int x, int y) {
        this.x = x;
        this.y = y;
    }

    public Point(int x) {
        this(x, 0); // appelle l’autre constructeur
    }
}
```

### 5.2 `super`
`super` fait référence à la **classe parente**.

Usages :

1) Appeler un constructeur parent
```java
public class Animal {
    protected final String name;

    public Animal(String name) {
        this.name = name;
    }
}

public class Dog extends Animal {
    public Dog(String name) {
        super(name);
    }
}
```

2) Appeler une méthode parent redéfinie
```java
public class BaseService {
    public void start() {
        System.out.println("Base start");
    }
}

public class ChildService extends BaseService {
    @Override
    public void start() {
        super.start();
        System.out.println("Child start");
    }
}
```

---

## 6) Héritage : quand et comment

### Définition
L’**héritage** permet de créer une classe *spécialisée* à partir d’une classe plus générale :

```java
public class Vehicle {
    public void move() { }
}

public class Car extends Vehicle {
    public void openTrunk() { }
}
```

- `Car` hérite des membres (selon visibilité) de `Vehicle`.
- `Car` peut **ajouter** des comportements.
- `Car` peut **redéfinir** (override) certains comportements.

### `extends` et héritage simple
Java n’autorise qu’un seul parent : héritage simple.

### `final`
- Une classe `final` ne peut pas être étendue.
- Une méthode `final` ne peut pas être redéfinie.

```java
public final class ImmutableConfig { }
```

### Risques de l’héritage
- Couplage fort parent/enfant.
- Fragilité : une modification du parent peut casser les enfants.
- Hiérarchies profondes difficiles à maintenir.

### Tester la relation « est-un » (is-a)
L’héritage est pertinent si la relation *is-a* est vraie et stable.

- `Car is a Vehicle` : plausible.
- `Square is a Rectangle` : souvent discuté car contraintes d’invariants.

---

## 7) Polymorphisme : substitution, override, dispatch dynamique

### Définition
Le **polymorphisme** permet d’utiliser un objet via un type plus général (souvent une super-classe ou une interface), tout en exécutant le comportement concret.

```java
Vehicle v = new Car();
v.move(); // exécute la version de Car si redéfinie
```

### Redéfinition (`@Override`)
Toujours utiliser `@Override` :
- sécurité (détecte erreurs de signature)
- lisibilité

```java
public class Vehicle {
    public void move() {
        System.out.println("Vehicle moves");
    }
}

public class Car extends Vehicle {
    @Override
    public void move() {
        System.out.println("Car drives");
    }
}
```

### Substitution (principe de Liskov, intuition)
Si `Car` étend `Vehicle`, alors un `Car` doit pouvoir être utilisé partout où un `Vehicle` est attendu **sans casser** le programme.

Exemple : si `Vehicle.move()` promet de déplacer l’objet, `Car.move()` doit aussi le faire (pas jeter une exception arbitraire, pas violer le contrat).

### Polymorphisme et collections
```java
List<Vehicle> fleet = List.of(new Car(), new Vehicle());
for (Vehicle v : fleet) {
    v.move();
}
```

---

## 8) Bonnes pratiques de conception : composition > héritage

### 8.1 Champs `private` par défaut
- Les champs devraient être `private`.
- On expose des méthodes, pas les données.

```java
public class Order {
    private final List<OrderLine> lines = new ArrayList<>();

    public void addLine(OrderLine line) {
        if (line == null) throw new IllegalArgumentException();
        lines.add(line);
    }

    public List<OrderLine> getLines() {
        return List.copyOf(lines); // protège l’encapsulation
    }
}
```

### 8.2 Préserver l’encapsulation des collections
Ne jamais exposer directement une liste modifiable interne.

- ✅ `List.copyOf(...)`
- ✅ `Collections.unmodifiableList(...)`

### 8.3 Favoriser la composition
**Composition** : une classe contient une autre classe (relation « a-un » / has-a).

Exemple : plutôt que `Car extends Engine`, on fait `Car has an Engine`.

```java
public class Engine {
    public void start() { }
}

public class Car {
    private final Engine engine;

    public Car(Engine engine) {
        this.engine = engine;
    }

    public void start() {
        engine.start();
    }
}
```

#### Pourquoi la composition est souvent meilleure ?
- moins de couplage
- plus flexible (on peut remplacer `Engine`)
- plus testable (injection de dépendances, mocks)

### 8.4 Héritage quand il exprime une abstraction stable
Utilisez l’héritage quand :
- vous contrôlez la hiérarchie
- le contrat est clair
- la spécialisation ne risque pas de violer les invariants

> En Java, on privilégie souvent un modèle : **interfaces + composition**.

---

## 9) Atelier fil rouge (mini-cas)

### Énoncé
Concevoir un mini-modèle : **Bibliothèque**.

- Une `Book` possède : `isbn`, `title`, `available`.
- Une `Member` possède : `id`, `name`.
- Une `Library` gère les emprunts.

Contraintes :
- Encapsulation stricte (champs `private`).
- Pas de setter inutile.
- API métier : `borrow(isbn, memberId)`, `returnBook(isbn)`.

### Proposition d’implémentation (simplifiée)

```java
public final class Book {
    private final String isbn;
    private final String title;
    private boolean available = true;

    public Book(String isbn, String title) {
        if (isbn == null || isbn.isBlank()) throw new IllegalArgumentException("isbn");
        if (title == null || title.isBlank()) throw new IllegalArgumentException("title");
        this.isbn = isbn;
        this.title = title;
    }

    public String getIsbn() { return isbn; }
    public String getTitle() { return title; }
    public boolean isAvailable() { return available; }

    void markBorrowed() {
        if (!available) throw new IllegalStateException("Already borrowed");
        available = false;
    }

    void markReturned() {
        if (available) throw new IllegalStateException("Not borrowed");
        available = true;
    }
}
```

> Note : `markBorrowed/markReturned` sont **package-private** : elles font partie du modèle mais ne sont pas exposées comme API publique au monde entier.

```java
public final class Member {
    private final String id;
    private final String name;

    public Member(String id, String name) {
        if (id == null || id.isBlank()) throw new IllegalArgumentException("id");
        if (name == null || name.isBlank()) throw new IllegalArgumentException("name");
        this.id = id;
        this.name = name;
    }

    public String getId() { return id; }
    public String getName() { return name; }
}
```

```java
public class Library {
    private final Map<String, Book> booksByIsbn = new HashMap<>();
    private final Map<String, Member> membersById = new HashMap<>();

    public void addBook(Book book) {
        if (book == null) throw new IllegalArgumentException("book");
        booksByIsbn.put(book.getIsbn(), book);
    }

    public void register(Member member) {
        if (member == null) throw new IllegalArgumentException("member");
        membersById.put(member.getId(), member);
    }

    public void borrow(String isbn, String memberId) {
        Book book = requireBook(isbn);
        requireMember(memberId);
        book.markBorrowed();
    }

    public void returnBook(String isbn) {
        Book book = requireBook(isbn);
        book.markReturned();
    }

    private Book requireBook(String isbn) {
        Book b = booksByIsbn.get(isbn);
        if (b == null) throw new IllegalArgumentException("Unknown isbn: " + isbn);
        return b;
    }

    private Member requireMember(String id) {
        Member m = membersById.get(id);
        if (m == null) throw new IllegalArgumentException("Unknown memberId: " + id);
        return m;
    }
}
```

### Questions d’analyse
- Où place-t-on les invariants ? (réponse : au plus près des données)
- Pourquoi `Book` n’expose pas `setAvailable(boolean)` ?
- Quel est l’intérêt du package-private sur `markBorrowed()` ?

---

## 10) Quiz / points de contrôle

1. Pourquoi dit-on que `public` est un contrat ?
2. Quelle est la différence entre *package-private* et `protected` ?
3. À quoi sert `static` et quels risques introduit-il ?
4. Dans quel cas `super(...)` est-il obligatoire dans un constructeur ?
5. Donnez un exemple où la composition est préférable à l’héritage.

---

## Synthèse

- **Encapsulation** : champs `private`, méthodes métier, invariants, éviter setters automatiques.
- **Visibilités** : choisir le niveau minimal nécessaire, exploiter le package-private pour l’interne.
- **`static`** : utile pour constantes/utilitaires/factories, prudence avec l’état global.
- **`this/super`** : clarifier l’instance courante et interagir avec la super-classe.
- **Héritage/Polymorphisme** : puissants mais à utiliser avec des contrats stables.
- **Best practices** : **composition > héritage** quand on veut de la flexibilité.
