# Building Powerful Websites with Razor (Razor Pages)

> Formation .NET / C# — focus Razor Pages : **lifecycle**, **forms**, **validation**, **layouts**, **partials**, **CRUD**.

---

## 0. Public, prérequis et objectifs

### Public cible
- Développeurs .NET / C# souhaitant créer des applications Web rapidement et proprement avec **Razor Pages**.
- Formateurs/tech leads souhaitant disposer d’un support complet (cours + ateliers + corrigés guidés).

### Prérequis
- C# (POO, LINQ, async/await) et notions de Web (HTTP, HTML).
- .NET SDK installé (idéalement .NET 9) + IDE (Visual Studio 2022+ / Rider / VS Code).
- Connaissance basique d’Entity Framework Core recommandée (sinon section EF guidée incluse).

### Objectifs pédagogiques
À la fin, vous saurez :
- Expliquer le **cycle de vie** Razor Pages (routing → handler → binding → validation → rendering).
- Construire des **formulaires** robustes (Tag Helpers, binding, antiforgery).
- Mettre en place la **validation** client/serveur et des messages UX cohérents.
- Structurer une UI avec **layouts**, **sections**, **partials** et **View Components**.
- Implémenter un **CRUD complet** avec EF Core, gestion d’erreurs, concurrence, validation et UX.
- Produire un code maintenable : séparation, conventions, tests légers, bonnes pratiques.

---

## 1. Plan de la formation (macro)

1. Introduction à Razor Pages et structure d’un projet
2. Razor Pages **Lifecycle** (pipeline, routing, handlers, binding, validation)
3. Formulaires : Tag Helpers, binding avancé, anti-forgery, uploads
4. Validation : DataAnnotations, validation personnalisée, validation client
5. Layouts, sections, partials, view components et organisation UI
6. CRUD : EF Core, Pages Index/Details/Create/Edit/Delete, filtres, tri, pagination
7. Expérience utilisateur : messages, erreurs, PRG pattern, accessibilité
8. Sécurité et robustesse : validation côté serveur, anti-forgery, authorization
9. Atelier final : mini-projet “Catalog”
10. Annexes : snippets, checklists, pièges fréquents

> Durée suggérée : **1 à 2 jours** (selon profondeur des ateliers)

---

## 2. Setup rapide (projet fil rouge)

### 2.1 Création du projet

```bash
mkdir RazorMasterclass
cd RazorMasterclass

dotnet new webapp -n Catalog.Web
cd Catalog.Web
```

Structure typique :
- `Pages/` : pages Razor (`.cshtml`) et modèles (`.cshtml.cs`)
- `wwwroot/` : assets statiques
- `Program.cs` : composition (services + middleware)

### 2.2 Ajout d’Entity Framework Core (fil rouge CRUD)

```bash
dotnet add package Microsoft.EntityFrameworkCore.Sqlite
dotnet add package Microsoft.EntityFrameworkCore.Design
```

Créez un dossier `Data/` et `Models/`.

**Modèle** `Models/Product.cs` :

```csharp
using System.ComponentModel.DataAnnotations;

namespace Catalog.Web.Models;

public class Product
{
    public int Id { get; set; }

    [Required, StringLength(80)]
    public string Name { get; set; } = string.Empty;

    [StringLength(400)]
    public string? Description { get; set; }

    [Range(0, 1_000_000)]
    public decimal Price { get; set; }

    public bool IsActive { get; set; } = true;

    // Concurrency token (optionnel mais très utile dans un CRUD)
    [Timestamp]
    public byte[]? RowVersion { get; set; }
}
```

**DbContext** `Data/AppDbContext.cs` :

```csharp
using Catalog.Web.Models;
using Microsoft.EntityFrameworkCore;

namespace Catalog.Web.Data;

public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }

    public DbSet<Product> Products => Set<Product>();
}
```

**Program.cs** (extrait) :

