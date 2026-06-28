# Jakarta EE (ex Java EE)

## Définition

Jakarta EE est la plateforme standard Java pour développer des
applications d'entreprise (web, API REST, microservices, systèmes
distribués).\
C'est la successeure de Java EE, désormais maintenue par l'Eclipse
Foundation.

------------------------------------------------------------------------

## À quoi sert Jakarta EE ?

Elle fournit un ensemble de spécifications standardisées pour construire
:

-   Applications web (Servlet, JSP)
-   API REST (JAX-RS)
-   Gestion de la persistance (JPA)
-   Injection de dépendances (CDI)
-   Transactions (JTA)
-   Sécurité
-   Messaging (JMS)

Objectif : éviter de tout coder from scratch et garantir la portabilité
entre serveurs d'applications.

------------------------------------------------------------------------

## Serveurs compatibles

-   WildFly\
-   Payara Server\
-   Apache Tomcat (Servlet uniquement)

------------------------------------------------------------------------

## Architecture type

Controller (JAX-RS / Servlet)\
↓\
Service (CDI / EJB)\
↓\
Repository (JPA)\
↓\
Base de données

------------------------------------------------------------------------

## Changement Java EE → Jakarta EE

Transfert d'Oracle vers la Eclipse Foundation en 2017.

Changement majeur :

Avant :

``` java
import javax.persistence.Entity;
```

Maintenant :

``` java
import jakarta.persistence.Entity;
```

------------------------------------------------------------------------

## Jakarta EE vs Spring

  Jakarta EE                    Spring
  ----------------------------- ---------------------------------
  Standard officiel             Framework
  Basé sur des spécifications   Implémentation opinionated
  Serveur d'applications        Souvent embarqué (Spring Boot)
  Enterprise historique         Très populaire en microservices

------------------------------------------------------------------------

## Résumé entretien (3 phrases)

Jakarta EE est la plateforme standard Java pour développer des
applications d'entreprise.\
Elle fournit des spécifications comme JPA, CDI, JAX-RS pour structurer
les applications backend.\
C'est l'évolution de Java EE, désormais maintenue par la Eclipse
Foundation.
