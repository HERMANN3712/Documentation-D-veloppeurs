# 4) Contrôle de flux (Java)

> **Objectif** : maîtriser les structures de contrôle de flux en Java pour écrire un code clair, robuste et maintenable.

## Public visé & prérequis
- Développeurs (débutants à intermédiaires) ayant des bases en Java (types, variables, opérateurs)
- Savoir compiler/exécuter une application Java (IDE ou ligne de commande)

## Durée suggérée
- **2h à 3h** (avec exercices)

## Compétences visées
- Choisir la bonne structure de contrôle selon le besoin
- Écrire des conditions lisibles (`if/else`, `switch`)
- Mettre en place des boucles (`for`, `while`, `do-while`, `for-each`)
- Éviter les pièges courants (boucles infinies, erreurs de limites, `switch` mal utilisé)

---

# Plan de la formation
1. Introduction au contrôle de flux
2. `if / else` (conditions)
3. `switch` classique (instruction)
4. `switch` moderne (expression)
5. Boucle `for`
6. Boucle `while`
7. Boucle `do-while`
8. Boucle `for-each`
9. Bonnes pratiques & anti-patterns
10. Exercices (avec corrigés)

---

# 1. Introduction au contrôle de flux
Le **contrôle de flux** désigne l’ensemble des mécanismes qui permettent de **diriger l’exécution** d’un programme :
- exécuter un bloc **si une condition** est vraie (`if`, `switch`)
- **répéter** une suite d’instructions (`for`, `while`, `do-while`, `for-each`)

Dans les exemples ci-dessous, on utilise Java 17+ (les idées restent valables en Java 8+ sauf pour le *switch expression*).

---

# 2. `if / else`

## 2.1. Syntaxe
```java
if (condition) {
    // ...
} else if (autreCondition) {
    // ...
} else {
    // ...
}
```

- La condition doit être un **booléen** (`true`/`false`).
- Les accolades `{}` sont recommandées, même pour un bloc d’une ligne (meilleure lisibilité, moins d’erreurs).

## 2.2. Exemple simple
```java
int age = 20;

if (age >= 18) {
    System.out.println("Majeur");
} else {
    System.out.println("Mineur");
}
```

## 2.3. Exemple avec `else if`
```java
int score = 72;

if (score >= 90) {
    System.out.println("A");
} else if (score >= 80) {
    System.out.println("B");
} else if (score >= 70) {
    System.out.println("C");
} else {
    System.out.println("D");
}
```

## 2.4. Erreurs fréquentes
### Confusion `=` vs `==`
```java
// Mauvais (ne compile pas car condition attend boolean)
// if (x = 10) { ... }

// Bon
if (x == 10) {
}
```

### Comparaison d’objets (String)
```java
String a = "ok";
String b = new String("ok");

// Mauvais (compare les références)
if (a == b) {
}

// Bon (compare le contenu)
if (a.equals(b)) {
}
```

## 2.5. Opérateur ternaire (complément)
Utile pour une affectation simple.
```java
int n = 5;
String parity = (n % 2 == 0) ? "even" : "odd";
```
> À éviter si l’expression devient difficile à lire.

---

# 3. `switch` classique (instruction)

## 3.1. Quand utiliser `switch` ?
- Lorsque vous comparez **une même valeur** à plusieurs cas possibles.
- Alternative plus lisible qu’une succession de `else if`.

## 3.2. Syntaxe (version classique)
```java
switch (valeur) {
    case 1:
        // ...
        break;
    case 2:
        // ...
        break;
    default:
        // ...
}
```

### Le rôle de `break`
Sans `break`, Java continue dans les `case` suivants (*fall-through*). C’est parfois voulu, mais souvent source de bugs.

## 3.3. Exemple
```java
int day = 3;
String name;

switch (day) {
    case 1:
        name = "Monday";
        break;
    case 2:
        name = "Tuesday";
        break;
    case 3:
        name = "Wednesday";
        break;
    default:
        name = "Unknown";
}

System.out.println(name);
```

