# Website Development Using MVC Pattern (ASP.NET Core MVC)

> **Public cible** : développeurs .NET/C# (débutant à intermédiaire) souhaitant concevoir des applications web maintenables avec **ASP.NET Core MVC**, intégrant **validation**, **ViewModels**, **Entity Framework Core**, **routing** et **développement d’API**.

---

## 1. Objectifs pédagogiques

À l’issue de cette formation, vous serez capable de :

- Expliquer le **pattern MVC** et ses bénéfices (séparation des responsabilités, testabilité, maintenabilité).
- Implémenter des **Controllers** robustes (actions, binding, filtres, redirections, gestion d’erreurs).
- Concevoir des **Views Razor** (layouts, partials, tag helpers, formulaires).
- Créer et utiliser des **ViewModels** pour éviter l’over-posting et structurer la donnée.
- Mettre en place la **validation** (DataAnnotations, validation serveur/client, validation personnalisée).
- Intégrer une base de données via **EF Core** (DbContext, migrations, requêtes, tracking, transactions).
- Exposer des **API** (routing, conventions REST, DTO, gestion des erreurs, pagination).
- Configurer et maîtriser le **routing** (conventionnel, attribut, contraintes, endpoints).

---

## 2. Prérequis et environnement

### Prérequis

- Connaissances de base en C# (classes, interfaces, async/await, LINQ).
- Notions de HTTP (GET/POST/PUT/DELETE), JSON, formulaires.

### Outils recommandés

- **.NET 9 SDK**
- Visual Studio 2022/2025 ou Rider / VS Code
- SQL Server / PostgreSQL / SQLite (selon votre contexte)
- `dotnet-ef` (outils migrations)

Installation des outils EF Core :

```bash
dotnet tool install --global dotnet-ef
```

---

## 3. Plan de formation (structure)

1. **Introduction au pattern MVC**
2. **Controllers** : actions, binding, DI, résultats, filtres
3. **Views** : Razor, layouts, partial views, tag helpers
4. **ViewModels** : projection, mapping, sécurité, design
5. **Validation** : DataAnnotations, validation personnalisée, UX
6. **EF Core integration** : DbContext, migrations, CRUD, performance
7. **API development** : contrôleurs API, DTO, erreurs, pagination
8. **Routing** : conventionnel, attribut, endpoints, contraintes
9. **Atelier fil rouge** : mini-application « Catalog + Orders » (MVC + EF Core + API)

---

# 4. Contenu détaillé

## Module 1 — Comprendre MVC (Model-View-Controller)

### 1.1 Définitions

- **Model** : données et logique métier (entités, services, règles).
- **View** : rendu UI (HTML) et interactions côté client.
- **Controller** : orchestration (réception requêtes, validation, appel métier, choix de la vue/réponse).

### 1.2 Pourquoi MVC ?

- Séparation des responsabilités
- Lisibilité et maintenabilité
- Testabilité (contrôleurs et services)
- Réutilisabilité : modèles et services indépendants des vues

### 1.3 MVC en ASP.NET Core

Un projet MVC se compose typiquement de :

- `Controllers/`
- `Views/`
- `Models/` (souvent réservé aux ViewModels/DTO ou aux entités selon conventions)
- `Program.cs` : configuration DI + pipeline HTTP

Création d’un projet MVC :

```bash
dotnet new mvc -n MvcMasterclass
cd MvcMasterclass
```

Lancement :

```bash
dotnet run
```

---

## Module 2 — Controllers

### 2.1 Rôle et structure

Un contrôleur :

- reçoit une requête HTTP,
- utilise la DI (services, repositories, DbContext),
- prépare une réponse : `View()`, `RedirectToAction()`, `Json()`, `NotFound()`…

Exemple :

```csharp
using Microsoft.AspNetCore.Mvc;

public class HomeController : Controller
{
    public IActionResult Index() => View();
}
```

### 2.2 Actions et résultats

Types de résultats courants :

- `ViewResult` : rendu Razor
- `RedirectResult` / `RedirectToActionResult`
- `ContentResult`
- `JsonResult`
- `StatusCodeResult` (`NotFound()`, `BadRequest()`, `Unauthorized()`, etc.)