```csharp
using Catalog.Web.Data;
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddRazorPages();

builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlite(builder.Configuration.GetConnectionString("Default")
        ?? "Data Source=catalog.db"));

var app = builder.Build();

if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/Error");
    app.UseHsts();
}

app.UseHttpsRedirection();
app.UseStaticFiles();

app.UseRouting();

// app.UseAuthentication();
// app.UseAuthorization();

app.MapRazorPages();

app.Run();
```

Migrations :

```bash
dotnet ef migrations add Initial

dotnet ef database update
```

---

## 3. Razor Pages : concepts essentiels

Razor Pages est une approche orientée **pages** (plutôt que contrôleurs MVC) :
- Une page = un couple :
  - `Pages/Products/Index.cshtml`
  - `Pages/Products/Index.cshtml.cs` (*PageModel*)
- Le *PageModel* expose :
  - des propriétés (modèle pour la vue)
  - des handlers (OnGet/OnPost/OnPut/…)
  - de la validation (ModelState)

### 3.1 Conventions de routing
- `Pages/Index.cshtml` → `/`
- `Pages/Products/Index.cshtml` → `/Products`
- `Pages/Products/Details.cshtml` → `/Products/Details`
- Paramètres : `Pages/Products/Details.cshtml` avec `@page "{id:int}"` → `/Products/Details/12`

Exemple page :

```cshtml
@page "{id:int}"
@model Catalog.Web.Pages.Products.DetailsModel

<h1>Détails</h1>
<p>Id: @Model.Product.Id</p>
```

---

## 4. Razor Pages Lifecycle (pipeline, handlers, binding)

### 4.1 La vision “de bout en bout”

Quand une requête arrive :
1. **Routing** sélectionne une page Razor (`MapRazorPages`)
2. Création du **PageModel** + injection de dépendances
3. Exécution des **filters** (si présents)
4. **Model Binding** : hydratation des paramètres et propriétés (GET/POST)
5. **Validation** : DataAnnotations + validations custom → `ModelState`
6. Sélection d’un **handler** (`OnGet`, `OnPost`, `OnPostDelete`, …)
7. Production d’un résultat : `Page()`, `RedirectToPage()`, `NotFound()`, etc.
8. **Rendering** du `.cshtml` avec le modèle

### 4.2 Handlers : conventions
Handlers typiques :
- `OnGet()` : afficher page
- `OnPost()` : traiter un formulaire
- `OnPostCreateAsync()` , `OnPostDeleteAsync()` : handlers “nommés”

Exemple :

```csharp
public class IndexModel : PageModel
{
    public void OnGet() { }

    public IActionResult OnPost() => Page();

    public IActionResult OnPostDelete(int id)
    {
        // ...
        return RedirectToPage();
    }
}
```

Dans la vue, on cible un handler nommé via `asp-page-handler` :

```cshtml
<form method="post" asp-page-handler="Delete">
    <input type="hidden" name="id" value="@p.Id" />
    <button class="btn btn-danger">Supprimer</button>
</form>
```

### 4.3 Model Binding : bonnes pratiques

#### 4.3.1 `[BindProperty]`
Pour binder une propriété du PageModel lors d’un POST :

```csharp
[BindProperty]
public ProductInput Input { get; set; } = new();
```

#### 4.3.2 Préférer un modèle d’entrée (DTO)
Cela limite l’**overposting** (soumission d’un champ non attendu).

```csharp
public class ProductInput
{
    [Required, StringLength(80)]
    public string Name { get; set; } = string.Empty;

    [Range(0, 1_000_000)]
    public decimal Price { get; set; }

    public bool IsActive { get; set; }
}
```

> Ne liez pas directement une entité EF complète si vous n’êtes pas certain des champs exposés.

#### 4.3.3 Binding sur paramètres
Exemple :

```csharp
public async Task<IActionResult> OnGetAsync(int id)
{
    // id vient de la route ou de la query string
    return Page();
}
```

### 4.4 Pattern PRG (Post-Redirect-Get)
Après un POST réussi, **rediriger** (évite resoumission).

```csharp
if (!ModelState.IsValid)
    return Page();

// save...
return RedirectToPage("./Index");
```

---

## 5. Formulaires (Tag Helpers, binding, antiforgery)

