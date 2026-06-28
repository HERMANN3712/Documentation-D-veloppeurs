# 14) Tests & qualité (Java)

> Formation : **Tests : JUnit 5** · **Mocks : Mockito** · **Qualité : Checkstyle / SpotBugs / PMD / Sonar** · **Logging : SLF4J + Logback / Log4j2**  
> Durée indicative : 1 à 2 jours (selon profondeur et ateliers)  
> Niveau : intermédiaire (connaissance Java + Maven/Gradle)  

---

## Objectifs pédagogiques

À la fin de cette formation, vous saurez :

- Écrire des **tests unitaires** lisibles et robustes avec **JUnit 5 (Jupiter)**.
- Structurer et maintenir une **stratégie de tests** (unitaires, intégration, end-to-end).
- Utiliser **Mockito** pour isoler les unités (mocks, stubs, spies), vérifier des interactions et contrôler les dépendances.
- Mesurer et améliorer la **qualité** (style, bugs potentiels, complexité) avec **Checkstyle**, **SpotBugs**, **PMD**.
- Mettre en place une chaîne de qualité (local + CI) avec **Sonar** (SonarQube/SonarCloud).
- Implémenter un **logging** cohérent et exploitable avec **SLF4J** et un backend (**Logback** ou **Log4j2**), en respectant les bonnes pratiques.

---

## Pré-requis & outillage

### Pré-requis techniques
- Java 17+ recommandé (Java 11+ possible)
- Maven ou Gradle
- IDE : IntelliJ IDEA / Eclipse

### Dépendances (exemples Maven)

```xml
<dependencies>
  <!-- JUnit 5 -->
  <dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter</artifactId>
    <version>5.10.2</version>
    <scope>test</scope>
  </dependency>

  <!-- Mockito -->
  <dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-junit-jupiter</artifactId>
    <version>5.10.0</version>
    <scope>test</scope>
  </dependency>

  <!-- AssertJ (optionnel mais très utile) -->
  <dependency>
    <groupId>org.assertj</groupId>
    <artifactId>assertj-core</artifactId>
    <version>3.25.3</version>
    <scope>test</scope>
  </dependency>
</dependencies>

<build>
  <plugins>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-surefire-plugin</artifactId>
      <version>3.2.5</version>
      <configuration>
        <useModulePath>false</useModulePath>
      </configuration>
    </plugin>
  </plugins>
</build>
```

> Pour Gradle, adaptez en `testImplementation` / `testRuntimeOnly`.

---

## Plan de la formation

1. **Panorama tests & qualité**
   - Pyramide de tests, ROI, dette de tests
   - Testabilité, conception et découplage
2. **JUnit 5 (Jupiter) – fondamentaux**
   - Structure des tests, assertions, organisation
   - Lifecycle, tags, nested tests
   - Tests paramétrés
3. **JUnit 5 – pratiques avancées**
   - Exceptions, timeouts, répétitions
   - Extensions, injection de dépendances, Testcontainers (intro)
4. **Mockito – isolation et doubles de test**
   - Mocks vs stubs vs fakes vs spies
   - Stubbing, verification, captors
   - Bonnes pratiques et anti-patterns
5. **Qualité : Checkstyle / SpotBugs / PMD**
   - Règles, configuration, exécution dans la build
   - Gestion des faux positifs et baselines
6. **Sonar (Qube/Cloud) – qualité pilotée**
   - Quality Gates, couverture, duplications, security hotspots
   - Intégration CI/CD
7. **Logging : SLF4J + Logback/Log4j2**
   - Niveaux, traces, corrélation, MDC
   - Format, rotation, performance
   - Bonnes pratiques et pièges
8. **Atelier de synthèse**
   - Mise en place complète sur un mini-projet

---

# 1. Panorama tests & qualité

## 1.1 Pourquoi tester ?
- Réduire les régressions
- Documenter le comportement attendu
- Faciliter le refactoring
- Raccourcir le cycle de feedback (local/CI)

## 1.2 Pyramide de tests (et variantes)

- **Unitaires** : rapides, isolés, très nombreux
- **Intégration** : valident assemblage (DB, FS, HTTP), moins nombreux
- **End-to-end** : valident scénario utilisateur, rares et coûteux

