# 🎯 FICHE MÉMO ENTRETIEN --- JAVA BACKEND SUR LINUX

------------------------------------------------------------------------

# 1️⃣ Stack Classique Java Backend (Linux)

**Environnement :** - Linux (Ubuntu / Debian / CentOS) - OpenJDK 17 /
21 - Maven ou Gradle - systemd pour service - Nginx en reverse proxy -
PostgreSQL / MySQL

------------------------------------------------------------------------

# 2️⃣ Java Backend AVEC Framework (Spring Boot)

## Pourquoi utilisé en entreprise ?

✔ Productivité rapide\
✔ Injection de dépendances\
✔ REST simplifié\
✔ Sécurité intégrée\
✔ JPA / Hibernate\
✔ Écosystème mature

## Commandes importantes

Build :

``` bash
mvn clean package
```

Run :

``` bash
java -jar app.jar
```

Service Linux :

``` bash
sudo systemctl start myapp
```

------------------------------------------------------------------------

# 3️⃣ Java Backend SANS Spring

## 🔹 Java HTTP natif

-   com.sun.net.httpserver.HttpServer
-   Approche pédagogique
-   Pas scalable entreprise

## 🔹 Undertow / Jetty

-   Serveur léger embarqué
-   Très performant
-   Adapté microservices optimisés

## 🔹 Tomcat + Servlet

-   Standard historique
-   Basé sur Servlet API
-   Architecture WAR

## 🔹 Jakarta EE

-   Standard entreprise officiel
-   JAX-RS (REST)
-   JPA
-   CDI
-   Déployé sur WildFly / Payara

------------------------------------------------------------------------

# 4️⃣ Architecture Backend Propre

Structure recommandée :

/controller\
/service\
/repository\
/model\
/config

Principe : Controller → Service → Repository → DB

Compatible : - Clean Architecture - DDD - Hexagonale

------------------------------------------------------------------------

# 5️⃣ Déploiement Linux (important en entretien)

## Jar exécutable

``` bash
java -jar app.jar
```

## Service systemd

-   Auto start au boot
-   Gestion logs
-   Restart automatique

## Reverse proxy Nginx

-   HTTPS
-   Redirection port 80 → 8080
-   Sécurisation

------------------------------------------------------------------------

# 6️⃣ Comparaison rapide (à dire à l'oral)

  Solution           Avantage                Quand l'utiliser
  ------------------ ----------------------- ----------------------------
  Java HTTP natif    Contrôle total          Pédagogique
  Undertow / Jetty   Ultra léger             Microservice performant
  Tomcat + Servlet   Standard historique     Projet classique
  Jakarta EE         Standard entreprise     Architecture robuste
  Spring Boot        Productivité maximale   Standard actuel entreprise

------------------------------------------------------------------------

# 7️⃣ Réponse type en entretien

💬 Exemple :

> "En backend Java sous Linux, j'utilise OpenJDK avec Maven.\
> Je peux développer avec Spring Boot pour la productivité,\
> ou sans framework via Undertow ou Servlet API selon le besoin.\
> Le déploiement se fait en jar exécutable, souvent derrière Nginx,\
> avec un service systemd pour la production."

------------------------------------------------------------------------

# 8️⃣ Points techniques à maîtriser

✔ JVM\
✔ Garbage Collector\
✔ Threads / Concurrency\
✔ JDBC\
✔ Pool de connexions\
✔ Logs (Logback / Log4j)\
✔ Variables d'environnement Linux\
✔ Docker (bonus moderne)

------------------------------------------------------------------------

# 🔥 Niveau Senior --- Différence clé

Un senior explique : - Pourquoi choisir ou non un framework - Impact
performance / mémoire - Maintenabilité long terme - Couplage faible -
Observabilité (logs, métriques) - Sécurité

------------------------------------------------------------------------

# 📌 Résumé Final

Backend Java sur Linux =

JVM + Serveur HTTP + Architecture propre + DB + Déploiement maîtrisé.

Le choix du framework dépend du contexte, pas d'une préférence
personnelle.
