# Cours Complet Java (Fondamentaux → Java 21)

------------------------------------------------------------------------

# 1. Fondamentaux

## Structure d'un programme

``` java
public class Main {
    public static void main(String[] args) {
        System.out.println("Hello World");
    }
}
```

## Types primitifs

byte, short, int, long\
float, double\
char\
boolean

## Variables

-   locale
-   attribut
-   static
-   final

------------------------------------------------------------------------

# 2. Programmation Orientée Objet

## Classe & Objet

``` java
class User {
    private String name;

    public User(String name) {
        this.name = name;
    }

    public String getName() {
        return name;
    }
}
```

## Concepts

-   Encapsulation
-   Héritage
-   Polymorphisme
-   Abstraction
-   Interfaces

------------------------------------------------------------------------

# 3. Collections Framework

Interfaces : - List - Set - Map - Queue

Implémentations : - ArrayList - LinkedList - HashSet - HashMap - TreeMap

------------------------------------------------------------------------

# 4. Java 8 (Révolution)

## Lambda

``` java
list.forEach(x -> System.out.println(x));
```

## Streams

``` java
list.stream()
    .filter(x -> x > 10)
    .map(x -> x * 2)
    .forEach(System.out::println);
```

## Optional

``` java
Optional<String> name = Optional.ofNullable(getName());
```

## Date/Time API

LocalDate, LocalDateTime, ZonedDateTime

------------------------------------------------------------------------

# 5. Concurrence

## Thread

``` java
Thread t = new Thread(() -> {});
t.start();
```

## ExecutorService

``` java
ExecutorService executor = Executors.newFixedThreadPool(4);
```

Synchronisation : - synchronized - volatile - ReentrantLock -
ConcurrentHashMap

------------------------------------------------------------------------

# 6. JVM & Mémoire

Heap → objets\
Stack → variables locales\
Garbage Collectors : - G1 - ZGC - Shenandoah

------------------------------------------------------------------------

# 7. Exceptions

``` java
try {
} catch (Exception e) {
} finally {
}
```

-   checked
-   unchecked
-   custom

------------------------------------------------------------------------

# 8. Nouveautés depuis Java 8

## Java 9

Modules (Project Jigsaw)

## Java 10

var (inférence locale)

``` java
var name = "Java";
```

## Java 11 (LTS)

-   HTTP Client
-   String.isBlank()
-   Files.readString()

## Java 12-13

Switch expressions

## Java 14-16

Records Pattern matching instanceof

``` java
record User(String name, int age) {}
```

## Java 17 (LTS)

Sealed classes

``` java
public sealed class Shape
    permits Circle, Rectangle {}
```

## Java 21 (LTS)

### Virtual Threads (Project Loom)

``` java
Thread.startVirtualThread(() -> {
    System.out.println("Lightweight thread");
});
```

### Pattern Matching Switch

### Record Patterns

------------------------------------------------------------------------

# Résumé évolution

Java 8 → Lambda & Streams\
Java 9 → Modules\
Java 10 → var\
Java 14 → Records\
Java 17 → Sealed Classes\
Java 21 → Virtual Threads

------------------------------------------------------------------------

# Conclusion

Java moderne est : - Plus fonctionnel - Moins verbeux - Plus performant
en concurrence - Adapté au cloud et microservices