Objectif : **maximiser la valeur** avec un coût maîtrisé.

## 1.3 Qu’est-ce qu’un “bon” test ?
- **Lisible** : le test raconte une histoire
- **Déterministe** : pas de dépendance au temps réseau/ordre d’exécution
- **Rapide** : encourage l’exécution fréquente
- **Isolé** : une cause ↔ un échec clair
- **Maintenable** : peu fragile aux refactorings internes

### Anti-patterns fréquents
- Test qui dépend d’un ordre
- Tests “flaky” (horloge, threads, réseau)
- Trop de mocks (test du code + test de Mockito)
- Assertions faibles (ex: `assertTrue(result != null)`)

---

# 2. JUnit 5 (Jupiter) – fondamentaux

## 2.1 Anatomie d’un test

```java
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class CalculatorTest {

  @Test
  void add_shouldReturnSum() {
    var calc = new Calculator();
    int result = calc.add(2, 3);
    assertEquals(5, result);
  }
}
```

### Convention de nommage
- `method_shouldExpected_whenCondition` (ex: `applyDiscount_shouldReducePrice_whenCustomerIsVip`)
- En français si projet francophone, mais homogène.

## 2.2 Assertions utiles

- `assertEquals(expected, actual)`
- `assertAll(...)` : regrouper plusieurs assertions
- `assertThrows(...)` : vérifier une exception
- `assertTimeout(...)` : limite de temps

Exemple `assertAll` :

```java
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class UserTest {
  @Test
  void user_shouldExposeFullName() {
    var u = new User("Ada", "Lovelace");

    assertAll(
      () -> assertEquals("Ada", u.firstName()),
      () -> assertEquals("Lovelace", u.lastName()),
      () -> assertEquals("Ada Lovelace", u.fullName())
    );
  }
}
```

> Astuce : AssertJ améliore la lisibilité (`assertThat(...)`).

## 2.3 Cycle de vie

- `@BeforeEach` / `@AfterEach`
- `@BeforeAll` / `@AfterAll` (souvent `static`)

```java
import org.junit.jupiter.api.*;

class LifecycleTest {
  @BeforeAll
  static void initAll() {}

  @BeforeEach
  void init() {}

  @Test
  void test1() {}

  @AfterEach
  void tearDown() {}

  @AfterAll
  static void tearDownAll() {}
}
```

## 2.4 Organiser les tests

### `@Nested`

```java
import org.junit.jupiter.api.*;
import static org.junit.jupiter.api.Assertions.*;

class PasswordPolicyTest {

  PasswordPolicy policy = new PasswordPolicy();

  @Nested
  class WhenPasswordIsTooShort {
    @Test
    void shouldReject() {
      assertFalse(policy.isValid("abc"));
    }
  }

  @Nested
  class WhenPasswordHasEnoughChars {
    @Test
    void shouldAccept() {
      assertTrue(policy.isValid("abcd1234"));
    }
  }
}
```

### `@Tag` et sélection

```java
import org.junit.jupiter.api.Tag;
import org.junit.jupiter.api.Test;

class TagTest {
  @Test
  @Tag("fast")
  void fastTest() {}

  @Test
  @Tag("integration")
  void integrationTest() {}
}
```

Maven Surefire : configurer l’inclusion/exclusion de tags.

---

# 3. JUnit 5 – pratiques avancées

## 3.1 Tests paramétrés

### `@ValueSource`

```java
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.ValueSource;
import static org.junit.jupiter.api.Assertions.*;

class EmailValidatorTest {

  @ParameterizedTest
  @ValueSource(strings = {"a@b.com", "john.doe@company.org"})
  void isValid_shouldAcceptValidEmails(String email) {
    assertTrue(EmailValidator.isValid(email));
  }
}
```

### `@CsvSource` / `@MethodSource`

```java
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.CsvSource;
import static org.junit.jupiter.api.Assertions.*;

class DiscountServiceTest {

  @ParameterizedTest
  @CsvSource({
    "VIP,100,90",
    "REGULAR,100,100"
  })
  void apply_shouldComputePrice(String customerType, int input, int expected) {
    var service = new DiscountService();
    int result = service.apply(CustomerType.valueOf(customerType), input);
    assertEquals(expected, result);
  }
}
```