Exemple :

```csharp
public IActionResult Details(int id)
{
    if (id <= 0) return BadRequest();
    // charger des données...
    return View();
}
```

### 2.3 Binding (liaison modèle) et paramètres

ASP.NET Core peut binder :

- route (`/products/5` → `id=5`)
- query string (`?q=abc`)
- form data (POST)
- body JSON (API)

Exemples :

```csharp
public IActionResult Search(string q, int page = 1) { ... }

[HttpPost]
public IActionResult Create(ProductCreateVm vm) { ... }
```

Attributs utiles :

- `[FromRoute]`, `[FromQuery]`, `[FromForm]`, `[FromBody]`

### 2.4 DI dans les contrôleurs

Injection via constructeur :

```csharp
public class ProductsController : Controller
{
    private readonly IProductService _service;

    public ProductsController(IProductService service)
        => _service = service;
}
```

Enregistrement dans `Program.cs` :

```csharp
builder.Services.AddScoped<IProductService, ProductService>();
```

### 2.5 Filtres et gestion des erreurs

Filtres :

- Authorization
- Resource
- Action
- Exception
- Result

Exemple d’attribut :

```csharp
[ResponseCache(Duration = 60)]
public IActionResult CachedView() => View();
```

Gestion d’erreurs (idées) :

- `UseExceptionHandler("/Home/Error")`
- vues d’erreur spécifiques (`404`, `500`)
- pages HTTP status code

---

## Module 3 — Views (Razor)

### 3.1 Principes

Les Views Razor (`.cshtml`) mélangent HTML et C#.

Typiquement :

- `Views/{Controller}/{Action}.cshtml`
- `Views/Shared/` (Layouts, partials)

### 3.2 Layouts

`Views/Shared/_Layout.cshtml` : encapsule l’ossature HTML.

Rendu du contenu :

```cshtml
<body>
    @RenderBody()
</body>
```

Sections :

```cshtml
@RenderSection("Scripts", required: false)
```

### 3.3 Strongly-typed views

Utiliser `@model` :

```cshtml
@model ProductDetailsVm

<h1>@Model.Name</h1>
<p>@Model.Price</p>
```

### 3.4 HTML Helpers et Tag Helpers

Exemples (Tag Helpers) :

```cshtml
<a asp-controller="Products" asp-action="Details" asp-route-id="@Model.Id">Détails</a>

<form asp-action="Create" method="post">
    <input asp-for="Name" />
    <span asp-validation-for="Name"></span>

    <button type="submit">Créer</button>
</form>
```

Activer validation client :

- scripts jQuery validate (si utilisé)
- ou validation côté serveur uniquement (toujours nécessaire)

### 3.5 Partial Views

Réutiliser des fragments :

- `Views/Shared/_ProductCard.cshtml`

```cshtml
@model ProductListItemVm
<div class="card">
  <h3>@Model.Name</h3>
</div>
```

Dans une vue :

```cshtml
<partial name="_ProductCard" model="item" />
```

---

## Module 4 — ViewModels

### 4.1 Pourquoi des ViewModels ?

- Éviter d’exposer directement vos entités EF (couplage)
- Éviter l’**over-posting**
- Adapter la donnée à l’écran (ex : propriétés calculées)
- Simplifier validation et UX

### 4.2 Exemples de ViewModels

#### ViewModel création

```csharp
public class ProductCreateVm
{
    public string Name { get; set; } = "";
    public decimal Price { get; set; }
    public int CategoryId { get; set; }
}
```

#### ViewModel listing

```csharp
public class ProductListItemVm
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public decimal Price { get; set; }
    public string CategoryName { get; set; } = "";
}
```

### 4.3 Mapping (projection)

Projection LINQ (efficace) :

```csharp
var items = await _db.Products
    .Select(p => new ProductListItemVm
    {
        Id = p.Id,
        Name = p.Name,
        Price = p.Price,
        CategoryName = p.Category.Name
    })
    .ToListAsync();
```

Éviter : charger tout en mémoire puis mapper.

### 4.4 Sécurité : over-posting