## 3.4. Regrouper plusieurs `case`
```java
int n = 2;

switch (n) {
    case 1:
    case 2:
        System.out.println("Small");
        break;
    case 3:
        System.out.println("Medium");
        break;
    default:
        System.out.println("Other");
}
```

---

# 4. `switch` moderne (expression)
Depuis Java 14+ (standard), `switch` peut être utilisé comme une **expression** qui retourne une valeur.

## 4.1. Avantages
- Plus concis
- Pas de `break`
- Moins d’erreurs de *fall-through*
- Très adapté aux affectations

## 4.2. Syntaxe avec flèche `->`
> Exemple demandé :
```java
int n = 2;
String label = switch (n) {
    case 1, 2 -> "small";
    case 3 -> "medium";
    default -> "other";
};

System.out.println(label);
```

- Chaque `case` produit une valeur.
- Le `switch` finit par un `;` car c’est une expression.

## 4.3. Bloc de code et `yield`
Si un cas nécessite plusieurs instructions, on utilise un bloc `{}` et `yield`.
```java
int n = 3;

String label = switch (n) {
    case 1, 2 -> {
        System.out.println("Small value detected");
        yield "small";
    }
    case 3 -> "medium";
    default -> {
        String computed = "other:" + n;
        yield computed;
    }
};
```

## 4.4. Bonnes pratiques
- Couvrir un `default` (ou gérer exhaustivement les cas)
- Préférer le switch expression pour les affectations simples
- Garder les cas courts pour préserver la lisibilité

---

# 5. Boucle `for`

## 5.1. Syntaxe
```java
for (initialisation; condition; incrément) {
    // ...
}
```

- **Initialisation** : exécutée une seule fois au début
- **Condition** : testée avant chaque itération
- **Incrément** : exécuté après chaque itération

## 5.2. Exemple : compter de 0 à 4
```java
for (int i = 0; i < 5; i++) {
    System.out.println(i);
}
```

## 5.3. Exemple : parcourir un tableau par index
```java
int[] values = {10, 20, 30};

for (int i = 0; i < values.length; i++) {
    System.out.println("values[" + i + "]=" + values[i]);
}
```

## 5.4. Pièges courants
- Mauvaise condition (`i <= length` au lieu de `i < length`) → `ArrayIndexOutOfBoundsException`.

---

# 6. Boucle `while`

## 6.1. Syntaxe
```java
while (condition) {
    // ...
}
```

La condition est testée **avant** d’entrer dans le bloc : la boucle peut s’exécuter **0 fois**.

## 6.2. Exemple : lire jusqu’à une valeur
```java
int n = 1;

while (n < 100) {
    n = n * 2;
}

System.out.println(n); // 128
```

## 6.3. Boucles infinies
Une boucle `while (true)` est valide si on a une condition de sortie claire (`break`, `return`, exception).

---

# 7. Boucle `do-while`

## 7.1. Syntaxe
```java
do {
    // ...
} while (condition);
```

Contrairement à `while`, le bloc s’exécute **au moins une fois**.

## 7.2. Exemple : saisie avec validation (simplifié)
```java
import java.util.Scanner;

Scanner sc = new Scanner(System.in);
int value;

do {
    System.out.print("Enter a number between 1 and 10: ");
    value = sc.nextInt();
} while (value < 1 || value > 10);

System.out.println("OK: " + value);
```

---

# 8. Boucle `for-each`

## 8.1. But
Parcourir une collection ou un tableau **sans gérer explicitement l’index**.

## 8.2. Syntaxe
```java
for (Type element : iterable) {
    // ...
}
```

## 8.3. Exemple : tableau
```java
int[] values = {10, 20, 30};

for (int v : values) {
    System.out.println(v);
}
```

## 8.4. Exemple : liste
```java
import java.util.List;

List<String> names = List.of("Ada", "Linus", "Grace");

for (String name : names) {
    System.out.println(name.toUpperCase());
}
```