### Bonnes pratiques
- Garder les cas **courts** et **justifiés**
- Nommer les scénarios (`@ParameterizedTest(name = "...")`)

## 3.2 Exceptions, timeouts, répétitions

```java
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class ExceptionTest {
  @Test
  void parse_shouldThrow_whenInvalid() {
    assertThrows(IllegalArgumentException.class, () -> Parser.parse("??"));
  }
}
```

```java
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;
import java.time.Duration;

class TimeoutTest {
  @Test
  void compute_shouldFinishQuickly() {
    assertTimeout(Duration.ofMillis(50), () -> Heavy.compute());
  }
}
```

## 3.3 Extensions (intro)
JUnit 5 permet d’étendre le framework via `Extension`.
- Exemple d’usage : gérer des ressources, injecter des paramètres, initialiser un conteneur.

> En pratique : de nombreux frameworks (Spring, Mockito, Testcontainers) fournissent leurs extensions.

---

# 4. Mockito – isolation et doubles de test

## 4.1 Les doubles de test
- **Mock** : objet simulé, on vérifie interactions (`verify`)
- **Stub** : renvoie des valeurs contrôlées
- **Fake** : implémentation alternative simplifiée (ex: repo in-memory)
- **Spy** : enveloppe un objet réel (dangereux si mal maîtrisé)

## 4.2 Exemple : service dépendant d’un repository

### Code “production”

```java
interface UserRepository {
  boolean existsByEmail(String email);
  void save(User user);
}

class UserService {
  private final UserRepository repo;

  UserService(UserRepository repo) {
    this.repo = repo;
  }

  void register(String email) {
    if (repo.existsByEmail(email)) {
      throw new IllegalStateException("Email already used");
    }
    repo.save(new User(email));
  }
}

record User(String email) {}
```

### Test avec Mockito

```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.*;
import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

@ExtendWith(org.mockito.junit.jupiter.MockitoExtension.class)
class UserServiceTest {

  @Mock UserRepository repo;
  @InjectMocks UserService service;

  @Test
  void register_shouldSave_whenEmailNotUsed() {
    when(repo.existsByEmail("a@b.com")).thenReturn(false);

    service.register("a@b.com");

    verify(repo).save(new User("a@b.com"));
  }

  @Test
  void register_shouldThrow_whenEmailAlreadyUsed() {
    when(repo.existsByEmail("a@b.com")).thenReturn(true);

    assertThrows(IllegalStateException.class, () -> service.register("a@b.com"));

    verify(repo, never()).save(any());
  }
}
```

### Points clés
- La règle d’or : **mockez les dépendances** (ports), pas la classe testée.
- Garder les stubbings minimaux : uniquement ce qui est nécessaire au scénario.

## 4.3 Stubbing & matchers

```java
when(repo.existsByEmail(anyString())).thenReturn(false);
```

Attention : si vous utilisez un matcher (`anyString()`), utilisez des matchers pour **tous** les paramètres de l’appel.

## 4.4 Vérification des interactions

```java
verify(repo, times(1)).save(any(User.class));
verifyNoMoreInteractions(repo);
```

> `verifyNoMoreInteractions` est utile, mais peut rendre les tests trop fragiles si l’implémentation évolue.

## 4.5 ArgumentCaptor

```java
@Captor ArgumentCaptor<User> userCaptor;

@Test
void register_shouldSaveWithExpectedEmail() {
  when(repo.existsByEmail("a@b.com")).thenReturn(false);

  service.register("a@b.com");

  verify(repo).save(userCaptor.capture());
  assertEquals("a@b.com", userCaptor.getValue().email());
}
```

## 4.6 Spies (avec prudence)

```java
var list = spy(new ArrayList<String>());
list.add("x");
verify(list).add("x");
```

### Quand éviter les spies ?
- Quand l’objet réel a des effets de bord (I/O, time)
- Quand vous testez surtout des interactions et plus la logique

## 4.7 Anti-patterns Mockito
- Mock d’objets de valeur (DTO, `String`, `List`), sauf cas particulier
- Tests qui reproduisent l’implémentation (trop de `verify`)
- Stubbings “globaux” dans `@BeforeEach` sans lien avec le scénario

---

# 5. Qualité : Checkstyle / SpotBugs / PMD