### 5.1 Tag Helpers indispensables
- `asp-page`, `asp-page-handler`
- `asp-for` : bind sur propriété
- `asp-validation-for`, `asp-validation-summary`
- `asp-route-*` : paramètres d’URL

Exemple formulaire :

```cshtml
@page
@model Catalog.Web.Pages.Products.CreateModel

<h1>Créer un produit</h1>

<form method="post" class="mt-3">
    <div class="mb-3">
        <label asp-for="Input.Name" class="form-label"></label>
        <input asp-for="Input.Name" class="form-control" />
        <span asp-validation-for="Input.Name" class="text-danger"></span>
    </div>

    <div class="mb-3">
        <label asp-for="Input.Price" class="form-label"></label>
        <input asp-for="Input.Price" class="form-control" />
        <span asp-validation-for="Input.Price" class="text-danger"></span>
    </div>

    <div class="form-check mb-3">
        <input asp-for="Input.IsActive" class="form-check-input" />
        <label asp-for="Input.IsActive" class="form-check-label"></label>
    </div>

    <div asp-validation-summary="ModelOnly" class="text-danger"></div>

    <button class="btn btn-primary" type="submit">Créer</button>
    <a class="btn btn-secondary" asp-page="./Index">Annuler</a>
</form>

@section Scripts {
    <partial name="_ValidationScriptsPartial" />
}
```

> Le token anti-forgery est ajouté automatiquement sur les formulaires Razor Pages (sauf désactivation explicite).

### 5.2 Multiples actions sur la même page

Approche : plusieurs boutons, un handler par action.

```cshtml
<form method="post">
    <button class="btn btn-primary" asp-page-handler="Save">Enregistrer</button>
    <button class="btn btn-warning" asp-page-handler="SaveAndClose">Enregistrer et fermer</button>
</form>
```

```csharp
public IActionResult OnPostSave() => Page();
public IActionResult OnPostSaveAndClose() => RedirectToPage("./Index");
```

### 5.3 Upload de fichiers (option utile)

```cshtml
<form method="post" enctype="multipart/form-data">
    <input type="file" name="upload" />
    <button type="submit">Envoyer</button>
</form>
```

```csharp
public async Task<IActionResult> OnPostAsync(IFormFile upload)
{
    if (upload is null || upload.Length == 0)
    {
        ModelState.AddModelError(string.Empty, "Fichier manquant");
        return Page();
    }

    using var stream = System.IO.File.Create(Path.Combine("uploads", upload.FileName));
    await upload.CopyToAsync(stream);

    return RedirectToPage();
}
```

---

## 6. Validation (serveur + client + personnalisée)

### 6.1 Validation serveur (ModelState)

- `DataAnnotations` alimentent `ModelState` via la validation.
- Toujours re-tester `ModelState.IsValid` avant d’écrire en base.

```csharp
if (!ModelState.IsValid)
    return Page();
```

### 6.2 Validation client
- Avec `jquery.validate` + unobtrusive validation (template par défaut).
- Inclure :

```cshtml
@section Scripts {
    <partial name="_ValidationScriptsPartial" />
}
```

> La validation client **améliore** l’UX mais ne remplace jamais la validation serveur.

### 6.3 Messages d’erreurs : ciblés et globaux

- Ciblé : `asp-validation-for="Input.Name"`
- Global :

```csharp
ModelState.AddModelError(string.Empty, "Une erreur inattendue est survenue");
```

### 6.4 Validation personnalisée

#### 6.4.1 Attribut personnalisé

```csharp
public class NotForbiddenNameAttribute : ValidationAttribute
{
    private readonly string[] _forbidden;

    public NotForbiddenNameAttribute(params string[] forbidden)
        => _forbidden = forbidden;

    protected override ValidationResult? IsValid(object? value, ValidationContext validationContext)
    {
        var name = value as string;
        if (name is null) return ValidationResult.Success;

        if (_forbidden.Any(f => string.Equals(f, name, StringComparison.OrdinalIgnoreCase)))
            return new ValidationResult($"Le nom '{name}' est interdit.");

        return ValidationResult.Success;
    }
}
```

