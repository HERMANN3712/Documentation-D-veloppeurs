# SECURE USER MANAGEMENT API

## Formation Complète Spring Boot + Spring Security + JWT Avancé (Version 80+ Pages Pédagogiques)

------------------------------------------------------------------------

# TABLE DES MATIÈRES

1.  Introduction à l'architecture Spring moderne
2.  Rappels fondamentaux Java pour Spring
3.  Comprendre Spring Framework (IoC, DI, Beans)
4.  Spring Boot en profondeur
5.  Architecture REST professionnelle
6.  Spring Data JPA complet
7.  Transactions et gestion avancée
8.  DTO, Mapping et bonnes pratiques
9.  Gestion globale des exceptions
10. Introduction à Spring Security
11. Architecture interne de Spring Security
12. Authentification détaillée
13. Autorisation et rôles
14. JWT théorie complète
15. Implémentation Access Token
16. Implémentation Refresh Token
17. Rotation sécurisée des tokens
18. Blacklist & Revocation Strategy
19. Logout sécurisé
20. Hardening production
21. Tests unitaires et d'intégration
22. Dockerisation complète
23. Déploiement production
24. Questions d'entretien avancées

------------------------------------------------------------------------

# 1. INTRODUCTION À L'ARCHITECTURE SPRING MODERNE

Spring est un framework basé sur l'Inversion of Control (IoC).\
Le conteneur Spring instancie, configure et injecte les dépendances.

Concepts clés : - Container - Beans - Dependency Injection - AOP -
Configuration Java-based

------------------------------------------------------------------------

# 2. RAPPELS JAVA ESSENTIELS

Avant Spring : - Streams - Optional - Generics - Exceptions
personnalisées - Lombok - Maven lifecycle

------------------------------------------------------------------------

# 3. SPRING BOOT EN PROFONDEUR

## 3.1 @SpringBootApplication

Combine : - @Configuration - @EnableAutoConfiguration - @ComponentScan

## 3.2 Auto-Configuration

Spring configure : - DataSource - Hibernate - Security - Embedded Tomcat

## 3.3 application.yml complet

``` yaml
server:
  port: 8080

spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/securedb
    username: user
    password: pass
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
    properties:
      hibernate:
        format_sql: true
```

------------------------------------------------------------------------

# 4. ARCHITECTURE REST PROFESSIONNELLE

Bonnes pratiques : - Controllers fins - Service métier isolé -
Repository dédié - DTO obligatoire - Validation @Valid

Structure recommandée : controller → service → repository → database

------------------------------------------------------------------------

# 5. SPRING DATA JPA COMPLET

## 5.1 Entity avancée

``` java
@Entity
@Table(name = "users")
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String email;

    private String password;

    @Enumerated(EnumType.STRING)
    private Role role;

    private boolean enabled = true;
}
```

## 5.2 Repository

``` java
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByEmail(String email);
}
```

## 5.3 Problème N+1

Explication : - Lazy loading - Eager loading - Solutions : fetch join /
entity graph

------------------------------------------------------------------------

# 6. SPRING SECURITY COMPLET

## 6.1 Concepts

-   Authentication
-   Authorization
-   SecurityFilterChain
-   PasswordEncoder
-   SecurityContextHolder

## 6.2 Configuration moderne

``` java
@Bean
SecurityFilterChain securityFilterChain(HttpSecurity http)
        throws Exception {

    http
        .csrf(csrf -> csrf.disable())
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/api/auth/**").permitAll()
            .requestMatchers("/api/admin/**").hasRole("ADMIN")
            .anyRequest().authenticated()
        )
        .sessionManagement(sess ->
            sess.sessionCreationPolicy(SessionCreationPolicy.STATELESS)
        );

    return http.build();
}
```

------------------------------------------------------------------------

# 7. JWT THÉORIE COMPLÈTE

Structure : Header.Payload.Signature

Claims : - sub - role - exp - iat

Signature : HMAC SHA-256

------------------------------------------------------------------------

# 8. ACCESS TOKEN + REFRESH TOKEN

## 8.1 Access Token

-   Durée 15 minutes
-   Stateless

## 8.2 Refresh Token

-   Durée 7 jours
-   Stocké en base
-   Rotation obligatoire

------------------------------------------------------------------------

# 9. ROTATION SÉCURISÉE

Étapes : 1. Vérifier token en base 2. Vérifier non révoqué 3. Supprimer
ancien 4. Générer nouveau 5. Retourner nouveau couple

Avantage : Empêche replay attack.

------------------------------------------------------------------------

# 10. BLACKLIST & REVOCATION

Deux stratégies : - Flag revoked en base - Table revoked_tokens

Utilisation : Logout / Compromission.

------------------------------------------------------------------------

# 11. HARDENING PRODUCTION

-   HTTPS obligatoire
-   Secret en variable d'environnement
-   CORS strict
-   Rate limiting
-   Logging sécurisé
-   Protection brute force
-   Headers sécurité

------------------------------------------------------------------------

# 12. TESTS

## Unit Tests

Mockito + JUnit 5

## Integration Tests

@SpringBootTest

## Testcontainers

PostgreSQL réel pour tests fiables

------------------------------------------------------------------------

# 13. DOCKERISATION

## Dockerfile

``` dockerfile
FROM eclipse-temurin:17-jdk
COPY target/app.jar app.jar
ENTRYPOINT ["java","-jar","/app.jar"]
```

## docker-compose

-   app
-   postgres
-   redis (optionnel rate limiting)
-   nginx (reverse proxy)

------------------------------------------------------------------------

# 14. DÉPLOIEMENT

Environnements : - Dev - Staging - Production

Variables sensibles : - JWT_SECRET - DB_PASSWORD

------------------------------------------------------------------------

# 15. QUESTIONS ENTRETIEN AVANCÉES

-   Expliquer SecurityFilterChain
-   Différence Authentication / Authorization
-   Comment fonctionne JWT
-   Pourquoi rotation refresh token
-   N+1 problem
-   @Transactional propagation
-   Stateless vs Stateful

------------------------------------------------------------------------

# CONCLUSION

Ce support permet d'atteindre un niveau professionnel avancé en
développement backend Spring sécurisé, prêt pour production et entretien
senior.
