# Formation 6 — Interfaces, abstraction, records (Java)

> **Public cible** : développeurs débutants/intermédiaires Java (ou venant de .NET/C#) souhaitant structurer un design OO propre.
>
> **Pré-requis** : classes, héritage, encapsulation, exceptions, collections, notions de SOLID.
>
> **Version Java** : exemples compatibles **Java 17+** (records disponibles depuis Java 16).

---

## 1. Objectifs pédagogiques

À la fin de ce module, vous saurez :

- Définir et utiliser une **interface** comme **contrat** (API) indépendant de l’implémentation.
- Choisir entre **interface** et **classe abstraite** selon le besoin (partage de comportement, état, extension, composition).
- Concevoir des modèles de données (DTO) avec des **records** (immutables, compacts, sûrs).
- Appliquer ces concepts dans un mini-cas de conception (notifications, services, DTOs) en suivant de bonnes pratiques.

---

## 2. Plan de la formation

1. **Interfaces**
   1. Définition, rôle de contrat
   2. Implémentation et polymorphisme
   3. Méthodes `default`, `static`, privées
   4. Interfaces fonctionnelles et lambda (rappel)
   5. Bonnes pratiques

2. **Classe abstraite**
   1. Définition et objectifs
   2. État + comportements partagés
   3. Méthodes abstraites vs concrètes
   4. Template Method (pattern)
   5. Bonnes pratiques

3. **Interface vs classe abstraite**
   1. Grille de décision
   2. Pièges usuels
   3. Composition vs héritage

4. **Records (Java 16+)**
   1. Définition : data class immutable
   2. Génération automatique (constructeur, accessors, `equals/hashCode/toString`)
   3. Validation, constructeur compact
   4. Personnalisation et limites
   5. Records pour DTO et interop JSON

5. **Atelier / Mise en pratique**
   1. API de notification via interface
   2. Mutualisation via classe abstraite
   3. DTO en record
   4. Tests rapides et design review

---

# 3. Interfaces

## 3.1. Définition : une interface = un contrat

Une **interface** décrit *ce qu’un type sait faire*, sans imposer *comment il le fait*.

- Une interface déclare des **méthodes** (signatures), éventuellement des `default`.
- Une classe **implémente** une ou plusieurs interfaces.
- On programme contre l’interface (abstraction), pas contre l’implémentation.

### Exemple minimal (contrat)

```java
public interface Notifier {
    void send(String msg);
}
```

Ici, `Notifier` garantit qu’un objet sait **envoyer** un message.

---

## 3.2. Implémentation et polymorphisme

### Implémentations

```java
public class EmailNotifier implements Notifier {
    @Override
    public void send(String msg) {
        System.out.println("[EMAIL] " + msg);
    }
}

public class SmsNotifier implements Notifier {
    @Override
    public void send(String msg) {
        System.out.println("[SMS] " + msg);
    }
}
```

### Consommation (injection d’une interface)

```java
public class AlertService {
    private final Notifier notifier;

    public AlertService(Notifier notifier) {
        this.notifier = notifier;
    }

    public void alert(String message) {
        notifier.send(message);
    }
}
```

### Usage

```java
Notifier notifier = new EmailNotifier();
AlertService service = new AlertService(notifier);
service.alert("Incident détecté");
```

**Bénéfice** : `AlertService` ne dépend pas de `EmailNotifier` ou `SmsNotifier` mais du **contrat** `Notifier`.

---

## 3.3. Méthodes `default`, `static`, et privées

### `default` : fournir un comportement par défaut

```java
public interface Notifier {
    void send(String msg);

    default void sendWithPrefix(String prefix, String msg) {
        send(prefix + ": " + msg);
    }
}
```

- Utile pour faire évoluer une interface sans casser les implémentations.
- Attention : trop de logique dans une interface peut devenir confus.

### `static` : utilitaires liés à l’interface

```java
public interface Notifier {
    void send(String msg);

    static String normalize(String msg) {
        return msg == null ? "" : msg.trim();
    }
}
```

### `private` (Java 9+) : factorisation interne

```java
public interface Notifier {
    void send(String msg);

    default void sendSafe(String msg) {
        send(normalizeInternal(msg));
    }

    private String normalizeInternal(String msg) {
        return msg == null ? "" : msg.trim();
    }
}
```

---

## 3.4. Interfaces fonctionnelles (rappel)

Une **interface fonctionnelle** = une interface avec **une seule méthode abstraite**. Ex : `Runnable`, `Supplier<T>`, `Function<T,R>`.

```java
@FunctionalInterface
public interface Notifier {
    void send(String msg);
}

Notifier console = msg -> System.out.println("[CONSOLE] " + msg);
console.send("Hello");
```

> `@FunctionalInterface` n’est pas obligatoire, mais sécurise la contrainte à la compilation.

---

## 3.5. Bonnes pratiques (interfaces)

- **Favoriser des interfaces petites** (ISP — *Interface Segregation Principle*).
- Nommer selon rôle : `Notifier`, `Repository`, `Validator`, `Clock`.
- Éviter les « interfaces fourre-tout ».
- Préférer **composition** (avoir un `Notifier`) plutôt qu’un héritage inutile.

---

# 4. Classe abstraite

## 4.1. Définition

Une **classe abstraite** est une classe **non instanciable** qui peut contenir :

- **État** (attributs)
- **Constructeur**
- Méthodes **concrètes** (implémentées)
- Méthodes **abstraites** (à implémenter dans les sous-classes)

Elle sert à partager de la logique et/ou un état commun.

---

## 4.2. État + comportements partagés

Exemple : plusieurs notificateurs partagent des validations, un format, une journalisation.

```java
import java.time.Clock;
import java.time.Instant;

public abstract class AbstractNotifier {
    protected final Clock clock;

    protected AbstractNotifier(Clock clock) {
        this.clock = clock;
    }

    public final void send(String msg) {
        String normalized = normalize(msg);
        String enriched = "[" + Instant.now(clock) + "] " + normalized;
        doSend(enriched);
    }

    protected abstract void doSend(String msg);

    protected String normalize(String msg) {
        return msg == null ? "" : msg.trim();
    }
}
```

- `send` est `final` : on impose l’algorithme.
- `doSend` est abstraite : chaque sous-classe gère le transport.

### Implémentations

```java
import java.time.Clock;

public class EmailNotifier extends AbstractNotifier {
    public EmailNotifier(Clock clock) {
        super(clock);
    }

    @Override
    protected void doSend(String msg) {
        System.out.println("[EMAIL] " + msg);
    }
}

public class SmsNotifier extends AbstractNotifier {
    public SmsNotifier(Clock clock) {
        super(clock);
    }

    @Override
    protected void doSend(String msg) {
        System.out.println("[SMS] " + msg);
    }
}
```

---

## 4.3. Méthodes abstraites vs concrètes

- Méthode **abstraite** : signature sans corps → obligation d'implémentation.
- Méthode **concrète** : comportement partagé, réutilisable.

### Exemple

```java
public abstract class Validator {
    public final boolean isValid(String value) {
        return value != null && validate(value);
    }

    protected abstract boolean validate(String value);
}
```

---

## 4.4. Pattern Template Method

Le pattern **Template Method** consiste à :

- définir un **squelette d’algorithme** dans la classe abstraite
- déléguer certaines étapes à des méthodes surchargées (abstraites/concrètes)

Dans `AbstractNotifier`, `send()` est le template, `doSend()` est l’étape variable.

---

## 4.5. Bonnes pratiques (classes abstraites)

- N’utiliser l’héritage que si la relation est un vrai **"est-un"** (*is-a*).
- Conserver une classe abstraite **cohérente** : un seul domaine de responsabilité.
- Attention au couplage : l’héritage fige des choix.
- Préférer des champs `protected` avec parcimonie (encapsulation). Souvent mieux : `private` + `protected` getters.

---

# 5. Interface vs Classe abstraite

## 5.1. Différences essentielles

| Sujet | Interface | Classe abstraite |
|---|---|---|
| Rôle | Contrat | Factorisation d’état/implémentation |
| Héritage multiple | Oui (une classe peut implémenter plusieurs interfaces) | Non (une classe ne peut étendre qu’une seule classe) |
| État (champs) | Pas d’état d’instance (seulement `public static final`) | Oui |
| Constructeurs | Non | Oui |
| Évolution | `default` aide à ajouter des méthodes | Ajout de méthodes souvent moins risqué (mais impact sur sous-classes) |

---

## 5.2. Grille de décision

Choisir **une interface** quand :

- vous exposez une **API** (contrat) à différentes implémentations
- vous voulez permettre des implémentations multiples (plugins)
- vous voulez favoriser test/mocking

Choisir **une classe abstraite** quand :

- vous avez un **comportement commun** non trivial
- vous avez un **état** commun (dépendances partagées, cache, config)
- vous souhaitez imposer un **template** (ordre des opérations)

> Règle pratique : **interface d’abord**, classe abstraite ensuite si vous avez de la vraie factorisation.

---

## 5.3. Composition vs héritage

Le design moderne privilégie souvent :

- **composition** : `Service` *a* un `Notifier`
- plutôt que héritage : `Service` *est* un `Notifier` (souvent faux)

Exemple composition :

```java
public class AuditNotifier implements Notifier {
    private final Notifier inner;

    public AuditNotifier(Notifier inner) {
        this.inner = inner;
    }

    @Override
    public void send(String msg) {
        System.out.println("AUDIT: sending '" + msg + "'");
        inner.send(msg);
    }
}
```

Ici on ajoute un comportement **sans** modifier les classes existantes (principe Open/Closed).

---

# 6. Records (Java 16+)

## 6.1. Définition : une “data class” immutable

Un **record** est idéal pour modéliser un objet immutable centré sur ses données.

Exemple :

```java
public record User(String id, String email) {}
```

Ce code génère automatiquement :

- un **constructeur canonique** `new User(String id, String email)`
- des **accessors** `id()` et `email()`
- `equals`, `hashCode`, `toString`

> On l’utilise fréquemment comme **DTO** : données de transport entre couches (API ↔ service ↔ persistence).

---

## 6.2. Utilisation

```java
User u = new User("42", "dev@example.com");
System.out.println(u.id());
System.out.println(u);
```

- Les champs sont `private final`.
- Pas de setters : l’état ne change pas.

---

## 6.3. Validation : constructeur compact

Vous pouvez valider/normaliser les composants.

```java
public record User(String id, String email) {
    public User {
        if (id == null || id.isBlank()) {
            throw new IllegalArgumentException("id is required");
        }
        if (email == null || !email.contains("@")) {
            throw new IllegalArgumentException("email is invalid");
        }
        email = email.trim().toLowerCase();
    }
}
```

- `public User { ... }` = **constructeur compact**.
- Le compilateur injecte l’affectation des paramètres vers les champs après votre code.

---

## 6.4. Ajouter des méthodes

Un record peut contenir des méthodes :

```java
public record Money(long cents, String currency) {
    public Money {
        if (cents < 0) throw new IllegalArgumentException("cents must be >= 0");
        if (currency == null || currency.isBlank()) throw new IllegalArgumentException("currency required");
    }

    public String display() {
        return String.format("%s %.2f", currency, cents / 100.0);
    }
}
```

---

## 6.5. Limites et points d’attention

- Un record est **final** : impossible d’en hériter.
- Les champs sont *par conception* immutables (références finales) ; mais si un champ est une collection mutable, l’immuabilité n’est pas « profonde ».

Exemple piège :

```java
public record Group(String name, java.util.List<String> members) {}
```

`members` peut être modifiée si on passe une liste mutable.

Solution : copier en immuable (défensive) :

```java
import java.util.List;

public record Group(String name, List<String> members) {
    public Group {
        members = List.copyOf(members);
    }
}
```

---

## 6.6. Records et DTO (cas d’usage)

### DTO API entrant

```java
public record CreateUserRequest(String email) {}
```

### DTO API sortant

```java
public record UserResponse(String id, String email) {}
```

**Avantages** :

- code minimal
- `toString/equals/hashCode` fiables
- immutabilité (bonne pour la concurrence et la prédictibilité)

> Notes JSON : Jackson supporte très bien les records (avec versions récentes). Pour des besoins avancés, on ajoute des annotations (`@JsonProperty`, etc.).

---

# 7. Atelier : mini design “Notification + DTO”

## 7.1. Objectif

- Définir une API stable via **interface**.
- Mutualiser la normalisation/logging via **classe abstraite**.
- Transporter des données via **record**.

---

## 7.2. Étape A — Contrat d’envoi

```java
public interface Notifier {
    void send(String msg);
}
```

---

## 7.3. Étape B — Record DTO

```java
public record Notification(String subject, String body) {
    public Notification {
        subject = subject == null ? "" : subject.trim();
        body = body == null ? "" : body.trim();
    }

    public String format() {
        return subject + "\n" + body;
    }
}
```

---

## 7.4. Étape C — Classe abstraite pour template + état

```java
import java.time.Clock;

public abstract class AbstractNotifier implements Notifier {
    protected final Clock clock;

    protected AbstractNotifier(Clock clock) {
        this.clock = clock;
    }

    @Override
    public final void send(String msg) {
        doSend("[" + java.time.Instant.now(clock) + "] " + normalize(msg));
    }

    protected String normalize(String msg) {
        return msg == null ? "" : msg.trim();
    }

    protected abstract void doSend(String msg);
}
```

---

## 7.5. Étape D — Implémentations

```java
import java.time.Clock;

public class ConsoleNotifier extends AbstractNotifier {
    public ConsoleNotifier(Clock clock) {
        super(clock);
    }

    @Override
    protected void doSend(String msg) {
        System.out.println(msg);
    }
}
```

---

## 7.6. Étape E — Service applicatif

```java
public class NotificationService {
    private final Notifier notifier;

    public NotificationService(Notifier notifier) {
        this.notifier = notifier;
    }

    public void notify(Notification notification) {
        notifier.send(notification.format());
    }
}
```

---

## 7.7. Étape F — Test rapide (sans framework)

```java
import java.time.Clock;

public class Demo {
    public static void main(String[] args) {
        Notifier notifier = new ConsoleNotifier(Clock.systemUTC());
        NotificationService service = new NotificationService(notifier);

        Notification dto = new Notification("Bienvenue", "Votre compte est créé.");
        service.notify(dto);
    }
}
```

---

# 8. Quiz / Questions de validation

1. Pourquoi dit-on qu’une interface est un **contrat** ?
2. Dans quel cas un `default` method est-il utile ? Quels risques ?
3. Une classe peut-elle étendre deux classes abstraites ? Pourquoi ?
4. Quels éléments un record génère-t-il automatiquement ?
5. Pourquoi les records sont-ils adaptés aux DTOs ?
6. Différence entre immutabilité “de surface” et “profonde” avec un champ `List` ?

---

# 9. Synthèse

- **Interface** : abstraction + contrat, favorise polymorphisme, testabilité, évolutivité.
- **Classe abstraite** : partage d’état et de comportements, utile pour template method et factorisation.
- **Record** : data class immutable concise, parfaite pour DTOs et modèles de données simples.

---

## Annexes — Rappels rapides (pour profils .NET/C#)

- Interface Java ≈ interface C# (avec `default` methods proche d’une implémentation partielle).
- Classe abstraite Java ≈ abstract class C#.
- Record Java ≈ record C# (idée similaire : immutabilité + génération de membres), mais syntaxe et capacités diffèrent.
