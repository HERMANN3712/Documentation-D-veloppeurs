# 11) Concurrence & async (Java)

> Public cible : développeur·se confirmé·e (ex .NET/C#) souhaitant maîtriser la concurrence en Java moderne.
>
> Objectifs : comprendre les problèmes classiques (race conditions, visibilité), choisir les bons outils (Executors/Future, CompletableFuture, collections concurrentes), appliquer les bonnes pratiques (immutabilité, thread-safety) et savoir diagnostiquer.

---

## Plan de la formation

1. **Introduction & modèle de concurrence Java**
   - Process vs thread, CPU-bound vs I/O-bound
   - Threads Java, scheduling, états d’un thread
   - Pourquoi l’async ? Latence, scalabilité, réactivité

2. **Les fondamentaux : risques et vocabulaire**
   - Conditions de course (race conditions)
   - Atomicité, interleavings, invariants
   - Visibilité mémoire, réordonnancement, *happens-before*
   - Thread-safety : définition et niveaux
   - Immutabilité : pourquoi c’est la stratégie n°1

3. **Executors & Future**
   - `Executor`, `ExecutorService`, `ScheduledExecutorService`
   - Soumission de tâches : `execute`, `submit`, `invokeAll`, `invokeAny`
   - `Future` : récupération, timeout, annulation
   - Gestion du pool : types, sizing, queues, `ThreadFactory`
   - Bonnes pratiques : fermeture, exceptions, backpressure

4. **CompletableFuture (programmation asynchrone moderne)**
   - Différences avec `Future`
   - Chaînage : `thenApply`, `thenCompose`, `thenAccept`, `thenRun`
   - Combinaison : `allOf`, `anyOf`, `thenCombine`, `handle`, `exceptionally`
   - Choix du scheduler : `ForkJoinPool.commonPool` vs executor dédié
   - Timeouts : `orTimeout`, `completeOnTimeout`
   - Patterns : pipeline, fan-out/fan-in, retry (simple)

5. **Collections concurrentes (focus : ConcurrentHashMap)**
   - Pourquoi `HashMap` n’est pas thread-safe
   - `ConcurrentHashMap` : garanties, performances, contention
   - Opérations atomiques : `compute`, `computeIfAbsent`, `merge`, `putIfAbsent`
   - Pièges : opérations composées, itérations, valeurs mutables

6. **Synchronisation : synchronized / Lock**
   - Moniteur Java et `synchronized`
   - `ReentrantLock` : fonctionnalités et trade-offs
   - Lecture/écriture : `ReadWriteLock`
   - Attente/notification : `wait/notify` (aperçu) et `Condition`
   - Deadlocks : causes, prévention (ordre de locks, timeouts)

7. **`volatile` (visibilité) et atomiques (aperçu)**
   - Quand `volatile` est suffisant (flags, publication)
   - Quand ce n’est pas suffisant (compteurs)
   - Mention rapide : `AtomicInteger`, `LongAdder`

8. **Synthèse : guider vos choix**
   - CPU-bound vs I/O-bound
   - Préférer l’immutabilité et les opérations atomiques
   - Stratégie de test & debug

9. **Atelier (exercices guidés)**
   - Corriger une race condition
   - Mettre en place un pool et gérer un timeout
   - Pipeline async avec `CompletableFuture`
   - Cache concurrent avec `ConcurrentHashMap.computeIfAbsent`

---

## 1. Introduction & modèle de concurrence Java

### 1.1 Threading en Java
- Un **thread** exécute du code de manière concurrente au sein d’un même processus.
- La JVM s’appuie sur les threads OS (généralement 1:1) : la planification reste largement dépendante de l’OS.

**États (simplifiés)** : NEW → RUNNABLE → (BLOCKED/WAITING/TIMED_WAITING) → TERMINATED.

### 1.2 Pourquoi l’async ?
Deux motivations principales :
- **Parallélisme** (utiliser plusieurs cœurs) : tâches CPU-bound.
- **Concurrence** (masquer la latence) : tâches I/O-bound (HTTP, DB, disque).

> En Java, le *vrai* async I/O existe via NIO (ou frameworks), mais dans ce module on couvre surtout la **concurrence au niveau des tâches** via pools et `CompletableFuture`.

---

## 2. Fondamentaux : risques et vocabulaire

### 2.1 Conditions de course (race conditions)
Une race condition survient quand :
- plusieurs threads accèdent à une même donnée,
- au moins un thread écrit,
- et l’ordre d’exécution n’est pas contrôlé.

Exemple classique : compteur non atomique.

```java
class Counter {
    private int value = 0;

    public void inc() {
        value++; // read-modify-write non atomique
    }

    public int get() {
        return value;
    }
}
```

Le `value++` se décompose en :
1) lecture, 2) +1, 3) écriture. Deux threads peuvent interférer.