Usage :

```csharp
[NotForbiddenName("Test", "Demo")]
public string Name { get; set; } = string.Empty;
```

#### 6.4.2 Validation cross-field via `IValidatableObject`

```csharp
public class ProductInput : IValidatableObject
{
    [Required]
    public string Name { get; set; } = string.Empty;

    public decimal Price { get; set; }

    public bool IsActive { get; set; }

    public IEnumerable<ValidationResult> Validate(ValidationContext validationContext)
    {
        if (!IsActive && Price > 0)
            yield return new ValidationResult("Un produit inactif devrait avoir un prix à 0.",
                new[] { nameof(Price), nameof(IsActive) });
    }
}
```

---

## 7. Layouts, sections, partials (structurer l’UI)

### 7.1 Layout : `_Layout.cshtml`
Le layout définit le “chrome” (header, nav, footer…) et expose `@RenderBody()`.

Exemple (simplifié) :

```cshtml
<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="utf-8" />
    <title>@ViewData["Title"] - Catalog</title>
    <link rel="stylesheet" href="~/css/site.css" />
</head>
<body>
    <header>
        <nav>
            <a asp-page="/Index">Home</a>
            <a asp-page="/Products/Index">Produits</a>
        </nav>
    </header>

    <main class="container">
        @RenderBody()
    </main>

    <footer>
        <small>&copy; @DateTime.UtcNow.Year</small>
    </footer>

    @await RenderSectionAsync("Scripts", required: false)
</body>
</html>
```

### 7.2 Sections
- Une page peut injecter des scripts/styles via `@section`.
- Le layout rend la section optionnelle/obligatoire.

### 7.3 Partials : réutilisation de fragments

Partial : `Pages/Shared/_ProductForm.cshtml`

```cshtml
@model Catalog.Web.Pages.Products.ProductInput

<div class="mb-3">
    <label asp-for="Name" class="form-label"></label>
    <input asp-for="Name" class="form-control" />
    <span asp-validation-for="Name" class="text-danger"></span>
</div>

<div class="mb-3">
    <label asp-for="Price" class="form-label"></label>
    <input asp-for="Price" class="form-control" />
    <span asp-validation-for="Price" class="text-danger"></span>
</div>

<div class="form-check mb-3">
    <input asp-for="IsActive" class="form-check-input" />
    <label asp-for="IsActive" class="form-check-label"></label>
</div>
```

Utilisation :

```cshtml
<partial name="_ProductForm" model="Model.Input" />
```

> Objectif : réduire la duplication Create/Edit.

### 7.4 View Components (quand le partial ne suffit pas)
- Un partial ne contient pas de logique (ou très peu).
- Un View Component encapsule logique + rendu, idéal pour un widget (ex: panier, menu dynamique).

---

## 8. CRUD complet avec Razor Pages + EF Core

On implémente :
- Index (liste)
- Details
- Create
- Edit
- Delete

Dossier : `Pages/Products/`

### 8.1 Index : listing + recherche simple

`Pages/Products/Index.cshtml.cs`

```csharp
using Catalog.Web.Data;
using Catalog.Web.Models;
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.RazorPages;
using Microsoft.EntityFrameworkCore;

namespace Catalog.Web.Pages.Products;

public class IndexModel(AppDbContext db) : PageModel
{
    public IReadOnlyList<Product> Products { get; private set; } = Array.Empty<Product>();

    [BindProperty(SupportsGet = true)]
    public string? Q { get; set; }

    public async Task OnGetAsync()
    {
        var query = db.Products.AsNoTracking();

        if (!string.IsNullOrWhiteSpace(Q))
            query = query.Where(p => p.Name.Contains(Q));

        Products = await query
            .OrderBy(p => p.Name)
            .ToListAsync();
    }
}
```

`Pages/Products/Index.cshtml`

