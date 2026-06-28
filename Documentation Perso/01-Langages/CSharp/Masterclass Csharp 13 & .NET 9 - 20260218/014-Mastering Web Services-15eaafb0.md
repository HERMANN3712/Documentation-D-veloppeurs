# Mastering Web Services (C# / .NET 9)

**Public** : Développeurs et formateurs .NET/C# (intermédiaire → avancé)  
**Durée suggérée** : 2 jours (14h) ou 3 demi-journées (12h)  
**Format** : Alternance théorie + démos + labs guidés  
**Pré-requis** : C# moderne, ASP.NET Core (minimal APIs ou controllers), notions HTTP/JSON, auth de base

---

## Objectifs pédagogiques

À l’issue de la formation, vous saurez :

- Concevoir, documenter et consommer des **REST APIs** robustes (ressources, verbes, statuts, idempotence, erreurs).
- Mettre en place et exploiter **gRPC** (contrats protobuf, streaming, interop).
- Exposer une API **GraphQL** et maîtriser le modèle *schema-first*/*code-first*.
- Utiliser **HttpClient** correctement (HttpClientFactory, handlers, resilience, pooling).
- Gérer la **versioning** d’API (URL, header, media type) et publier une politique claire.
- Implémenter la **pagination** (offset, cursor), tri/filtre, HATEOAS optionnel.
- Sécuriser des Web Services : **OAuth2/OIDC**, JWT, scopes/claims, CORS, rate limiting, validation, OWASP API.

---

## Plan détaillé

1. **Fondations : HTTP, contrats et bonnes pratiques**
2. **REST APIs** (design, erreurs, documentation, test)
3. **HttpClient** (consommation robuste, resilience, observabilité)
4. **API Versioning** (strategies + compatibilité)
5. **Pagination** (offset vs cursor, tri/filtre)
6. **Sécurité des Web Services** (authN/authZ, OWASP, rate limiting)
7. **gRPC** (protos, unary, streaming, interop)
8. **GraphQL** (schema, resolvers, performance, auth)
9. **Capstone** : architecture de service et mise en production (checklist)

---

# 1) Fondations : HTTP, contrats et bonnes pratiques

## 1.1 HTTP en pratique

- **Méthodes** : GET (safe), POST (création/commande), PUT (remplacement), PATCH (modification partielle), DELETE.
- **Idempotence** :
  - GET/PUT/DELETE sont *idempotents* (même requête répétée → même résultat).
  - POST ne l’est pas par défaut → prévoir **idempotency key** si nécessaire.
- **Codes de statut** (sélection utile en API) :
  - 200 OK, 201 Created (+ `Location`), 202 Accepted, 204 No Content
  - 400 Bad Request (validation), 401 Unauthorized, 403 Forbidden
  - 404 Not Found, 409 Conflict, 412 Precondition Failed, 422 Unprocessable Entity
  - 429 Too Many Requests, 500/503 côté serveur

## 1.2 Contrats, compatibilité et évolutivité

- Définir un **contrat** stable : champs requis vs optionnels, valeurs par défaut, formats.
- Évolution **backward compatible** :
  - Ajouter des champs optionnels (OK)
  - Éviter de renommer/supprimer sans versioning
- **Content negotiation** : `Accept`, `Content-Type`.

## 1.3 Recommandations transverses

- **Corrélation** : `traceparent` (W3C), `X-Correlation-Id`.
- **Time-outs** explicites.
- **Observabilité** : logs structurés, métriques, traces (OpenTelemetry).

---

# 2) REST APIs

## 2.1 Ressources, routes et conventions

### Modèle ressource
- Ressource = un **nom** (souvent pluriel) + **identifiant** :
  - `/api/customers/{id}`
  - `/api/customers/{id}/orders`

### Verbes HTTP
- **Créer** : `POST /api/customers`
- **Lire** : `GET /api/customers/{id}`
- **Mettre à jour** : `PUT /api/customers/{id}` ou `PATCH`
- **Supprimer** : `DELETE /api/customers/{id}`

### Exemple minimal API (.NET 9)

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();
app.UseSwagger();
app.UseSwaggerUI();

var customers = new Dictionary<Guid, Customer>();

app.MapPost("/api/customers", (CreateCustomerRequest req) =>
{
    var id = Guid.NewGuid();
    var customer = new Customer(id, req.Name, req.Email);
    customers[id] = customer;

    return Results.Created($"/api/customers/{id}", customer);
});

app.MapGet("/api/customers/{id:guid}", (Guid id) =>
{
    return customers.TryGetValue(id, out var c)
        ? Results.Ok(c)
        : Results.NotFound(new ProblemDetails
        {
            Title = "Customer not found",
            Status = StatusCodes.Status404NotFound,
            Detail = $"No customer with id '{id}'"
        });
});

app.Run();

record CreateCustomerRequest(string Name, string Email);
record Customer(Guid Id, string Name, string Email);
```

## 2.2 Validation et erreurs : Problem Details (RFC 7807)

**But** : retourner une structure d’erreur standard.

Exemple :

```json
{
  "type": "https://httpstatuses.com/400",
  "title": "Validation failed",
  "status": 400,
  "detail": "Email is invalid",
  "instance": "/api/customers"
}
```

En ASP.NET Core :

- Utiliser `Results.Problem(...)` / `ProblemDetails`
- Centraliser via un middleware de gestion d’exceptions.

## 2.3 Documentation : OpenAPI/Swagger

Bonnes pratiques :

- Décrire les **codes de réponses** et exemples.
- Documenter **auth** (bearer, oauth2), scopes.
- Versionner votre **OpenAPI**.

## 2.4 ETags et concurrence optimiste

- Le client récupère une ressource avec un **ETag**.
- À l’update, il envoie `If-Match: <etag>`.
- En cas de divergence : `412 Precondition Failed`.

## 2.5 Tests d’API

- **Tests d’intégration** : `WebApplicationFactory` (ASP.NET Core Testing).
- Assertions : statuts, payload, headers, erreurs.

---

# 3) HttpClient : consommation robuste

## 3.1 Les pièges classiques

- Créer `new HttpClient()` par requête → **épuisement de sockets**.
- Absence de timeout/global policy.
- Gestion naïve des erreurs (variante “ça marche en dev”).

## 3.2 HttpClientFactory

### Enregistrement

```csharp
builder.Services.AddHttpClient("Catalog", client =>
{
    client.BaseAddress = new Uri("https://api.example.com/");
    client.DefaultRequestHeaders.Accept.ParseAdd("application/json");
    client.Timeout = TimeSpan.FromSeconds(10);
});
```

### Utilisation

```csharp
public sealed class CatalogClient(IHttpClientFactory factory)
{
    public async Task<ProductDto?> GetProductAsync(Guid id, CancellationToken ct)
    {
        var http = factory.CreateClient("Catalog");
        using var resp = await http.GetAsync($"products/{id}", ct);

        if (resp.StatusCode == HttpStatusCode.NotFound)
            return null;

        resp.EnsureSuccessStatusCode();
        return await resp.Content.ReadFromJsonAsync<ProductDto>(cancellationToken: ct);
    }
}

public sealed record ProductDto(Guid Id, string Name);
```

## 3.3 DelegatingHandlers : cross-cutting

- Ajout de headers (correlation id)
- Auth (Bearer token)
- Logs
- Retry/timeout/circuit breaker (via librairies de resilience)

```csharp
public sealed class CorrelationIdHandler : DelegatingHandler
{
    protected override Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken ct)
    {
        request.Headers.TryAddWithoutValidation("X-Correlation-Id", Activity.Current?.TraceId.ToString());
        return base.SendAsync(request, ct);
    }
}
```

Enregistrement :

```csharp
builder.Services.AddTransient<CorrelationIdHandler>();

builder.Services.AddHttpClient("Catalog", c => c.BaseAddress = new Uri("https://api.example.com/"))
    .AddHttpMessageHandler<CorrelationIdHandler>();
```

## 3.4 Timeout, cancellation et backpressure

- `CancellationToken` partout.
- Timeout côté client **et** côté serveur.
- Protéger les endpoints : limiter payload, limiter concurrence.

## 3.5 Observabilité

- `HttpClient` + OpenTelemetry (traces + metrics)
- Logs structurés : méthode, durée, statut.

---

# 4) API Versioning

## 4.1 Pourquoi versionner ?

- Changements non compatibles : suppression/renommage, modification d’un champ requis, changement sémantique.
- Politique : support N versions, date de dépréciation.

## 4.2 Stratégies

1. **URL versioning** : `/api/v1/customers`
   - Simple, explicite, mais “duplique” des routes.
2. **Header** : `X-API-Version: 1`
   - Propre au niveau URL, moins visible.
3. **Media type** : `Accept: application/vnd.company.resource+json;v=1`
   - Très “REST-like”, plus complexe.

## 4.3 Recommandations

- Préférer **URL** pour API publique simple.
- **Header** ou **media type** si besoin d’URLs stables.
- Version de **contrat** ≠ version d’implémentation.

## 4.4 Gestion de compatibilité

- Tolérer champs inconnus côté client.
- Éviter breaking changes, sinon versionner.
- Déprécation : headers `Deprecation`, `Sunset` (RFC 8594) quand applicable.

---

# 5) Pagination, tri et filtrage

## 5.1 Pourquoi ?

- Performance (éviter réponses massives)
- Maîtrise des coûts (DB, réseau)
- UX

## 5.2 Offset pagination (page/size)

### Requête
`GET /api/orders?page=3&pageSize=50`

### Réponse
Inclure métadonnées :

```json
{
  "items": [ /* ... */ ],
  "page": 3,
  "pageSize": 50,
  "total": 1240
}
```

**Avantages** : simple, compatible SQL (`Skip/Take`).  
**Inconvénients** : instable si insertions/suppressions, coûts élevés avec gros offsets.

## 5.3 Cursor pagination

### Requête
`GET /api/orders?limit=50&cursor=eyJpZCI6...`

### Réponse

```json
{
  "items": [ /* ... */ ],
  "nextCursor": "eyJpZCI6..."
}
```

**Avantages** : stable, performant.  
**Inconvénients** : plus complexe, nécessite un ordre déterministe.

## 5.4 Tri et filtrage

- Tri : `sort=createdAt,-total`
- Filtre : `status=Paid&minTotal=100`
- Recommandation : documenter le langage, valider et whitelister.

## 5.5 HATEOAS (optionnel)

Ajouter des liens : `self`, `next`, `prev`. Utile pour discoverability, pas obligatoire.

---

# 6) Sécurité des Web Services

## 6.1 Menaces courantes (OWASP API Top 10 - synthèse)

- BOLA (Broken Object Level Authorization)
- Auth broken / token leak
- Excessive data exposure
- Rate limiting manquant
- Injection / SSRF

## 6.2 Authentification et autorisation

### JWT Bearer (typique)
- AuthN : vérifier signature, issuer, audience, expiration.
- AuthZ : policies/scopes/claims.

Minimal API :

```csharp
builder.Services.AddAuthentication("Bearer")
    .AddJwtBearer("Bearer", options =>
    {
        options.Authority = "https://identity.example.com";
        options.Audience = "webservices";
    });

