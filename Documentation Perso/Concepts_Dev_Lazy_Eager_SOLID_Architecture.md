# Concepts clés en développement logiciel

## LAZY vs EAGER Loading

-   **Lazy Loading** : Les données associées sont chargées uniquement
    lorsqu'elles sont réellement utilisées.
-   **Eager Loading** : Les données associées sont chargées
    immédiatement avec la requête principale.

## N+1 Problem

Le problème N+1 survient lorsqu'une requête principale est suivie de N
requêtes supplémentaires pour charger des données liées, provoquant un
fort impact sur les performances (souvent en ORM).

------------------------------------------------------------------------

# Méthodes & Architectures (résumé en moins de deux phrases chacune)

## SOLID

Ensemble de cinq principes de conception orientée objet visant à
produire un code maintenable, extensible et testable.

## Architecture Hexagonale

Architecture centrée sur le domaine qui sépare le cœur métier des
dépendances techniques via des ports et adaptateurs.

## CQRS (Command Query Responsibility Segregation)

Séparation des opérations d'écriture (Command) et de lecture (Query)
pour améliorer performance, scalabilité et clarté du modèle.

## Microservices

Architecture distribuée où une application est découpée en services
indépendants, déployables et communicant via API.

## Clean Architecture

Architecture en couches concentriques où le domaine métier est
indépendant des frameworks et des détails techniques.

## DDD (Domain-Driven Design)

Approche de conception centrée sur le domaine métier, utilisant un
langage commun et des modèles riches pour représenter la logique métier.