Mauvais exemple : binder une entité complète en POST.

Bon exemple : binder uniquement `ProductCreateVm`, puis créer l’entité côté serveur.

---

## Module 5 — Validation

### 5.1 Validation côté serveur (indispensable)

DataAnnotations :

```csharp
using System.ComponentModel.DataAnnotations;

public class ProductCreateVm
{
    [Required]
    [StringLength(100, MinimumLength = 3)]
    public string Name { get; set; } = "";

    [Range(0.01, 100000)]
    public decimal Price { get; set; }
}
```

Dans le contrôleur :

```csharp
[HttpPost]
[ValidateAntiForgeryToken]
public async Task<IActionResult> Create(ProductCreateVm vm)
{
    if (!ModelState.IsValid)
        return View(vm);

    // persister...
    return RedirectToAction(nameof(Index));
}
```

### 5.2 Validation personnalisée

#### Option A : attribut personnalisé

```csharp
public class NotInFutureAttribute : ValidationAttribute
{
    public override bool IsValid(object? value)
        => value is not DateTime dt || dt <= DateTime.UtcNow;
}
```

#### Option B : `IValidatableObject`

```csharp
public class OrderCreateVm : IValidatableObject
{
    public DateTime OrderDate { get; set; }

    public IEnumerable<ValidationResult> Validate(ValidationContext validationContext)
    {
        if (OrderDate > DateTime.UtcNow)
            yield return new ValidationResult("OrderDate ne peut pas être dans le futur", new[] { nameof(OrderDate) });
    }
}
```

### 5.3 Messages et UX

Dans la vue :

```cshtml
<div asp-validation-summary="ModelOnly"></div>
<span asp-validation-for="Name"></span>
```

### 5.4 Anti-forgery

Par défaut en MVC, utilisez :

- `<form ...>` avec tag helper (injecte le token)
- `[ValidateAntiForgeryToken]`

---

## Module 6 — Intégration EF Core

### 6.1 Concepts

- `DbContext` : unité de travail
- `DbSet<TEntity>` : table logique
- migrations : versionnement schéma

### 6.2 Modèle de données (exemple fil rouge)

```csharp
public class Category
{
    public int Id { get; set; }
    public string Name { get; set; } = "";

    public List<Product> Products { get; set; } = new();
}

public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public decimal Price { get; set; }

    public int CategoryId { get; set; }
    public Category Category { get; set; } = null!;
}
```

### 6.3 DbContext

```csharp
using Microsoft.EntityFrameworkCore;

public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }

    public DbSet<Product> Products => Set<Product>();
    public DbSet<Category> Categories => Set<Category>();
}
```

### 6.4 Configuration dans `Program.cs`

SQL Server :

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("Default")));
```

SQLite (simple pour démo) :

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlite(builder.Configuration.GetConnectionString("Default")));
```

`appsettings.json` :

```json
{
  "ConnectionStrings": {
    "Default": "Data Source=app.db"
  }
}
```

### 6.5 Migrations

```bash
dotnet ef migrations add InitialCreate
dotnet ef database update
```

### 6.6 CRUD dans un contrôleur MVC

Exemple `Index` (listing) :

```csharp
public class ProductsController : Controller
{
    private readonly AppDbContext _db;

    public ProductsController(AppDbContext db) => _db = db;

    public async Task<IActionResult> Index()
    {
        var items = await _db.Products
            .AsNoTracking()
            .OrderBy(p => p.Name)
            .Select(p => new ProductListItemVm
            {
                Id = p.Id,
                Name = p.Name,
                Price = p.Price,
                CategoryName = p.Category.Name
            })
            .ToListAsync();

        return View(items);
    }
}
```

Exemple création :

```csharp
[HttpGet]
public async Task<IActionResult> Create()
{
    // charger dropdown categories etc.
    return View(new ProductCreateVm());
}

[HttpPost]
[ValidateAntiForgeryToken]
public async Task<IActionResult> Create(ProductCreateVm vm)
{
    if (!ModelState.IsValid)
        return View(vm);

    var product = new Product
    {
        Name = vm.Name,
        Price = vm.Price,
        CategoryId = vm.CategoryId
    };

    _db.Products.Add(product);
    await _db.SaveChangesAsync();

    return RedirectToAction(nameof(Index));
}
```

