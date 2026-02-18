
# MASTERCLASS JAVA MODERNE (Java 8 → 21+)
## Formation intensive 3 jours – Niveau confirmé / senior

---
# JOUR 1 — Fondamentaux Avancés & Architecture du Langage

## 1. Architecture Java

### JVM (Java Virtual Machine)
- ClassLoader
- Bytecode verification
- JIT Compiler (C1 / C2)
- HotSpot optimizations
- Tiered compilation

### JDK vs JRE vs JVM
- JDK = outils + compilateur + runtime
- JRE = runtime uniquement
- JVM = moteur d'exécution

---
## 2. Modèle Mémoire Java (JMM)

### Heap
- Young Generation (Eden + Survivor)
- Old Generation
- Metaspace

### Stack
- Frames
- Variables locales
- Références

### Garbage Collectors
- Serial GC
- Parallel GC
- G1
- ZGC
- Shenandoah

---
## 3. POO Avancée

### Immutabilité
- Classes final
- Champs final
- Defensive copies

### equals() & hashCode()
Contrat :
- réflexif
- symétrique
- transitif
- cohérent

### Comparable vs Comparator

---
## 4. Generics Deep Dive

- Type erasure
- Wildcards (? extends / ? super)
- PECS principle

Exemple :

```java
List<? extends Number>
List<? super Integer>
```

---
# JOUR 2 — Java Moderne (8 → 21)

## Java 8 – Révolution fonctionnelle

### Lambda

```java
(x, y) -> x + y
```

### Functional Interfaces
- Predicate
- Function
- Consumer
- Supplier

### Streams avancés
- groupingBy
- partitioningBy
- parallelStream (pièges)
- reduce vs collect

### Optional avancé
- orElse vs orElseGet
- map / flatMap

---
## Java 9 → 11

### Modules (Jigsaw)
- module-info.java
- requires
- exports

### HTTP Client (Java 11)

```java
HttpClient client = HttpClient.newHttpClient();
```

---
## Java 14 → 17

### Records

```java
record User(String name, int age) {}
```

### Sealed Classes

```java
public sealed class Shape permits Circle, Rectangle {}
```

### Pattern Matching instanceof

```java
if (obj instanceof String s) {}
```

---
## Java 21 — Moderne & Concurrence

### Virtual Threads (Project Loom)

```java
Thread.startVirtualThread(() -> {});
```

Avantages :
- Threads légers
- Millions de threads
- Simplifie la concurrence

---
# JOUR 3 — Concurrence, Performance & Clean Code

## Concurrence Avancée

### Problèmes classiques
- Deadlock
- Livelock
- Starvation

### Synchronisation
- synchronized
- ReentrantLock
- ReadWriteLock
- Atomic classes

### CompletableFuture

```java
CompletableFuture.supplyAsync(() -> "data")
    .thenApply(String::toUpperCase);
```

---
## Performance & Profiling

### Outils
- JVisualVM
- JFR
- JMC

### Optimisations
- éviter boxing
- limiter allocations
- StringBuilder
- GC tuning

---
## Clean Code & Architecture

### SOLID
- SRP
- OCP
- LSP
- ISP
- DIP

### Bonnes pratiques senior
- Composition > héritage
- Immutabilité
- Logging structuré
- Gestion centralisée des exceptions
- Tests unitaires (JUnit 5)

---
# ANNEXES

## Évolution Java

| Version | Feature clé |
|----------|-------------|
| 8 | Lambda, Streams |
| 9 | Modules |
| 10 | var |
| 14 | Records |
| 17 | Sealed Classes |
| 21 | Virtual Threads |

---
# Conclusion

Java moderne est :
- Expressif
- Fonctionnel
- Performant
- Adapté au cloud