builder.Services.AddAuthorization(o =>
{
    o.AddPolicy("orders.read", p => p.RequireClaim("scope", "orders.read"));
});

app.UseAuthentication();
app.UseAuthorization();

app.MapGet("/api/orders/{id:guid}", (Guid id) => Results.Ok(new { id }))
   .RequireAuthorization("orders.read");
```

### OAuth2 / OpenID Connect
- OIDC : identité (login), OAuth2 : délégation d’accès.
- Scopes : permissions.

## 6.3 CORS

- Autoriser seulement les origines nécessaires.
- Ne pas combiner `AllowAnyOrigin()` avec `AllowCredentials()`.

## 6.4 Rate limiting & throttling

Objectifs :
- Réduire abus, DDoS applicatif
- Protéger ressources (DB, downstream)

En ASP.NET Core : `UseRateLimiter` (selon vos besoins) + stratégies (fixed window, token bucket).

## 6.5 Validation d’entrée

- Validation DTO (DataAnnotations / FluentValidation)
- Limiter tailles : body, headers, query.
- Encoder/normaliser.

## 6.6 Secrets et config

- Ne jamais stocker de secrets en clair.
- Utiliser Key Vault / secrets manager.
- Rotation et scopes minimalistes.

---

# 7) gRPC

## 7.1 Quand choisir gRPC ?

- Communication service-to-service (faible latence, contrats stricts)
- Streaming (client/server/bidirectional)
- Interop (C#, Java, Go, etc.)

**Limites** :
- Navigateur : nécessite gRPC-Web
- Debugging plus “binaire” qu’en REST

## 7.2 Protobuf : contrat

Exemple `catalog.proto` :

```proto
syntax = "proto3";

