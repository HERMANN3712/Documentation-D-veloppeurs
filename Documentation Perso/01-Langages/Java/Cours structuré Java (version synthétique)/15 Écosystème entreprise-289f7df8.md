# 15) Écosystème entreprise (Java)

> Public cible : développeurs (ex. .NET/C#) souhaitant comprendre et pratiquer l’écosystème Java entreprise.
>
> Pré-requis : bases Java (syntaxe, OOP), notions HTTP/REST, SQL, Git.
>
> Format suggéré : 1 à 2 jours (7–14h) – adaptable.

---

## Objectifs pédagogiques

À l’issue de cette formation, vous serez capable de :

- Choisir et démarrer un projet **Web/API** en **Spring Boot** ou **Jakarta EE**.
- Mettre en place la **persistance** avec **JDBC** et/ou **JPA/Hibernate**.
- Construire, tester et packager avec **Maven** ou **Gradle**.
- Concevoir une chaîne **CI/CD** (pipelines) pour build, tests, qualité et livraison.
- Instrumenter une application pour l’**observabilité** : **logs**, **metrics**, **tracing** (Micrometer / OpenTelemetry selon la stack).

---

## Plan de la formation

1. **Panorama de l’écosystème entreprise Java**
2. **Web/API**
   - 2.1 Spring Boot (REST, validation, erreurs)
   - 2.2 Jakarta EE (JAX-RS, CDI, MicroProfile)
3. **Persistance**
   - 3.1 JDBC (DAO, transactions)
   - 3.2 JPA/Hibernate (entités, relations, requêtes)
4. **Build & dépendances**
   - 4.1 Maven
   - 4.2 Gradle
5. **CI/CD & pipelines**
   - 5.1 Stratégie (branches, versioning)
   - 5.2 Pipeline type (build/test/quality/package/deploy)
6. **Observabilité**
   - 6.1 Logging (SLF4J/Logback ou JUL/Log4j2)
   - 6.2 Metrics (Micrometer, Prometheus)
   - 6.3 Tracing (OpenTelemetry, propagation)
7. **Atelier fil rouge** (API + DB + CI/CD + Observabilité)

---

# 1. Panorama de l’écosystème entreprise Java

## 1.1 Java en entreprise : repères rapides

- **JDK** : environnement d’exécution + compilation. En entreprise, on fixe une version (ex. 17/21 LTS).
- **Frameworks Web** : Spring (dominant), Jakarta EE (standard), frameworks complémentaires (Quarkus, Micronaut).
- **Gestion de dépendances** : Maven/Gradle (analogue à NuGet, mais avec conventions Java).
- **Observabilité** : codifiée (logs structurés, métriques Prometheus, traces distribuées).

## 1.2 Spring vs Jakarta EE (comment choisir)

### Spring Boot
- Très productif, écosystème riche (Spring Data, Spring Security, Spring Cloud).
- Approche “opinionated” : conventions fortes.
- Démarrage très rapide (starter dependencies, auto-configuration).

### Jakarta EE
- Standard (spécification) avec implémentations (WildFly, Payara, Open Liberty, etc.).
- API stables, orientées conteneur d’application.
- Souvent associé à **MicroProfile** pour le cloud-native (config, metrics, health, openapi).

**Heuristique de choix**
- Besoin d’intégration massive/écosystème : **Spring Boot**.
- Besoin de standardisation, portabilité serveur : **Jakarta EE**.

---

# 2. Web/API

Objectif : exposer des endpoints REST propres, documentés, testables.

## 2.1 Spring Boot : API REST

### 2.1.1 Démarrage d’un projet
- Dépendances typiques :
  - `spring-boot-starter-web`
  - `spring-boot-starter-validation`
  - `spring-boot-starter-actuator` (observabilité)
  - `spring-boot-starter-test`

Structure fréquente :
- `controller` (API)
- `service` (métier)
- `repository` (accès données)
- `dto` (contrats)

### 2.1.2 Contrôleur REST minimal

```java
@RestController
@RequestMapping("/api/customers")
public class CustomerController {

  private final CustomerService service;

  public CustomerController(CustomerService service) {
    this.service = service;
  }

  @GetMapping
  public List<CustomerDto> list() {
    return service.list();
  }

  @GetMapping("/{id}")
  public CustomerDto get(@PathVariable long id) {
    return service.get(id);
  }

  @PostMapping
  @ResponseStatus(HttpStatus.CREATED)
  public CustomerDto create(@Valid @RequestBody CreateCustomerRequest req) {
    return service.create(req);
  }
}
```

Points clés :
- `@RestController` = sérialisation JSON via Jackson.
- `@Valid` : validation Bean Validation (Jakarta Validation).
- `@ResponseStatus` : code HTTP explicite.

### 2.1.3 Validation

```java
public record CreateCustomerRequest(
  @NotBlank String name,
  @Email String email
) {}
```

### 2.1.4 Gestion d’erreurs (équivalent “middleware”)

Avec `@ControllerAdvice` :

```java
@RestControllerAdvice
public class ApiExceptionHandler {

  @ExceptionHandler(NotFoundException.class)
  @ResponseStatus(HttpStatus.NOT_FOUND)
  public Map<String, Object> notFound(NotFoundException ex) {
    return Map.of(
      "error", "NOT_FOUND",
      "message", ex.getMessage()
    );
  }
}
```

### 2.1.5 Tests API

- Tests unitaires du service (JUnit 5 + Mockito)
- Tests web : `@WebMvcTest` (slice test)
- Tests intégration : `@SpringBootTest` + Testcontainers (DB)

---

## 2.2 Jakarta EE : API REST (JAX-RS) + CDI

### 2.2.1 Concepts de base
- **JAX-RS** : API standard REST (`@Path`, `@GET`, `@POST`…)
- **CDI** : injection de dépendances (`@Inject`, scopes)
- **JSON-B / JSON-P** : sérialisation JSON (selon runtime)

### 2.2.2 Ressource JAX-RS

```java
@Path("/customers")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class CustomerResource {

  @Inject
  CustomerService service;

  @GET
  public List<CustomerDto> list() {
    return service.list();
  }

  @GET
  @Path("/{id}")
  public CustomerDto get(@PathParam("id") long id) {
    return service.get(id);
  }

  @POST
  public Response create(CreateCustomerRequest req) {
    CustomerDto created = service.create(req);
    return Response.status(Response.Status.CREATED).entity(created).build();
  }
}
```

### 2.2.3 Validation & erreurs
- Bean Validation via annotations (`@NotBlank`, `@Email`) et intégration container.
- Exception mappers :

```java
@Provider
public class NotFoundMapper implements ExceptionMapper<NotFoundException> {
  @Override
  public Response toResponse(NotFoundException ex) {
    return Response.status(Response.Status.NOT_FOUND)
      .entity(Map.of("error", "NOT_FOUND", "message", ex.getMessage()))
      .build();
  }
}
```

### 2.2.4 MicroProfile (optionnel mais fréquent)
- **Config** : externalisation configuration
- **Metrics** : métriques standard
- **Health** : endpoints health
- **OpenAPI** : doc auto

---

# 3. Persistance

Objectif : comprendre deux approches complémentaires :

- **JDBC** : bas niveau, contrôle total, performant mais verbeux.
- **JPA/Hibernate** : ORM, productif, attention aux pièges (N+1, lazy loading).

## 3.1 JDBC

### 3.1.1 Notions
- `DataSource` : pool de connexions.
- `Connection/PreparedStatement/ResultSet`.
- Transactions : `commit/rollback`.

### 3.1.2 Exemple DAO JDBC

```java
public class CustomerJdbcDao {

  private final DataSource ds;

  public CustomerJdbcDao(DataSource ds) {
    this.ds = ds;
  }

  public List<Customer> findAll() {
    String sql = "SELECT id, name, email FROM customers";

    try (Connection c = ds.getConnection();
         PreparedStatement ps = c.prepareStatement(sql);
         ResultSet rs = ps.executeQuery()) {

      List<Customer> out = new ArrayList<>();
      while (rs.next()) {
        out.add(new Customer(
          rs.getLong("id"),
          rs.getString("name"),
          rs.getString("email")
        ));
      }
      return out;

    } catch (SQLException e) {
      throw new RuntimeException(e);
    }
  }
}
```

### 3.1.3 Transactions JDBC

```java
try (Connection c = ds.getConnection()) {
  c.setAutoCommit(false);
  try {
    // opérations
    c.commit();
  } catch (Exception ex) {
    c.rollback();
    throw ex;
  }
}
```

**Bonnes pratiques**
- Toujours utiliser `PreparedStatement` (sécurité/perf).
- Gérer proprement la fermeture via try-with-resources.
- Externaliser le mapping (RowMapper) si réutilisé.

---

## 3.2 JPA / Hibernate

### 3.2.1 Concepts clés
- **Entity** : objet persisté, mappé table.
- **EntityManager** / `Session` Hibernate.
- **Persistence context** : cache 1er niveau.
- **Transactions** : frontières transactionnelles.

### 3.2.2 Exemple d’entité

```java
@Entity
@Table(name = "customers")
public class CustomerEntity {

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  @Column(nullable = false)
  private String name;

  @Column(nullable = false, unique = true)
  private String email;

  protected CustomerEntity() {}

  public CustomerEntity(String name, String email) {
    this.name = name;
    this.email = email;
  }

  // getters
}
```

### 3.2.3 Repository (Spring Data JPA) ou DAO JPA

**Spring Data JPA** :

```java
public interface CustomerRepository extends JpaRepository<CustomerEntity, Long> {
  Optional<CustomerEntity> findByEmail(String email);
}
```

**Jakarta EE (EntityManager)** :

```java
@ApplicationScoped
public class CustomerJpaDao {

  @PersistenceContext
  EntityManager em;

  public CustomerEntity find(long id) {
    return em.find(CustomerEntity.class, id);
  }
}
```

### 3.2.4 Relations & pièges

- `@OneToMany` / `@ManyToOne` : attention au `fetch = LAZY/EAGER`.
- Problème **N+1** : éviter de charger en boucle ; utiliser `JOIN FETCH`.

Exemple requête JPQL :

```java
TypedQuery<CustomerEntity> q = em.createQuery(
  "select c from CustomerEntity c where c.email = :email",
  CustomerEntity.class
);
q.setParameter("email", email);
```

### 3.2.5 Migration et schéma
- Solutions courantes : **Flyway** / **Liquibase**.
- En dev : génération de schéma possible, mais en prod : migrations versionnées.

---

# 4. Build & dépendances

## 4.1 Maven

### 4.1.1 Concepts
- `pom.xml` : coordonnées (groupId/artifactId/version), dépendances, plugins.
- Lifecycle : `validate`, `compile`, `test`, `package`, `verify`, `install`, `deploy`.

### 4.1.2 Exemple `pom.xml` minimal (extrait)

```xml
<project>
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.acme</groupId>
  <artifactId>customers-api</artifactId>
  <version>1.0.0</version>

  <properties>
    <maven.compiler.release>17</maven.compiler.release>
  </properties>

  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
  </dependencies>
</project>
```

### 4.1.3 Plugins fréquents
- Surefire/Failsafe : tests unitaires/intégration
- JaCoCo : couverture
- SpotBugs/Checkstyle : qualité

## 4.2 Gradle

### 4.2.1 Concepts
- DSL Groovy ou Kotlin (`build.gradle` / `build.gradle.kts`).
- Tâches déclaratives, builds rapides.

### 4.2.2 Exemple Gradle Kotlin DSL

```kotlin
plugins {
  id("java")
}

group = "com.acme"
version = "1.0.0"

java {
  toolchain {
    languageVersion.set(JavaLanguageVersion.of(17))
  }
}

dependencies {
  testImplementation(platform("org.junit:junit-bom:5.10.0"))
  testImplementation("org.junit.jupiter:junit-jupiter")
}

tasks.test {
  useJUnitPlatform()
}
```

---

# 5. CI/CD & pipelines

Objectif : rendre la livraison reproductible : build → tests → qualité → package → déploiement.

## 5.1 Stratégie de base

- **Branching** : trunk-based ou GitFlow (souvent trunk-based + PR).
- **Versioning** : SemVer (1.2.3) ou version pipeline (build number).
- **Artefacts** : jar/war, image Docker, libs.

## 5.2 Pipeline type (exemple générique)

### Étapes recommandées
1. **Checkout**
2. **Restore/Cache dépendances** (Maven local repo / Gradle cache)
3. **Build** (`mvn -B clean package` ou `./gradlew build`)
4. **Tests unitaires** + rapport
5. **Tests intégration** (optionnel, souvent avec Testcontainers)
6. **Analyse qualité** (Sonar, SpotBugs, Checkstyle)
7. **Packaging** (jar) + publication (Nexus/Artifactory)
8. **Build image** (Docker) + push registry
9. **Déploiement** (dev/staging/prod) + smoke tests

### Exemple (pseudo YAML)

```yaml
stages:
  - build
  - test
  - quality
  - package
  - deploy

build:
  stage: build
  script:
    - mvn -B -DskipTests clean compile

unit_tests:
  stage: test
  script:
    - mvn -B test
  artifacts:
    reports:
      junit: target/surefire-reports/*.xml

quality:
  stage: quality
  script:
    - mvn -B verify

package:
  stage: package
  script:
    - mvn -B -DskipTests package

deploy_dev:
  stage: deploy
  script:
    - ./deploy.sh dev
  when: manual
```

### Bonnes pratiques CI/CD
- Build **idempotent** et reproductible.
- Pas de secrets en dur (vault/variables protégées).
- Séparer tests unitaires vs intégration.
- “Shift-left” sécurité (SCA dépendances, scan image).

---

# 6. Observabilité

Objectif : diagnostiquer rapidement en production : **quoi**, **pourquoi**, **où**.

## 6.1 Logs

### 6.1.1 Principes
- Logs **structurés** (JSON) plutôt que texte libre.
- Niveaux : `ERROR`, `WARN`, `INFO`, `DEBUG`, `TRACE`.
- Corrélation : inclure un **correlation-id / trace-id**.

### 6.1.2 SLF4J (façade) + backend (Logback / Log4j2)

```java
private static final Logger log = LoggerFactory.getLogger(MyService.class);

log.info("Creating customer email={}", email);
```

### 6.1.3 MDC (corrélation)

```java
MDC.put("correlationId", correlationId);
try {
  // traitement
} finally {
  MDC.clear();
}
```

Avec Spring Boot, on peut propager via filtres servlet/interceptors.

---

## 6.2 Metrics

### 6.2.1 Pourquoi
- Mesurer : débit, latence, taux d’erreur, saturation ressources.
- Alimenter alerting (SLO/SLA).

### 6.2.2 Micrometer (Spring)

- Micrometer = façade de métriques (Prometheus, Datadog, etc.).
- Avec Spring Boot Actuator : endpoints ` /actuator/metrics` et ` /actuator/prometheus`.

Exemple compteur :

```java
private final Counter createdCounter;

public CustomerService(MeterRegistry registry) {
  this.createdCounter = Counter.builder("customers_created_total")
    .description("Number of created customers")
    .register(registry);
}

public CustomerDto create(CreateCustomerRequest req) {
  createdCounter.increment();
  // ...
}
```

**Jakarta EE / MicroProfile** : `@Counted`, `@Timed` selon implémentation.

---

## 6.3 Tracing distribué

### 6.3.1 Concepts
- **Trace** : parcours d’une requête à travers plusieurs services.
- **Span** : segment (DB query, appel HTTP sortant...
- Propagation : headers `traceparent` (W3C).

### 6.3.2 OpenTelemetry (OTel)

- Standard de collecte traces/logs/metrics.
- Instrumentation : auto (agents) + manuelle (spans custom).

Exemple conceptuel (span manuel) :

```java
Span span = tracer.spanBuilder("customer.create").startSpan();
try (Scope scope = span.makeCurrent()) {
  // code
} catch (Exception e) {
  span.recordException(e);
  throw e;
} finally {
  span.end();
}
```

### 6.3.3 Recommandations
- Échantillonnage (sampling) adapté.
- Ne pas tracer des données sensibles.
- Corréler logs et traces (traceId dans MDC).

---

# 7. Atelier fil rouge (proposition)

## 7.1 Énoncé
Construire une API “Customers” :
- Endpoints : liste, détail, création.
- Persistance : table `customers`.
- Tests : unitaires + intégration.
- Pipeline CI : build, tests, rapport.
- Observabilité : logs corrélés, métriques de base, tracing.

## 7.2 Découpage par étapes

1. **Scaffold** projet (Spring Boot ou Jakarta EE)
2. **Contrats API** (DTO + validation)
3. **Implémentation service**
4. **Accès DB** (JDBC puis refactor JPA)
5. **Migrations** (Flyway)
6. **Instrumentation** :
   - logs structurés + correlation id
   - metrics (counter/timer)
   - traces (OTel)
7. **CI pipeline**

## 7.3 Critères de réussite
- Endpoints répondent avec codes HTTP corrects.
- Erreurs standardisées (payload cohérent).
- Build reproductible en CI.
- Dashboards possibles (Prometheus/Grafana, Jaeger/Tempo selon stack).

---

## Annexes

### A. Glossaire rapide
- **DTO** : Data Transfer Object (contrat API).
- **ORM** : Object-Relational Mapping.
- **SLO/SLA** : objectifs/engagements de service.
- **MDC** : contexte de logs par thread.

### B. Anti-patterns courants
- Exceptions non maîtrisées → 500 partout.
- JPA : `EAGER` partout, `N+1`, lazy loading hors transaction.
- Logs trop verbeux en prod, ou sans corrélation.
- Pipeline fragile (dépend de l’état de la machine).

### C. Pour aller plus loin
- Spring Boot Actuator, Spring Security
- MicroProfile OpenAPI/Health/Metrics
- Testcontainers, Pact (contract testing)
- OpenTelemetry Collector + exporters