### 2.2 Atomicité vs visibilité
- **Atomicité** : une opération semble indivisible.
- **Visibilité** : un thread voit-il les écritures d’un autre thread rapidement et de manière ordonnée ?

En Java, un thread peut conserver des valeurs en cache (registre, cache CPU…) et le compilateur/CPU peut réordonner.

### 2.3 Le modèle mémoire Java (JMM) et *happens-before*
Sans entrer dans tous les détails :
- certaines constructions créent une relation **happens-before** garantissant visibilité + ordre.
- Exemples :
  - un `unlock` *happens-before* un `lock` du même moniteur/lock,
  - une écriture sur un champ `volatile` *happens-before* une lecture subséquente de ce champ,
  - publication sûre via initialisation statique, final, etc.

### 2.4 Thread-safety et immutabilité
**Thread-safe** : un composant peut être utilisé par plusieurs threads sans casser ses invariants.

Niveaux :
- *immutable* → intrinsèquement thread-safe
- *thread-confined* (utilisé par un seul thread)
- *synchronized* / *concurrent data structures*

**Immutabilité** :
- Pas de changements après construction.
- Permet le partage sans locks.

```java
public record Money(String currency, long cents) {}
```

> En pratique : rendez vos objets **immutables** dès que possible, et évitez de stocker des objets mutables dans des structures partagées.

---

## 3. Executors & Future

### 3.1 Pourquoi `ExecutorService` ?
Créer un `new Thread(...)` par tâche :
- coûteux,
- difficile à superviser,
- risque de surconsommation (trop de threads).

Les pools via `ExecutorService` :
- réutilisent des threads,
- contrôlent le nombre de tâches concurrentes,
- centralisent la gestion (shutdown, métriques, stratégie de rejet).

### 3.2 Créer un pool

```java
ExecutorService pool = Executors.newFixedThreadPool(8);
```

Types courants :
- `newFixedThreadPool(n)` : taille fixe, bon point de départ CPU-bound.
- `newCachedThreadPool()` : croissance dynamique, attention aux explosions.
- `newSingleThreadExecutor()` : garantit l’ordre, utile pour sérialiser.
- `newScheduledThreadPool(n)` : tâches planifiées.

> Pour un contrôle fin (queue, saturation), utilisez plutôt `ThreadPoolExecutor`.

### 3.3 Soumettre des tâches
- `execute(Runnable)` : fire-and-forget (exceptions via handler du thread).
- `submit(Callable<T>)` : retourne un `Future<T>`.

```java
Future<Integer> f = pool.submit(() -> {
    // calcul
    return 42;
});

Integer result = f.get();
```

### 3.4 `Future` : timeout, annulation, exceptions

```java
try {
    Integer r = f.get(200, java.util.concurrent.TimeUnit.MILLISECONDS);
} catch (java.util.concurrent.TimeoutException e) {
    f.cancel(true); // interruption si possible
}
```

- `get()` peut lever `ExecutionException` (wrapping de l’exception métier).
- `cancel(true)` demande l’interruption du thread → la tâche doit coopérer.

**Coopérer à l’interruption** :
- vérifier `Thread.currentThread().isInterrupted()`
- ou propager `InterruptedException`.

### 3.5 `invokeAll` et `invokeAny`

```java
List<Callable<String>> tasks = List.of(
    () -> "A",
    () -> "B"
);

List<Future<String>> results = pool.invokeAll(tasks);
```

- `invokeAll` attend la fin de toutes les tâches.
- `invokeAny` retourne la première qui réussit (annule souvent les autres).

### 3.6 Shutdown propre

```java
pool.shutdown();
if (!pool.awaitTermination(5, java.util.concurrent.TimeUnit.SECONDS)) {
    pool.shutdownNow();
}
```

Bonnes pratiques :
- éviter les fuites (toujours shutdown),
- gérer la saturation (queue pleine),
- nommer les threads avec `ThreadFactory`.