## 5.1 Checkstyle (conventions de code)

### Objectif
- Uniformiser le style : indentation, nommage, imports, Javadoc...
- Réduire les débats de style en PR

### Maven (exemple)

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-checkstyle-plugin</artifactId>
  <version>3.3.1</version>
  <executions>
    <execution>
      <phase>verify</phase>
      <goals>
        <goal>check</goal>
      </goals>
    </execution>
  </executions>
  <configuration>
    <configLocation>checkstyle.xml</configLocation>
    <consoleOutput>true</consoleOutput>
    <failsOnError>true</failsOnError>
  </configuration>
</plugin>
```

### Bonnes pratiques
- Partir d’un règlement existant (Google Java Style) puis adapter
- Activer progressivement (baseline) pour éviter “l’enfer” au démarrage

## 5.2 SpotBugs (bug patterns)

### Objectif
Détecter des bugs probables (null deref, equals/hashCode, mauvaise API, etc.).

### Maven

```xml
<plugin>
  <groupId>com.github.spotbugs</groupId>
  <artifactId>spotbugs-maven-plugin</artifactId>
  <version>4.8.3.0</version>
  <executions>
    <execution>
      <phase>verify</phase>
      <goals>
        <goal>check</goal>
      </goals>
    </execution>
  </executions>
</plugin>
```

### Gérer les faux positifs
- Supprimer la cause (code)
- Ou annoter : `@SuppressFBWarnings("...")` (avec commentaire)

## 5.3 PMD (maintenabilité, complexité, mauvaises pratiques)

### Objectif
- Identifier code smells (complexité cyclomatique, `if` imbriqués, duplication logique)

### Maven

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-pmd-plugin</artifactId>
  <version>3.22.0</version>
  <executions>
    <execution>
      <phase>verify</phase>
      <goals>
        <goal>check</goal>
      </goals>
    </execution>
  </executions>
</plugin>
```

### Stratégie d’adoption
- Commencer avec quelques règles à forte valeur
- Ajuster les règles selon le contexte (legacy vs greenfield)

---

# 6. Sonar (SonarQube/SonarCloud) – qualité pilotée

## 6.1 Concepts
- **Bugs** : comportements incorrects probables
- **Vulnerabilities** : failles de sécurité
- **Code Smells** : maintenabilité
- **Coverage** : couverture de tests (via rapports, ex: JaCoCo)
- **Duplications** : duplication de code
- **Security Hotspots** : points à revoir (pas forcément vulnérabilités confirmées)

## 6.2 Quality Gate
Un **Quality Gate** est un ensemble de conditions (ex: coverage sur nouveau code > 80%, pas de vulnérabilités, etc.)

Approche recommandée :
- Exiger un niveau élevé sur le **nouveau code**
- Plan de remédiation progressif sur l’existant

## 6.3 Intégration CI
- Lancer analyse Sonar en CI sur PR
- Bloquer le merge si le Quality Gate échoue

> Selon l’environnement (GitHub Actions, GitLab CI, Jenkins), la configuration varie.

## 6.4 Couverture : JaCoCo (indispensable)

### Maven (exemple minimal)

```xml
<plugin>
  <groupId>org.jacoco</groupId>
  <artifactId>jacoco-maven-plugin</artifactId>
  <version>0.8.11</version>
  <executions>
    <execution>
      <goals>
        <goal>prepare-agent</goal>
      </goals>
    </execution>
    <execution>
      <id>report</id>
      <phase>test</phase>
      <goals>
        <goal>report</goal>
      </goals>
    </execution>
  </executions>
</plugin>
```

### Bonnes pratiques “coverage”
- La couverture est un **indicateur**, pas un objectif absolu
- Visons surtout : **couverture pertinente** (branches, cas limites, erreurs)

---

# 7. Logging : SLF4J + Logback / Log4j2

## 7.1 Pourquoi SLF4J ?
SLF4J est une **façade** : votre code dépend d’une API stable (`org.slf4j.Logger`), et vous pouvez changer le backend (Logback, Log4j2) sans réécrire vos logs.

## 7.2 Niveaux de logs
- `TRACE` : très verbeux (debug profond)
- `DEBUG` : diagnostic
- `INFO` : événements métier/technique normaux
- `WARN` : anomalie gérée
- `ERROR` : erreur nécessitant investigation

