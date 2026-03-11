# Formation 13 — Modules (Java 9+) & packaging

Public cible : développeurs (ex. .NET/C#) souhaitant comprendre le système de modules Java (JPMS) et les options modernes de packaging/déploiement.

Durée indicative : 1 jour (7h) — ajustable.

## Objectifs pédagogiques

À l’issue de la formation, vous saurez :

- Expliquer **pourquoi** JPMS (Java Platform Module System) a été introduit à partir de Java 9.
- Créer et structurer un projet modulaire avec `module-info.java`.
- Utiliser `exports` / `requires` (et variantes) pour contrôler l’encapsulation et les dépendances.
- Packager une application en **JAR exécutable** (manifest `Main-Class`).
- Construire un runtime minimal avec **jlink**.
- Déployer en conteneur Docker avec une image **JRE/JDK** adaptée (ou runtime jlink).

## Pré-requis

- Java SE (idéalement 17+ LTS).
- Connaissances de base Java (classes, packages, jar).
- Outils : JDK, Maven ou Gradle, Docker (optionnel mais recommandé).

## Plan de la formation

1. **Contexte et motivations du système de modules (Java 9+)**
2. **Anatomie d’un module : `module-info.java`**
3. **`exports` : API publique vs implémentation interne**
4. **`requires` : dépendances et lisibilité**
5. **Services JPMS (`uses` / `provides`) (option avancée)**
6. **Packaging : JAR exécutable (`Main-Class`)**
7. **Packaging : création d’un runtime minimal avec `jlink`**
8. **Déploiement en container : images JRE/JDK, multi-stage, jlink**
9. **Atelier guidé : de la modularisation au conteneur**
10. **Checklist et bonnes pratiques**

---

## 1) Contexte : pourquoi des modules ?

### 1.1 Limites du classpath « historique »

Avant Java 9, la plupart des applications utilisaient le **classpath** (liste de JAR). Problèmes classiques :

- **JAR Hell** : conflits de versions, classes dupliquées, ordre de résolution ambigu.
- **Encapsulation faible** : tout est accessible si dans le classpath, même ce qui devrait rester interne.
- **Temps de démarrage & empreinte** : monolithisme du JRE/JDK, difficile de construire un runtime minimal.

### 1.2 Objectifs de JPMS

JPMS apporte :

- **Encapsulation forte** : ce qui n’est pas exporté n’est pas accessible.
- **Dépendances explicites** : un module déclare ce qu’il requiert.
- **Outils** : possibilité de créer une image runtime minimaliste (via `jlink`).
- **Meilleure maintenabilité** sur grands systèmes : frontières propres entre composants.

> À retenir : sur un petit projet, JPMS peut sembler « verbeux ». Sur un grand système, il aide à maîtriser l’architecture et réduit les couplages.

---

## 2) Anatomie d’un module : `module-info.java`

Un **module** est une unité de déploiement et de compilation au-dessus des packages.

### 2.1 Structure typique

```
my-app/
  src/
    main/
      java/
        module-info.java
        com/example/app/Main.java
        com/example/app/internal/...
```

### 2.2 Exemple minimal

```java
module com.example.app {
}
```

Cela déclare un module nommé `com.example.app`. Sans `exports`, il n’expose aucune API aux autres modules.

### 2.3 Nom de module

- En pratique, on utilise un **reverse domain name** (comme les packages) : `com.company.product.module`.
- Le nom de module devient l’identité au niveau JPMS.

---

## 3) `exports` : exposer une API publique

### 3.1 Exposer un package

```java
module com.example.lib {
    exports com.example.lib.api;
}
```

- Tous les types publics de `com.example.lib.api` deviennent accessibles aux modules qui requièrent `com.example.lib`.
- Les packages **non exportés** restent inaccessibles (encapsulation forte).

### 3.2 API vs interne

Organisation recommandée :

- `...api` ou `...public` : API stable
- `...internal` : implementation interne non exportée

Exemple :

```text
com.example.lib.api      // exporté
com.example.lib.internal // non exporté
```

### 3.3 `exports ... to ...` (export qualifié)

Vous pouvez exporter un package **uniquement** à certains modules :

```java
module com.example.lib {
    exports com.example.lib.spi to com.example.plugin;
}
```

Cas d’usage : SPI/Plugins où on limite l’exposition.

---

## 4) `requires` : déclarer des dépendances

### 4.1 Dépendance simple

```java
module com.example.app {
    requires com.example.lib;
}
```

Cela signifie :

- `com.example.app` lit (`reads`) `com.example.lib`.
- Le compilateur et le runtime savent que ces modules sont requis.

### 4.2 Accès aux packages

- `requires` donne la **lisibilité** (readability) entre modules.
- L’accès réel dépend aussi de `exports` (accessibility).

> Raccourci mental : `requires` = “je peux voir le module” ; `exports` = “le module me montre ses packages”.

### 4.3 `requires transitive`

Quand vous publiez une bibliothèque, vous pouvez propager une dépendance à vos consommateurs :

```java
module com.example.framework {
    requires transitive com.fasterxml.jackson.databind;
    exports com.example.framework.api;
}
```

Ainsi, un module qui fait `requires com.example.framework` “voit” aussi Jackson sans le déclarer explicitement.

À utiliser avec parcimonie (risque d’augmenter le couplage).

### 4.4 `requires static`

Dépendance **optionnelle** au runtime, utile pour des annotations/compilation conditionnelle :

```java
module com.example.lib {
    requires static lombok;
}
```

- Requis à la compilation si présent
- Pas obligatoire au runtime

---

## 5) Services JPMS (`uses` / `provides`) — option avancée

JPMS intègre un mécanisme de **Service Provider Interface** (SPI) plus structuré que `META-INF/services`.

### 5.1 Déclarer l’usage d’un service

```java
module com.example.app {
    uses com.example.spi.PaymentProvider;
}
```

### 5.2 Déclarer une implémentation fournie

```java
module com.example.paypal {
    requires com.example.spi;
    provides com.example.spi.PaymentProvider
        with com.example.paypal.PaypalPaymentProvider;
}
```

Au runtime, `ServiceLoader` peut découvrir cette implémentation.

---

## 6) Packaging : JAR exécutable (Manifest `Main-Class`)

### 6.1 JAR “classique”

Compiler puis créer un JAR :

```bash
javac -d out src/main/java/com/example/app/Main.java
jar --create --file app.jar -C out .
```

### 6.2 Ajouter un point d’entrée

Pour rendre le jar exécutable : définir `Main-Class` dans le manifest.

#### Option A — via `jar --main-class`

```bash
jar --create --file app.jar --main-class com.example.app.Main -C out .
java -jar app.jar
```

#### Option B — manifest explicite

Créer `manifest.mf` :

```
Main-Class: com.example.app.Main
```

Puis :

```bash
jar cfm app.jar manifest.mf -C out .
```

### 6.3 Cas modulaire

Si votre application est modulaire, vous pouvez exécuter via module path :

```bash
java --module-path mods -m com.example.app/com.example.app.Main
```

> En pratique, Maven/Gradle gèrent la compilation et l’assemblage. L’essentiel est de comprendre le rôle du manifest et/ou du module launcher.

---

## 7) `jlink` : construire un runtime minimal

### 7.1 Pourquoi `jlink` ?

`jlink` permet de construire une image runtime contenant :

- les modules Java de la plateforme nécessaires (`java.base`, `java.logging`, etc.)
- vos modules applicatifs
- éventuellement des libs modulaires

Résultat :

- image plus petite qu’un JRE généraliste
- surface d’attaque réduite
- démarrage parfois plus rapide

### 7.2 Pré-requis

- Application (idéalement) **modulaire**.
- Disposer des modules compilés (dans un répertoire, ex. `mods/`).

### 7.3 Exemple de commande

Supposons :

- modules applicatifs dans `mods/`
- vous voulez une image dans `image/`

```bash
jlink \
  --module-path "$JAVA_HOME/jmods:mods" \
  --add-modules com.example.app \
  --output image \
  --strip-debug \
  --no-man-pages \
  --no-header-files \
  --compress=2
```

Lancement :

```bash
./image/bin/java -m com.example.app/com.example.app.Main
```

### 7.4 Générer un launcher

```bash
jlink \
  --module-path "$JAVA_HOME/jmods:mods" \
  --add-modules com.example.app \
  --output image \
  --launcher myapp=com.example.app/com.example.app.Main

./image/bin/myapp
```

---

## 8) Déploiement container : image JRE/JDK adaptée

### 8.1 Choisir une base d’image

Options courantes :

- **JRE/JDK complet** (plus simple) : utile quand on veut un environnement standard.
- **JRE slim** : tailles réduites.
- **Distroless** : surface minimale, bonnes pratiques sécurité.
- **Runtime jlink** : souvent le plus petit et le plus contrôlé.

Points d’attention :

- version Java (17, 21...) alignée avec votre compilation
- libc : `glibc` vs `musl` (Alpine) — peut impacter la compatibilité
- certificats (CA) si appels HTTPS

### 8.2 Dockerfile — exécution d’un JAR

Exemple simple (JRE standard) :

```dockerfile
FROM eclipse-temurin:21-jre
WORKDIR /app
COPY app.jar /app/app.jar
ENTRYPOINT ["java","-jar","/app/app.jar"]
```

### 8.3 Dockerfile — multi-stage + jlink

Construire puis copier uniquement l’image runtime générée :

```dockerfile
# Stage 1: build
FROM eclipse-temurin:21-jdk AS build
WORKDIR /src

# Copiez vos sources/projet (ex: Maven/Gradle) puis compilez en modules
# Ici, on suppose qu'au final vous avez un dossier mods/ contenant les modules
# et potentiellement un module principal com.example.app.
COPY . .

# Exemple: commande fictive; adaptez à Maven/Gradle
# RUN ./mvnw -q -DskipTests package

# Exemple jlink
RUN jlink \
  --module-path "$JAVA_HOME/jmods:mods" \
  --add-modules com.example.app \
  --output /image \
  --strip-debug \
  --no-man-pages \
  --no-header-files \
  --compress=2 \
  --launcher myapp=com.example.app/com.example.app.Main

# Stage 2: runtime
FROM debian:bookworm-slim
WORKDIR /app
COPY --from=build /image /opt/java
ENV PATH="/opt/java/bin:${PATH}"
ENTRYPOINT ["myapp"]
```

Variantes :

- utiliser une image **distroless** (ex. `gcr.io/distroless/base-debian12`) si vous maîtrisez bien les dépendances.
- ajouter un user non-root, gérer `TZ`, `LANG`, etc.

### 8.4 JVM flags usuels en container

- Limitation mémoire/GC adaptée :

```bash
java -XX:MaxRAMPercentage=75 -XX:+UseG1GC -jar app.jar
```

- Observabilité :
  - logs, JFR (Java Flight Recorder), metrics

> À noter : les JVM modernes détectent les limites cgroup et s’adaptent, mais il reste utile de définir des limites explicites selon vos SLA.

---

## 9) Atelier guidé (fil rouge)

Objectif : construire 2 modules, puis packager et déployer.

### 9.1 Module `com.example.lib`

**Arborescence**

```
lib/
  src/main/java/
    module-info.java
    com/example/lib/api/Greeter.java
    com/example/lib/internal/GreeterImpl.java
```

**Greeter (API)**

```java
package com.example.lib.api;

public interface Greeter {
    String greet(String name);
}
```

**Impl interne**

```java
package com.example.lib.internal;

import com.example.lib.api.Greeter;

public class GreeterImpl implements Greeter {
    @Override
    public String greet(String name) {
        return "Hello " + name;
    }
}
```

**module-info.java**

```java
module com.example.lib {
    exports com.example.lib.api;
}
```

> Le package `internal` n’est pas exporté : un autre module ne doit pas pouvoir instancier `GreeterImpl`.

### 9.2 Module `com.example.app`

**Arborescence**

```
app/
  src/main/java/
    module-info.java
    com/example/app/Main.java
```

**Main**

```java
package com.example.app;

import com.example.lib.api.Greeter;

public class Main {
    public static void main(String[] args) {
        Greeter greeter = new com.example.lib.internal.GreeterImpl();
        System.out.println(greeter.greet("World"));
    }
}
```

Essayez de compiler : cela doit **échouer** car `com.example.lib.internal` n’est pas exporté.

### 9.3 Correction : exposer une фабrique (pattern)

Dans la lib, on ajoute une fabrique dans `api` (exporté) :

```java
package com.example.lib.api;

import com.example.lib.internal.GreeterImpl;

public final class Greeters {
    private Greeters() {}

    public static Greeter defaultGreeter() {
        return new GreeterImpl();
    }
}
```

Côté app :

```java
package com.example.app;

import com.example.lib.api.Greeter;
import com.example.lib.api.Greeters;

public class Main {
    public static void main(String[] args) {
        Greeter greeter = Greeters.defaultGreeter();
        System.out.println(greeter.greet("World"));
    }
}
```

**module-info.java (app)**

```java
module com.example.app {
    requires com.example.lib;
}
```

### 9.4 Packaging / Exécution

- Exécuter via module path
- Construire jar exécutable (si souhaité)
- Générer une image runtime `jlink`
- Créer une image Docker en multi-stage

---

## 10) Checklist & bonnes pratiques

### Modularisation

- Exporter **uniquement** l’API stable.
- Éviter `requires transitive` par défaut.
- Préférer des factories/SPI plutôt que d’exposer des implémentations.
- Sur bibliothèques non modulaires :
  - utiliser des **automatic modules** (nom dérivé du jar), mais c’est une transition, pas un objectif final.

### Packaging

- JAR exécutable : simple, universel.
- jlink : excellent pour prod si vous maîtrisez votre graphe de modules.
- Verrouiller la version Java (LTS) et l’image Docker correspondante.

### Container

- Utiliser un user non-root (sécurité).
- Vérifier certificats CA, timezone/locale.
- Monitorer : heap, threads, GC, temps de réponse.

---

## Annexes — commandes utiles

### Inspecter des dépendances de modules

```bash
jdeps --module-path mods --print-module-deps mods/com.example.app.jar
```

### Lister modules présents

```bash
java --list-modules
```

### Exécuter un module

```bash
java --module-path mods -m com.example.app/com.example.app.Main
```

---

## Résumé

- `module-info.java` formalise les frontières : `exports` (API) et `requires` (dépendances).
- Pour les grands systèmes, JPMS améliore l’encapsulation et la maintenabilité.
- Packaging :
  - **JAR** + `Main-Class` pour le standard.
  - **jlink** pour un runtime minimal.
  - **Docker** pour un déploiement reproductible, idéalement avec une image Java adaptée ou un runtime `jlink`.