```cshtml
@page
@model Catalog.Web.Pages.Products.IndexModel
@{
    ViewData["Title"] = "Produits";
}

<h1>Produits</h1>

<form method="get" class="mb-3">
    <input class="form-control" name="q" value="@Model.Q" placeholder="Rechercher..." />
</form>

<p>
    <a class="btn btn-primary" asp-page="./Create">Nouveau produit</a>
</p>

<table class="table table-striped">
    <thead>
        <tr>
            <th>Nom</th>
            <th>Prix</th>
            <th>Actif</th>
            <th></th>
        </tr>
    </thead>
    <tbody>
    @foreach (var p in Model.Products)
    {
        <tr>
            <td>@p.Name</td>
            <td>@p.Price</td>
            <td>@p.IsActive</td>
            <td class="text-end">
                <a class="btn btn-sm btn-outline-secondary" asp-page="./Details" asp-route-id="@p.Id">Détails</a>
                <a class="btn btn-sm btn-outline-primary" asp-page="./Edit" asp-route-id="@p.Id">Éditer</a>
                <a class="btn btn-sm btn-outline-danger" asp-page="./Delete" asp-route-id="@p.Id">Supprimer</a>
            </td>
        </tr>
    }
    </tbody>
</table>
```

### 8.2 Details

`Details.cshtml.cs`

```csharp
using Catalog.Web.Data;
using Catalog.Web.Models;
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.RazorPages;
using Microsoft.EntityFrameworkCore;

namespace Catalog.Web.Pages.Products;

public class DetailsModel(AppDbContext db) : PageModel
{
    public Product Product { get; private set; } = default!;

    public async Task<IActionResult> OnGetAsync(int id)
    {
        var p = await db.Products.AsNoTracking().FirstOrDefaultAsync(x => x.Id == id);
        if (p is null) return NotFound();

        Product = p;
        return Page();
    }
}
```

`Details.cshtml`

```cshtml
@page "{id:int}"
@model Catalog.Web.Pages.Products.DetailsModel
@{
    ViewData["Title"] = "Détails produit";
}

<h1>@Model.Product.Name</h1>

<dl>
    <dt>Prix</dt>
    <dd>@Model.Product.Price</dd>
    <dt>Actif</dt>
    <dd>@Model.Product.IsActive</dd>
</dl>

<p>
    <a class="btn btn-primary" asp-page="./Edit" asp-route-id="@Model.Product.Id">Éditer</a>
    <a class="btn btn-secondary" asp-page="./Index">Retour</a>
</p>
```

### 8.3 Create

`Create.cshtml.cs`

```csharp
using Catalog.Web.Data;
using Catalog.Web.Models;
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.RazorPages;

namespace Catalog.Web.Pages.Products;

public class CreateModel(AppDbContext db) : PageModel
{
    [BindProperty]
    public ProductInput Input { get; set; } = new();

    public void OnGet() { }

    public async Task<IActionResult> OnPostAsync()
    {
        if (!ModelState.IsValid)
            return Page();

        var entity = new Product
        {
            Name = Input.Name,
            Price = Input.Price,
            IsActive = Input.IsActive,
        };

        db.Products.Add(entity);
        await db.SaveChangesAsync();

        TempData["Flash"] = "Produit créé.";
        return RedirectToPage("./Index");
    }
}

public class ProductInput
{
    [System.ComponentModel.DataAnnotations.Required]
    [System.ComponentModel.DataAnnotations.StringLength(80)]
    public string Name { get; set; } = string.Empty;

    [System.ComponentModel.DataAnnotations.Range(0, 1_000_000)]
    public decimal Price { get; set; }

    public bool IsActive { get; set; } = true;
}
```

`Create.cshtml`

```cshtml
@page
@model Catalog.Web.Pages.Products.CreateModel
@{
    ViewData["Title"] = "Créer";
}

<h1>Créer un produit</h1>

<form method="post">
    <partial name="_ProductForm" model="Model.Input" />

    <div asp-validation-summary="ModelOnly" class="text-danger"></div>

    <button class="btn btn-primary" type="submit">Créer</button>
    <a class="btn btn-secondary" asp-page="./Index">Annuler</a>
</form>

@section Scripts {
    <partial name="_ValidationScriptsPartial" />
}
```