## 7.3 Bonnes pratiques

### 7.3.1 Ne pas construire les strings à la main

```java
logger.debug("User {} logged in", userId);
```

Éviter :

```java
logger.debug("User " + userId + " logged in");
```

### 7.3.2 Logger les exceptions correctement

```java
try {
  service.call();
} catch (Exception e) {
  logger.error("Call failed for orderId={}", orderId, e);
}
```

### 7.3.3 Corrélation (MDC)
Utiliser le **MDC** pour injecter un identifiant de requête et relier les logs.

```java
import org.slf4j.MDC;

MDC.put("requestId", requestId);
try {
  // traitement
} finally {
  MDC.remove("requestId");
}
```

## 7.4 Logback : configuration de base

`src/main/resources/logback.xml`

```xml
<configuration>
  <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
    <encoder>
      <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} %-5level [%thread] %logger{36} - %msg %n</pattern>
    </encoder>
  </appender>

  <root level="INFO">
    <appender-ref ref="CONSOLE"/>
  </root>
</configuration>
```

## 7.5 Log4j2 : configuration de base

`src/main/resources/log4j2.xml`

```xml
<Configuration status="WARN">
  <Appenders>
    <Console name="Console" target="SYSTEM_OUT">
      <PatternLayout pattern="%d{yyyy-MM-dd HH:mm:ss.SSS} %-5p [%t] %c{1.} - %m%n"/>
    </Console>
  </Appenders>

  <Loggers>
    <Root level="info">
      <AppenderRef ref="Console"/>
    </Root>
  </Loggers>
</Configuration>
```

## 7.6 Structurer les logs
- Préférer des logs **structurés** (clé=valeur) pour l’analyse (ELK, Grafana Loki, Datadog)
- Exemple : `orderId=... customerId=... durationMs=...`

## 7.7 Performance & sécurité
- Ne pas logger de secrets (tokens, mots de passe)
- Éviter `DEBUG`/`TRACE` en prod sauf besoin temporaire
- Échantillonner certains logs volumineux

---

# 8. Atelier de synthèse (fil rouge)

## Objectif
Mettre en place un mini-projet Maven/Gradle avec :
- Tests unitaires JUnit 5
- Mockito pour isoler des dépendances
- Rapports JaCoCo
- Règles Checkstyle/PMD/SpotBugs dans `verify`
- Analyse Sonar (local ou CI)
- Logging SLF4J + configuration Logback/Log4j2

## Sujet proposé : “Order Processing”

### Exigences
- `OrderService.placeOrder(customerId, items)`
- Vérifier la disponibilité via `InventoryClient`
- Persister via `OrderRepository`
- Publier un événement via `EventPublisher`
- Logger les étapes clés (`INFO`) et détails (`DEBUG`)

### Étapes pédagogiques
1. Écrire les tests unitaires (happy path + erreurs)
2. Introduire Mockito (mocks + captor)
3. Ajouter JaCoCo, interpréter la couverture
4. Ajouter Checkstyle/PMD/SpotBugs et corriger les alertes
5. Ajouter Sonar et configurer un Quality Gate “nouveau code”
6. Ajouter MDC `requestId` et vérifier la trace

---

## Annexes

### A. Checklist “qualité & tests”
- [ ] Les tests unitaires s’exécutent en < 2s (projet moyen)
- [ ] Pas de tests flaky
- [ ] Les tests racontent le scénario (noms + arrange/act/assert)
- [ ] Les règles de style sont automatiques (Checkstyle)
- [ ] Les bugs potentiels sont analysés (SpotBugs)
- [ ] Les code smells sont suivis (PMD + Sonar)
- [ ] Couverture suivie (JaCoCo) mais utilisée intelligemment
- [ ] Logs : niveaux cohérents, exceptions loggées, MDC, pas de secrets

### B. Arrange / Act / Assert (AAA)

```java
@Test
void example() {
  // Arrange
  // Act
  // Assert
}
```

### C. Conseils de progression (legacy)
- Encapsuler les zones difficiles à tester
- Ajouter des tests “caractérisation” avant refactoring
- Monter progressivement le Quality Gate sur le nouveau code

---

*Fin du module 14 — Tests & qualité.*
