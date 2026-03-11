# Formation Java — 2) Mise en place & outillage

> Public cible : développeurs (ex. .NET/C#) découvrant ou standardisant un environnement Java.

## Objectifs pédagogiques
À l’issue de ce module, vous serez capable de :

- Choisir une version **LTS** de Java adaptée (17/21/25) et comprendre ses impacts.
- Installer/configurer un **JDK** et vérifier l’environnement (PATH, JAVA_HOME).
- Démarrer un projet Java avec un IDE : **IntelliJ IDEA**, **Eclipse** ou **VS Code**.
- Mettre en place un build standard : **Maven** (`pom.xml`) ou **Gradle** (`build.gradle`).
- Respecter la **structure standard** d’un projet Java (sources, tests, ressources).
- Exécuter les tests et packager l’application (JAR) de manière reproductible.

## Pré-requis
- Connaissances générales en développement logiciel (CLI, Git, dépendances).
- Avoir un poste Windows/macOS/Linux avec droits d’installation.
- Connexion Internet (téléchargements JDK, dépendances Maven/Gradle).

---

## Plan du module

1. [Choisir une version LTS (17/21/25)](#1-choisir-une-version-lts-172125)
2. [Installation et configuration du JDK](#2-installation-et-configuration-du-jdk)
3. [Choisir et configurer un IDE](#3-choisir-et-configurer-un-ide)
   - IntelliJ IDEA
   - Eclipse
   - VS Code
4. [Outillage de build : Maven ou Gradle](#4-outillage-de-build--maven-ou-gradle)
   - Maven (`pom.xml`)
   - Gradle (`build.gradle`)
5. [Structure standard d’un projet Java](#5-structure-standard-dun-projet-java)
6. [Atelier : créer, builder, tester, packager](#6-atelier--créer-builder-tester-packager)
7. [Checklist de fin de module](#7-checklist-de-fin-de-module)

---

## 1) Choisir une version LTS (17/21/25)

### 1.1 Qu’est-ce qu’une LTS ?
Java évolue par versions. Certaines sont **LTS** (*Long Term Support*) : elles reçoivent des correctifs de sécurité et de stabilité sur une période plus longue.

- **Java 17 (LTS)** : très répandue, socle stable pour beaucoup d’écosystèmes.
- **Java 21 (LTS)** : successeur moderne, adoption en cours, meilleur support des nouveautés récentes.
- **Java 25 (LTS)** : future/à venir selon calendrier, adoption progressive.

> Recommandation générale : si vous démarrez un projet aujourd’hui et que votre écosystème le permet, privilégiez **Java 21**. Sinon, **Java 17** reste un excellent choix.

### 1.2 Critères de choix (projet, runtime, dépendances)
À vérifier avant de fixer la version :

- **Compatibilité des frameworks** (Spring, Quarkus, Micronaut, etc.).
- **Politique interne** (certification, support, contraintes de production).
- **Runtime cible** (Docker base image, OS serveur).
- **Dépendances legacy** (certaines libs imposent une version minimale).
- **Outils CI/CD** (agents de build disponibles avec la bonne version).

### 1.3 Notions utiles : JDK vs JRE
- **JDK** (Java Development Kit) : nécessaire pour **compiler**, **tester** et **builder**.
- **JRE** (Java Runtime Environment) : historique (aujourd’hui on déploie généralement un JDK minimal ou un runtime issu du JDK).

### 1.4 Distribution du JDK
Le langage est standard, mais le JDK existe via plusieurs distributions :

- **Eclipse Temurin (Adoptium)** : très courant, gratuit.
- **Oracle JDK** : licences différentes selon usage.
- **Amazon Corretto**, **Microsoft Build of OpenJDK**, etc.

> Conseil : standardiser en équipe sur une distribution (souvent Temurin) + version LTS.

---

## 2) Installation et configuration du JDK

### 2.1 Installer le JDK
Téléchargez et installez un JDK LTS (ex. Temurin 21) :

- Windows : installateur MSI/ZIP
- macOS : pkg ou via gestionnaire de paquets (brew)
- Linux : paquet distribution (apt/yum) ou tar.gz

### 2.2 Vérifier l’installation
Dans un terminal :

```bash
java -version
javac -version
```

Vous devez voir la version LTS choisie.

### 2.3 Variables d’environnement : JAVA_HOME et PATH
#### Pourquoi ?
Beaucoup d’outils (Maven, Gradle, IDE, CI) s’appuient sur :

- `JAVA_HOME` : chemin du JDK
- `PATH` : pour appeler `java`, `javac`, etc.

#### Exemple (Linux/macOS)
```bash
export JAVA_HOME=$(/usr/libexec/java_home -v 21)
export PATH="$JAVA_HOME/bin:$PATH"
```

#### Exemple (Windows - PowerShell)
```powershell
setx JAVA_HOME "C:\Program Files\Eclipse Adoptium\jdk-21"
# Puis ajouter %JAVA_HOME%\bin au PATH via les variables système
```

### 2.4 Multi-versions : recommandations
En entreprise, vous pouvez devoir supporter plusieurs versions.

- Utiliser un **version manager** (sdkman sur Linux/macOS) ou une convention par projet.
- En CI, fixer explicitement la version Java (Docker image, runner toolcache).

---

## 3) Choisir et configurer un IDE

### 3.1 IDE : critères de sélection
- **Ergonomie** et productivité (refactoring, navigation, inspections)
- **Support Maven/Gradle**
- **Intégration tests (JUnit)**
- **Qualité du debug**
- **Coût/licence**

### 3.2 IntelliJ IDEA
**Points forts** : excellent refactoring, inspections, expérience Maven/Gradle.

#### Configuration recommandée
- Installer le JDK et le déclarer dans :
  - *Project SDK*
  - *Gradle JVM* ou *Maven importer JDK*
- Activer l’import automatique du build (Gradle/Maven).

#### Bonnes pratiques
- Ne pas « bricoler » la structure : laisser l’IDE refléter `src/main/java`, `src/test/java`.
- Lancer les tests via l’IDE **ou** via CLI (pour se rapprocher de la CI).

### 3.3 Eclipse
**Points forts** : historique Java, intégration large, gratuit.

#### Configuration recommandée
- Installer un plugin Gradle si nécessaire (Buildship est généralement intégré).
- Configurer le JDK utilisé par workspace et projet.
- Importer projets Maven/Gradle via les assistants dédiés.

### 3.4 VS Code
**Points forts** : léger, polyvalent, très bon pour polyglotte.

#### Extensions utiles
- Extension Pack for Java
- Maven for Java / Gradle for Java
- Test Runner for Java

#### Attention
VS Code dépend beaucoup de l’installation d’un JDK correct et d’extensions.

---

## 4) Outillage de build : Maven ou Gradle

### 4.1 Pourquoi un outil de build ?
Un build tool standardise :

- La **compilation**
- La **gestion des dépendances**
- L’exécution des **tests**
- Le **packaging** (JAR)
- L’intégration CI/CD

> Objectif : un projet doit pouvoir se builder sans l’IDE.

---

## 4.A) Maven (`pom.xml`)

### 4.A.1 Concepts Maven
- Convention + configuration via `pom.xml`
- Cycle de vie : `validate`, `compile`, `test`, `package`, `verify`, `install`, `deploy`

### 4.A.2 Structure minimale d’un `pom.xml`
Exemple (projet simple) :

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>com.example</groupId>
  <artifactId>demo</artifactId>
  <version>1.0.0</version>
  <packaging>jar</packaging>

  <properties>
    <maven.compiler.release>21</maven.compiler.release>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  </properties>

  <dependencies>
    <dependency>
      <groupId>org.junit.jupiter</groupId>
      <artifactId>junit-jupiter</artifactId>
      <version>5.10.2</version>
      <scope>test</scope>
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-surefire-plugin</artifactId>
        <version>3.2.5</version>
      </plugin>
    </plugins>
  </build>
</project>
```

### 4.A.3 Commandes Maven essentielles
Dans le dossier contenant le `pom.xml` :

```bash
mvn -v
mvn clean
mvn test
mvn package
```

- `clean` : supprime `target/`
- `test` : exécute les tests
- `package` : produit un JAR dans `target/`

### 4.A.4 Dépôt local Maven
Maven télécharge les dépendances dans le dépôt local :

- Linux/macOS : `~/.m2/repository`
- Windows : `C:\Users\<user>\.m2\repository`

---

## 4.B) Gradle (`build.gradle`)

### 4.B.1 Concepts Gradle
- Build déclaratif en Groovy (`build.gradle`) ou Kotlin (`build.gradle.kts`)
- Performances et flexibilité
- Très utilisé dans certains écosystèmes (Android, multi-modules)

### 4.B.2 Exemple minimal (`build.gradle` Groovy)

```groovy
plugins {
    id 'java'
}

group = 'com.example'
version = '1.0.0'

java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(21)
    }
}

repositories {
    mavenCentral()
}

dependencies {
    testImplementation 'org.junit.jupiter:junit-jupiter:5.10.2'
}

test {
    useJUnitPlatform()
}
```

### 4.B.3 Commandes Gradle essentielles
Avec le wrapper (recommandé) :

```bash
./gradlew -v
./gradlew clean
./gradlew test
./gradlew build
```

- `build` : compile + test + assemble
- Artefacts : `build/libs/`

### 4.B.4 Pourquoi utiliser le Gradle Wrapper ?
Le wrapper (`gradlew`, `gradlew.bat` + dossier `gradle/wrapper`) fixe la version de Gradle et rend le build reproductible.

---

## 4.2 Maven vs Gradle : aide au choix

| Critère | Maven | Gradle |
|---|---|---|
| Simplicité | Très conventionnel | Très flexible |
| Fichier de build | XML (verbeux) | DSL (Groovy/Kotlin) |
| Standard entreprise | Très répandu | Très répandu aussi, selon contexte |
| Performance | Bonne | Très bonne (build cache, incrémental) |
| Courbe d’apprentissage | Douce | Un peu plus technique |

**Conseil pragmatique** :
- Projet « classique » (API, service, lib) : Maven est souvent le plus simple à standardiser.
- Projet multi-modules complexe ou besoin de customisation : Gradle peut être plus adapté.

---

## 5) Structure standard d’un projet Java

### 5.1 Convention de répertoires
#### Maven / Gradle (standard de fait)

```
project-root/
├─ src/
│  ├─ main/
│  │  ├─ java/
│  │  │  └─ com/example/App.java
│  │  └─ resources/
│  │     └─ application.properties
│  └─ test/
│     ├─ java/
│     │  └─ com/example/AppTest.java
│     └─ resources/
│        └─ ...
├─ pom.xml               (si Maven)
└─ build.gradle          (si Gradle)
```

### 5.2 Rôle des dossiers
- `src/main/java` : code de production
- `src/test/java` : tests unitaires/intégration
- `src/main/resources` : ressources embarquées (fichiers de config, templates, etc.)
- `src/test/resources` : données/fixtures de test

### 5.3 Namespaces/packages
En Java, on organise le code par **packages** (souvent reverse domain) :

- `com.example.monproduit`

> Bonnes pratiques : aligner les packages sur les modules fonctionnels, éviter les packages « fourre-tout ».

### 5.4 Différence mentale .NET vs Java (repères rapides)
- `.csproj` ↔ `pom.xml` / `build.gradle`
- NuGet ↔ Maven Central
- `bin/obj` ↔ `target/` (Maven) / `build/` (Gradle)
- `Properties/` (.NET) ↔ `src/main/resources` (Java)

---

## 6) Atelier : créer, builder, tester, packager

### 6.1 Objectif
Créer un projet minimal, valider l’exécution des tests et générer un JAR.

Vous pouvez choisir Maven **ou** Gradle.

---

### 6.2 Version Maven (exemple guidé)

#### Étape 1 — Créer l’arborescence

```bash
mkdir -p demo/src/main/java/com/example
mkdir -p demo/src/test/java/com/example
cd demo
```

#### Étape 2 — Ajouter `pom.xml`
Créez `pom.xml` (reprendre l’exemple de la section Maven).

#### Étape 3 — Ajouter une classe
`src/main/java/com/example/App.java` :

```java
package com.example;

public class App {
    public static int add(int a, int b) {
        return a + b;
    }

    public static void main(String[] args) {
        System.out.println("Hello Java tooling");
    }
}
```

#### Étape 4 — Ajouter un test JUnit 5
`src/test/java/com/example/AppTest.java` :

```java
package com.example;

import org.junit.jupiter.api.Test;

import static org.junit.jupiter.api.Assertions.*;

class AppTest {

    @Test
    void add_shouldSum() {
        assertEquals(3, App.add(1, 2));
    }
}
```

#### Étape 5 — Builder

```bash
mvn clean test
mvn package
```

Résultat attendu : un JAR dans `target/`.

---

### 6.3 Version Gradle (exemple guidé)

#### Étape 1 — Créer l’arborescence

```bash
mkdir -p demo/src/main/java/com/example
mkdir -p demo/src/test/java/com/example
cd demo
```

#### Étape 2 — Ajouter `build.gradle`
Créez `build.gradle` (reprendre l’exemple de la section Gradle).

#### Étape 3 — Ajouter la classe et le test
Utilisez les mêmes fichiers `App.java` et `AppTest.java`.

#### Étape 4 — Utiliser le wrapper
Si le wrapper n’existe pas, initiez-le :

```bash
gradle wrapper
```

Puis :

```bash
./gradlew clean test
./gradlew build
```

Résultat attendu : un JAR dans `build/libs/`.

---

## 7) Checklist de fin de module

- [ ] Une version **LTS** est choisie (17/21/25) et documentée.
- [ ] `java -version` et `javac -version` pointent sur la bonne version.
- [ ] `JAVA_HOME` est configuré et stable.
- [ ] Un IDE est installé et utilise le bon JDK.
- [ ] Le projet build en CLI sans l’IDE (Maven/Gradle).
- [ ] La structure `src/main/java`, `src/test/java`, `src/main/resources` est respectée.
- [ ] Les tests s’exécutent en local et en CI (à terme).

---

## Annexes

### A. Commandes de diagnostic utiles

```bash
java -version
javac -version
mvn -v
./gradlew -v
```

### B. Erreurs fréquentes et correctifs
- **"Unsupported class file major version"** : mismatch entre version Java de compilation et runtime.
  - Corriger `maven.compiler.release` (Maven) ou `toolchain` (Gradle) + vérifier `JAVA_HOME`.
- **IDE compile mais CLI échoue** : l’IDE utilise un JDK différent.
  - Aligner JDK IDE et JDK CLI.
- **Dépendances non résolues** : proxy/SSL/miroir.
  - Configurer `settings.xml` (Maven) ou `gradle.properties` selon contexte entreprise.
