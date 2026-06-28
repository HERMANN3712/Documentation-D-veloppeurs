# 10) IO et NIO.2 (java.nio.file) — Path & Files

> Public cible : développeurs (p. ex. .NET/C#) souhaitant maîtriser les I/O modernes en Java.
>
> Objectifs : lire/écrire des fichiers, parcourir des répertoires, gérer l’encodage et le buffering, manipuler des chemins de façon portable, comprendre les différences IO vs NIO.2, intégrer du JSON (souvent via Jackson) dans un projet.

---

## Plan de la formation

1. **Introduction : IO classique vs NIO.2**
   - Pourquoi NIO.2 (Java 7+) ?
   - Concepts clés : `Path`, `Files`, `FileSystem`, streams de répertoire
   - Correspondances/culture .NET (au besoin)
2. **`Path` : construction, normalisation, résolution**
   - `Paths.get(...)`, `Path.of(...)`
   - `resolve`, `relativize`, `normalize`, `toAbsolutePath`
   - Extraction : `getFileName`, `getParent`, `getRoot`
   - Séparateurs et portabilité
3. **`Files` : opérations de base sur fichiers/répertoires**
   - Existence, attributs, taille, dates
   - Création : fichiers/répertoires
   - Copie, déplacement, suppression
4. **Lecture/écriture : API "simple" (tout-en-mémoire) vs flux (streaming)**
   - `readAllBytes`, `readString`, `readAllLines`
   - `write`, `writeString`, options (`StandardOpenOption`)
   - Quand éviter le "all-in-memory" ?
5. **Buffering et performances**
   - `BufferedReader/Writer`
   - `Files.newBufferedReader/newBufferedWriter`
   - `InputStream/OutputStream` bufferisés
   - `transferTo`, copie efficace
6. **Parcours de répertoires**
   - `DirectoryStream` (lazy)
   - `Files.list`, `Files.walk`
   - Filtrage, profondeur, consommation/fermeture des streams
7. **Filtrage, recherche et correspondance de motifs**
   - `PathMatcher` (glob/regex)
   - Recherche de fichiers (ex : `walk` + filter)
8. **Gestion d’erreurs, robustesse et sécurité**
   - Exceptions fréquentes (`NoSuchFileException`, `AccessDeniedException`…)
   - Liens symboliques
   - Chemins utilisateurs et traversal (sécuriser `resolve`)
9. **JSON en projet avec Jackson + NIO.2**
   - Lire/écrire JSON depuis/vers un `Path`
   - Streaming JSON vs chargement complet
10. **Exercices guidés & mini-projet**
   - Tooling CLI: scan dossier, extraire métadonnées, JSON report
   - Bonnes pratiques de structuration

---

## 1. Introduction : IO classique vs NIO.2

### 1.1. IO "classique" (java.io)
Historiquement, Java utilisait `java.io.File`, `FileInputStream`, `FileReader`, etc.
- `File` représente un chemin **mais** l’API est limitée (peu d’options, gestion d’erreurs moins précise).
- Beaucoup de méthodes retournent `boolean` au lieu de lever des exceptions détaillées.

### 1.2. NIO.2 (java.nio.file)
Depuis Java 7, l’API NIO.2 apporte :
- `Path` : représentation moderne d’un chemin.
- `Files` : façade utilitaire riche (création, copies, lectures, attributs…).
- `FileSystem` : abstraction de systèmes de fichiers (local, zipfs, etc.).
- Gestion d’exceptions plus fine et options explicites.

> À retenir : **dans la majorité des cas "fichier/répertoire" en Java moderne ⇒ `Path` + `Files`.**

---

## 2. `Path` : construction, normalisation, résolution

### 2.1. Créer un `Path`
Deux façons courantes :

```java
import java.nio.file.Path;

Path p1 = Path.of("data", "input.txt");
Path p2 = java.nio.file.Paths.get("/var", "log", "app.log");
```

- `Path.of(...)` (Java 11+) est idiomatique.
- `Paths.get(...)` existe depuis plus longtemps.

### 2.2. Inspection d’un chemin

```java
Path p = Path.of("data", "input.txt");

System.out.println(p.getFileName()); // input.txt
System.out.println(p.getParent());   // data
System.out.println(p.getRoot());     // null (chemin relatif)
```

### 2.3. Absolu, normalisation

```java
Path p = Path.of("data", "..", "data", "input.txt");

Path normalized = p.normalize();
Path absolute = normalized.toAbsolutePath();
```

- `normalize()` simplifie `.` et `..`.
- `toAbsolutePath()` résout depuis le **working directory**.

### 2.4. Composer des chemins : `resolve` / `relativize`

```java
Path base = Path.of("/tmp/app");
Path file = base.resolve("config").resolve("app.json");
// /tmp/app/config/app.json

Path rel = base.relativize(file);
// config/app.json
```

### 2.5. Attention à la portabilité
- Sous Windows : `C:\...`, séparateur `\`.
- Sous Unix : `/...`, séparateur `/`.

**Bon réflexe :** ne pas concaténer des chaînes avec `/` ou `\`. Utiliser `Path.of(...)` / `resolve(...)`.

---

## 3. `Files` : opérations de base sur fichiers/répertoires

### 3.1. Tests et attributs

```java
import java.nio.file.*;

Path p = Path.of("data/input.txt");

boolean exists = Files.exists(p);
boolean readable = Files.isReadable(p);
long size = Files.size(p);
```

- `Files.exists` peut retourner `false` si absence **ou** si droits insuffisants (selon contexte).

### 3.2. Création

```java
Path dir = Path.of("data", "out");
Files.createDirectories(dir); // idempotent si existe

Path file = dir.resolve("result.txt");
Files.createFile(file); // échoue si existe déjà
```

### 3.3. Copier / déplacer / supprimer

```java
Path src = Path.of("data/input.txt");
Path dst = Path.of("data/out/input-copy.txt");

Files.copy(src, dst, StandardCopyOption.REPLACE_EXISTING);

Path moved = Path.of("data/out/input-moved.txt");
Files.move(dst, moved, StandardCopyOption.REPLACE_EXISTING);

Files.delete(moved); // lève une exception si absent
// Files.deleteIfExists(moved); // alternative
```

- `StandardCopyOption.ATOMIC_MOVE` peut exister (selon FS) pour déplacer atomiquement.

---

## 4. Lecture/écriture : all-in-memory vs streaming

### 4.1. Lecture "simple" (pratique, mais attention mémoire)

```java
Path p = Path.of("data/input.txt");

byte[] bytes = Files.readAllBytes(p);
String text = Files.readString(p); // Java 11+
```

Pour un fichier de quelques Ko/Mo, c’est parfait.
Pour un fichier de plusieurs centaines de Mo/Go, cela peut :
- consommer trop de mémoire,
- provoquer des ralentissements/GC,
- planter l’application.

### 4.2. Lecture ligne à ligne

```java
try (var lines = Files.lines(Path.of("data/input.txt"))) {
    long count = lines.filter(l -> !l.isBlank()).count();
    System.out.println(count);
}
```

- `Files.lines(...)` retourne un `Stream<String>` **à fermer**.
- Traite en flux, mais attention : le stream lit au fil de l’eau, et dépend d’un `BufferedReader`.

### 4.3. Écriture simple

```java
Path out = Path.of("data/out/result.txt");
Files.createDirectories(out.getParent());

Files.writeString(out, "Hello\n", StandardOpenOption.CREATE, StandardOpenOption.TRUNCATE_EXISTING);
Files.writeString(out, "World\n", StandardOpenOption.CREATE, StandardOpenOption.APPEND);
```

Options utiles :
- `CREATE` : crée si absent.
- `TRUNCATE_EXISTING` : remplace le contenu.
- `APPEND` : ajoute à la fin.
- `CREATE_NEW` : échoue si le fichier existe.

---

## 5. Buffering et performances

### 5.1. Pourquoi bufferiser ?
Les opérations I/O peuvent faire beaucoup d’appels système si on lit/écrit de petits morceaux.
Le buffering regroupe les accès, ce qui :
- réduit la latence,
- améliore le débit,
- limite le coût CPU.

### 5.2. BufferedReader/Writer via `Files`

```java
import java.nio.charset.StandardCharsets;
import java.nio.file.*;

Path in = Path.of("data/input.txt");
Path out = Path.of("data/out/upper.txt");
Files.createDirectories(out.getParent());

try (var reader = Files.newBufferedReader(in, StandardCharsets.UTF_8);
     var writer = Files.newBufferedWriter(out, StandardCharsets.UTF_8,
             StandardOpenOption.CREATE, StandardOpenOption.TRUNCATE_EXISTING)) {

    String line;
    while ((line = reader.readLine()) != null) {
        writer.write(line.toUpperCase());
        writer.newLine();
    }
}
```

- Contrôlez explicitement l’encodage (`UTF-8` recommandé).
- Utilisez `try-with-resources` pour la fermeture.

### 5.3. Streams binaires + buffers

```java
Path src = Path.of("data/big.bin");
Path dst = Path.of("data/out/big-copy.bin");
Files.createDirectories(dst.getParent());

try (var in = new java.io.BufferedInputStream(Files.newInputStream(src));
     var out = new java.io.BufferedOutputStream(Files.newOutputStream(dst,
             StandardOpenOption.CREATE, StandardOpenOption.TRUNCATE_EXISTING))) {

    in.transferTo(out); // Java 9+
}
```

### 5.4. Règle pratique
- Petits fichiers texte : `readString` / `writeString`.
- Traitements volumineux : flux + buffering.
- Traitements ligne à ligne : `newBufferedReader` ou `Files.lines`.

---

## 6. Parcours de répertoires

### 6.1. Lister un répertoire (non récursif) : `DirectoryStream`

```java
Path dir = Path.of("data");

try (DirectoryStream<Path> stream = Files.newDirectoryStream(dir, "*.txt")) {
    for (Path p : stream) {
        System.out.println(p);
    }
}
```

- Très adapté pour des répertoires volumineux (itération lazy).
- Supporte un filtre glob simple via le 2ᵉ argument.

### 6.2. `Files.list` (Stream) — non récursif

```java
try (var paths = Files.list(Path.of("data"))) {
    paths.filter(Files::isRegularFile)
         .forEach(System.out::println);
}
```

- Retourne un `Stream<Path>` **à fermer**.

### 6.3. `Files.walk` — récursif

```java
Path root = Path.of("data");

try (var paths = Files.walk(root, 5)) { // profondeur max = 5
    paths.filter(Files::isRegularFile)
         .filter(p -> p.toString().endsWith(".json"))
         .forEach(System.out::println);
}
```

- `walk` est puissant, mais peut être coûteux (beaucoup d’entrées).
- Toujours limiter la profondeur si possible.

---

## 7. Filtrage avancé : `PathMatcher` (glob/regex)

### 7.1. Créer un matcher

```java
import java.nio.file.*;

FileSystem fs = FileSystems.getDefault();
PathMatcher matcher = fs.getPathMatcher("glob:**/*.json");

Path p = Path.of("data/config/app.json");
System.out.println(matcher.matches(p)); // true
```

- Préfixes : `glob:` ou `regex:`.
- `**` traverse les sous-dossiers (en glob).

### 7.2. Exemple : trouver tous les JSON récursivement

```java
Path root = Path.of("data");
PathMatcher jsonMatcher = FileSystems.getDefault().getPathMatcher("glob:**/*.json");

try (var paths = Files.walk(root)) {
    paths.filter(Files::isRegularFile)
         .filter(jsonMatcher::matches)
         .forEach(System.out::println);
}
```

---

## 8. Gestion d’erreurs, robustesse et sécurité

### 8.1. Exceptions courantes
- `NoSuchFileException` : fichier absent.
- `FileAlreadyExistsException` : création avec collision.
- `AccessDeniedException` : droits insuffisants.
- `DirectoryNotEmptyException` : suppression d’un dossier non vide.

Exemple :

```java
try {
    Files.delete(Path.of("data/out"));
} catch (DirectoryNotEmptyException e) {
    System.err.println("Le dossier n'est pas vide");
}
```

### 8.2. Liens symboliques
Par défaut, certaines opérations suivent les liens symboliques.
- Utilisez `LinkOption.NOFOLLOW_LINKS` si nécessaire.

```java
boolean isLink = Files.isSymbolicLink(Path.of("data/link"));
```

### 8.3. Sécuriser la résolution de chemins (anti path traversal)
Si vous prenez une saisie utilisateur (`"../../etc/passwd"`), ne la `resolve`z pas naïvement.

```java
Path base = Path.of("/srv/app/uploads").toAbsolutePath().normalize();
Path candidate = base.resolve(userProvidedName).normalize();

if (!candidate.startsWith(base)) {
    throw new SecurityException("Path traversal détecté");
}
```

---

## 9. JSON en projet avec Jackson + NIO.2

### 9.1. Dépendance (Maven)

```xml
<dependency>
  <groupId>com.fasterxml.jackson.core</groupId>
  <artifactId>jackson-databind</artifactId>
  <version>2.17.2</version>
</dependency>
```

### 9.2. Modèle Java

```java
public record AppConfig(String appName, int port) {}
```

### 9.3. Lire un JSON depuis un `Path`

Deux approches :

**A) Passer un `InputStream` (streaming, recommandé)**

```java
import com.fasterxml.jackson.databind.ObjectMapper;
import java.nio.file.*;

ObjectMapper mapper = new ObjectMapper();
Path jsonPath = Path.of("config/app.json");

AppConfig cfg;
try (var in = Files.newInputStream(jsonPath)) {
    cfg = mapper.readValue(in, AppConfig.class);
}
```

**B) Lire en string puis parser (simple, mais full memory)**

```java
String json = Files.readString(Path.of("config/app.json"));
AppConfig cfg = mapper.readValue(json, AppConfig.class);
```

### 9.4. Écrire un JSON vers un `Path`

```java
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.SerializationFeature;
import java.nio.file.*;

ObjectMapper mapper = new ObjectMapper().enable(SerializationFeature.INDENT_OUTPUT);
AppConfig cfg = new AppConfig("demo", 8080);

Path out = Path.of("data/out/app.json");
Files.createDirectories(out.getParent());

try (var outStream = Files.newOutputStream(out,
        StandardOpenOption.CREATE, StandardOpenOption.TRUNCATE_EXISTING)) {
    mapper.writeValue(outStream, cfg);
}
```

### 9.5. Bonnes pratiques JSON + I/O
- Préférer `InputStream/OutputStream` (évite de charger tout le fichier JSON en String).
- Gérer l’encodage : Jackson écrit en UTF-8 par défaut pour flux.
- Toujours créer les dossiers parent (`createDirectories`).

---

## 10. Exercices guidés & mini-projet

### Exercice 1 — Copier un fichier texte en filtrant les lignes vides
**Énoncé :** lire `data/input.txt`, écrire `data/out/clean.txt` sans lignes blanches.

- Utiliser `Files.newBufferedReader` et `Files.newBufferedWriter`.
- Encodage UTF-8.

### Exercice 2 — Scanner un dossier et produire un rapport JSON
**Énoncé :** parcourir récursivement `data/scan`, repérer tous les fichiers `*.json`, et produire un rapport `report.json` contenant :
- chemin relatif,
- taille,
- dernier timestamp de modification.

**Pistes :**
- `Files.walk(root)`
- `Files.size(path)`
- `Files.getLastModifiedTime(path)`
- record `FileInfo(path, size, lastModified)`
- Jackson `writeValue(OutputStream, ...)`

### Mini-projet — CLI "file-audit"
**But :** un outil console :
- prend un dossier en argument,
- filtre par glob (ex : `**/*.log`),
- calcule : nb fichiers, taille totale, top 10 des plus gros,
- exporte un JSON.

**Contraintes :**
- Ne pas charger les fichiers en mémoire (uniquement les métadonnées).
- Fermer les streams (`try-with-resources`).

---

## Annexes

### A. Cheatsheet `StandardOpenOption`
- `CREATE` : créer si absent
- `CREATE_NEW` : créer uniquement si absent
- `TRUNCATE_EXISTING` : vider si existant
- `APPEND` : ajouter en fin
- `WRITE`, `READ`

### B. Quand utiliser quoi ?
- **Manipulation de chemin** : `Path` (jamais concat de String)
- **Opérations fichier/dossier** : `Files.*`
- **Text line-by-line** : `newBufferedReader` / `Files.lines`
- **Gros binaires** : `newInputStream` + buffer + `transferTo`
- **JSON** : Jackson + flux NIO (`Files.newInputStream/newOutputStream`)

---

## QCM rapide (validation)
1. Pourquoi `Path` est-il préféré à `File` ?
2. Quelle différence entre `Files.list` et `Files.walk` ?
3. Pourquoi faut-il fermer le `Stream` retourné par `Files.lines` ?
4. Dans quel cas `readAllBytes` est-il une mauvaise idée ?
5. Comment sécuriser un `resolve` face au path traversal ?

---

## Fin

Prochaines étapes suggérées :
- WatchService (surveillance de fichiers) si votre cours l’aborde ensuite.
- AsynchronousFileChannel pour des scénarios haut débit (plus avancé).
