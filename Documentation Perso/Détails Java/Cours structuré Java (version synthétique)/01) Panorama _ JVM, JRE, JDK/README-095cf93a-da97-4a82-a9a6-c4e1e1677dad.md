# Formation – 1) Panorama : JVM, JRE, JDK

> Public cible : développeur/formatteur (profil .NET/C#).  
> Objectif : comprendre clairement l’écosystème d’exécution Java, les rôles et différences entre **JVM**, **JRE** et **JDK**, ainsi que le pipeline de compilation/exécution (**`javac` → bytecode → JVM (interpréteur + JIT)**).

---

## 0. Pré-requis

- Notions de compilation vs exécution (ex. C# : compilation en IL, exécution via CLR).
- Notions de ligne de commande.

---

## 1. Objectifs pédagogiques

À l’issue de cette formation, vous serez capable de :

1. Définir précisément **JVM**, **JRE**, **JDK**.
2. Expliquer le **pipeline** de transformation : source Java → bytecode → exécution.
3. Identifier quels composants sont nécessaires *selon le scénario* (dev, exécution prod, CI/CD, etc.).
4. Comprendre les notions d’**interprétation** et de **JIT** dans la JVM.
5. Faire le lien avec des concepts familiers (CLR/.NET) sans confusion.

---

## 2. Plan de la formation

1. **Panorama et vocabulaire**
2. **JVM : la machine virtuelle**
   - Bytecode, classloader, interpréteur, JIT
3. **JRE : l’environnement d’exécution**
   - Contenu et finalité
4. **JDK : le kit de développement**
   - Outils inclus (javac, jar, javadoc, etc.)
5. **Pipeline complet : `javac` → `.class` → JVM**
6. **Cas d’usage : que faut-il installer ?**
7. **Exercices guidés (CLI)**
8. **Résumé / mémo**

---

## 3. Panorama et vocabulaire (mise à plat)

Java est souvent présenté comme :

- Un **langage** (fichiers `.java`)
- Un **format binaire portable** : le **bytecode** (fichiers `.class`)
- Une **plateforme d’exécution** basée sur une machine virtuelle : la **JVM**
- Un environnement et des outils : **JRE** et **JDK**

### 3.1 Les trois sigles à retenir

- **JVM (Java Virtual Machine)** : exécute le bytecode (`.class`).
- **JRE (Java Runtime Environment)** : le **runtime** (inclut la JVM + bibliothèques runtime) pour *exécuter* des applications Java.
- **JDK (Java Development Kit)** : *JRE + outils de développement* (compilation, packaging, debug, etc.).

> Phrase mémo : **JDK = outils de dev + runtime**, **JRE = runtime**, **JVM = le moteur d’exécution du bytecode**.

---

## 4. JVM : la machine virtuelle Java

### 4.1 Rôle

La **JVM** est la “machine” logicielle qui :

- charge les classes (`.class`) via un **ClassLoader**,
- vérifie le bytecode (
  - sécurité,
  - cohérence,
  - types…
),
- exécute le bytecode :
  - soit via un **interpréteur**,
  - soit via un compilateur **JIT** (Just-In-Time) qui produit du code machine optimisé.

### 4.2 Bytecode : l’unité d’exécution

Le compilateur Java (`javac`) produit du **bytecode**, indépendant du système d’exploitation et du CPU.

- Source : `Hello.java`
- Résultat : `Hello.class`

Le bytecode correspond à un set d’instructions standardisées, conçues pour être exécutées par une JVM.

### 4.3 Interpréteur + JIT : deux modes complémentaires

La JVM exécute le bytecode via :

1. **Interpréteur**
   - démarre rapidement,
   - exécute instruction par instruction,
   - utile au démarrage (cold start), ou si le code est peu exécuté.

2. **JIT (Just-In-Time compiler)**
   - observe l’exécution,
   - détecte les “hot spots” (fonctions/boucles fréquemment exécutées),
   - compile en **code machine natif**,
   - applique des optimisations (inlining, escape analysis, etc.).

> Conséquence : une application Java peut s’améliorer en performance après un “échauffement” (warm-up), car le JIT optimise au fil de l’exécution.

### 4.4 Mémoire et exécution (conceptuel)

Sans entrer trop profondément (ce sera typiquement une autre séance), retenez que la JVM gère :

- une **pile** (stack) par thread,
- un **tas** (heap) pour les objets,
- un **garbage collector** (GC) pour recycler la mémoire.

---

## 5. JRE : Java Runtime Environment (le runtime)

### 5.1 Définition

Le **JRE** est l’environnement minimal pour **exécuter** une application Java.

Il comprend typiquement :

- une **JVM**,
- les **bibliothèques standard** nécessaires au runtime (API Java de base : collections, IO, réseau, sécurité…),
- des composants de support (ex. modules nécessaires à l’exécution).

### 5.2 Ce que le JRE ne contient pas

Le JRE **ne contient pas** (ou pas nécessairement) les outils de développement :

- pas de compilateur `javac` (pas faits pour compiler),
- pas d’outils de packaging/debug au sens du kit complet.

> En pratique, les distributions modernes (OpenJDK) mettent l’accent sur le **JDK**; le “JRE séparé” est devenu moins central, mais le concept reste important : **runtime vs dev kit**.

---

## 6. JDK : Java Development Kit (outil + runtime)

### 6.1 Définition

Le **JDK** est destiné aux développeurs :

- il inclut le **JRE** (donc la JVM + bibliothèques runtime),
- il ajoute les **outils** nécessaires au cycle de développement.

En résumé :

> **JDK = outils de dev + runtime**

### 6.2 Outils majeurs (à connaître)

Selon la distribution/version, on retrouve notamment :

- `javac` : compilateur Java → `.class`
- `java` : lanceur d’application (démarre la JVM et exécute la classe / le JAR)
- `jar` : empaquetage `.jar` (Java ARchive)
- `javadoc` : génération de documentation
- `jshell` : REPL (shell interactif)
- `jdb` : debug (selon distributions)
- `jlink`, `jpackage` : création d’images runtime / packaging (versions modernes)

> À retenir pour ce module : **`javac` compile**, **`java` exécute**.

---

## 7. Pipeline complet : `javac` → bytecode → JVM

### 7.1 Vue d’ensemble (diagramme)

```text
Fichier source           Compilation                Exécution
-----------             -------------              -------------------------
Hello.java   --javac-->  Hello.class (bytecode)  --java--> JVM
                                                     |-- Interpréteur
                                                     '-- JIT (optimisations)
```

### 7.2 Détail des étapes

1. **Écriture du code** (ex. `Hello.java`)
2. **Compilation** via `javac`
   - vérifie la syntaxe et les types,
   - compile vers du bytecode `.class`.
3. **Exécution** via `java`
   - charge les classes,
   - vérifie le bytecode,
   - exécute via l’interpréteur,
   - compile à la volée via le JIT si pertinent.

---

## 8. Cas d’usage : que faut-il installer ?

### 8.1 Scénarios typiques

- **Développer en Java** (local dev) :
  - installer un **JDK** (indispensable)

- **Compiler en CI/CD** :
  - agent CI doit avoir un **JDK**

- **Exécuter une application Java** (serveur/prod) :
  - conceptuellement un **JRE** suffit,
  - en pratique, on installe souvent un **JDK** ou une **image runtime** minimaliste (modules/containeurs) selon les besoins.

### 8.2 Analogie rapide avec .NET (pour se repérer)

- **JVM** ≈ **CLR** (moteur d’exécution)
- **Bytecode Java** ≈ **IL (MSIL/CIL)**
- **JRE** ≈ **Runtime .NET** (exécution)
- **JDK** ≈ **SDK .NET** (outils de compilation/build + runtime)

> Attention : l’analogie aide, mais les implémentations et écosystèmes diffèrent (outillage, packaging, GC, JIT, etc.).

---

## 9. Exercices guidés (CLI)

Les exercices ci-dessous supposent que vous avez un **JDK** installé et que `java`/`javac` sont dans le `PATH`.

### 9.1 Vérifier l’installation

```bash
java -version
javac -version
```

Vous devriez voir une version (ex. 17, 21…).

### 9.2 Compiler un programme simple

Créer `Hello.java` :

```java
public class Hello {
    public static void main(String[] args) {
        System.out.println("Hello JVM/JRE/JDK");
    }
}
```

Compiler :

```bash
javac Hello.java
```

Résultat : un fichier `Hello.class`.

### 9.3 Exécuter sur la JVM

```bash
java Hello
```

La commande `java` démarre la **JVM** et exécute la méthode `main`.

### 9.4 Observer le bytecode (optionnel, si outil disponible)

Selon JDK, vous pouvez utiliser `javap` :

```bash
javap -c Hello
```

Vous verrez une représentation des instructions bytecode.

---

## 10. Points clés à retenir (mémo)

- **JVM** : exécute le **bytecode** `.class` (interpréteur + JIT).
- **JRE** : **runtime** pour exécuter (inclut la JVM + libs).
- **JDK** : **JRE + outils de dev** (`javac`, `jar`, `javadoc`, …).
- Pipeline :

```text
.javac  →  .class (bytecode)  →  JVM (interpréteur + JIT)
```

---

## 11. Quiz rapide (auto-évaluation)

1. Quelle différence entre **JRE** et **JDK** ?
2. Quel fichier produit `javac` ?
3. Quel composant exécute réellement le bytecode ?
4. Pourquoi parle-t-on d’**échauffement** (warm-up) en Java ?
5. À quoi sert la commande `java` ?

---

## 12. Prolongements (suite logique)

- Classpath / Modules (JPMS)
- Packaging : JAR, fat JAR, jlink/jpackage
- Garbage Collector et tuning JVM
- JVM languages (Kotlin, Scala) : même bytecode/JVM

