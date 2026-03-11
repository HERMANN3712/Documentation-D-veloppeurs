# Formation 12 — JVM : mémoire, GC, performance

## Public & prérequis
- **Public** : développeurs Java (ou venant de .NET/C#) souhaitant comprendre le modèle mémoire JVM, l’impact du GC et les techniques de profiling.
- **Prérequis** : notions de base Java, classes/objets, collections, outils en ligne de commande.

## Objectifs pédagogiques
À l’issue, vous saurez :
1. Expliquer **stack vs heap** et les zones mémoire de la JVM.
2. Comprendre les grandes familles de **garbage collectors**, leurs compromis (débit vs latence).
3. Diagnostiquer les problèmes : **pauses GC**, allocation excessive, fuite mémoire, fragmentation.
4. Mettre en place un **profiling** adapté (JFR/Java Flight Recorder, logs GC, jcmd/jmap/jstat).
5. Appliquer des **bonnes pratiques** pour réduire les allocations et stabiliser les performances.

## Durée suggérée
- **3 h** (format cours + démos)
- Option atelier : **+1 h** (analyse de cas réels et tuning guidé)

## Plan de la formation
1. [Modèle mémoire JVM (vue d’ensemble)](#1-modèle-mémoire-jvm-vue-densemble)
2. [Stack vs Heap : ce qui vit où et pourquoi](#2-stack-vs-heap--ce-qui-vit-où-et-pourquoi)
3. [Garbage Collection : principes, algorithmes, pauses](#3-garbage-collection--principes-algorithmes-pauses)
4. [Tuning GC : à doser, mesurer, itérer](#4-tuning-gc--à-doser-mesurer-itérer)
5. [Profiling & observabilité : logs GC, JFR, outils](#5-profiling--observabilité--logs-gc-jfr-outils)
6. [Bonnes pratiques de performance mémoire](#6-bonnes-pratiques-de-performance-mémoire)
7. [Ateliers / exercices (avec corrigés)](#7-ateliers--exercices-avec-corrigés)
8. [Checklist de diagnostic et mémo commandes](#8-checklist-de-diagnostic-et-mémo-commandes)

---

## 1) Modèle mémoire JVM (vue d’ensemble)

### 1.1 Les grandes zones mémoire
Selon l’implémentation (HotSpot le plus souvent), on retrouve typiquement :

- **Heap** : stockage des objets/allocation dynamique, géré par le GC.
  - **Young generation** (souvent : Eden + Survivor spaces)
  - **Old generation** (tenured)
  - (Selon GC : régions, humongous, etc.)
- **Stacks (par thread)** : frames d’appels, variables locales, références.
- **Metaspace** (ex-PermGen) : métadonnées de classes, structures JVM.
- **Code cache** : code JIT compilé.
- **Native memory** : mémoire hors heap (direct buffers, bibliothèques natives, threads, etc.).

> Point clé : « mémoire JVM » ≠ seulement le heap. Un process Java peut OOM côté heap **ou** côté natif.

### 1.2 OOM : différentes causes
- `java.lang.OutOfMemoryError: Java heap space` : heap saturé (fuite, charge, sizing insuffisant).
- `java.lang.OutOfMemoryError: Metaspace` : trop de classes, classloaders qui fuient.
- `java.lang.OutOfMemoryError: Direct buffer memory` : buffers NIO directs (off-heap) trop nombreux.
- `unable to create new native thread` : limites OS / stack par thread / ressources.

---

## 2) Stack vs Heap : ce qui vit où et pourquoi

### 2.1 Stack (pile d’exécution)
La stack est **par thread** :
- Une **frame** par appel de méthode.
- Contient notamment :
  - paramètres,
  - variables locales (primitives et références),
  - infos de retour, etc.

Caractéristiques :
- Allocation/désallocation **très rapide** (push/pop).
- Taille bornée (configurée, souvent via `-Xss`).
- Si récursion profonde ou trop de frames : `StackOverflowError`.

### 2.2 Heap (tas)
Le heap est **partagé par tous les threads** :
- Contient les **objets** et tableaux.
- Problème principal : les objets ont une durée de vie variable ⇒ on a besoin du GC.

Caractéristiques :
- **Accès** potentiellement plus coûteux (cache locality, pointeurs, etc.).
- **Fragmentation** possible selon le collecteur.
- Sizing via `-Xms` / `-Xmx`.

### 2.3 Primitives, références et objets (rappel utile)
- Une **primitive** (`int`, `long`, `double`, etc.) peut être stockée en local sur la stack.
- Un **objet** est sur le heap.
- Une **référence** vers l’objet est copiée sur la stack (variable locale) ou stockée dans d’autres objets.

Exemple :
```java
void f() {
  int x = 42;              // primitive locale (stack)
  Person p = new Person(); // p (référence) sur stack, Person sur heap
}
```

### 2.4 Escape analysis et allocations optimisées (JIT)
Le JIT peut parfois :
- **scalar replacement** (remplacer un objet par des champs locaux)
- **allocation elimination** (éviter d’allouer)

Mais :
- dépend du code,
- dépend du warm-up,
- dépend du compilateur JIT et des optimisations.

Conseil : **ne pas compter** systématiquement sur l’élimination d’allocations ; mesurer.

---

## 3) Garbage Collection : principes, algorithmes, pauses

### 3.1 Pourquoi un GC ?
En Java, on ne libère pas explicitement la mémoire des objets. Le GC :
- Identifie les objets **inaccessibles** (garbage)
- Récupère l’espace
- Réduit la fragmentation (selon algo)

### 3.2 Définition de “vivant” : reachability
Un objet est “vivant” s’il est **atteignable** depuis des racines (GC roots) :
- stacks des threads,
- classes statiques,
- JNI refs,
- registres internes JVM.

### 3.3 Approches classiques
- **Mark-Sweep** : marquage puis balayage. Peut fragmenter.
- **Mark-Compact** : marquage puis compactage (réduit la fragmentation, coût plus élevé).
- **Copying** : copie des survivants (souvent utilisé en young gen).

### 3.4 Générations : pourquoi ça marche souvent
Hypothèse empirique : **la majorité des objets meurent jeunes**.
Donc on segmente :
- **Young** : collectes fréquentes, rapides (minor GC)
- **Old** : collectes plus rares, plus coûteuses (major/full GC)

### 3.5 Pauses et latence
Un GC peut :
- être **Stop-The-World (STW)** : tous les threads applicatifs s’arrêtent,
- ou **concurrent** : travail en parallèle/concurrence avec l’application mais avec des phases STW.

Impact :
- **latence** (pauses visibles),
- **débit** (temps CPU consommé par le GC),
- **variabilité** (jitter).

### 3.6 Familles de GC (HotSpot, vue pragmatique)
> Les valeurs par défaut varient selon versions de Java et contextes.

- **G1 GC** (souvent le choix généraliste serveur)
  - heap découpé en **regions**
  - vise des pauses prévisibles (`MaxGCPauseMillis`)
  - bon compromis latence/débit

- **ZGC**
  - très faible latence, majoritairement concurrent
  - adapté aux heaps très grands et objectifs de latence stricts

- **Shenandoah**
  - similaire dans l’esprit (faible latence)

- **Parallel GC**
  - vise le débit (throughput), pauses plus longues mais globalement efficace pour batch

> Choix = workload + objectifs (latence p99, débit, CPU disponible, taille heap).

---

## 4) Tuning GC : à doser, mesurer, itérer

### 4.1 Règle d’or : tuning minimal, guidé par la mesure
- Changer **un paramètre** à la fois.
- Mesurer sur un environnement réaliste (charge, dataset, CPU, limites conteneur).
- Surveiller p50/p95/p99 de latence, CPU, allocation rate, GC time.

### 4.2 Dimensionnement du heap
- `-Xms` : taille initiale
- `-Xmx` : taille max

Recommandations courantes :
- Fixer `-Xms` proche de `-Xmx` pour éviter les redimensionnements (selon contexte).
- En conteneur (Kubernetes), vérifier la prise en compte de la mémoire limite.

### 4.3 Objectif de pause
Avec G1 par exemple :
- `-XX:MaxGCPauseMillis=200` (exemple)

Attention :
- Ce n’est pas une garantie.
- Trop agressif peut augmenter le travail GC, réduire le throughput.

### 4.4 Logs GC : votre première “sonde”
Activer des logs exploitables (Java 9+) :
```bash
-Xlog:gc*:file=gc.log:time,uptime,level,tags
```
À regarder :
- fréquence des minor GC,
- temps des pauses,
- full GC (à éviter en prod si possible),
- allocation rate,
- promotion failures.

### 4.5 Symptômes fréquents et réponses
- **Pauses fréquentes** : young trop petit, allocation rate élevé, trop d’objets temporaires.
- **Full GC** : old saturé, fuite mémoire, promotions excessives.
- **CPU GC élevé** : trop de travail concurrent, heap mal dimensionné, objets très vivant.

### 4.6 “Tuning à doser” : pourquoi trop toucher est risqué
- Complexité du GC moderne
- Effets secondaires non-intuitifs
- Variabilité selon JDK, flags, OS, conteneurs

Conclusion : commencer par :
1. **Mesurer**
2. **Corriger le code** (allocations inutiles)
3. Puis seulement ajuster quelques paramètres simples

---

## 5) Profiling & observabilité : logs GC, JFR, outils

### 5.1 Choisir l’outil selon le contexte
- **Prod** : privilégier outils à faible overhead (JFR, logs GC, métriques)
- **Préprod/dev** : profiler plus intrusif autorisé (allocation profiling intensif)

### 5.2 Java Flight Recorder (JFR)
JFR fournit des événements :
- allocations,
- pauses GC,
- contention/locks,
- threads,
- IO,
- compilation JIT.

Démarrer un enregistrement (exemples) :
```bash
# démarrage simple
jcmd <pid> JFR.start name=profile settings=profile filename=recording.jfr

# arrêt
jcmd <pid> JFR.stop name=profile
```

Analyser :
- JDK Mission Control (JMC)
- focus : allocation hotspots, GC pauses, threads bloquants.

### 5.3 Outils standards
- `jcmd` : couteau suisse (heap, JFR, flags, etc.)
- `jstat` : stats GC en continu
- `jmap` : heap dump (attention en prod)
- `jstack` : thread dumps

Exemples :
```bash
jcmd <pid> VM.flags
jcmd <pid> GC.heap_info
jstat -gc <pid> 1s
jcmd <pid> GC.class_histogram
```

### 5.4 Heap dump & analyse de fuite mémoire
À utiliser quand :
- old gen augmente continuellement,
- OOM,
- suspicion de rétention.

Production : préférer déclenchement contrôlé :
```bash
jcmd <pid> GC.heap_dump /path/heap.hprof
```
Analyse via Eclipse MAT / VisualVM / YourKit.

---

## 6) Bonnes pratiques de performance mémoire

### 6.1 Limiter les allocations inutiles
Sources fréquentes :
- création d’objets temporaires dans des boucles,
- `String` concaténations répétées,
- boxing/unboxing (`Integer` vs `int`),
- streams/lambdas sur chemins critiques (selon cas).

#### Exemple : concaténation de String
Mauvais (dans une boucle) :
```java
String s = "";
for (int i = 0; i < n; i++) {
  s += i; // crée beaucoup d'objets temporaires
}
```
Meilleur :
```java
StringBuilder sb = new StringBuilder(n * 2);
for (int i = 0; i < n; i++) {
  sb.append(i);
}
String s = sb.toString();
```

### 6.2 Dimensionner les collections (ArrayList, HashMap)
`ArrayList` : croissance progressive implique copies.
- Si taille connue/estimée, fournir une capacité :
```java
List<Item> items = new ArrayList<>(expectedSize);
```

`HashMap` : rehash coûteux.
- Pré-dimensionner et comprendre le `loadFactor`.

### 6.3 Choisir la bonne structure de données
- `ArrayList` vs `LinkedList` : `LinkedList` coûte cher en allocations et locality.
- Privilégier `ArrayDeque` pour une file/pile.
- Éviter les objets “wrapper” dans les chemins chauds (collections de primitives : envisager libs spécialisées si nécessaire).

### 6.4 Réduire la rétention involontaire
Causes typiques :
- caches non bornés,
- listeners non détachés,
- `static` qui garde des graphes d’objets,
- références implicites via classes internes.

Bonnes pratiques :
- caches bornés (taille/TTL),
- éviter `static` non nécessaire,
- utiliser `WeakReference`/`SoftReference` avec prudence.

### 6.5 Ajuster le niveau d’objets et de granularité
- regrouper des allocations
- éviter les “micro-objets” si cela crée une pression GC
- mais ne pas micro-optimiser sans mesure

### 6.6 IO et buffers
- Réutiliser des buffers lorsque c’est sûr.
- Attention aux `ByteBuffer.allocateDirect()` : hors heap mais peut causer OOM direct.

---

## 7) Ateliers / exercices (avec corrigés)

### Exercice 1 — Lire des logs GC et identifier un problème
**Objectif** : repérer minor GC trop fréquents et surallocation.
1. Activer les logs GC.
2. Lancer une charge (ou un test JMH / boucle de traitement).
3. Identifier : fréquence, temps de pause, full GC.

**Corrigé (attendu)** :
- Si minor GC toutes les quelques ms avec temps de pause non négligeable ⇒ allocation rate trop élevé.
- Actions : réduire allocations (StringBuilder, dimensionner collections), augmenter young/heap si pertinent.

### Exercice 2 — Allocation hotspot via JFR
**Objectif** : trouver la méthode qui alloue le plus.
1. Démarrer un JFR.
2. Exécuter un scénario.
3. Ouvrir dans JMC : onglet allocations.

**Corrigé (attendu)** :
- Top frames d’allocation.
- Corriger la construction d’objets inutiles (ex. `toString()` implicite, concaténations, streams).

### Exercice 3 — Fuite mémoire simple (cache non borné)
Code :
```java
class Cache {
  private static final Map<String, byte[]> cache = new HashMap<>();
  static byte[] get(String k) {
    return cache.computeIfAbsent(k, key -> new byte[1024 * 1024]);
  }
}
```
**Questions** :
- Pourquoi fuite/rétention ?
- Comment corriger ?

**Corrigé** :
- `static` + map non bornée ⇒ croissance infinie.
- Corriger : limite taille, TTL, eviction (Caffeine), ou supprimer `static`.

---

## 8) Checklist de diagnostic et mémo commandes

### 8.1 Checklist
- [ ] La mémoire process augmente ? Heap ou natif ?
- [ ] GC time (%) élevé ? Pauses visibles p99 ?
- [ ] Full GC présents ?
- [ ] Allocation rate trop élevé ?
- [ ] Old gen augmente continuellement (rétention) ?
- [ ] Metaspace augmente (classloaders) ?
- [ ] Direct buffers (off-heap) ?

### 8.2 Mémo commandes
```bash
# Flags et infos heap
jcmd <pid> VM.flags
jcmd <pid> GC.heap_info

# Histogramme des classes
jcmd <pid> GC.class_histogram

# JFR
jcmd <pid> JFR.start name=rec settings=profile filename=rec.jfr
jcmd <pid> JFR.stop name=rec

# Heap dump
jcmd <pid> GC.heap_dump /tmp/heap.hprof

# Stats GC
jstat -gc <pid> 1s

# Thread dump
jstack <pid>
```

---

## Annexes (pistes pour aller plus loin)
- Lire/rechercher : “GC tuning is workload dependent”
- Tester vos optimisations avec **JMH** (éviter les micro-benchmarks biaisés)
- Garder en tête : performance = **latence** + **débit** + **stabilité**, pas juste “plus rapide”