### 6.7 Performance et bonnes pratiques

- Utiliser **projection** (`Select`) pour limiter les colonnes.
- Utiliser `AsNoTracking()` sur les requêtes en lecture.
- Éviter le `N+1` : inclure ou projeter (`p.Category.Name`).
- Gérer la concurrence optimiste (`RowVersion`) si nécessaire.

---

## Module 7 — API development (dans une application MVC)

### 7.1 Pourquoi ajouter une API ?

- UI moderne (SPA/HTMX/AJAX)
- intégration tiers (mobile, partenaires)
- séparation UI / backend

### 7.2 Ajouter les contrôleurs API

Dans `Program.cs` :

```csharp
builder.Services.AddControllersWithViews();
```

Vous pouvez combiner MVC + API dans la même app.

Créer un contrôleur API :

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;

[ApiController]
[Route("api/products")]
public class ProductsApiController : ControllerBase
{
    private readonly AppDbContext _db;

    public ProductsApiController(AppDbContext db) => _db = db;

    [HttpGet]
    public async Task<ActionResult<List<ProductListItemVm>>> GetAll([FromQuery] int page = 1, [FromQuery] int pageSize = 20)
    {
        page = Math.Max(page, 1);
        pageSize = Math.Clamp(pageSize, 1, 200);

        var query = _db.Products.AsNoTracking().OrderBy(p => p.Id);

        var items = await query
            .Skip((page - 1) * pageSize)
            .Take(pageSize)
            .Select(p => new ProductListItemVm
            {
                Id = p.Id,
                Name = p.Name,
                Price = p.Price,
                CategoryName = p.Category.Name
            })
            .ToListAsync();

        return Ok(items);
    }
}
```

### 7.3 DTO vs ViewModel

API : préférez **DTO** dédiés (stables, versionnables).

Exemple :

```csharp
public record ProductDto(int Id, string Name, decimal Price);
```

### 7.4 Gestion des erreurs et codes HTTP

- `400 BadRequest` : validation
- `404 NotFound` : ressource inexistante
- `201 Created` : création
- `204 NoContent` : suppression

Exemple POST :

```csharp
public record CreateProductDto(string Name, decimal Price, int CategoryId);

[HttpPost]
public async Task<IActionResult> Create(CreateProductDto dto)
{
    if (string.IsNullOrWhiteSpace(dto.Name))
        return ValidationProblem("Name est requis");

    var entity = new Product
    {
        Name = dto.Name,
        Price = dto.Price,
        CategoryId = dto.CategoryId
    };

    _db.Products.Add(entity);
    await _db.SaveChangesAsync();

    return CreatedAtAction(nameof(GetById), new { id = entity.Id }, new ProductDto(entity.Id, entity.Name, entity.Price));
}

[HttpGet("{id:int}")]
public async Task<ActionResult<ProductDto>> GetById(int id)
{
    var p = await _db.Products.AsNoTracking().FirstOrDefaultAsync(x => x.Id == id);
    if (p is null) return NotFound();

    return Ok(new ProductDto(p.Id, p.Name, p.Price));
}
```

### 7.5 Versioning (piste)

- Versioning par URL (`/api/v1/products`)
- Versioning par header
- Conserver DTO par version

---

## Module 8 — Routing

### 8.1 Routing MVC (conventionnel)

Dans `Program.cs` :

```csharp
app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Home}/{action=Index}/{id?}");
```

- Route par défaut
- `{id?}` optionnel

### 8.2 Attribute routing

Plus précis et recommandé pour API.

```csharp
[Route("products")]
public class ProductsController : Controller
{
    [HttpGet("")]
    public IActionResult Index() => View();

