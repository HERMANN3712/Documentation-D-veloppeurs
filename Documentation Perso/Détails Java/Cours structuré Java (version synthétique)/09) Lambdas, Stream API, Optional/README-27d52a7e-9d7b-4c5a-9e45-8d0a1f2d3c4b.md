# 9) Lambdas, Stream API, Optional (Java)

> Formation Java — Niveau intermédiaire. Guide pédagogique orienté pratique.

## Objectifs pédagogiques
À l’issue de cette séance, vous saurez :

- Expliquer ce qu’est une **lambda** et une **interface fonctionnelle**.
- Utiliser les **Streams** pour traiter des collections de manière déclarative : `filter`, `map`, `reduce`.
- Utiliser les primitives streams (`mapToInt`) et les agrégations (`sum`, `average`, `max`).
- Comprendre et appliquer **Optional** correctement : `map`, `flatMap`, `orElse`, `orElseGet`, `orElseThrow`.
- Adopter de bonnes pratiques : Optional pour les **retours** (APIs), pas partout (champs/DTO), et éviter les nulls.

## Pré-requis
- Java 8+ (idéalement 11/17)
- Connaissances : collections (`List`, `Set`, `Map`), classes, exceptions.

## Plan
1. [Lambdas et interfaces fonctionnelles](#1-lambdas-et-interfaces-fonctionnelles)
2. [Stream API : pipeline de traitements](#2-stream-api--pipeline-de-traitements)
3. [Streams + primitives : `mapToInt`, `sum`](#3-streams--primitives--maptoint-sum)
4. [Optional : éviter les null](#4-optional--éviter-les-null)
5. [Bonnes pratiques et pièges courants](#5-bonnes-pratiques-et-pièges-courants)
6. [Exercices](#6-exercices)
7. [Corrigés (propositions)](#7-corrigés-propositions)

---

## 1) Lambdas et interfaces fonctionnelles

### 1.1 Pourquoi les lambdas ?
Avant Java 8, pour passer un comportement (une fonction) en paramètre, on utilisait :

- des classes anonymes (`new Runnable(){...}`)
- des patterns (Strategy)

Les **lambdas** rendent cela plus lisible et concis.

### 1.2 Définition
Une **lambda** est une *fonction anonyme* pouvant être assignée à une **interface fonctionnelle**.

Une **interface fonctionnelle** = interface avec **une seule méthode abstraite** (SAM : *Single Abstract Method*).

Exemples d’interfaces fonctionnelles standards (package `java.util.function`) :

- `Function<T, R>` : transforme un T en R
- `Consumer<T>` : consomme un T (pas de retour)
- `Supplier<T>` : fournit un T (pas de paramètre)
- `Predicate<T>` : teste un T (retourne boolean)

### 1.3 Exemple fondamental : `Function<String, Integer>`

```java
import java.util.function.Function;

public class DemoLambda {
    public static void main(String[] args) {
        Function<String, Integer> len = s -> s.length();

        System.out.println(len.apply("hello")); // 5
    }
}
```

Décorticage :

- `s` est le paramètre (type inféré : `String`)
- `s.length()` est l’expression retournée (type `Integer` / `int` auto-boxé)
- `apply` exécute la fonction

### 1.4 Syntaxes possibles

#### Cas expression simple
```java
Function<String, Integer> len = s -> s.length();
```

#### Bloc avec plusieurs instructions
```java
Function<String, Integer> len = s -> {
    if (s == null) return 0;
    return s.length();
};
```

#### Plusieurs paramètres
```java
java.util.function.BiFunction<Integer, Integer, Integer> add = (a, b) -> a + b;
```

### 1.5 Méthodes références (method references)
Alternative lisible à certaines lambdas :

- `ClassName::staticMethod`
- `instance::method`
- `ClassName::instanceMethod`
- `ClassName::new` (constructeur)

Exemples :

```java
Function<String, Integer> len1 = s -> s.length();
Function<String, Integer> len2 = String::length;
```

```java
java.util.function.Consumer<String> printer = System.out::println;
```

### 1.6 Règles importantes

- Une lambda ne peut être assignée qu’à une **cible typée** (une interface fonctionnelle).
- Les variables capturées depuis l’extérieur doivent être **effectivement finales**.

```java
int base = 10;
Function<Integer, Integer> f = x -> x + base; // OK si base n'est pas modifiée
```

---

## 2) Stream API : pipeline de traitements

### 2.1 Pourquoi les streams ?
Les streams permettent :

- un style déclaratif ("quoi" plutôt que "comment")
- l’enchaînement d’opérations (pipeline)
- des optimisations internes (lazy evaluation)
- l’accès facile au parallélisme (`parallelStream`) avec prudence

> Un `Stream<T>` n’est pas une collection : il représente une **séquence de données** et un **pipeline**.

### 2.2 Étapes d’un pipeline

1. **Source** : `list.stream()`, `Stream.of(...)`, `Files.lines(...)`
2. **Opérations intermédiaires** (lazy) : `filter`, `map`, `sorted`, `distinct`...
3. **Opération terminale** : `collect`, `forEach`, `reduce`, `count`, `findFirst`...

### 2.3 Exemple de données

```java
import java.util.List;

record User(long id, String name, String email, boolean active) {}

List<User> users = List.of(
    new User(1, "Alice", "alice@corp.com", true),
    new User(2, "Bob", null, false),
    new User(3, "Chloé", "chloe@corp.com", true)
);
```

### 2.4 `filter` : garder certains éléments

```java
List<User> activeUsers = users.stream()
    .filter(User::active)
    .toList();
```

- `filter` prend un `Predicate<T>`.
- Ne modifie pas la liste source.

### 2.5 `map` : transformer les éléments

Extraire les noms :

```java
List<String> names = users.stream()
    .map(User::name)
    .toList();
```

Transformer en domaine/DTO :

```java
record UserDto(long id, String name) {}

List<UserDto> dtos = users.stream()
    .map(u -> new UserDto(u.id(), u.name()))
    .toList();
```

### 2.6 `reduce` : agréger en une seule valeur

`reduce` combine les éléments en appliquant une fonction associative.

Somme des ids (démonstration) :

```java
long sumIds = users.stream()
    .map(User::id)
    .reduce(0L, Long::sum);
```

Autre exemple : concaténation (à éviter pour perf, préférer `Collectors.joining`) :

```java
String allNames = users.stream()
    .map(User::name)
    .reduce("", (a, b) -> a + "," + b);
```

### 2.7 `collect` : le terminal le plus courant

Collecter en liste/ensemble :

```java
List<String> emails = users.stream()
    .map(User::email)
    .toList();
```

> Attention : ici `email` peut être `null` si le modèle l’autorise. En pratique, préférez des invariants (email non-null) ou un `Optional` au retour d’une méthode.

---

## 3) Streams + primitives : `mapToInt`, `sum`

### 3.1 Pourquoi des streams de primitives ?
Les `Stream<Integer>` impliquent boxing/unboxing. Java fournit :

- `IntStream`
- `LongStream`
- `DoubleStream`

Ils évitent l’auto-boxing et fournissent des méthodes d’agrégation pratiques.

### 3.2 Exemple : longueurs de chaînes → somme

```java
import java.util.List;

List<String> words = List.of("java", "stream", "lambda");

int totalLength = words.stream()
    .mapToInt(String::length)
    .sum();

System.out.println(totalLength); // 4 + 6 + 6 = 16
```

- `mapToInt` transforme un `Stream<T>` en `IntStream`
- `sum` est un terminal disponible sur `IntStream`

### 3.3 Autres agrégations utiles

```java
int maxLen = words.stream().mapToInt(String::length).max().orElse(0);

double avgLen = words.stream().mapToInt(String::length).average().orElse(0.0);

long count = words.stream().filter(w -> w.length() >= 5).count();
```

### 3.4 `reduce` vs `sum`
On peut faire :

```java
int total = words.stream()
    .map(String::length)
    .reduce(0, Integer::sum);
```

Mais `mapToInt(...).sum()` est plus direct et évite le boxing.

---

## 4) Optional : éviter les null

### 4.1 Objectif
`Optional<T>` représente la présence ou l’absence d’une valeur.

- évite des `NullPointerException`
- rend l’absence explicite dans la signature
- encourage une API plus sûre

### 4.2 Création d’un Optional

```java
Optional<String> a = Optional.of("x");
Optional<String> b = Optional.ofNullable(null);
Optional<String> c = Optional.empty();
```

- `of` : interdit `null` (sinon NPE immédiate)
- `ofNullable` : accepte `null`

### 4.3 Utilisation : `map` et `orElse`

Exemple conforme à l’énoncé :

```java
String email = repo.find(id)
    .map(User::email)
    .orElse("unknown");
```

Interprétation :

- `repo.find(id)` retourne un `Optional<User>`
- si un `User` est présent : on transforme en `email`
- sinon : on renvoie une valeur par défaut

### 4.4 `orElse` vs `orElseGet`

```java
String v1 = opt.orElse(computeDefault());
String v2 = opt.orElseGet(() -> computeDefault());
```

- `orElse(...)` **évalue** l’argument immédiatement (même si opt est présent)
- `orElseGet(...)` n’évalue le supplier **que si nécessaire**

=> Utilisez `orElseGet` si le défaut est coûteux.

### 4.5 `orElseThrow`

```java
User u = repo.find(id)
    .orElseThrow(() -> new IllegalArgumentException("User not found: " + id));
```

### 4.6 `flatMap` (cas Optional imbriqués)
Si `User::emailOptional` retourne déjà un `Optional<String>` :

```java
Optional<String> email = repo.find(id)
    .flatMap(User::emailOptional);
```

- `map` donnerait `Optional<Optional<String>>`
- `flatMap` aplatit.

### 4.7 `ifPresent` / `ifPresentOrElse`

```java
repo.find(id).ifPresent(u -> System.out.println(u.email()));

repo.find(id).ifPresentOrElse(
    u -> System.out.println("found " + u.id()),
    () -> System.out.println("not found")
);
```

---

## 5) Bonnes pratiques et pièges courants

### 5.1 Conseil clé (à retenir)
- ✅ Utiliser `Optional` pour les **retours** de méthodes qui peuvent ne pas produire de valeur.
- ❌ Éviter `Optional` **partout** (en champs d’entités, champs de DTO, champs sérialisés JSON).

#### Pourquoi éviter `Optional` en champs ?
- surcharge mémoire/complexité
- sérialisation/ORM parfois compliquée
- la sémantique devient confuse (Optional d’Optional, etc.)

> En pratique : **modèle interne strict (non-null)** + `Optional` en surface (API / repository).

### 5.2 Ne pas utiliser `Optional.get()` sans test

À éviter :

```java
String x = opt.get(); // NoSuchElementException si vide
```

Préférez : `orElse`, `orElseThrow`, `ifPresent`.

### 5.3 Streams : attention aux effets de bord
Évitez dans `map` / `filter` :

- mutation d’objets
- I/O
- accès DB

Préférez isoler les effets de bord en terminal (`forEach`) ou hors stream.

### 5.4 Streams : compréhension de la lazy evaluation

```java
users.stream()
    .filter(u -> {
        System.out.println("filter " + u.id());
        return u.active();
    })
    .map(u -> {
        System.out.println("map " + u.id());
        return u.email();
    })
    .findFirst();
```

- Le pipeline s’exécute seulement au terminal (`findFirst`).
- `findFirst` peut court-circuiter.

### 5.5 `parallelStream` : prudence
- utile pour gros volumes + calcul CPU + absence d’effets de bord
- attention au coût de parallélisation et au contexte (serveur web)

---

## 6) Exercices

### Exercice 1 — Lambda
Créer une `Function<String, Integer>` appelée `len` qui retourne la longueur d’une chaîne.

- Tester avec `"bonjour"`.
- Variante : gérer `null` en renvoyant 0.

### Exercice 2 — Streams `filter/map`
Avec une liste de `User`, produire la liste des emails des utilisateurs actifs.

Contraintes :
- Ignorer les emails `null`.

### Exercice 3 — Streams `mapToInt/sum`
Avec une liste de mots, calculer la somme des longueurs.

### Exercice 4 — Optional
Simuler un repository :

- `Optional<User> find(long id)`

Puis produire :

```java
repo.find(id).map(User::email).orElse("unknown");
```

Variante : lever une exception si l’utilisateur est absent.

---

## 7) Corrigés (propositions)

### Corrigé Ex.1

```java
Function<String, Integer> len = s -> s.length();

Function<String, Integer> lenSafe = s -> s == null ? 0 : s.length();
```

### Corrigé Ex.2

```java
List<String> emails = users.stream()
    .filter(User::active)
    .map(User::email)
    .filter(Objects::nonNull)
    .toList();
```

### Corrigé Ex.3

```java
int totalLength = words.stream()
    .mapToInt(String::length)
    .sum();
```

### Corrigé Ex.4

```java
String email = repo.find(id)
    .map(User::email)
    .orElse("unknown");

User user = repo.find(id)
    .orElseThrow(() -> new IllegalStateException("User not found"));
```

---

## Récapitulatif

- **Lambda** : `Function<String,Integer> len = s -> s.length();`
- **Streams** : `filter` / `map` / `reduce`, et `mapToInt(...).sum()` pour les primitives
- **Optional** : `repo.find(id).map(User::email).orElse("unknown")`
- **Conseil** : Optional pour les retours, pas partout en champs.
