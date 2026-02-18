# Masterclass — Introduction to C# 13 & .NET 9

> **Audience** : développeurs .NET (débutants à intermédiaires) et/ou équipes qui migrent vers .NET 9 / C# 13  
> **Prérequis** : bases de programmation (variables, conditions, boucles), notions de POO recommandées, installation d’un IDE (VS / Rider / VS Code)  
> **Durée suggérée** : 1 jour (6–7h) ou 2 demi‑journées  
> **Objectif** : comprendre l’écosystème .NET 9, les types d’applications, les nouveautés de C# 13, les outils (SDK/CLI/IDE), et adopter des bases solides pour CI/CD, déploiement et cross‑platform.

---

## Sommaire

1. [Objectifs pédagogiques](#1-objectifs-pédagogiques)
2. [Plan de la formation (agenda)](#2-plan-de-la-formation-agenda)
3. [Module 1 — Panorama .NET 9 & C# 13](#3-module-1--panorama-net-9--c-13)
4. [Module 2 — Le runtime .NET : SDK, runtime, packs, NuGet](#4-module-2--le-runtime-net--sdk-runtime-packs-nuget)
5. [Module 3 — Types d’applications .NET modernes](#5-module-3--types-dapplications-net-modernes)
6. [Module 4 — Tooling : CLI, IDE, analyzers, formatters](#6-module-4--tooling--cli-ide-analyzers-formatters)
7. [Module 5 — Cross‑platform : exécuter et packager sur Windows/macOS/Linux](#7-module-5--crossplatform--exécuter-et-packager-sur-windowsmacoslinux)
8. [Module 6 — CI/CD : build, test, qualité, sécurité](#8-module-6--cicd--build-test-qualité-sécurité)
9. [Module 7 — Stratégies de déploiement](#9-module-7--stratégies-de-déploiement)
10. [Atelier fil rouge (mini‑projet)](#10-atelier-fil-rouge-mini-projet)
11. [Checklist de fin de formation](#11-checklist-de-fin-de-formation)
12. [Annexes : commandes utiles](#12-annexes--commandes-utiles)

---

## 1. Objectifs pédagogiques

À la fin de cette formation, les participants seront capables de :

- Expliquer le rôle de **C#**, du **.NET SDK**, du **runtime** et de l’écosystème **NuGet**.
- Identifier les principaux **types d’applications** (.NET) et choisir une architecture adaptée.
- Utiliser efficacement le **dotnet CLI** (création, build, run, test, publish, pack).
- Comprendre les grandes lignes des nouveautés de **C# 13** et des évolutions de **.NET 9** (orientation pratique).
- Mettre en place une pipeline de **CI/CD** simple (build + tests + analyse + artefacts).
- Choisir une stratégie de **déploiement** (framework‑dependent, self‑contained, single‑file, containers, cloud).
- Développer et livrer en **cross‑platform**.
- Utiliser l’IDE (Visual Studio / Rider / VS Code) de manière productive.

---

## 2. Plan de la formation (agenda)

> Ajustez selon votre contexte (présentiel/distanciel). Les modules 6–7 peuvent être condensés pour une session plus courte.

| Module | Titre | Durée | Format |
|---:|---|---:|---|
| 1 | Panorama .NET 9 & C# 13 | 45 min | Cours + démo |
| 2 | Runtime .NET : SDK, runtime, packs, NuGet | 50 min | Cours + démo |
| 3 | Types d’applications .NET modernes | 60 min | Cours + étude de cas |
| 4 | Tooling : CLI, IDE, analyzers, formatters | 75 min | Démo + exercices |
| 5 | Cross‑platform & packaging | 45 min | Démo |
| 6 | CI/CD : build, tests, qualité, sécurité | 70 min | Cours + exemple pipeline |
| 7 | Stratégies de déploiement | 60 min | Cours + démo publish |
| 10 | Atelier fil rouge | 90 min | Pratique guidée |
| 11 | Checklist & wrap‑up | 15 min | Discussion |

---

## 3. Module 1 — Panorama .NET 9 & C# 13

### 3.1. Qu’est‑ce que .NET ?

- **.NET** est une plateforme de développement et d’exécution.
- Elle inclut :
  - un **runtime** (CLR/CoreCLR) : exécution du code IL, GC, JIT/AOT, gestion mémoire.
  - des **libraries** (BCL) : collections, I/O, réseau, JSON, etc.
  - un **SDK** : outils de build, templates, dotnet CLI.
  - une chaîne d’outils : MSBuild, analyzers, test runners, etc.

### 3.2. C# dans l’écosystème .NET

- **C#** est le langage principal pour construire des apps .NET.
- Le mode de compilation :
  1. C# → **IL** (Intermediate Language)
  2. IL → exécution via **JIT** ou compilation **AOT** selon l’app/plateforme

### 3.3. Pourquoi C# 13 / .NET 9 ?

- Continuité de la modernisation : productivité, performance, cloud‑native, AOT, outillage.
- Focus :
  - **qualité** (analyzers, nullability, warnings)
  - **déploiement** simplifié (publish, containers)
  - **cross‑platform** stable

> Note : les détails exacts des nouveautés peuvent varier avec les builds/RTM. L’approche ci‑dessous vise à comprendre *comment explorer* et *adopter* les nouveautés via tooling, documentation, et tests de migration.

### 3.4. Bonnes pratiques de migration (approche)

- Migrer le **SDK target** d’abord, sans changer le code.
- Activer la **build** + **tests** en CI rapidement.
- Ensuite seulement, activer/resserrer :
  - **nullable**
  - analyzers de style/qualité
  - warnings → erreurs (progressivement)

---

## 4. Module 2 — Le runtime .NET : SDK, runtime, packs, NuGet

### 4.1. Le .NET SDK vs runtime

- **Runtime** : permet d’exécuter une application.
- **SDK** : permet de développer/construire/publier.

Vérification locale :

```bash
# Liste des SDK installés
 dotnet --list-sdks

# Liste des runtimes installés
 dotnet --list-runtimes

# Info complète
 dotnet --info
```

### 4.2. Target Framework Monikers (TFM)

- Les projets ciblent un framework via `<TargetFramework>`.
- Exemples typiques :

```xml
<PropertyGroup>
  <TargetFramework>net9.0</TargetFramework>
  <ImplicitUsings>enable</ImplicitUsings>
  <Nullable>enable</Nullable>
</PropertyGroup>
```

- **Multi‑targeting** pour librairies :

```xml
<PropertyGroup>
  <TargetFrameworks>net8.0;net9.0</TargetFrameworks>
</PropertyGroup>
```

### 4.3. NuGet : dépendances, restore, lock files

- `dotnet restore` résout les dépendances.
- Fichier `*.csproj` → `PackageReference`.

```xml
<ItemGroup>
  <PackageReference Include="Microsoft.Extensions.Http" Version="9.0.0" />
</ItemGroup>
```

Recommandations :

- Utiliser un **lock file** (`packages.lock.json`) pour reproductibilité.
- Central Package Management (CPM) via `Directory.Packages.props` (utile en mono‑repo).

Exemple `Directory.Packages.props` :

```xml
<Project>
  <PropertyGroup>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
  </PropertyGroup>
  <ItemGroup>
    <PackageVersion Include="Serilog" Version="4.0.0" />
  </ItemGroup>
</Project>
```

### 4.4. Structure d’une solution .NET

- `*.sln` : agrège des projets.
- `src/` : code applicatif, `tests/` : tests.
- `Directory.Build.props` / `Directory.Build.targets` : configuration commune.

---

## 5. Module 3 — Types d’applications .NET modernes

### 5.1. Console / Worker services

- **Console** : scripts, CLI, batchs.
- **Worker Service** : services long‑running, background processing.

Création :

```bash
dotnet new console -n Demo.Console
dotnet new worker -n Demo.Worker
```

Points clés :

- Logging, configuration, DI via `Microsoft.Extensions.*`.
- Hébergement générique (`HostBuilder`).

### 5.2. Web APIs (ASP.NET Core)

- **Minimal APIs** : rapides à démarrer.
- **Controllers** : structuration, API versioning, conventions.

Création :

```bash
dotnet new webapi -n Demo.Api
```

Concepts :

- Middleware pipeline
- Endpoints
- OpenAPI/Swagger
- Authentication/Authorization

### 5.3. Applications web (MVC / Razor / Blazor)

- **Razor Pages** : pages orientées UI simple.
- **MVC** : pattern plus classique.
- **Blazor** : UI en C# (Server / WebAssembly / hybrid).

### 5.4. Desktop

- **Windows** : WPF/WinForms (net9.0‑windows).
- **Cross‑platform** : MAUI (selon besoins et maturité du parc).

### 5.5. Bibliothèques, packages, SDKs internes

- Créer des packages **NuGet** internes.
- Encourager le **versioning sémantique**.

```bash
dotnet new classlib -n Demo.Lib
```

### 5.6. Choisir le bon type d’app : grille de décision

| Besoin | Choix recommandé |
|---|---|
| API HTTP + perf + cloud | ASP.NET Core Web API / Minimal APIs |
| Traitement asynchrone, queue | Worker Service |
| Outil interne, scripting | Console |
| UI web moderne en C# | Blazor |
| UI Windows | WPF/WinForms |
| Partage de logique | Class Library + NuGet |

---

## 6. Module 4 — Tooling : CLI, IDE, analyzers, formatters

### 6.1. dotnet CLI : workflow essentiel

Créer une solution + projets :

```bash
dotnet new sln -n Demo
mkdir src tests
dotnet new webapi -n Demo.Api -o src/Demo.Api
dotnet new xunit -n Demo.Api.Tests -o tests/Demo.Api.Tests

dotnet sln Demo.sln add src/Demo.Api/Demo.Api.csproj
dotnet sln Demo.sln add tests/Demo.Api.Tests/Demo.Api.Tests.csproj

dotnet add tests/Demo.Api.Tests/Demo.Api.Tests.csproj reference src/Demo.Api/Demo.Api.csproj
```

Build / run / test :

```bash
dotnet build
dotnet run --project src/Demo.Api
dotnet test
```

### 6.2. Gestion des configurations (appsettings)

- `appsettings.json` + `appsettings.Development.json`.
- Variables d’environnement : `ASPNETCORE_ENVIRONMENT`.

### 6.3. EditorConfig & formatage

- `.editorconfig` : conventions de code (espaces, naming, style).
- `dotnet format` (si installé) pour appliquer.

Extrait `.editorconfig` :

```ini
root = true

[*.cs]
dotnet_style_qualification_for_field = false:suggestion
csharp_style_var_for_built_in_types = true:suggestion
```

### 6.4. Analyzers et qualité

- Les analyzers .NET fournissent :
  - avertissements de qualité/sécurité
  - recommandations de style
- Stratégie : commencer en `suggestion` puis durcir.

Dans le `.csproj` :

```xml
<PropertyGroup>
  <AnalysisLevel>latest</AnalysisLevel>
  <TreatWarningsAsErrors>false</TreatWarningsAsErrors>
</PropertyGroup>
```

### 6.5. IDE : Visual Studio / Rider / VS Code

**Points communs** :

- Navigation (Go to definition, Find usages)
- Refactoring (rename, extract method)
- Debugger (breakpoints, watch)
- Profiling (CPU/memory)

**Usage recommandé en formation** :

- Apprendre les raccourcis essentiels.
- Savoir basculer entre debug local et exécution CLI.
- Configurer le **launchSettings.json** (profils, variables).

---

## 7. Module 5 — Cross‑platform : exécuter et packager sur Windows/macOS/Linux

### 7.1. Les RID (Runtime Identifiers)

- RID = cible OS/arch (ex : `win-x64`, `linux-x64`, `osx-arm64`).
- Utile pour publier en **self‑contained**.

Liste indicative :

- Windows : `win-x64`, `win-arm64`
- Linux : `linux-x64`, `linux-arm64`
- macOS : `osx-x64`, `osx-arm64`

### 7.2. Publier une application

Framework‑dependent (nécessite runtime installé) :

```bash
dotnet publish -c Release -o out
```

Self‑contained (embarque le runtime) :

```bash
dotnet publish -c Release -r linux-x64 --self-contained true -o out
```

Single‑file (optionnel selon type d’app) :

```bash
dotnet publish -c Release -r win-x64 --self-contained true \
  /p:PublishSingleFile=true -o out
```

### 7.3. AOT (vue d’ensemble)

- Certaines applis bénéficient de l’**Ahead‑of‑Time** (NativeAOT) :
  - démarrage plus rapide
  - empreinte plus petite
  - contraintes de réflexion/dynamic code

Approche recommandée :

- Commencer par une app simple (console/minimal API).
- Mesurer, puis ajuster (trimming, source generators, etc.).

---

## 8. Module 6 — CI/CD : build, test, qualité, sécurité

### 8.1. Objectifs CI

- Reproductibilité : même build sur poste et CI.
- Rapidité : cache NuGet.
- Qualité : tests, couverture, analyzers.
- Sécurité : scan de dépendances.

### 8.2. Pipeline type (exemple GitHub Actions)

> À adapter pour Azure DevOps / GitLab CI.

```yaml
name: ci
on:
  push:
  pull_request:

jobs:
  build-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '9.0.x'

      - name: Restore
        run: dotnet restore

      - name: Build
        run: dotnet build -c Release --no-restore

      - name: Test
        run: dotnet test -c Release --no-build --collect:"XPlat Code Coverage"
```

Ajouts fréquents :

- artefacts (`dotnet publish` puis upload)
- analyse (Sonar, Roslyn)
- SAST/Dependency scanning

### 8.3. Versioning et release

- Stratégie : SemVer, GitVersion, ou version basée tag.
- Publier packages : `dotnet pack` + push vers feed.

```bash
dotnet pack -c Release
```

---

## 9. Module 7 — Stratégies de déploiement

### 9.1. Choisir : framework-dependent vs self-contained

- **Framework-dependent**
  - ✅ artefacts plus petits
  - ✅ patch runtime géré séparément
  - ❗ nécessite runtime installé

- **Self-contained**
  - ✅ autonomie (runtime inclus)
  - ✅ utile sur serveurs verrouillés
  - ❗ artefacts plus gros

### 9.2. Déploiement on‑prem / VM

- Service systemd (Linux) pour worker/API.
- IIS (Windows) pour ASP.NET Core (ou Kestrel + reverse proxy).

### 9.3. Déploiement container (Docker)

Objectif : empaqueter une API dans une image.

Dockerfile (exemple multi‑stage) :

```dockerfile
# build
FROM mcr.microsoft.com/dotnet/sdk:9.0 AS build
WORKDIR /src
COPY . .
RUN dotnet publish src/Demo.Api/Demo.Api.csproj -c Release -o /app/publish

# run
FROM mcr.microsoft.com/dotnet/aspnet:9.0
WORKDIR /app
COPY --from=build /app/publish .
ENV ASPNETCORE_URLS=http://+:8080
EXPOSE 8080
ENTRYPOINT ["dotnet", "Demo.Api.dll"]
```

Commandes :

```bash
docker build -t demo-api:net9 .
docker run -p 8080:8080 demo-api:net9
```

### 9.4. Déploiement cloud (survol)

- Azure App Service / Container Apps
- AWS ECS/Fargate
- Kubernetes

Points clés :

- logs structurés
- health checks
- config via variables/secret manager
- observabilité (OpenTelemetry)

---

## 10. Atelier fil rouge (mini‑projet)

### 10.1. Objectif

Construire une mini‑solution **API + bibliothèque + tests**, puis l’exécuter en local et la publier.

### 10.2. Étapes guidées

1) Créer la solution :

```bash
dotnet new sln -n Catalog
mkdir -p src tests
```

2) Créer une bibliothèque domaine :

```bash
dotnet new classlib -n Catalog.Domain -o src/Catalog.Domain
```

3) Créer une API :

```bash
dotnet new webapi -n Catalog.Api -o src/Catalog.Api
```

4) Créer un projet de tests :

```bash
dotnet new xunit -n Catalog.Domain.Tests -o tests/Catalog.Domain.Tests
```

5) Références :

```bash
dotnet sln Catalog.sln add src/Catalog.Domain/Catalog.Domain.csproj
dotnet sln Catalog.sln add src/Catalog.Api/Catalog.Api.csproj
dotnet sln Catalog.sln add tests/Catalog.Domain.Tests/Catalog.Domain.Tests.csproj

dotnet add src/Catalog.Api/Catalog.Api.csproj reference src/Catalog.Domain/Catalog.Domain.csproj

dotnet add tests/Catalog.Domain.Tests/Catalog.Domain.Tests.csproj reference src/Catalog.Domain/Catalog.Domain.csproj
```

6) Ajouter un modèle simple `Product` dans `Catalog.Domain` :

```csharp
namespace Catalog.Domain;

public sealed record Product(Guid Id, string Name, decimal Price);
```

7) Exposer un endpoint minimal dans `Catalog.Api` :

```csharp
using Catalog.Domain;

var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.MapGet("/products", () => new[]
{
    new Product(Guid.NewGuid(), "Keyboard", 49.99m),
    new Product(Guid.NewGuid(), "Mouse", 19.99m)
});

app.Run();
```

8) Écrire un test simple dans `Catalog.Domain.Tests` :

```csharp
using Catalog.Domain;

namespace Catalog.Domain.Tests;

public class ProductTests
{
    [Fact]
    public void Product_should_keep_values()
    {
        var id = Guid.NewGuid();
        var p = new Product(id, "Keyboard", 49.99m);

        Assert.Equal(id, p.Id);
        Assert.Equal("Keyboard", p.Name);
        Assert.Equal(49.99m, p.Price);
    }
}
```

9) Exécuter :

```bash
dotnet test
dotnet run --project src/Catalog.Api
```

10) Publier :

```bash
dotnet publish src/Catalog.Api -c Release -o out
```

### 10.3. Critères de réussite

- La solution compile (`dotnet build`).
- Les tests passent (`dotnet test`).
- L’endpoint `/products` répond en local.
- Un dossier `out/` est généré avec les artefacts.

---

## 11. Checklist de fin de formation

- [ ] Je sais différencier **SDK**, **runtime**, **TFM**, **RID**.
- [ ] Je sais créer une solution multi‑projets avec **dotnet new** / **dotnet sln**.
- [ ] Je sais exécuter **build/test/publish**.
- [ ] Je sais choisir une stratégie de packaging (framework‑dependent vs self‑contained).
- [ ] Je sais les bases d’une pipeline CI (restore/build/test) et comment l’étendre.
- [ ] Je sais utiliser mon IDE pour refactor/debug/profiler.

---

## 12. Annexes : commandes utiles

### 12.1. Nettoyage

```bash
dotnet clean
```

### 12.2. Ajouter un package

```bash
dotnet add src/Catalog.Api package Serilog.AspNetCore
```

### 12.3. Lister les packages

```bash
dotnet list package
```

### 12.4. Mettre à jour les templates

```bash
dotnet new update
```

### 12.5. Créer un `.gitignore`

```bash
dotnet new gitignore
```

---

## Notes formateur (optionnel)

- Préparer une machine « propre » (sans caches) pour montrer `restore`.
- Prévoir variantes : Visual Studio vs Rider vs VS Code.
- Laisser 15 minutes pour questions : migration de solutions existantes, nugets internes, AOT/containers.
