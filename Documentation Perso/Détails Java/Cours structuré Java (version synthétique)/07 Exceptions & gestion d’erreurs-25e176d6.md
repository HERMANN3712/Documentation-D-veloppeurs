# Formation 7) Exceptions & gestion d’erreurs (Java)

> Public : développeurs (profil .NET/C# bienvenu) souhaitant maîtriser le modèle d’exceptions Java et les bons réflexes de gestion d’erreurs.

## Objectifs pédagogiques

À l’issue de cette formation, vous serez capable de :

- Expliquer le rôle des exceptions (vs codes de retour) et le flux d’exécution.
- Utiliser correctement `try / catch / finally`.
- Utiliser `try-with-resources` et comprendre `AutoCloseable`.
- Distinguer **checked** et **unchecked** (ex. `IOException` vs `RuntimeException`).
- Mettre en place de bons réflexes :
  - exceptions explicites pour l’I/O,
  - validations pour les contrats (préconditions),
  - messages et causes (`cause`) exploitables.

## Pré-requis

- Connaissances de base en Java (classes, méthodes, interfaces).
- Notions sur I/O (fichiers / flux) utiles mais pas indispensables.

## Durée suggérée

- 1h30 à 3h selon profondeur des exercices.

---

## Plan de la formation

1. [Pourquoi gérer les erreurs ?](#1-pourquoi-gérer-les-erreurs-)
2. [`try / catch / finally` : fondations](#2-try--catch--finally--fondations)
3. [Bonnes pratiques de `catch`](#3-bonnes-pratiques-de-catch)
4. [`try-with-resources` & `AutoCloseable`](#4-try-with-resources--autocloseable)
5. [Checked vs unchecked](#5-checked-vs-unchecked)
6. [Bon réflexe : exceptions explicites pour l’I/O](#6-bon-réflexe--exceptions-explicites-pour-lio)
7. [Bon réflexe : validations pour les contrats](#7-bon-réflexe--validations-pour-les-contrats)
8. [Atelier / exercices guidés](#8-atelier--exercices-guidés)
9. [Synthèse & checklist](#9-synthèse--checklist)

---

## 1) Pourquoi gérer les erreurs ?

### 1.1. Les erreurs arrivent : qu’est-ce qu’on fait ?

Dans un programme, certaines situations sont *exceptionnelles* :

- un fichier n’existe pas,
- un réseau est indisponible,
- une donnée d’entrée ne respecte pas un contrat,
- une dépendance externe renvoie un résultat incohérent.

**But de la gestion d’erreurs** :

- rendre l’application **robuste** (ne pas tomber brutalement),
- produire des diagnostics **compréhensibles**,
- décider quoi faire : **réessayer**, **remonter**, **compenser**, **dégrader**.

### 1.2. Exceptions vs codes de retour

Deux grandes approches :

- **Code de retour** : `return -1`, `null`, `Optional.empty()`...
- **Exceptions** : interruption du flux normal, remontée de la pile d’appels.

Les exceptions se prêtent bien à :

- séparer le code métier du code de gestion d’erreur,
- remonter l’info au bon niveau de décision,
- conserver un **stacktrace**.

---

## 2) `try / catch / finally` : fondations

### 2.1. Syntaxe et flux d’exécution

```java
try {
    // Code susceptible de lever une exception
} catch (SomeException ex) {
    // Gestion d’erreur si SomeException survient
} finally {
    // Exécuté quoi qu'il arrive (sauf cas extrêmes: kill process)
}
```

Règles clés :

- Le bloc `try` encapsule le code risqué.
- Un `catch` ne s’exécute que si une exception compatible est levée.
- `finally` s’exécute **qu’il y ait exception ou non**, y compris si on `return` dans le `try`.

### 2.2. Exemple simple

```java
public int parseAndDivide(String a, String b) {
    try {
        int x = Integer.parseInt(a);
        int y = Integer.parseInt(b);
        return x / y;
    } catch (NumberFormatException ex) {
        throw new IllegalArgumentException("Entrée non numérique", ex);
    } catch (ArithmeticException ex) {
        throw new IllegalArgumentException("Division par zéro", ex);
    }
}
```

Points importants :

- On **traduit** des erreurs techniques en exceptions plus proches du besoin (`IllegalArgumentException`).
- On conserve la cause via `new ... (message, ex)`.

### 2.3. `finally` pour garantir le nettoyage

```java
public void writeLineLegacy(Path path, String line) {
    BufferedWriter writer = null;
    try {
        writer = Files.newBufferedWriter(path);
        writer.write(line);
        writer.newLine();
    } catch (IOException ex) {
        throw new UncheckedIOException("Erreur d'écriture", ex);
    } finally {
        if (writer != null) {
            try {
                writer.close();
            } catch (IOException closeEx) {
                // Selon la stratégie : log, suppression, ou remontée
            }
        }
    }
}
```

Ce style « legacy » est remplacé avantageusement par `try-with-resources`.

---

## 3) Bonnes pratiques de `catch`

### 3.1. Catcher le plus spécifique possible

Mauvais (trop large) :

```java
try {
    // ...
} catch (Exception ex) {
    // gestion floue
}
```

Meilleur :

- catcher `IOException`, `SQLException`, etc.
- ou un type fonctionnel (`IllegalArgumentException`) si c’est une **erreur de contrat**.

### 3.2. Ne pas avaler une exception

Anti-pattern :

```java
try {
    doSomething();
} catch (IOException ex) {
    // rien
}
```

Conséquences :

- votre programme continue dans un état potentiellement invalide,
- le diagnostic est perdu.

### 3.3. Préserver la cause (exception chaining)

Toujours éviter :

```java
throw new RuntimeException("Erreur"); // cause perdue
```

Préférer :

```java
throw new RuntimeException("Erreur", ex);
```

### 3.4. Messages utiles

Bon message :

- indique **quoi**, **où**, et **avec quelles données** (sans secrets).

Exemple :

```java
throw new IOException("Impossible de lire le fichier: " + path, ex);
```

---

## 4) `try-with-resources` & `AutoCloseable`

### 4.1. Le problème : libérer les ressources

Ressources typiques :

- fichiers (`InputStream`, `Reader`, `BufferedReader`),
- sockets,
- connexions BDD,
- tout ce qui doit être `close()`.

### 4.2. Solution moderne : `try-with-resources`

```java
public String readFirstLine(Path path) throws IOException {
    try (BufferedReader reader = Files.newBufferedReader(path)) {
        return reader.readLine();
    }
}
```

Caractéristiques :

- l’objet doit implémenter `AutoCloseable` (ou `Closeable`).
- `close()` est appelé automatiquement, même en cas d’exception.

### 4.3. Comprendre `AutoCloseable`

Signature :

```java
public interface AutoCloseable {
    void close() throws Exception;
}
```

Beaucoup de classes I/O utilisent `Closeable`, une sous-interface (orientée `IOException`).

### 4.4. Exceptions « supprimées » (suppressed)

Si une exception survient dans le `try` **et** que `close()` lève aussi, Java :

- propage l’exception principale (du bloc `try`),
- stocke les exceptions de fermeture comme **suppressed**.

Exemple d’inspection :

```java
try {
    // ...
} catch (Exception ex) {
    for (Throwable s : ex.getSuppressed()) {
        System.err.println("suppressed: " + s);
    }
}
```

---

## 5) Checked vs unchecked

Java distingue :

- **Checked exceptions** : doivent être déclarées (`throws`) ou capturées (`catch`).
- **Unchecked exceptions** : héritent de `RuntimeException` ; pas d’obligation de déclaration.

### 5.1. Exemple : `IOException` (checked)

`IOException` représente des erreurs d’I/O (fichier, flux, réseau).

```java
public byte[] load(Path path) throws IOException {
    return Files.readAllBytes(path);
}
```

Ici, le compilateur vous oblige à :

- gérer : `try/catch`,
- ou propager : `throws IOException`.

### 5.2. Exemple : `RuntimeException` (unchecked)

Exemples :

- `NullPointerException`
- `IllegalArgumentException`
- `IndexOutOfBoundsException`

Ces exceptions signalent souvent :

- une violation de contrat (préconditions non respectées),
- un bug.

### 5.3. Guidance : quand utiliser quoi ?

- **Checked** :
  - pour des erreurs attendues provenant de l’environnement (I/O, réseau),
  - quand l’appelant peut prendre une décision concrète.
- **Unchecked** :
  - pour les erreurs de programmation / invariants,
  - pour signaler une mauvaise utilisation d’une API (contrat).

> Note : dans la pratique, certains projets évitent les checked au profit d’exceptions applicatives runtime. Ici, l’objectif est de connaître le modèle standard et les réflexes sains.

---

## 6) Bon réflexe : exceptions explicites pour l’I/O

### 6.1. Ne pas masquer le contexte I/O

En I/O, il est souvent utile d’être explicite :

- `FileNotFoundException`,
- `AccessDeniedException`,
- `NoSuchFileException`,
- etc.

Exemple :

```java
public Properties loadConfig(Path path) throws IOException {
    Properties p = new Properties();
    try (InputStream in = Files.newInputStream(path)) {
        p.load(in);
    }
    return p;
}
```

Ici, laisser `IOException` remonter est souvent pertinent :

- le niveau supérieur (CLI, service) peut afficher un message user-friendly,
- ou déclencher une stratégie de fallback.

### 6.2. Transformer *localement* si nécessaire (API boundary)

Quand vous êtes à une frontière (ex: contrôleur web, service exposé), vous pouvez traduire :

```java
public Config getConfig(Path path) {
    try {
        return Config.from(loadConfig(path));
    } catch (IOException ex) {
        throw new IllegalStateException("Configuration illisible: " + path, ex);
    }
}
```

Règle :

- traduire en ajoutant du contexte,
- conserver la cause.

### 6.3. Checked → unchecked : `UncheckedIOException`

Java fournit :

- `UncheckedIOException` pour envelopper une `IOException` dans un runtime.

```java
try {
    Files.lines(path).forEach(System.out::println);
} catch (IOException ex) {
    throw new UncheckedIOException(ex);
}
```

Utile lorsque :

- une API fonctionnelle (streams) gère mal les checked exceptions.

---

## 7) Bon réflexe : validations pour les contrats

### 7.1. Valider les préconditions

Les préconditions sont des règles d’utilisation :

- paramètre non null,
- format attendu,
- intervalle,
- état cohérent.

Approche classique : `IllegalArgumentException` / `IllegalStateException`.

```java
public void transfer(Account from, Account to, BigDecimal amount) {
    if (from == null || to == null) {
        throw new IllegalArgumentException("Comptes requis");
    }
    if (amount == null || amount.signum() <= 0) {
        throw new IllegalArgumentException("Montant invalide: " + amount);
    }

    // ...
}
```

### 7.2. Pourquoi la validation plutôt que des exceptions techniques ?

- Une validation exprime un **contrat** clair.
- L’erreur est détectée **le plus tôt possible**.
- Elle évite des erreurs secondaires (ex: NPE).

### 7.3. `Objects.requireNonNull`

```java
import java.util.Objects;

public User save(User user) {
    Objects.requireNonNull(user, "user");
    Objects.requireNonNull(user.id(), "user.id");
    // ...
    return user;
}
```

### 7.4. Contrats et exceptions : règle de séparation

- **I/O** : exceptions explicites (souvent checked) car dépendances externes.
- **Contrat** : validation + `IllegalArgumentException` (unchecked).

---

## 8) Atelier / exercices guidés

### Exercice 1 — `try/catch/finally`

Écrire une méthode qui :

- parse un entier,
- affiche "OK" en `finally`,
- remonte une erreur de contrat si parsing impossible.

Attendus :

- `catch (NumberFormatException)`
- `finally` toujours exécuté.

### Exercice 2 — Lecture de fichier avec `try-with-resources`

But : lire toutes les lignes d’un fichier et compter celles non vides.

Contraintes :

- utiliser `try-with-resources`,
- propager `IOException` (checked).

### Exercice 3 — Traduction à la frontière

Simuler un service :

- méthode bas niveau : `load(Path) throws IOException`
- méthode haut niveau : `get()` ne déclare pas `throws`

Objectif :

- traduire en `IllegalStateException` en conservant la cause.

### Exercice 4 — Validations de contrats

Écrire une méthode `register(email, age)` :

- `email` non null, contient `@`.
- `age` >= 18.

Attendu :

- `IllegalArgumentException` pour les violations.

---

## 9) Synthèse & checklist

### Check-list rapide

- [ ] Je catch des exceptions **spécifiques**.
- [ ] Je n’avale jamais une exception sans stratégie.
- [ ] Je conserve la cause (`new X(msg, ex)`).
- [ ] J’utilise `try-with-resources` pour toute ressource `close()`.
- [ ] Je laisse remonter les exceptions I/O (ou je traduis à la frontière).
- [ ] Je valide les contrats avec des exceptions unchecked (`IllegalArgumentException`).

### Mémo

- `IOException` : **checked** → gérer ou déclarer.
- `RuntimeException` : **unchecked** → souvent bug/contrat.
- Ressources : `AutoCloseable` + `try-with-resources`.

---

## Annexes (référence rapide)

### A1. Modèle recommandé pour une couche I/O

- couche I/O (DAL / filesystem) :
  - méthodes `throws IOException`
  - messages contextualisés
- couche service :
  - validation de contrats
  - traduction globale éventuelle
- couche UI/API :
  - mapping vers réponse utilisateur (HTTP 400/500, message)

### A2. Exemple complet (I/O + validation)

```java
public final class UserRepository {
    private final Path baseDir;

    public UserRepository(Path baseDir) {
        this.baseDir = java.util.Objects.requireNonNull(baseDir, "baseDir");
    }

    public void save(String userId, String json) throws IOException {
        if (userId == null || userId.isBlank()) {
            throw new IllegalArgumentException("userId requis");
        }
        if (json == null || json.isBlank()) {
            throw new IllegalArgumentException("json requis");
        }

        Path file = baseDir.resolve(userId + ".json");
        try (var writer = java.nio.file.Files.newBufferedWriter(file)) {
            writer.write(json);
        }
    }
}
```
