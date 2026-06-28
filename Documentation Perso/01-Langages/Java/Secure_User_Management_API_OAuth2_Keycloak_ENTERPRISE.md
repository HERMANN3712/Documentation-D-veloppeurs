# SECURE USER MANAGEMENT API

## Spring Boot + Spring Security + JWT + OAuth2 + Keycloak

### Version Enterprise Avancée

------------------------------------------------------------------------

# TABLE DES MATIÈRES

1.  Introduction à la sécurité moderne
2.  Architecture OAuth2 complète
3.  JWT vs OAuth2
4.  Introduction à Keycloak
5.  Installation et configuration Keycloak
6.  Configuration Spring Boot avec OAuth2 Resource Server
7.  Intégration Keycloak avec Spring Security
8.  Gestion des rôles et des claims
9.  Multi-realm & multi-tenant
10. Refresh Token avec Keycloak
11. Authorization Server vs Resource Server
12. Protection API avec scopes
13. Method Security avancée
14. Tests sécurité OAuth2
15. Dockerisation complète avec Keycloak
16. Hardening Production Enterprise
17. Questions d'entretien Senior

------------------------------------------------------------------------

# 1. INTRODUCTION À LA SÉCURITÉ MODERNE

Dans une architecture enterprise moderne :

-   Authentification externalisée
-   API stateless
-   Identity Provider (IdP)
-   Centralisation des utilisateurs
-   SSO (Single Sign-On)

------------------------------------------------------------------------

# 2. OAUTH2 ARCHITECTURE

Acteurs :

-   Resource Owner (Utilisateur)
-   Client (Frontend)
-   Authorization Server (Keycloak)
-   Resource Server (Spring Boot API)

Flux principaux :

-   Authorization Code Flow (Web apps)
-   Client Credentials Flow (Machine-to-machine)
-   Refresh Token Flow

------------------------------------------------------------------------

# 3. JWT vs OAuth2

JWT : - Token auto-signé - Validation locale

OAuth2 : - Protocole d'autorisation - Gestion des permissions -
Centralisation IAM

------------------------------------------------------------------------

# 4. INTRODUCTION À KEYCLOAK

Keycloak est :

-   Identity & Access Management
-   Open Source
-   Compatible OAuth2 / OpenID Connect
-   Gestion des utilisateurs
-   Rôles & permissions
-   SSO

------------------------------------------------------------------------

# 5. INSTALLATION KEYCLOAK (Docker)

``` yaml
version: "3.8"

services:
  keycloak:
    image: quay.io/keycloak/keycloak:latest
    command: start-dev
    environment:
      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD: admin
    ports:
      - "8080:8080"
```

------------------------------------------------------------------------

# 6. CONFIGURATION SPRING BOOT (RESOURCE SERVER)

application.yml

``` yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: http://localhost:8080/realms/myrealm
```

------------------------------------------------------------------------

# 7. SECURITY CONFIGURATION

``` java
@Bean
SecurityFilterChain securityFilterChain(HttpSecurity http)
        throws Exception {

    http
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/api/public/**").permitAll()
            .requestMatchers("/api/admin/**").hasRole("ADMIN")
            .anyRequest().authenticated()
        )
        .oauth2ResourceServer(oauth2 -> oauth2.jwt());

    return http.build();
}
```

------------------------------------------------------------------------

# 8. EXTRACTION DES ROLES KEYCLOAK

Keycloak stocke les rôles dans :

realm_access.roles

Custom converter nécessaire.

``` java
@Bean
public JwtAuthenticationConverter jwtAuthenticationConverter() {
    JwtAuthenticationConverter converter =
        new JwtAuthenticationConverter();

    converter.setJwtGrantedAuthoritiesConverter(jwt -> {
        Collection<String> roles =
            jwt.getClaimAsStringList("roles");

        return roles.stream()
            .map(r -> new SimpleGrantedAuthority("ROLE_" + r))
            .toList();
    });

    return converter;
}
```

------------------------------------------------------------------------

# 9. METHOD SECURITY

``` java
@PreAuthorize("hasRole('ADMIN')")
@GetMapping("/admin/data")
public String adminData() {
    return "secured";
}
```

Activer :

``` java
@EnableMethodSecurity
```

------------------------------------------------------------------------

# 10. REFRESH TOKEN AVEC KEYCLOAK

Keycloak gère :

-   Access Token
-   Refresh Token
-   Rotation automatique
-   Expiration configurable

Spring Boot ne génère plus les tokens → validation seulement.

------------------------------------------------------------------------

# 11. SCOPES & PERMISSIONS

Configurer dans Keycloak :

-   Scopes personnalisés
-   Client roles
-   Realm roles
-   Mappers

Spring peut vérifier :

``` java
@PreAuthorize("hasAuthority('SCOPE_read')")
```

------------------------------------------------------------------------

# 12. TESTS OAUTH2

Utiliser :

-   @SpringBootTest
-   MockJwt
-   Testcontainers pour Keycloak

------------------------------------------------------------------------

# 13. DOCKER COMPOSE COMPLET

Services :

-   app (Spring Boot)
-   postgres
-   keycloak
-   nginx (reverse proxy SSL)

------------------------------------------------------------------------

# 14. HARDENING ENTERPRISE

-   HTTPS obligatoire
-   Reverse proxy
-   Rate limiting
-   Key rotation
-   Token signature RS256 (asymétrique)
-   Monitoring sécurité
-   Audit logs
-   Secrets via Vault

------------------------------------------------------------------------

# 15. ARCHITECTURE ENTERPRISE FINALE

Frontend → Keycloak (Auth) → Spring Boot (Resource Server) → Database

Séparation claire des responsabilités.

------------------------------------------------------------------------

# 16. COMPÉTENCES SENIOR ACQUISES

-   OAuth2 complet
-   OpenID Connect
-   Keycloak administration
-   Resource Server configuration
-   JWT validation RS256
-   Architecture IAM enterprise
-   SSO
-   Multi-tenant

------------------------------------------------------------------------

# 17. QUESTIONS ENTRETIEN SENIOR

-   Différence Authorization Server vs Resource Server
-   RS256 vs HS256
-   Pourquoi externaliser l'authentification
-   Comment fonctionne le flow Authorization Code
-   Comment sécuriser une API microservices
-   Gestion des clés publiques
-   Key rotation

------------------------------------------------------------------------

# CONCLUSION

Cette version apporte un niveau enterprise complet en sécurité backend
avec OAuth2 et Keycloak, adapté aux architectures microservices modernes
et aux environnements sécurisés professionnels.