## 8.5. Limites
- Pas d’index directement (on peut le reconstruire mais ce n’est pas l’objectif)
- Modification structurelle de certaines collections pendant l’itération → `ConcurrentModificationException`

---

# 9. Bonnes pratiques & anti-patterns

## 9.1. Lisibilité
- Préférer des conditions simples et nommées :
```java
boolean isAdult = age >= 18;
if (isAdult) { ... }
```

## 9.2. Éviter les niveaux d’imbrication excessifs
- Utiliser des **retours anticipés** (*guard clauses*) :
```java
if (user == null) {
    throw new IllegalArgumentException("user required");
}
// suite logique
```

## 9.3. Choisir la bonne boucle
- `for` : nombre d’itérations connu / index requis
- `while` : condition d’arrêt basée sur un état qui évolue
- `do-while` : au moins une exécution obligatoire
- `for-each` : parcours simple, intention claire

## 9.4. `switch` : privilégier la clarté
- `switch` expression pour affecter une valeur
- `switch` classique si vous avez des effets de bord multi-lignes (même si `yield` gère aussi ce cas)

---

# 10. Exercices (avec corrigés)

## Exercice 1 — `if/else`
Écrire une fonction qui retourne `"positive"`, `"negative"` ou `"zero"` selon la valeur.

**Énoncé**
```java
static String signLabel(int n) {
    // TODO
}
```

**Corrigé**
```java
static String signLabel(int n) {
    if (n > 0) {
        return "positive";
    } else if (n < 0) {
        return "negative";
    } else {
        return "zero";
    }
}
```

---

## Exercice 2 — `switch` moderne
Retourner un libellé pour une taille.
- 1,2 → `"small"`
- 3 → `"medium"`
- sinon → `"other"`

**Corrigé**
```java
static String sizeLabel(int n) {
    return switch (n) {
        case 1, 2 -> "small";
        case 3 -> "medium";
        default -> "other";
    };
}
```

---

## Exercice 3 — `for`
Calculer la somme des entiers de 1 à `n` (inclus).

**Corrigé**
```java
static int sumTo(int n) {
    int sum = 0;
    for (int i = 1; i <= n; i++) {
        sum += i;
    }
    return sum;
}
```

---

## Exercice 4 — `while`
Trouver le plus petit `p` tel que `2^p >= target`.

**Corrigé**
```java
static int minPowerOfTwoExponent(int target) {
    int p = 0;
    int v = 1;
    while (v < target) {
        v *= 2;
        p++;
    }
    return p;
}
```

---

## Exercice 5 — `do-while`
Simuler un menu console qui s’affiche au moins une fois et s’arrête quand l’utilisateur choisit `0`.

**Corrigé (simplifié)**
```java
import java.util.Scanner;

static void menuLoop() {
    Scanner sc = new Scanner(System.in);
    int choice;

    do {
        System.out.println("1) Say hello");
        System.out.println("0) Exit");
        System.out.print("Choice: ");
        choice = sc.nextInt();

        if (choice == 1) {
            System.out.println("Hello");
        }
    } while (choice != 0);
}
```

---

## Exercice 6 — `for-each`
Afficher tous les noms dont la longueur est >= 4.

**Corrigé**
```java
import java.util.List;

static void printLongNames(List<String> names) {
    for (String name : names) {
        if (name.length() >= 4) {
            System.out.println(name);
        }
    }
}
```

---

# Annexes

## A. `break` et `continue`
- `break` : sortie immédiate de la boucle
- `continue` : passe directement à l’itération suivante

```java
for (int i = 0; i < 10; i++) {
    if (i % 2 == 0) continue; // ignore les pairs
    if (i > 7) break;         // stop à partir de 9
    System.out.println(i);    // 1,3,5,7
}
```

## B. Conseils pédagogiques (pour formateur)
- Faire tracer l’exécution (papier ou debug) pour comprendre conditions et itérations
- Insister sur la lisibilité : noms de variables et conditions explicites
- Faire comparer `switch` classique vs `switch` expression sur un même problème