### 8.4 Edit (avec gestion concurrence RowVersion)

`Edit.cshtml.cs`

```csharp
using Catalog.Web.Data;
using Catalog.Web.Models;
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.RazorPages;
using Microsoft.EntityFrameworkCore;

namespace Catalog.Web.Pages.Products;

public class EditModel(AppDbContext db) : PageModel
{
    [BindProperty]
    public ProductEditInput Input { get; set; } = new();

    public async Task<IActionResult> OnGetAsync(int id)
    {
        var p = await db.Products.AsNoTracking().FirstOrDefaultAsync(x => x.Id == id);
        if (p is null) return NotFound();

        Input = new ProductEditInput
        {
            Id = p.Id,
            Name = p.Name,
            Price = p.Price,
            IsActive = p.IsActive,
            RowVersion = p.RowVersion
        };

        return Page();
    }

    public async Task<IActionResult> OnPostAsync()
    {
        if (!ModelState.IsValid)
            return Page();

        var entity = await db.Products.FirstOrDefaultAsync(x => x.Id == Input.Id);
        if (entity is null) return NotFound();

        // Concurrency
        db.Entry(entity).Property(x => x.RowVersion).OriginalValue = Input.RowVersion;

        entity.Name = Input.Name;
        entity.Price = Input.Price;
        entity.IsActive = Input.IsActive;

        try
        {
            await db.SaveChangesAsync();
        }
        catch (DbUpdateConcurrencyException)
        {
            ModelState.AddModelError(string.Empty,
                "Ce produit a été modifié par quelqu'un d'autre. Rechargez la page et réessayez.");
            return Page();
        }

        TempData["Flash"] = "Produit mis à jour.";
        return RedirectToPage("./Index");
    }
}

public class ProductEditInput
{
    public int Id { get; set; }

    [System.ComponentModel.DataAnnotations.Required]
    [System.ComponentModel.DataAnnotations.StringLength(80)]
    public string Name { get; set; } = string.Empty;

    [System.ComponentModel.DataAnnotations.Range(0, 1_000_000)]
    public decimal Price { get; set; }

    public bool IsActive { get; set; }

    public byte[]? RowVersion { get; set; }
}
```

`Edit.cshtml`

```cshtml
@page "{id:int}"
@model Catalog.Web.Pages.Products.EditModel
@{
    ViewData["Title"] = "Éditer";
}

<h1>Éditer</h1>

<form method="post">
    <input type="hidden" asp-for="Input.Id" />
    <input type="hidden" asp-for="Input.RowVersion" />

    <partial name="_ProductForm" model="Model.Input" />

    <div asp-validation-summary="ModelOnly" class="text-danger"></div>

    <button class="btn btn-primary" type="submit">Enregistrer</button>
    <a class="btn btn-secondary" asp-page="./Index">Retour</a>
</form>

@section Scripts {
    <partial name="_ValidationScriptsPartial" />
}
```

### 8.5 Delete (confirmation + POST)

`Delete.cshtml.cs`

```csharp
using Catalog.Web.Data;
using Catalog.Web.Models;
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.RazorPages;
using Microsoft.EntityFrameworkCore;

namespace Catalog.Web.Pages.Products;

public class DeleteModel(AppDbContext db) : PageModel
{
    public Product Product { get; private set; } = default!;

    public async Task<IActionResult> OnGetAsync(int id)
    {
        var p = await db.Products.AsNoTracking().FirstOrDefaultAsync(x => x.Id == id);
        if (p is null) return NotFound();
        Product = p;
        return Page();
    }

    public async Task<IActionResult> OnPostAsync(int id)
    {
        var p = await db.Products.FirstOrDefaultAsync(x => x.Id == id);
        if (p is null) return RedirectToPage("./Index");

        db.Products.Remove(p);
        await db.SaveChangesAsync();

        TempData["Flash"] = "Produit supprimé.";
        return RedirectToPage("./Index");
    }
}
```

`Delete.cshtml`