---

## 4. CompletableFuture

### 4.1 Limites de `Future`
- Pas de composition (enchaîner des étapes sans bloquer).
- Gestion des erreurs et combinaisons peu ergonomiques.

`CompletableFuture` apporte :
- composition fonctionnelle,
- callbacks,
- combinators,
- exécution asynchrone configurable.

### 4.2 Création

```java
CompletableFuture<String> cf = CompletableFuture.supplyAsync(() -> {
    return "data";
});
```

Par défaut : `ForkJoinPool.commonPool`.

### 4.3 Chaînage : thenApply vs thenCompose

```java
CompletableFuture<Integer> length =
    CompletableFuture.supplyAsync(() -> "hello")
        .thenApply(String::length); // transformation
```

`thenCompose` pour éviter les futures imbriqués :

```java
CompletableFuture<String> userName = fetchUserId()
    .thenCompose(id -> fetchUserName(id));
```

- `thenApply` : T → U
- `thenCompose` : T → CompletableFuture<U>

### 4.4 Combinaisons

**Combiner deux résultats** :

```java
CompletableFuture<Integer> a = CompletableFuture.supplyAsync(() -> 20);
CompletableFuture<Integer> b = CompletableFuture.supplyAsync(() -> 22);

CompletableFuture<Integer> sum = a.thenCombine(b, Integer::sum);
```

**Fan-out / fan-in** :

```java
CompletableFuture<Void> all = CompletableFuture.allOf(a, b);
all.join();
```

- `allOf` : attend tout, ne donne pas les valeurs directement.
- `anyOf` : première complétée (succès ou échec).

### 4.5 Gestion d’erreurs

```java
CompletableFuture<String> safe =
    CompletableFuture.supplyAsync(() -> {
        throw new RuntimeException("boom");
    })
    .exceptionally(ex -> "fallback");
```

`handle` permet de traiter succès/échec :

```java
cf.handle((value, ex) -> ex == null ? value : "fallback");
```

### 4.6 Choisir l’executor
Pour éviter de saturer le `commonPool` (surtout en I/O bloquantes), fournir un executor dédié :

```java
ExecutorService ioPool = Executors.newFixedThreadPool(32);

CompletableFuture<String> cf = CompletableFuture.supplyAsync(() -> callBlockingApi(), ioPool);
```

Règle pratique :
- CPU-bound : pool proche du nombre de cœurs.
- I/O bloquant : pool plus large (mais surveiller).

### 4.7 Timeouts

```java
CompletableFuture<String> cf = CompletableFuture
    .supplyAsync(() -> callSlow())
    .orTimeout(300, java.util.concurrent.TimeUnit.MILLISECONDS)
    .exceptionally(ex -> "timeout/fallback");
```

Ou `completeOnTimeout("fallback", ...)`.

### 4.8 Attention : blocage et `join/get`
- `join()` renvoie une exception non checked (`CompletionException`).
- Évitez de bloquer (get/join) au milieu d’un pipeline : vous perdez l’intérêt de l’async.

---

## 5. Collections concurrentes : `ConcurrentHashMap`

### 5.1 Pourquoi pas `HashMap` ?
`HashMap` n’est pas thread-safe :
- corruption interne possible,
- perte d’écritures,
- comportements non déterministes.

### 5.2 Garanties de `ConcurrentHashMap`
- Opérations individuelles thread-safe.
- Bonne scalabilité via verrouillage fin (segmentation/stripes selon implémentation).
- Itérateurs *weakly consistent* : pas d’exception `ConcurrentModificationException`, mais la vue peut être partielle.

### 5.3 Opérations atomiques composées
**Piège** : check-then-act non atomique.

```java
if (!map.containsKey(k)) {
    map.put(k, compute());
}
```

Correct : `computeIfAbsent`.

```java
V v = map.computeIfAbsent(k, key -> compute());
```

Autres opérations utiles :
- `putIfAbsent`
- `compute`
- `merge`

Exemple : compteur par clé avec `merge`.

```java
map.merge(userId, 1L, Long::sum);
```

### 5.4 Piège : valeurs mutables
Même si la map est concurrente, **les valeurs** peuvent être mutables et non thread-safe.

Stratégies :
- valeurs immutables,
- valeurs atomiques (`AtomicInteger`),
- confinement (ne pas partager),
- synchronisation autour de la valeur.