    [HttpGet("{id:int}")]
    public IActionResult Details(int id) => View();
}
```

### 8.3 Contraintes de route

- `{id:int}`
- `{slug:regex(...)}`
- contraintes personnalisées

### 8.4 Génération d’URL

Dans Razor :

```cshtml
<a asp-action="Details" asp-route-id="@item.Id">Voir</a>
```

Dans contrôleur :

```csharp
return RedirectToAction(nameof(Details), new { id = product.Id });
```

### 8.5 Endpoints

Dans ASP.NET Core moderne, le routage est basé sur `Map...` :

- `app.MapControllerRoute(...)`
- `app.MapControllers()`

Combiner :

```csharp
app.MapControllers();
app.MapControllerRoute(name: "default", pattern: "{controller=Home}/{action=Index}/{id?}");
```

> Attention : l’ordre peut compter selon vos conventions/app.

---

# 5. Atelier fil rouge (MVC + EF Core + API)

## 5.1 Objectif

Construire une mini-application :

- Catalogue de produits (MVC)
- Création/édition via formulaires (validation)
- Stockage en base avec EF Core
- Exposition d’une API pour lister et créer des produits

## 5.2 Étapes guidées

1. Créer projet MVC et ajouter EF Core provider (SQLite ou SQL Server)
2. Créer entités `Product`, `Category`
3. Créer `AppDbContext`, configurer la DI + connection string
4. Lancer migrations
5. Créer `ProductsController` MVC avec actions :
   - `Index` (GET)
   - `Create` (GET/POST)
   - `Edit` (GET/POST)
   - `Details` (GET)
6. Créer les ViewModels :
   - `ProductListItemVm`, `ProductCreateVm`, `ProductEditVm`, `ProductDetailsVm`
7. Ajouter validation DataAnnotations + validation personnalisée
8. Créer `ProductsApiController` :
   - `GET /api/products`
   - `GET /api/products/{id}`
   - `POST /api/products`
9. Ajouter routing attribut + tester via `curl`/Postman

---

# 6. Bonnes pratiques et checklists

## 6.1 Checklists MVC

- [ ] Les actions GET ne modifient pas la donnée
- [ ] Les actions POST vérifient `ModelState.IsValid`
- [ ] Utiliser `[ValidateAntiForgeryToken]` sur les POST
- [ ] Utiliser des ViewModels dédiés (pas d’entités EF en input)
- [ ] Utiliser PRG pattern (Post/Redirect/Get) après succès

## 6.2 Checklists EF Core

- [ ] `AsNoTracking()` sur les lectures
- [ ] Projection `Select` pour les listes
- [ ] Migrations versionnées
- [ ] Relations configurées (FK et navigation)

## 6.3 Checklists API

- [ ] DTO dédiés
- [ ] Codes HTTP corrects
- [ ] Validation + messages d’erreurs cohérents
- [ ] Pagination sur les listes

---

# 7. Exercices (avec attendus)

## Exercice 1 — Formulaire de création produit

**But** : page MVC `Products/Create`.

- Champs : `Name`, `Price`, `CategoryId`
- Validations :
  - Name requis, 3-100
  - Price > 0

**Attendu** :
- En cas d’erreur, réafficher le formulaire avec messages.
- En cas de succès, redirection vers `Index`.

## Exercice 2 — Liste paginée côté API

**But** : `GET /api/products?page=1&pageSize=20`

**Attendu** :
- page >= 1
- pageSize clamp 1..200
- réponse `200` avec liste

## Exercice 3 — Routing et contraintes

**But** : créer une route `GET /products/{id:int}`.

**Attendu** :
- si `id` non int, pas de match
- sinon, action `Details`

---

# 8. Annexes

## 8.1 Exemple minimal `Program.cs`

```csharp
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllersWithViews();

builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlite(builder.Configuration.GetConnectionString("Default")));

var app = builder.Build();

if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/Home/Error");
    app.UseHsts();
}

app.UseHttpsRedirection();
app.UseStaticFiles();

app.UseRouting();

app.UseAuthorization();

app.MapControllers();
app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Home}/{action=Index}/{id?}");

app.Run();
```

## 8.2 Ressources

- Documentation ASP.NET Core MVC : https://learn.microsoft.com/aspnet/core/mvc/
- EF Core : https://learn.microsoft.com/ef/core/
- Routing : https://learn.microsoft.com/aspnet/core/fundamentals/routing

---

**Fin de la formation — Website Development Using MVC Pattern**