```cshtml
@page "{id:int}"
@model Catalog.Web.Pages.Products.DeleteModel
@{
    ViewData["Title"] = "Supprimer";
}

<h1>Supprimer</h1>

<div class="alert alert-warning">
    Confirmez la suppression de <strong>@Model.Product.Name</strong> ?
</div>

<form method="post">
    <button class="btn btn-danger" type="submit">Oui, supprimer</button>
    <a class="btn btn-secondary" asp-page="./Index">Annuler</a>
</form>
```

---

## 9. UX : messages, erreurs, conventions

### 9.1 Flash messages via TempData
Dans `_Layout.cshtml` (ou un partial) :

```cshtml
@if (TempData["Flash"] is string msg)
{
    <div class="alert alert-success">@msg</div>
}
```

### 9.2 Gestion erreurs
- `NotFound()` pour ressources inexistantes
- `ModelState.AddModelError` pour erreurs fonctionnelles
- `UseExceptionHandler` en prod

### 9.3 Accessibilité
- Labels associés (`label asp-for`)
- Messages d’erreur lisibles
- Boutons explicites

---

## 10. Sécurité minimale

### 10.1 Anti-forgery
- Activé par défaut sur POST.
- Ne pas désactiver sauf besoin particulier.

### 10.2 Overposting
- Utiliser DTO d’entrée.
- Mapper explicitement vers l’entité.

### 10.3 Authorization (option)
- Protéger les pages CRUD :

```csharp
// builder.Services.AddAuthentication().AddCookie();
// builder.Services.AddAuthorization();

// app.UseAuthentication();
// app.UseAuthorization();
```

Et côté page :

```csharp
// [Authorize]
public class CreateModel : PageModel { }
```

---

## 11. Atelier final (mini-projet)

### Énoncé
Créer un mini catalogue “Catalog” :
- Produits CRUD complet
- Recherche sur Index (`q`)
- Layout cohérent
- Partials pour formulaire
- Validation (DataAnnotations + une règle custom)
- Pattern PRG + Flash messages

### Critères de réussite
- Aucun POST n’écrit en base si `ModelState` invalide
- Validation client active
- Pas de duplication Create/Edit (partial)
- Gestion 404 sur Details/Edit/Delete
- Concurrence gérée (RowVersion) sur Edit

---

## 12. Checklists & pièges fréquents

### Checklist Forms
- [ ] `method="post"`
- [ ] `asp-for` sur inputs
- [ ] `asp-validation-for` et `asp-validation-summary`
- [ ] `_ValidationScriptsPartial` inclus
- [ ] PRG : Redirect après succès

### Checklist CRUD
- [ ] `AsNoTracking()` sur read-only
- [ ] DTO d’entrée anti-overposting
- [ ] Gestion NotFound
- [ ] Gestion concurrence (si nécessaire)

### Pièges
- Oublier `SupportsGet=true` pour binder sur querystring en GET
- Lier directement une entité EF complète en POST
- Oublier d’inclure les scripts de validation
- Renvoyer `Page()` après POST réussi (au lieu de redirect)

---

## 13. Annexes : snippets utiles

### RedirectToPage avec route

```csharp
return RedirectToPage("./Details", new { id = entity.Id });
```

### Retourner un JSON depuis une page (cas API-like)

```csharp
public IActionResult OnGetPing() => new JsonResult(new { ok = true });
```

### Handler nommé + bouton

```cshtml
<button asp-page-handler="Deactivate" class="btn btn-warning">Désactiver</button>
```

```csharp
public async Task<IActionResult> OnPostDeactivateAsync(int id) { /*...*/ return RedirectToPage(); }
```

---

## 14. Conclusion

Razor Pages permet de construire rapidement des applications Web maintenables grâce à :
- un modèle orienté page,
- des conventions simples,
- un binding/validation puissants,
- une UI structurée (layouts/partials),
- et une implémentation CRUD très directe avec EF Core.

Prochaines étapes recommandées :
- Ajout d’authentification (ASP.NET Core Identity)
- Paging/Sorting avancés
- Tests (PageModel unit tests + tests d’intégration WebApplicationFactory)
- Composants UI (View Components / HTMX / minimal JS)
