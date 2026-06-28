# 8) Collections & Generics (Java)

> Public : développeurs (ex. .NET/C#) découvrant/approfondissant Java.
>
> Objectif : maîtriser les collections Java, leurs implémentations courantes, et l’usage des génériques + wildcards pour écrire du code **type-safe**, lisible et performant.

---

## Plan de la formation

1. **Introduction & motivation**
   - Pourquoi des collections ?
   - Différences clés vs tableaux
   - Notions : ordre, unicité, clé/valeur
2. **Panorama de l’API Java Collections Framework**
   - Interfaces principales : `List`, `Set`, `Map`
   - Itération : `Iterator`, boucle `for-each`, streams
3. **Collections : `List` (ordre)**
   - Propriétés et cas d’usage
   - Implémentations : `ArrayList` et `LinkedList`
   - Complexités (Big-O) et bonnes pratiques
4. **Collections : `Set` (unicité)**
   - Règles d’unicité, `equals`/`hashCode`
   - Implémentations : `HashSet` et `TreeSet`
   - Notion d’ordre naturel/comparateur
5. **Collections : `Map` (clé/valeur)**
   - Propriétés et cas d’usage
   - Implémentations : `HashMap` et `TreeMap`
   - Parcours et opérations fréquentes (`getOrDefault`, `computeIfAbsent`…)
6. **Génériques (Generics)**
   - Pourquoi : sécurité de type, élimination des casts
   - Exemple : `List<String>`
   - Types génériques courants : `List<T>`, `Map<K,V>`
7. **Wildcards**
   - `? extends T` (lecture)
   - `? super T` (écriture)
   - Règle PECS : *Producer Extends, Consumer Super*
8. **Atelier / exercices corrigés**
   - Choisir la bonne structure de données
   - Refactoring sans casts
   - Réécriture avec wildcards

---

## 1) Introduction & motivation

En Java, les collections sont la base de la plupart des applications : manipulation de séries d’objets, indexation par clé, déduplication, tri, etc.

### Tableaux vs collections

- **Tableaux** (`T[]`) : taille fixe, type homogène, performances bonnes, mais peu flexibles.
- **Collections** : taille dynamique, API riche (recherche, tri, filtrage, opérations en masse), plusieurs sémantiques (ordre/unicité/clé-valeur).

### Trois sémantiques majeures

- **List** : **ordre** + doublons autorisés
- **Set** : **unicité** (pas de doublon)
- **Map** : **clé/valeur** (clé unique ↔ valeur)

---

## 2) Panorama : Java Collections Framework

### Interfaces principales

- `List<E>` : séquence ordonnée d’éléments
- `Set<E>` : ensemble d’éléments uniques
- `Map<K,V>` : dictionnaire clé → valeur

### Itérer sur une collection

```java
List<String> names = List.of("Ada", "Linus", "Grace");

for (String n : names) {
    System.out.println(n);
}
```

Avec `Iterator` : utile quand on veut supprimer durant l’itération.

```java
Iterator<String> it = names.iterator();
while (it.hasNext()) {
    String n = it.next();
}
```

---

## 3) `List` : ordre

### Propriétés

- Conserve l’**ordre d’insertion** (en général)
- Accès par **index** (0..n-1)
- Autorise les **doublons**

### Cas d’usage typiques

- Résultats ordonnés (tri, pagination)
- Historique d’événements
- Datasets où l’index est utile

---

### 3.1 `ArrayList` (tableau dynamique)

`ArrayList` est l’implémentation la plus courante.

```java
List<String> todo = new ArrayList<>();
todo.add("Learn generics");
todo.add("Practice collections");

String first = todo.get(0); // O(1)
```

#### Caractéristiques

- Stockage contigu (comme un tableau)
- Redimensionnement automatique

#### Complexités (approximatives)

| Opération | Coût | Notes |
|---|---:|---|
| `get(i)` | O(1) | accès direct |
| `add(e)` (fin) | amorti O(1) | redimensionnement occasionnel |
| `add(i,e)` | O(n) | décale les éléments |
| `remove(i)` | O(n) | décale |

#### À retenir

- Très bon choix par défaut
- Excellent pour : lecture par index, ajout en fin

---

### 3.2 `LinkedList` (liste chaînée)

```java
List<String> queue = new LinkedList<>();
queue.add("A");
queue.add("B");
```

#### Caractéristiques

- Nœuds chaînés (pointeurs précédent/suivant)
- Bon pour des insertions/suppressions au début/fin… **si on a déjà le nœud**

#### Complexités (approximatives)

| Opération | Coût | Notes |
|---|---:|---|
| `get(i)` | O(n) | il faut parcourir |
| `addFirst/ addLast` | O(1) | opérations deque |
| `add(i,e)` | O(n) | parcours + insertion |

#### À retenir

- Souvent sur-utilisée : si vous faites beaucoup de `get(i)`, **évitez**.
- Plutôt adaptée pour des usages type **queue/deque**.

> Remarque : En Java, si vous avez un besoin de queue, regardez aussi `ArrayDeque` (souvent plus performant que `LinkedList`).

---

## 4) `Set` : unicité

### Propriétés

- Un élément ne peut apparaître qu’une fois (unicité)
- S’appuie sur :
  - `equals()` pour comparer
  - `hashCode()` pour optimiser (dans les sets basés sur hash)

### Quand utiliser un `Set`

- Déduplication (`distinct`)
- Test d’appartenance rapide (`contains`)
- Gestion d’identifiants uniques

---

### 4.1 `HashSet`

```java
Set<String> tags = new HashSet<>();
tags.add("java");
tags.add("collections");
tags.add("java"); // ignoré, déjà présent

System.out.println(tags.contains("java"));
```

#### Caractéristiques

- Non ordonné (l’ordre n’est pas garanti)
- Performant pour `add/contains/remove` en moyenne

#### Complexités (moyenne)

| Opération | Coût |
|---|---:|
| `add` | O(1) |
| `contains` | O(1) |
| `remove` | O(1) |

#### Attention : `equals` / `hashCode`

Si vous mettez des objets métiers dans un `HashSet`, vous devez définir correctement `equals` et `hashCode`.

```java
class User {
    private final String email;

    User(String email) { this.email = email; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof User other)) return false;
        return email.equals(other.email);
    }

    @Override
    public int hashCode() {
        return email.hashCode();
    }
}
```

---

### 4.2 `TreeSet` (ensemble trié)

`TreeSet` maintient les éléments **triés** (ordre naturel ou `Comparator`).

```java
Set<Integer> sorted = new TreeSet<>();
sorted.add(10);
sorted.add(3);
sorted.add(7);
// [3, 7, 10]
```

#### Caractéristiques

- Tri permanent (coût log(n))
- Offre des opérations : `first()`, `last()`, `higher()`, `lower()`, sous-ensembles…

#### Complexités

| Opération | Coût |
|---|---:|
| `add` | O(log n) |
| `contains` | O(log n) |
| `remove` | O(log n) |

#### Tri personnalisé

```java
Set<String> byLength = new TreeSet<>(Comparator.comparingInt(String::length));
byLength.add("aaa");
byLength.add("b");
byLength.add("cc");
```

> Attention : si le comparateur considère deux éléments comme “égaux” (compare == 0), l’un des deux ne sera pas stocké.

---

## 5) `Map` : clé/valeur

### Propriétés

- Clés **uniques**
- Valeurs duplicables
- Les opérations se font principalement par clé : `get(key)`, `put(key,value)`

### Cas d’usage

- Index par identifiant
- Cache, lookup rapide
- Association (email → user, code → libellé, etc.)

---

### 5.1 `HashMap`

```java
Map<String, Integer> scores = new HashMap<>();
scores.put("Ada", 10);
scores.put("Linus", 7);

int ada = scores.get("Ada");
Integer missing = scores.get("Grace"); // null si absent
```

#### Opérations utiles

`getOrDefault` :

```java
int grace = scores.getOrDefault("Grace", 0);
```

`putIfAbsent` :

```java
scores.putIfAbsent("Grace", 5);
```

`computeIfAbsent` (très utile pour initialiser une liste) :

```java
Map<String, List<String>> errorsByField = new HashMap<>();
errorsByField.computeIfAbsent("email", k -> new ArrayList<>())
             .add("Invalid format");
```

#### Complexités (moyennes)

| Opération | Coût |
|---|---:|
| `get/put/remove` | O(1) |

#### Parcourir une map

```java
for (Map.Entry<String, Integer> e : scores.entrySet()) {
    System.out.println(e.getKey() + "=" + e.getValue());
}
```

---

### 5.2 `TreeMap` (map triée par clé)

```java
Map<String, Integer> sortedScores = new TreeMap<>();
sortedScores.put("Linus", 7);
sortedScores.put("Ada", 10);
// clés triées : Ada, Linus
```

#### Caractéristiques

- Clés triées (ordre naturel ou `Comparator`)
- Utile pour : exports triés, plages de clés (`subMap`, `headMap`, `tailMap`)

#### Complexités

| Opération | Coût |
|---|---:|
| `get/put/remove` | O(log n) |

---

## 6) Generics (génériques)

Les génériques apportent :

- **Sécurité de type** à la compilation
- Moins de casts
- API plus expressive

### Exemple : `List<String>`

```java
List<String> names = new ArrayList<>();
names.add("Ada");

String n = names.get(0); // pas de cast
```

Sans génériques (raw type), on perd la sécurité :

```java
List raw = new ArrayList();
raw.add("Ada");
raw.add(123);

String x = (String) raw.get(1); // ClassCastException à l'exécution
```

### Génériques sur `Map<K,V>`

```java
Map<String, Integer> ages = new HashMap<>();
ages.put("Ada", 36);
Integer age = ages.get("Ada");
```

---

## 7) Wildcards

Les wildcards servent à exprimer de la variance : accepter “une famille de types génériques”.

Rappel : en Java, `List<Dog>` **n’est pas** un sous-type de `List<Animal>` (invariance).

### 7.1 `? extends T` : lecture (Producer)

Utilisez `? extends T` quand la collection **produit** des `T` (vous allez lire des éléments).

```java
static double sum(List<? extends Number> values) {
    double s = 0;
    for (Number n : values) {
        s += n.doubleValue();
    }
    // values.add(1); // interdit : on ne sait pas quel sous-type exact
    return s;
}

sum(List.of(1, 2, 3));            // List<Integer>
sum(List.of(1.5, 2.5));           // List<Double>
```

**Pourquoi l’ajout est interdit ?**

`List<? extends Number>` peut être une `List<Integer>` ; ajouter un `Double` casserait le contrat. Java interdit donc `add` (sauf `null`).

---

### 7.2 `? super T` : écriture (Consumer)

Utilisez `? super T` quand la collection **consomme** des `T` (vous allez écrire/ajouter des éléments).

```java
static void addDefaults(List<? super Integer> target) {
    target.add(0);
    target.add(1);
}

List<Number> nums = new ArrayList<>();
addDefaults(nums);
```

Ici, `target` peut être une `List<Integer>`, `List<Number>` ou `List<Object>`. Dans tous les cas, ajouter un `Integer` est sûr.

**Lecture limitée** :

```java
Object x = target.get(0); // ok
// Integer y = target.get(0); // pas garanti
```

---

### 7.3 Règle PECS

- **P**roducer → `extends`
- **C**onsumer → `super`

> Si votre méthode ne fait que lire : `? extends T`.
> Si votre méthode ne fait qu’écrire : `? super T`.
> Si elle fait les deux, préférez souvent `List<T>` invariant (ou repensez l’API).

---

## 8) Atelier / exercices (avec corrigés)

### Exercice 1 — Choisir la bonne structure

**Énoncé** :
- Vous stockez des identifiants utilisateurs uniques, et vous faites beaucoup de `contains`.

**Solution** :
- `HashSet<String>` (unicité + membership rapide)

---

### Exercice 2 — Compter les occurrences

**Énoncé** :
- À partir d’une liste de mots, produire une map `mot → nombre`.

```java
List<String> words = List.of("java", "java", "collections", "map", "map");
```

**Corrigé** :

```java
Map<String, Integer> counts = new HashMap<>();
for (String w : words) {
    counts.put(w, counts.getOrDefault(w, 0) + 1);
}
```

Variante moderne avec `merge` :

```java
Map<String, Integer> counts = new HashMap<>();
for (String w : words) {
    counts.merge(w, 1, Integer::sum);
}
```

---

### Exercice 3 — Grouper des valeurs

**Énoncé** :
- Grouper une liste d’erreurs par champ (`field → List<message>`)

**Corrigé** :

```java
record Error(String field, String message) {}

List<Error> errors = List.of(
    new Error("email", "Invalid format"),
    new Error("email", "Required"),
    new Error("age", "Must be >= 18")
);

Map<String, List<String>> byField = new HashMap<>();
for (Error e : errors) {
    byField.computeIfAbsent(e.field(), k -> new ArrayList<>())
           .add(e.message());
}
```

---

### Exercice 4 — Wildcards (`extends` / `super`)

**Énoncé** :
- Écrire une méthode qui copie tous les éléments d’une liste source vers une liste destination.

**Corrigé** (API classique) :

```java
static <T> void copy(List<? extends T> src, List<? super T> dst) {
    for (T x : src) {
        dst.add(x);
    }
}

List<Integer> src = List.of(1, 2, 3);
List<Number> dst = new ArrayList<>();
copy(src, dst);
```

Ici :
- `src` est un **producer** → `extends`
- `dst` est un **consumer** → `super`

---

## Synthèse

- **List** : ordre, index, doublons → `ArrayList` par défaut, `LinkedList` plutôt pour deque/queue.
- **Set** : unicité → `HashSet` rapide, `TreeSet` trié.
- **Map** : clé/valeur → `HashMap` rapide, `TreeMap` trié.
- **Generics** : type-safety (`List<String>`, `Map<K,V>`).
- **Wildcards** :
  - `? extends T` : lecture (producer)
  - `? super T` : écriture (consumer)
  - **PECS**

---

## Annexes (références utiles)

- Java Collections Framework (Oracle) : https://docs.oracle.com/javase/8/docs/technotes/guides/collections/overview.html
- `List`, `Set`, `Map` Javadoc :
  - https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/List.html
  - https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/Set.html
  - https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/Map.html
