# Java en Backend sur Linux

## 1. Installation de Java sur Linux

### Vérifier si Java est installé

``` bash
java -version
```

### Installer OpenJDK (Ubuntu/Debian)

``` bash
sudo apt update
sudo apt install openjdk-21-jdk
```

Vérification :

``` bash
javac -version
```

------------------------------------------------------------------------

## 2. Backend avec Spring Boot

### Création via Spring Initializr

-   Project : Maven
-   Language : Java
-   Dependencies :
    -   Spring Web
    -   Spring Data JPA
    -   PostgreSQL Driver

### Lancer l'application

``` bash
./mvnw spring-boot:run
```

Accès : http://localhost:8080

### Exemple Controller

``` java
@RestController
@RequestMapping("/api")
public class HelloController {

    @GetMapping("/hello")
    public String hello() {
        return "Hello from Linux backend!";
    }
}
```

### Build et exécution

``` bash
mvn clean package
java -jar target/app.jar
```

------------------------------------------------------------------------

## 3. Déploiement Linux via systemd

Fichier : /etc/systemd/system/myapp.service

``` ini
[Unit]
Description=My Spring Boot App
After=network.target

[Service]
User=ubuntu
ExecStart=/usr/bin/java -jar /opt/app/app.jar
SuccessExitStatus=143

[Install]
WantedBy=multi-user.target
```

Commandes :

``` bash
sudo systemctl daemon-reload
sudo systemctl start myapp
sudo systemctl enable myapp
```

------------------------------------------------------------------------

# Java Backend sur Linux SANS Spring

## 1. Serveur HTTP natif Java

``` java
import com.sun.net.httpserver.HttpServer;
import java.io.IOException;
import java.io.OutputStream;
import java.net.InetSocketAddress;

public class Main {
    public static void main(String[] args) throws IOException {
        HttpServer server = HttpServer.create(new InetSocketAddress(8080), 0);

        server.createContext("/hello", exchange -> {
            String response = "Hello from pure Java!";
            exchange.sendResponseHeaders(200, response.length());
            OutputStream os = exchange.getResponseBody();
            os.write(response.getBytes());
            os.close();
        });

        server.start();
        System.out.println("Server started on port 8080");
    }
}
```

Compilation :

``` bash
javac Main.java
java Main
```

------------------------------------------------------------------------

## 2. Exemple avec Undertow

``` java
import io.undertow.Undertow;

public class Main {
    public static void main(String[] args) {
        Undertow server = Undertow.builder()
            .addHttpListener(8080, "0.0.0.0")
            .setHandler(exchange -> {
                exchange.getResponseSender().send("Hello from Undertow!");
            }).build();

        server.start();
    }
}
```

------------------------------------------------------------------------

## 3. Exemple Servlet avec Tomcat

``` java
@WebServlet("/hello")
public class HelloServlet extends HttpServlet {

    protected void doGet(HttpServletRequest req, HttpServletResponse resp)
        throws IOException {

        resp.getWriter().write("Hello from Servlet!");
    }
}
```

------------------------------------------------------------------------

## 4. Jakarta EE (standard entreprise)

Utilise : - JAX-RS (API REST) - JPA (Hibernate) - CDI (Injection)

Déployé sur WildFly ou Payara.

Exemple REST :

``` java
@Path("/hello")
public class HelloResource {

    @GET
    @Produces(MediaType.TEXT_PLAIN)
    public String hello() {
        return "Hello Jakarta EE!";
    }
}
```

------------------------------------------------------------------------

## 5. Connexion Base de Données JDBC

``` java
Connection conn = DriverManager.getConnection(
    "jdbc:postgresql://localhost:5432/test",
    "postgres",
    "password"
);

PreparedStatement stmt = conn.prepareStatement("SELECT * FROM users");
ResultSet rs = stmt.executeQuery();
```

------------------------------------------------------------------------

# Architecture Backend Recommandée

    /controller
    /service
    /repository
    /model
    /config

------------------------------------------------------------------------

# 🎯 En entretien (important)

Si on te demande pourquoi utiliser ou non un framework :

### Avec Spring :

-   Productivité élevée
-   Écosystème riche
-   Sécurité intégrée
-   Convention over configuration

### Sans Spring :

-   Contrôle total sur le cycle HTTP
-   Moins d'abstraction
-   Moins de dépendances
-   Adapté aux microservices très légers ou systèmes embarqués

Argument senior : \> Le choix dépend du contexte projet, des contraintes
de performance, de la maintenabilité et de l'équipe.

------------------------------------------------------------------------

# 📌 En résumé

  Technologie        Usage principal                    Niveau
  ------------------ ---------------------------------- -----------------
  Java HTTP natif    Projet simple / pédagogique        Junior
  Undertow / Jetty   Microservice léger et performant   Confirmé
  Tomcat + Servlet   Web classique standard             Confirmé
  Jakarta EE         Architecture entreprise complète   Senior
  Spring Boot        Productivité maximale entreprise   Standard actuel