option csharp_namespace = "Catalog";

package catalog;

service CatalogService {
  rpc GetProduct (GetProductRequest) returns (ProductReply);
}

message GetProductRequest {
  string id = 1;
}

message ProductReply {
  string id = 1;
  string name = 2;
}
```

## 7.3 Serveur gRPC (ASP.NET Core)

- Ajouter `Grpc.AspNetCore`
- Mapper service : `app.MapGrpcService<CatalogGrpcService>();`

Implémentation :

```csharp
public sealed class CatalogGrpcService : CatalogService.CatalogServiceBase
{
    public override Task<ProductReply> GetProduct(GetProductRequest request, ServerCallContext context)
    {
        return Task.FromResult(new ProductReply { Id = request.Id, Name = "Keyboard" });
    }
}
```

## 7.4 Client gRPC

```csharp
using var channel = GrpcChannel.ForAddress("https://localhost:5001");
var client = new CatalogService.CatalogServiceClient(channel);
var reply = await client.GetProductAsync(new GetProductRequest { Id = "..." });
```

## 7.5 Streaming

- Server streaming : push d’événements
- Client streaming : upload chunks
- Bidirectional : chat, temps réel

Points d’attention :
- Cancellation
- Backpressure
- Deadlines

## 7.6 Sécurité gRPC

- TLS obligatoire en prod
- JWT bearer côté serveur
- Interceptors pour logs/correlation

---

# 8) GraphQL

## 8.1 Pourquoi GraphQL ?

- Le client choisit les champs : réduit “over/under-fetching”.
- Un point d’entrée unique : `/graphql`.
- Typage fort via schema.

**Risques** :
- Requêtes coûteuses (nécessite limites/complexity)
- Cache plus délicat

## 8.2 Concepts

- **Schema** = Query/Mutation/Subscription
- **Resolver** = fonction qui hydrate un champ
- **N+1** : problème classique → DataLoader/batching

## 8.3 Exemple de schema (conceptuel)

```graphql
type Query {
  customer(id: ID!): Customer
  customers(limit: Int!, cursor: String): CustomerConnection!
}