---

## 6. Synchronisation : `synchronized` / `Lock`

### 6.1 `synchronized`
- Verrouille un moniteur (sur `this` ou un objet lock).
- Garantit mutual exclusion + happens-before à la sortie/entrée.

```java
class SafeCounter {
    private int value;

    public synchronized void inc() {
        value++;
    }

    public synchronized int get() {
        return value;
    }
}
```

Bonnes pratiques :
- verrouiller sur un objet privé dédié :

```java
private final Object lock = new Object();

synchronized (lock) {
   // ...
}
```

### 6.2 `ReentrantLock`
Avantages :
- `tryLock()` (avec timeout),
- lock interruptible,
- plusieurs `Condition`.

```java
var lock = new java.util.concurrent.locks.ReentrantLock();

lock.lock();
try {
    // section critique
} finally {
    lock.unlock();
}
```

### 6.3 ReadWriteLock
Utile si beaucoup de lectures, peu d’écritures.

```java
var rw = new java.util.concurrent.locks.ReentrantReadWriteLock();

rw.readLock().lock();
try {
    // lecture
} finally {
    rw.readLock().unlock();
}
```

### 6.4 Deadlocks
Causes typiques :
- ordre de locks différent,
- locks non relâchés,
- appels en chaîne.

Prévention :
- **ordre global** d’acquisition,
- `tryLock` + timeout,
- limiter l’étendue des sections critiques.

---

## 7. `volatile` (visibilité)

### 7.1 À quoi sert `volatile` ?
- Garantit la **visibilité** des écritures et empêche certains réordonnancements.
- Ne rend pas une opération composée atomique.

Cas typique : **flag d’arrêt**.

```java
class Worker {
    private volatile boolean running = true;

    public void stop() { running = false; }

    public void run() {
        while (running) {
            // work
        }
    }
}
```

### 7.2 Ce que `volatile` ne fait pas
Un compteur `volatile int` reste non atomique :

```java
volatile int x;

x++; // non atomique
```

Solutions :
- `synchronized`
- `AtomicInteger`
- `LongAdder` (forte contention)

---

## 8. Points clés (récap)

- **Race conditions** : apparaissent dès que vous faites du *read-modify-write* sans coordination.
- **Immutabilité** : meilleure arme (simple, performante, testable).
- **Thread-safety** : concerne aussi les valeurs stockées, pas seulement les conteneurs.
- **Executors/Future** : pour gérer des tâches et des pools, avec timeouts et annulation.
- **CompletableFuture** : pour composer des workflows asynchrones sans bloquer.
- **synchronized/Lock** : pour sections critiques, attention aux deadlocks.
- **volatile** : visibilité et publication, pas d’atomicité.

---

## 9. Atelier (exercices guidés)

### Exercice 1 — Diagnostiquer et corriger une race condition
1. Implémenter un compteur partagé incrémenté par 10 threads.
2. Observer que la valeur finale est incorrecte.
3. Corriger via :
   - `synchronized`, puis
   - `AtomicInteger`.

### Exercice 2 — Pool + timeout
1. Créer un `ExecutorService`.
2. Soumettre une tâche qui dort 2s.
3. Récupérer avec `get(200ms)` et annuler.
4. Assurer un shutdown propre.

### Exercice 3 — Pipeline `CompletableFuture`
1. `supplyAsync` (fetch)
2. `thenApply` (parse)
3. `thenCompose` (fetch détails)
4. `exceptionally` (fallback)

### Exercice 4 — Cache concurrent
1. Créer un cache `ConcurrentHashMap<Key, Value>`.
2. Alimenter via `computeIfAbsent`.
3. Discuter : valeurs immutables vs mutables.

---

## Annexes — Correspondances .NET (repères)

- `Task` / `Task<T>` ~ `CompletableFuture` (composition)
- `ThreadPool` ~ `ExecutorService`
- `lock(obj)` ~ `synchronized`
- `ConcurrentDictionary` ~ `ConcurrentHashMap`
- `volatile` existe aussi, mêmes idées (visibilité) mais modèle mémoire spécifique.

---

### Références
- Javadoc : `java.util.concurrent` (Executors, Locks, ConcurrentHashMap)
- Brian Goetz et al., *Java Concurrency in Practice* (référence incontournable)
