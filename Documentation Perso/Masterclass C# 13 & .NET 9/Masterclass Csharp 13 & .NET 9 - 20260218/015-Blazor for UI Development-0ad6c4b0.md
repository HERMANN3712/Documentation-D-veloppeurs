# Masterclass — Blazor for UI Development (.NET 9)

> Public cible : développeurs .NET/C# (confirmés) souhaitant maîtriser Blazor pour construire des interfaces web modernes en .NET 9.
>
> Format : cours + démos + ateliers.

---

## 0. Objectifs pédagogiques

À l’issue de cette formation, vous saurez :

- Expliquer clairement **Blazor Server vs Blazor WebAssembly** (architecture, perf, sécurité, coûts, contraintes).
- Construire une application à base de **composants** (paramètres, événements, cycle de vie, rendu).
- Mettre en place le **routing** (pages, paramètres de route, navigation, redirections, layouts).
- Concevoir des **formulaires** robustes : validation, `EditForm`, DataAnnotations, validation custom.
- Intégrer une couche données avec **EF Core** (DbContext, migrations, services, patterns de repo/service, async, tracking).
- Comprendre et utiliser les **render modes** en **.NET 9** (rendu serveur, WebAssembly, auto, rendu statique) et leurs implications.

---

## 1. Prérequis & setup

### 1.1 Prérequis

- C# (POO, async/await)
- Connaissance basique du web (HTTP, HTML/CSS)
- Notions de DI dans ASP.NET Core

### 1.2 Outils

- .NET 9 SDK
- Visual Studio (2022+) ou Rider/VS Code
- Node/non requis (Blazor n’en dépend pas)
- SQL Server / SQLite (SQLite recommandé pour les ateliers)

### 1.3 Création du projet

> Dans .NET 9, Blazor est pleinement intégré à ASP.NET Core.

Commande type (à adapter selon vos templates/install) :

```bash
# exemple générique
mkdir BlazorMasterclass
cd BlazorMasterclass

dotnet new blazor -n BlazorMasterclass
cd BlazorMasterclass

dotnet run
```

Structure typique :

- `Components/` : composants Razor
- `Pages/` : pages routables
- `wwwroot/` : assets statiques
- `Program.cs` : pipeline + DI + modes de rendu

---

## 2. Blazor : panorama & principes

### 2.1 Qu’est-ce que Blazor ?