type Customer {
  id: ID!
  name: String!
  email: String!
  orders: [Order!]!
}
```

## 8.4 Implémentation .NET

Approches courantes :
- Hot Chocolate (très utilisé)
- GraphQL.NET

Principes de mise en œuvre :
- Définir types/queries/mutations
- Brancher DI + DB context
- Mettre en place pagination/filtrage/tri

## 8.5 Pagination GraphQL (Connection)

- Pattern Relay : `edges`, `node`, `pageInfo`.
- Avantage : standard pour cursor pagination.

## 8.6 Sécurité GraphQL

- AuthN/AuthZ par champ/type
- Limites : profondeur, complexité, taille
- Désactiver introspection en prod si nécessaire (selon contexte)

---

# 9) Capstone : checklist d’API prête pour la prod

## 9.1 Checklist design

- Routes cohérentes, noms, statuts corrects
- Erreurs standardisées (ProblemDetails)
- Contrat documenté (OpenAPI) + exemples
- Pagination/tri/filtre documentés
- Versioning + politique de dépréciation

## 9.2 Checklist perf & fiabilité

- Timeouts (client/serveur)
- Resilience sur appels sortants (retry prudent, circuit breaker)
- Cache (HTTP cache/ETag) si pertinent
- Observabilité (traces, logs, metrics)

## 9.3 Checklist sécurité

- AuthN/AuthZ + scopes
- Validation d’entrée
- Rate limiting
- CORS minimal
- Secrets gérés correctement

---

# Labs (guidés)

> Les labs sont proposés pour un déroulé 2 jours. Adaptez selon votre timing.

## Lab 1 — Concevoir une REST API "Customers"

**Objectifs** : routes, statuts, ProblemDetails, OpenAPI.

1. Créer endpoints : Create/Get/List/Update/Patch/Delete.
2. Ajouter validation (email, name non vide).
3. Retourner `201 Created` + `Location`.
4. Centraliser erreurs : exceptions → ProblemDetails.

**Critères de succès** :
- Codes de réponse corrects
- Swagger complet

## Lab 2 — Consommer l’API via HttpClientFactory

1. Créer un client typé `CustomersClient`.
2. Gérer 404 (retour null) et erreurs (throw custom exception).
3. Ajouter `DelegatingHandler` correlation id.
4. (Option) Ajouter stratégie de retry uniquement sur 503/429.

## Lab 3 — Ajouter versioning (v1, v2)

1. Exposer v1 : `name`, `email`.
2. Exposer v2 : `name`, `email`, `phone?`.
3. Documenter les deux versions (OpenAPI séparées).

## Lab 4 — Pagination offset puis cursor

1. Implémenter `GET /api/customers?page&pageSize`.
2. Puis une alternative cursor : `limit&cursor`.
3. Comparer comportement lors d’insertion en milieu de liste.

## Lab 5 — Sécuriser l’API

1. Ajouter JwtBearer.
2. Ajouter policies par scope (`customers.read`, `customers.write`).
3. Ajouter rate limiting.
4. Vérifier CORS.

## Lab 6 — gRPC CatalogService

1. Créer `.proto`.
2. Implémenter unary + server streaming.
3. Créer un client console.

## Lab 7 — GraphQL Gateway

1. Créer `/graphql` avec Query `customer(id)`.
2. Ajouter cursor pagination.
3. Protéger une query par policy.

---

# Annexes

## A) Glossaire rapide

- **BFF** : Backend For Frontend
- **Idempotency key** : clé client pour rendre un POST idempotent
- **HATEOAS** : liens hypermédia dans la réponse
- **BOLA** : Broken Object Level Authorization

## B) Modèle de réponse d’erreur recommandé

- Toujours inclure : `traceId`, `instance`, `status`, `title`.
- Ne pas exposer stack trace en prod.

## C) Ressources

- RFC 7231 (HTTP/1.1 semantics)
- RFC 7807 (Problem Details)
- OWASP API Security Top 10
- gRPC docs + Protobuf language guide
- GraphQL specification
