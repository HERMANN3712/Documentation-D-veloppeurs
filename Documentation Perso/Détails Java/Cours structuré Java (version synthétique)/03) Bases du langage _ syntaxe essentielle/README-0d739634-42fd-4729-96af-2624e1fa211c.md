# Formation Java — 3) Bases du langage : syntaxe essentielle

> **Public visé** : développeurs (notamment venant de .NET/C#) débutant ou consolidant leurs bases en Java.
>
> **Objectif** : comprendre la syntaxe minimale et les notions essentielles (point d’entrée, types primitifs, références, `String`, comparaisons, inférence de type `var`, constantes `final`) pour écrire du code Java correct et lisible.

---

## Plan de la formation

1. **Hello World et structure minimale d’un programme**
   - Classe, méthode `main`, signature
   - Affichage console (`System.out.println`)
   - Conventions de nommage et fichiers
2. **Types primitifs (valeurs)**
   - Liste des primitifs : `byte`, `short`, `int`, `long`, `float`, `double`, `char`, `boolean`
   - Valeurs littérales, suffixes (`L`, `f`), séparateurs (`_`)
   - Portée / usage typique
3. **Types références**
   - `String`, tableaux, objets
   - Notion de référence vs valeur
4. **`String` : immutabilité et conséquences**
   - Pourquoi `String` est immutable
   - Concaténation, `StringBuilder`
5. **Comparer des objets : `==` vs `equals()`**
   - `==` (identité / référence)
   - `equals()` (égalité logique)
   - Cas particulier des `String` (pool) et bonnes pratiques
6. **Inférence de type avec `var` (Java 10+)**
   - Quand l’utiliser
   - Limites et conventions
7. **Constantes / non-réassignation avec `final`**
   - `final` sur variables locales, champs, paramètres
   - Constantes `static final`
8. **Atelier guidé**
   - Exercices progressifs
   - Vérifications et corrections

---

## 1) Hello World et structure minimale d’un programme

### 1.1. Exemple minimal

Voici un « Hello World » dans un style proche de ce que vous avez fourni (note : en Java, la convention veut que la classe soit en **PascalCase** et que le fichier ait le même nom que la classe publique) :

```java
public class Main {
    public static void main(String[] args) {
        System.out.println("Hello");
    }
}
```

### 1.2. Décomposition ligne par ligne

- `public class Main { ... }`
  - Déclare une **classe** nommée `Main`.
  - En Java, tout le code (ou presque) vit dans des classes.

- `public static void main(String[] args)`
  - **Point d’entrée** de l’application.
  - `public` : accessible depuis la JVM.
  - `static` : la méthode appartient à la classe, pas à une instance.
  - `void` : ne renvoie rien.
  - `String[] args` : arguments de ligne de commande.

- `System.out.println("Hello");`
  - Affiche une ligne dans la console.
  - `System` : classe utilitaire.
  - `out` : flux de sortie standard (`PrintStream`).
  - `println` : imprime puis ajoute un retour à la ligne.

### 1.3. Compilation et exécution (rappel)

Si vous disposez du JDK :

```bash
javac Main.java
java Main
```

> Dans les environnements modernes (IDE, build tools), cela peut être caché mais comprendre ce modèle aide au diagnostic.

---

## 2) Types primitifs (valeurs)

Java distingue **types primitifs** (valeurs) et **types référence** (références vers des objets).

### 2.1. Les 8 types primitifs

| Type | Taille | Valeurs | Usage typique |
|------|--------|---------|---------------|
| `byte` | 8 bits | -128 à 127 | I/O binaire, buffers |
| `short` | 16 bits | -32768 à 32767 | rare (mémoire) |
| `int` | 32 bits | ~ -2.1B à 2.1B | entier par défaut |
| `long` | 64 bits | ~ -9e18 à 9e18 | timestamps, ids, gros calculs |
| `float` | 32 bits | IEEE 754 | calculs flottants où précision moins critique |
| `double` | 64 bits | IEEE 754 | flottant par défaut |
| `char` | 16 bits | unité UTF-16 | caractères (attention aux emojis/surrogates) |
| `boolean` | JVM | `true`/`false` | conditions |

### 2.2. Littéraux et détails pratiques

#### Entiers

```java
int a = 42;
long b = 42L;      // suffixe L recommandé
int million = 1_000_000; // séparateurs (lisibilité)
```

> Sans `L`, un littéral entier est un `int` par défaut. `42L` force un `long`.

#### Flottants

```java
double pi = 3.14159; // double par défaut
float f = 3.14f;     // suffixe f obligatoire pour un float
```

#### `char`

```java
char c1 = 'A';
char c2 = '\n';
char c3 = '\u03A9'; // Ω en Unicode
```

> `char` représente une **unité UTF-16**, pas forcément un « caractère utilisateur » complet (certaines lettres/emoji utilisent deux `char`).

#### `boolean`

```java
boolean enabled = true;
boolean isEmpty = false;
```

---

## 3) Types références

### 3.1. Qu’est-ce qu’un type référence ?

Un type référence ne contient pas directement la valeur, mais une **référence** vers un objet (en mémoire managée) :

- `String`
- tableaux (`int[]`, `String[]`, …)
- objets de classes (`Person`, `List`, `Map`, …)

Exemple :

```java
int x = 5;              // primitif : valeur directe
String s = "hello";     // référence vers un objet String
int[] arr = {1, 2, 3};  // référence vers un tableau
```

### 3.2. Valeur `null`

Les références peuvent valoir `null` (absence de référence) :

```java
String name = null;
// name.length(); // NullPointerException si décommenté
```

> Bonne pratique : valider les entrées, utiliser `Objects.requireNonNull`, et/ou `Optional` selon le contexte (on y reviendra dans d’autres modules).

---

## 4) `String` : immutabilité

### 4.1. `String` est immutable

Une `String` en Java ne peut pas être modifiée après création.

```java
String s = "Hello";
s = s + " World";
```

Ici, `s + " World"` crée **une nouvelle** `String`. La variable `s` est réassignée vers le nouvel objet.

### 4.2. Pourquoi cette immutabilité ?

- **Sécurité** (ex. chaînes utilisées comme clés, chemins, etc.)
- **Thread-safety** implicite
- **Optimisations** (pool de chaînes, partage d’instances)

### 4.3. Concaténation : attention aux boucles

Dans une boucle, concaténer avec `+` peut créer beaucoup d’objets intermédiaires.

```java
String result = "";
for (int i = 0; i < 10_000; i++) {
    result += i; // potentiellement coûteux
}
```

Préférez `StringBuilder` :

```java
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 10_000; i++) {
    sb.append(i);
}
String result = sb.toString();
```

---

## 5) Comparer des objets : `==` vs `equals()`

### 5.1. `==` compare l’identité (la référence)

Pour les types références, `==` teste si deux variables pointent vers **le même objet**.

```java
String a = new String("test");
String b = new String("test");

System.out.println(a == b);      // false (références différentes)
System.out.println(a.equals(b)); // true  (contenu identique)
```

### 5.2. `equals()` compare l’égalité logique (contenu)

- Les classes peuvent redéfinir `equals()`.
- `String` redéfinit `equals()` pour comparer les caractères.

### 5.3. Cas particulier des `String` et le pool

Les littéraux peuvent être internés (pool de chaînes) :

```java
String s1 = "abc";
String s2 = "abc";

System.out.println(s1 == s2); // souvent true (même instance internée)
```

Mais ne vous basez **jamais** sur ce comportement pour la logique métier : utilisez `equals()`.

### 5.4. Bonnes pratiques

- Toujours comparer les chaînes avec `equals()` (ou `equalsIgnoreCase()` si pertinent) :

```java
if ("admin".equals(role)) {
    // pattern null-safe : évite NPE si role == null
}
```

- Si vous écrivez vos propres objets : implémenter `equals()` (et `hashCode()`) correctement (module dédié recommandé).

---

## 6) Inférence de type avec `var` (Java 10+)

### 6.1. Principe

`var` permet au compilateur d’inférer le type d’une **variable locale** à partir de l’expression d’initialisation.

```java
var message = "Hello";   // message est de type String
var count = 10;          // count est de type int
var list = new java.util.ArrayList<String>(); // type inféré
```

### 6.2. Limites

- `var` est autorisé uniquement pour des **variables locales** (pas pour champs, paramètres — hors évolutions spécifiques).
- Il faut une initialisation explicite :

```java
// var x; // interdit : pas d’initialisation
```

- Lisibilité : si le type n’est pas évident, explicitez-le.

### 6.3. Recommandations

- OK quand le type est évident à droite (`new HashMap<>()`, `"..."`, `List.of(...)`).
- Pas OK si cela masque un type important (ex. API complexe, types génériques imbriqués) et nuit à la compréhension.

---

## 7) Constantes et non-réassignation avec `final`

### 7.1. `final` sur une variable

- Pour un **primitif** : la valeur ne peut plus changer.

```java
final int maxRetries = 3;
// maxRetries = 4; // erreur de compilation
```

- Pour une **référence** : la référence ne peut plus être réassignée, mais l’objet peut rester mutable.

```java
final java.util.List<String> names = new java.util.ArrayList<>();
names.add("Alice");  // autorisé : on modifie l'objet
// names = new ArrayList<>(); // interdit : réassignation
```

### 7.2. Constantes de classe : `static final`

Convention : **UPPER_SNAKE_CASE**.

```java
public class Config {
    public static final int DEFAULT_TIMEOUT_MS = 3_000;
}
```

### 7.3. `final` et intent

`final` sert à :
- exprimer une intention (valeur stable)
- sécuriser du code
- faciliter certaines optimisations

---

## 8) Atelier guidé (exercices)

### Exercice 1 — Hello World + args

1. Créez une classe `Main`.
2. Affichez `Hello`.
3. Si un argument est fourni, affichez `Hello <arg>`.

**Solution possible** :

```java
public class Main {
    public static void main(String[] args) {
        if (args.length > 0) {
            System.out.println("Hello " + args[0]);
        } else {
            System.out.println("Hello");
        }
    }
}
```

### Exercice 2 — Primitifs

1. Déclarez un `int` et un `long`.
2. Utilisez des séparateurs `_` dans un littéral.
3. Déclarez un `float` correctement.

**Exemple** :

```java
int users = 120_000;
long timestamp = 1_700_000_000L;
float ratio = 0.75f;
```

### Exercice 3 — `==` vs `equals()`

1. Créez deux `String` via `new String("x")`.
2. Comparez-les avec `==` puis `equals()`.
3. Expliquez la différence.

### Exercice 4 — `final`

1. Créez une liste `final`.
2. Ajoutez un élément.
3. Tentez de réassigner la liste (constatez l’erreur).

---

## Résumé

- **Hello World** : classe + `public static void main(String[] args)`.
- **Primitifs** : `byte short int long float double char boolean` (valeurs).
- **Références** : `String`, tableaux, objets (+ `null`).
- **String immutable** : chaque modification crée une nouvelle instance.
- **Comparaison** : `==` compare la référence, `equals()` compare le contenu (logique).
- **`var` (Java 10+)** : inférence pour variables locales initialisées.
- **`final`** : empêche la réassignation (et `static final` pour constantes).

---

## Annexes — Correspondances utiles pour un profil C#/.NET

| Concept | C# | Java |
|--------|----|------|
| Point d’entrée | `static void Main(string[] args)` | `public static void main(String[] args)` |
| Chaîne | `string` (immutable) | `String` (immutable) |
| Concaténation en boucle | `StringBuilder` | `StringBuilder` |
| Référence vs valeur | `class` vs `struct` | références vs primitifs (et wrappers) |
| Constante | `const` / `readonly` | `static final` / `final` |

> Note : Java possède aussi des **wrappers** (`Integer`, `Long`, etc.) et l’**autoboxing** — souvent abordés juste après ce module.