Blazor est une stack UI où l’UI est composée de **composants Razor** (C# + markup). Le runtime gère :

- la **reconciliation** (diff) du DOM via un rendu component-based
- l’**interop JS** lorsque nécessaire
- le **binding** et l’eventing (événements UI -> C#)

### 2.2 Modèle de composant

Un composant Blazor est :

- un fichier `.razor`
- un arbre de rendu (render tree)
- un état local + paramètres + événements

Exemple minimal :

```razor
<h3>Hello @Name</h3>

@code {
    [Parameter] public string Name { get; set; } = "World";
}
```

---

## 3. Blazor Server vs WebAssembly

### 3.1 Blazor Server

**Principe** :

- Le rendu des composants s’exécute **sur le serveur**.
- Le navigateur maintient une connexion (SignalR) pour transporter les événements et les mises à jour UI (diffs).

**Avantages** :

- Temps de démarrage souvent excellent (pas de gros download initial)
- Accès direct aux ressources serveur (DbContext, secrets, réseau interne)
- Contrôle centralisé des versions (déploiement simple)

**Inconvénients** :

- Dépendance réseau (latence, stabilité)
- Coût serveur lié aux connexions (mémoire par circuit, concurrence)
- Attention à la scalabilité (sticky sessions, backplanes, etc.)

### 3.2 Blazor WebAssembly (WASM)

**Principe** :

- Le code .NET (assemblies) s’exécute **dans le navigateur** via WebAssembly.
- L’UI et la logique s’exécutent côté client.

**Avantages** :

- Interactions UI fluides même en latence élevée
- Scalabilité serveur meilleure (le client exécute du code)
- Mode offline possible (PWA)

**Inconvénients** :

- Download initial (temps de démarrage)
- Accès aux ressources serveur via API HTTP (pas de DbContext direct)
- Exposition du code client (attention au secret management)

### 3.3 Critères de choix

| Critère | Blazor Server | Blazor WASM |
|---|---|---|
| Démarrage | très bon | variable (download) |
| Sensibilité latence | forte | faible |
| Accès DB direct | oui | non |
| Scalabilité | à travailler | plus simple |
| Offline | non | possible |

### 3.4 Stratégie moderne : rendu hybride

Avec les évolutions récentes (ASP.NET Core), on peut **combiner** des modes de rendu selon les pages/composants (voir section Render Modes).

---

## 4. Components (composants Blazor)

### 4.1 Composition & réutilisation

Un composant peut importer et imbriquer d’autres composants.

```razor
<OrderSummary OrderId="42" />
```

### 4.2 Paramètres (`[Parameter]`) et paramètres en cascade

#### Paramètre standard

```razor
<ProductCard Product="product" OnAdd="AddToCart" />

@code {
    Product product = new("Keyboard", 99.90m);

    void AddToCart(Product p) { /* ... */ }
}
```

Dans `ProductCard.razor` :

```razor
<div class="card">
  <h4>@Product.Name</h4>
  <p>@Product.Price €</p>
  <button class="btn btn-primary" @onclick="() => OnAdd.InvokeAsync(Product)">
    Add
  </button>
</div>

@code {
    [Parameter, EditorRequired] public Product Product { get; set; } = default!;
    [Parameter] public EventCallback<Product> OnAdd { get; set; }
}
```

#### Cascading values

Utile pour distribuer un contexte (thème, culture, utilisateur, panier). Exemple conceptuel :

```razor
<CascadingValue Value="cart">
  <CartBadge />
  <CartDetails />
</CascadingValue>

@code {
    private Cart cart = new();
}
```

### 4.3 `EventCallback` vs `Action`

- `EventCallback` s’intègre au pipeline de rendu et supporte async.
- Utiliser `EventCallback<T>` dès que l’événement est déclenché depuis l’UI.

### 4.4 Cycle de vie

Méthodes clés :

- `OnInitialized` / `OnInitializedAsync`
- `OnParametersSet` / `OnParametersSetAsync`
- `ShouldRender`
- `OnAfterRender` / `OnAfterRenderAsync`
- `Dispose` / `IAsyncDisposable`

Exemple :

```razor
@code {
    protected override async Task OnInitializedAsync()
    {
        // Chargement initial
        await LoadAsync();
    }

    protected override bool ShouldRender()
        => true; // à optimiser si besoin
}
```

### 4.5 State management : règles et pratiques

- L’état doit être **local** si possible (simplicité)
- Pour l’état partagé :
  - service DI (singleton/scoped selon hosting)
  - cascading parameter
  - patterns inspirés Flux/Redux si besoin
- Éviter de déclencher des rendus en boucle (ex: `OnAfterRender`)

### 4.6 `RenderFragment` et slots

Permet l’injection de contenu.

```razor
<Modal Title="Confirm">
  <p>Voulez-vous continuer ?</p>
</Modal>
```

Dans `Modal.razor` :

```razor
<div class="modal">
  <h3>@Title</h3>
  <div class="body">
    @ChildContent
  </div>
</div>

@code {
  [Parameter] public string Title { get; set; } = "";
  [Parameter] public RenderFragment? ChildContent { get; set; }
}
```

---

## 5. Routing & navigation

### 5.1 Pages et attribut `@page`

Créer une page routable :

```razor
@page "/products"

<h3>Products</h3>
```

### 5.2 Paramètres de route

```razor
@page "/products/{id:int}"

<h3>Product @Id</h3>

@code {
    [Parameter] public int Id { get; set; }
}
```

### 5.3 Navigation avec `NavigationManager`

```razor
@inject NavigationManager Nav

<button @onclick="Go">Go to cart</button>

@code {
    void Go() => Nav.NavigateTo("/cart");
}
```

### 5.4 Layouts

Définir un layout :

```razor
@inherits LayoutComponentBase

<div class="app">
  <nav>...</nav>
  <main>@Body</main>
</div>
```

Appliquer à une page :

```razor
@layout MainLayout
@page "/"
```

### 5.5 Route constraints & bonnes pratiques

- Contraintes (`int`, `guid`) pour éviter des routes ambiguës
- Garder les routes stables (SEO, partage de liens)
- `NavLink` pour la navigation avec état actif

---

## 6. Forms : binding, validation, UX

### 6.1 `EditForm` et `EditContext`

Formulaire typique :

```razor
@page "/products/new"

<EditForm Model="model" OnValidSubmit="SaveAsync">
    <DataAnnotationsValidator />
    <ValidationSummary />

    <div class="mb-3">
        <label>Name</label>
        <InputText class="form-control" @bind-Value="model.Name" />
        <ValidationMessage For="() => model.Name" />
    </div>

    <div class="mb-3">
        <label>Price</label>
        <InputNumber class="form-control" @bind-Value="model.Price" />
        <ValidationMessage For="() => model.Price" />
    </div>

    <button class="btn btn-success" type="submit">Save</button>
</EditForm>

@code {
    private NewProductModel model = new();

    private Task SaveAsync()
    {
        // persist
        return Task.CompletedTask;
    }

    public sealed class NewProductModel
    {
        [Required, MinLength(2)]
        public string Name { get; set; } = "";

        [Range(0.01, 100000)]
        public decimal Price { get; set; }
    }
}
```

### 6.2 Validation custom

Approches :

- Attr. DataAnnotations custom
- Implémentation `IValidatableObject`
- Validation via `EditContext` + `ValidationMessageStore`

Exemple (conceptuel) avec `IValidatableObject` :

```csharp
public sealed class NewProductModel : IValidatableObject
{
    public string Name { get; set; } = "";
    public decimal Price { get; set; }

    public IEnumerable<ValidationResult> Validate(ValidationContext validationContext)
    {
        if (Name.Contains("test", StringComparison.OrdinalIgnoreCase))
            yield return new ValidationResult("Name cannot contain 'test'", new[] { nameof(Name) });
    }
}
```

### 6.3 Gestion UX

- Désactiver le bouton pendant la soumission (flag `isSaving`)
- Gérer les erreurs (try/catch + message)
- Préférer async, éviter le blocage UI

---

## 7. EF Core integration

> Objectif : brancher une persistance solide et idiomatique.

### 7.1 Modèle de données

```csharp
public sealed class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public decimal Price { get; set; }
    public DateTime CreatedAtUtc { get; set; } = DateTime.UtcNow;
}
```

### 7.2 DbContext

```csharp
public sealed class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }

    public DbSet<Product> Products => Set<Product>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Product>(b =>
        {
            b.Property(x => x.Name).HasMaxLength(200).IsRequired();
            b.HasIndex(x => x.Name);
        });
    }
}
```

### 7.3 Enregistrement DI

#### Dans un hosting Server (rendu serveur)

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlite(builder.Configuration.GetConnectionString("Default")));
```

- En Server, la durée de vie du `DbContext` est généralement **scoped** (par requête/circuit selon la config).

#### En WASM

- Le `DbContext` ne tourne généralement pas côté client.
- On consomme une API (Minimal API / Controllers) et on mappe des DTO.

### 7.4 Migrations

```bash
dotnet tool install --global dotnet-ef

dotnet ef migrations add InitialCreate
dotnet ef database update
```

### 7.5 Pattern service (recommandé)

Encapsuler EF Core derrière un service.

```csharp
public sealed class ProductService
{
    private readonly AppDbContext _db;

    public ProductService(AppDbContext db) => _db = db;

    public Task<List<Product>> GetAllAsync(CancellationToken ct)
        => _db.Products.AsNoTracking().OrderBy(x => x.Name).ToListAsync(ct);

    public async Task<Product> CreateAsync(Product product, CancellationToken ct)
    {
        _db.Products.Add(product);
        await _db.SaveChangesAsync(ct);
        return product;
    }
}
```

Enregistrement :

```csharp
builder.Services.AddScoped<ProductService>();
```

### 7.6 Affichage d’une liste (async)

```razor
@page "/products"
@inject ProductService Products

<h3>Products</h3>

@if (items is null)
{
    <p>Loading…</p>
}
else
{
    <ul>
        @foreach (var p in items)
        {
            <li>@p.Name — @p.Price €</li>
        }
    </ul>
}

@code {
    private List<Product>? items;

    protected override async Task OnInitializedAsync()
    {
        items = await Products.GetAllAsync(CancellationToken.None);
    }
}
```

### 7.7 Points d’attention (EF Core + Blazor)

- **AsNoTracking** pour les listes (perf)
- Concurrence : gérer les exceptions (`DbUpdateConcurrencyException`)
- Longs traitements : ne pas bloquer le thread UI
- Transactions : service dédié si besoin
- Validation : côté UI + côté DB

---

## 8. Render modes en .NET 9 (concepts et implications)

> Les render modes déterminent *où* et *comment* le composant est rendu : pré-rendu, interactif, serveur, WASM…

### 8.1 Pourquoi les render modes ?

Ils permettent d’optimiser :

- **Temps de chargement** : rendu statique initial puis activation interactive
- **Coût serveur** : limiter les circuits temps-réel
- **Expérience utilisateur** : interactivité localisée (ex : uniquement une zone)

### 8.2 Les modes courants (terminologie)

Selon les templates ASP.NET Core récents :

- **Static/SSR** (rendu statique côté serveur) : HTML généré, pas d’interactivité Blazor.
- **Interactive Server** : interactivité via SignalR, logique côté serveur.
- **Interactive WebAssembly** : interactivité côté navigateur.
- **Interactive Auto** : stratégie hybride (souvent SSR initial + bascule).

> Remarque : la terminologie exacte (noms des APIs/attributs) varie selon version/template. L’essentiel est de comprendre les conséquences en termes de dépendances, de latence, et de data access.

### 8.3 Choisir un render mode par scénario

- Page marketing / contenu : **SSR statique** (SEO, perf)
- Back-office intranet : **Interactive Server** (accès direct aux ressources)
- App grand public : **Interactive WASM** (scalabilité)
- App mixte : **Auto** (meilleur initial load + interactivité ensuite)

### 8.4 Impacts architecturaux

- **Injection de services** :
  - server : accès direct aux services backend
  - wasm : services HTTP/clients
- **Authentification** :
  - server : cookies/identity + accès direct
  - wasm : tokens, API, CORS
- **State** :
  - server : circuit (mémoire serveur)
  - wasm : stockage client (local storage) + cache + API

### 8.5 Exemple de stratégie d’application

- Layout + navigation rendus statiquement
- Sections interactives isolées en composants
- Pages “data-heavy” en Server (si intranet) ou WASM (si public)

---

## 9. Atelier fil rouge (proposition)

### 9.1 Objectif

Construire une mini application de catalogue produits avec :

- Liste + détails
- Création (formulaire)
- Persistance EF Core
- Au moins un composant réutilisable (card, modal, pager)
- Discussion render modes (séparer parties interactives)

### 9.2 Étapes

1. **Scaffold UI** : pages `/products`, `/products/{id}`
2. **Composants** : `ProductCard`, `ProductForm`
3. **Routing/layout** : menu + NavLink
4. **Forms** : validation + messages
5. **EF Core** : SQLite + migrations
6. **Optimisations** : AsNoTracking, cancellation token
7. **Rendu** : choisir un mode par page/composant (discussion guidée)

### 9.3 Critères de réussite

- Code lisible et testable (services, séparation UI/données)
- Routes robustes (paramètres, contraintes)
- Formulaires avec validation correcte
- Accès aux données async

---

## 10. Annexes

### 10.1 Checklist performance

- Minimiser les re-renders (state minimal, `ShouldRender` au besoin)
- Charger les données async et afficher des placeholders
- Éviter `DbContext` long-lived
- Utiliser `AsNoTracking` pour lecture
- Sur Server : surveiller la charge SignalR, latence, taille des diffs

### 10.2 Checklist architecture

- UI : composants petits, composables
- Services : logique métier hors UI
- DTOs : nécessaires pour WASM via API
- Validation : UI + back-end

### 10.3 Références

- Documentation officielle Blazor (Microsoft)
- EF Core docs (migrations, tracking, perf)
- Patterns ASP.NET Core (DI, configuration, minimal APIs)
