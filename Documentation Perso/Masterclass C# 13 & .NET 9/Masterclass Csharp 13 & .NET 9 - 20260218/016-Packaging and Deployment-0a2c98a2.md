# Masterclass C# 13 & .NET 9 — 016 — Packaging and Deployment

> Public cible : développeurs/.NET, formateurs, tech leads.  
> Pré-requis : C#/.NET (build, projets SDK-style), notions de CI/CD.  
> Durée indicative : 1 journée (6–7h) ou 2 demi-journées.

## Objectifs pédagogiques

À l’issue de cette formation, vous saurez :

- Créer des packages **NuGet** de qualité (metadata, multi-targeting, symbol packages, SourceLink).
- Mettre en place une **stratégie de versioning** cohérente (SemVer) et comprendre le versioning des assemblies.
- Produire et consommer des **assemblies strong-name** (signature, clés, implications, bonnes pratiques).
- Expliquer et utiliser **Native AOT** (scénarios, contraintes, trimming, interop).
- Déployer via **.NET CLI** : self-contained, framework-dependent, single-file, ready-to-run, runtime identifiers, linux/windows, conteneurs.

---

## Plan de la formation

1. **Packaging NuGet**
   - Anatomie d’un package
   - Pack via `.csproj` et `.nuspec`
   - Multi-targeting, assets, analyzers
   - Symbol packages, SourceLink
   - Versioning et publication (feed privé, nuget.org)
2. **Assembly Versioning**
   - `AssemblyVersion`, `AssemblyFileVersion`, `AssemblyInformationalVersion`
   - Versioning SDK-style / `GenerateAssemblyInfo`
   - Impact sur binding (GAC legacy, load contexts) et compatibilité
   - Stratégie SemVer + Git
3. **Strong-named assemblies**
   - Principe, signature, vérification
   - Création/gestion de clés
   - Signer bibliothèques et exécutables
   - Pitfalls, compatibilité, recommandations (2025+)
4. **Native AOT**
   - Ce que c’est, bénéfices et coûts
   - Publication AOT, RID, toolchain
   - Limitations (reflection, dynamic) et solutions
   - Trimming, RD.XML, attributes
5. **Déploiement avec .NET CLI**
   - `dotnet publish` en profondeur
   - Framework-dependent vs self-contained
   - Single-file, trimming, R2R
   - Variables d’environnement, config, secrets
   - Packaging final (zip, docker, systemd, Windows service)

---

# 1) NuGet packaging

## 1.1 Concepts clés

Un **package NuGet** est une archive `.nupkg` (zip) contenant :

- Des assemblies (`lib/`), éventuellement par TFM (ex: `lib/net9.0/…`)
- Des métadonnées (id, version, authors, description, license…)
- Des dépendances (par TFM)
- Des assets optionnels : `contentFiles/`, `build/`, `analyzers/`, `native/`, `runtimes/`, `tools/`

### Pourquoi packager ?

- Réutilisation et distribution interne
- Versioning et traçabilité
- CI/CD : livraison contrôlée
- Séparation des responsabilités (app vs libs)

---

## 1.2 Packager via `.csproj` (approche recommandée)

Dans les projets SDK-style, `dotnet pack` se base principalement sur le **fichier projet**.

### Exemple minimal

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net9.0</TargetFramework>

    <PackageId>Company.Product.MyLib</PackageId>
    <Version>1.2.3</Version>
    <Authors>Company</Authors>
    <Description>Lib utilitaire X…</Description>
    <PackageLicenseExpression>MIT</PackageLicenseExpression>
    <RepositoryUrl>https://github.com/company/product</RepositoryUrl>
    <PackageProjectUrl>https://company.tld/product</PackageProjectUrl>
    <PackageTags>csharp;dotnet;utilities</PackageTags>

    <GeneratePackageOnBuild>false</GeneratePackageOnBuild>
    <IncludeSymbols>true</IncludeSymbols>
    <SymbolPackageFormat>snupkg</SymbolPackageFormat>
  </PropertyGroup>
</Project>
```

### Commandes

```bash
# pack en Release
dotnet pack -c Release

# pack avec version fournie par la CI
dotnet pack -c Release -p:Version=1.2.3
```

> Bonnes pratiques : ne **pas** figer la version dans le `.csproj` si votre CI la fournit (GitVersion, Nerdbank.GitVersioning, MinVer, etc.).

---

## 1.3 Multi-targeting (TFM)

### Pourquoi ?

- Supporter plusieurs runtimes/versions : `netstandard2.0`, `net8.0`, `net9.0`, etc.
- Permettre à des applications plus anciennes de consommer votre package.

### Exemple

```xml
<PropertyGroup>
  <TargetFrameworks>netstandard2.0;net8.0;net9.0</TargetFrameworks>
</PropertyGroup>
```

### Code conditionnel

```csharp
#if NET9_0_OR_GREATER
// fonctionnalités .NET 9+
#else
// fallback
#endif
```

### Dépendances par TFM

```xml
<ItemGroup Condition="'$(TargetFramework)' == 'netstandard2.0'">
  <PackageReference Include="System.Text.Json" Version="8.0.5" />
</ItemGroup>

<ItemGroup Condition="'$(TargetFramework)' == 'net9.0'">
  <PackageReference Include="Some.Modern.Dependency" Version="1.0.0" />
</ItemGroup>
```

---

## 1.4 Contrôler ce qui est packagé

### Inclure des fichiers supplémentaires

```xml
<ItemGroup>
  <None Include="README.md" Pack="true" PackagePath="" />
</ItemGroup>
```

### `PackageReadmeFile`

NuGet peut afficher un README dans l’UI.

```xml
<PropertyGroup>
  <PackageReadmeFile>README.md</PackageReadmeFile>
</PropertyGroup>

<ItemGroup>
  <None Include="README.md" Pack="true" PackagePath="" />
</ItemGroup>
```

### `contentFiles` pour templates/config

```xml
<ItemGroup>
  <Content Include="contentFiles/**/*" Pack="true" PackagePath="contentFiles/any/any/" />
</ItemGroup>
```

> À utiliser avec prudence : `contentFiles` impacte le projet consommateur.

---

## 1.5 `nuspec` (quand et pourquoi)

Le `.nuspec` est utile si :

- Vous packager des artefacts non standard
- Vous voulez un contrôle fin (ex: fichiers dans `tools/`, `build/`)
- Vous packager sans projet `.csproj`

Mais la plupart du temps, **préférez le `.csproj`**.

---

## 1.6 Packages symboles et debugging

### Symbol packages (`.snupkg`)

- Permettent de déboguer dans le code source depuis un consumer.
- Publication sur : nuget.org (symbols), Azure Artifacts, GitHub Packages selon support.

### SourceLink

SourceLink associe le PDB à un commit Git, permettant au débogueur de récupérer les sources.

Ajoutez les packages :

```xml
<ItemGroup>
  <PackageReference Include="Microsoft.SourceLink.GitHub" Version="8.0.0" PrivateAssets="All" />
</ItemGroup>

<PropertyGroup>
  <PublishRepositoryUrl>true</PublishRepositoryUrl>
  <EmbedUntrackedSources>true</EmbedUntrackedSources>
</PropertyGroup>
```

---

## 1.7 Validation et qualité

### Commandes utiles

```bash
# inspecter un .nupkg
nuget locals all -list

# lister le contenu
unzip -l bin/Release/*.nupkg
```

### Checklist qualité

- Metadata complète (license, repository, description, tags)
- `PackageReadmeFile` et éventuellement `PackageIcon`
- Multi-targeting pertinent
- API stable et documentée (XML docs)
- Symbols + SourceLink
- Dépendances minimes et bien versionnées

---

# 2) Assembly versioning

Le versioning côté .NET combine :

- Version **NuGet** (SemVer) : `PackageVersion` / `Version`
- Version **assembly** : `AssemblyVersion`
- Version **fichier** (Windows) : `AssemblyFileVersion`
- Version **informational** (string libre, ex Git SHA) : `AssemblyInformationalVersion`

## 2.1 Les 3 attributs principaux

### `AssemblyVersion`
- Influence le **binding** de l’assembly.
- Typiquement : `MAJOR.0.0.0` ou `MAJOR.MINOR.0.0`.
- Changer `AssemblyVersion` peut casser la compatibilité binaire si des dépendances attendent une version précise.

### `AssemblyFileVersion`
- N’influence pas le binding ; utile pour diagnostiquer.
- Peut être incrémentée à chaque release.

### `AssemblyInformationalVersion`
- Peut inclure un suffixe (SemVer complet) : `1.2.3+githash`.

---

## 2.2 SDK-style: configuration via `.csproj`

Par défaut, le SDK génère les attributs (via `GenerateAssemblyInfo` = `true`).

### Exemple

```xml
<PropertyGroup>
  <Version>2.4.0</Version>
  <AssemblyVersion>2.0.0.0</AssemblyVersion>
  <FileVersion>2.4.0.0</FileVersion>
  <InformationalVersion>2.4.0+$(SourceRevisionId)</InformationalVersion>
</PropertyGroup>
```

`$(SourceRevisionId)` peut être injecté par la CI (GitHub Actions, Azure DevOps, etc.).

---

## 2.3 SemVer en pratique

- **MAJOR** : breaking changes
- **MINOR** : features compatibles
- **PATCH** : corrections
- Pré-release : `1.2.0-alpha.1`

### Recommandation courante

- NuGet `Version` = SemVer complet.
- `AssemblyVersion` stable (ex: `1.0.0.0` tant que MAJOR=1) pour limiter les soucis de binding.
- `FileVersion` suit les releases.

> Cette approche est surtout utile pour des libs partagées largement. Pour des applications, vous pouvez aligner plus directement.

---

## 2.4 Versioning automatisé (approches)

- **Nerdbank.GitVersioning** : version basée sur Git + config `version.json`
- **MinVer** : version minimale via tags Git
- **GitVersion** : versioning avancé basé sur branches

Objectif : une version **traçable** et **reproductible**.

---

# 3) Strong-named assemblies

## 3.1 Définition

Un assembly **strong-name** est signé avec une clé (paire RSA). Cela :

- Ajoute une **identité forte** : nom + version + culture + public key token
- Permet (historiquement) le chargement depuis le GAC (plutôt .NET Framework)
- Empêche la modification silencieuse de l’assembly (détection via signature)

> Dans .NET moderne, le strong-name n’est **pas** une sécurité “anti-tamper” complète. C’est une identité/garantie d’intégrité, mais pas un mécanisme anti reverse engineering.

---

## 3.2 Quand signer ?

À envisager si :

- Vous devez référencer un assembly strong-named depuis un autre strong-named (certaines contraintes legacy)
- Vous distribuez des plugins/SDK et voulez une identité forte
- Vous ciblez encore des scénarios .NET Framework

À éviter si :

- Vous n’en avez pas besoin (complexifie le build, la compatibilité, les forks)
- Vous publiez de l’open-source où la gestion de clés doit être claire

---

## 3.3 Génération de clé

### `sn.exe` (Windows SDK)

```bash
sn -k company.snk
sn -p company.snk company.publickey
```

### Bonnes pratiques

- Protéger la clé privée : coffre (Azure Key Vault), secrets CI, accès restreint.
- Utiliser une clé distincte par produit/organisation.
- Documenter la rotation (rare, mais planifier).

---

## 3.4 Signer un projet

```xml
<PropertyGroup>
  <SignAssembly>true</SignAssembly>
  <AssemblyOriginatorKeyFile>company.snk</AssemblyOriginatorKeyFile>
</PropertyGroup>
```

> En CI : évitez de committer la clé privée. Préférez injecter le fichier au build via secret.

### Vérifier

```bash
sn -v path/to/YourAssembly.dll
```

---

## 3.5 Interactions et pièges

- Un assembly strong-named **référence** typiquement des dépendances strong-named (surtout en .NET Framework). Dans .NET (Core/5+), c’est plus souple mais peut rester une contrainte selon le contexte.
- Le strong-name n’empêche pas de charger une version différente si vous contrôlez le runtime (LoadContext). Mais il fixe l’identité.
- Si vous devez “retargeter” une dépendance non strong-named, vous pouvez envisager :
  - Remplacer la dépendance
  - Utiliser une alternative strong-named
  - Éviter le strong-name côté lib si possible

---

# 4) Native AOT

## 4.1 Qu’est-ce que Native AOT ?

**Native AOT** (Ahead-of-Time) compile une application .NET en **binaire natif** (ex: `.exe` Windows, ELF Linux) avec :

- Démarrage souvent plus rapide
- Moins de dépendances (pas besoin du runtime séparé)
- Meilleure compatibilité avec des environnements verrouillés

Mais en échange :

- Build plus long
- Taille parfois plus grande (selon options)
- Contraintes sur reflection, dynamic code, certains patterns

---

## 4.2 Activer Native AOT

Dans un projet application :

```xml
<PropertyGroup>
  <TargetFramework>net9.0</TargetFramework>
  <PublishAot>true</PublishAot>
</PropertyGroup>
```

Puis :

```bash
dotnet publish -c Release -r win-x64
```

> Native AOT nécessite généralement un RID explicite (`-r`). Sur Linux, une toolchain (clang, etc.) est requise.

---

## 4.3 Reflection, trimming et compatibilité

Native AOT s’appuie fortement sur le **trimming** (suppression de code non utilisé). Cela peut casser :

- Reflection (types/membres accédés dynamiquement)
- Sérialisation basées sur reflection
- `Activator.CreateInstance`, `Type.GetType("...")`, DI scanning

### Stratégies

1. **Éviter la reflection** quand possible (générateurs de source).
2. Utiliser des bibliothèques AOT-friendly.
3. Annoter avec des attributs :

```csharp
using System.Diagnostics.CodeAnalysis;

[DynamicallyAccessedMembers(DynamicallyAccessedMemberTypes.PublicConstructors)]
static Type _type;
```

4. Pour System.Text.Json : préférer **source generation**.

---

## 4.4 Scénarios adaptés

- CLI tools
- microservices ultra denses (latence de start)
- apps embarquées
- fonctions serverless (cold start)

Moins adapté :

- apps très dynamiques (plugins à chaud, reflection intensive)
- certaines apps desktop complexes selon dépendances

---

# 5) .NET CLI deployment (dotnet publish)

## 5.1 Les modes de déploiement

### Framework-dependent deployment (FDD)

- Le runtime .NET est **préinstallé** sur la machine cible.
- Output plus petit.

```bash
dotnet publish -c Release
```

### Self-contained deployment (SCD)

- Inclut le runtime + librairies nécessaires.
- Plus gros, mais autonome.

```bash
dotnet publish -c Release -r win-x64 --self-contained true
```

---

## 5.2 Runtime Identifiers (RID)

Exemples :

- `win-x64`, `win-arm64`
- `linux-x64`, `linux-arm64`
- `osx-x64`, `osx-arm64`

Lister les RIDs supportés :

- Documentation .NET (RID catalog)
- `dotnet --info` (partiel)

---

## 5.3 Single-file

Objectif : produire un seul fichier exécutable (et parfois quelques fichiers annexes selon config).

```bash
dotnet publish -c Release -r win-x64 \
  -p:PublishSingleFile=true \
  -p:SelfContained=true
```

Points d’attention :

- Extraction au runtime possible (selon options)
- Certains scénarios (native libs, plugins) peuvent nécessiter des fichiers séparés

---

## 5.4 Trimming

Réduit la taille de l’output en supprimant le code non utilisé.

```bash
dotnet publish -c Release -r linux-x64 \
  -p:PublishTrimmed=true
```

Risques :

- Reflection et patterns dynamiques

Mitigation :

- Activer warnings d’analyse, tests end-to-end
- Utiliser source generators

---

## 5.5 ReadyToRun (R2R)

Compile partiellement en natif (JIT réduit au démarrage), sans les fortes contraintes de l’AOT.

```bash
dotnet publish -c Release -r win-x64 \
  -p:PublishReadyToRun=true
```

Tradeoff :

- Taille plus grande
- Gains variables selon l’app

---

## 5.6 Paramètres de publication essentiels (récap)

| Objectif | Paramètres |
|---|---|
| autonome | `-r <RID> --self-contained true` |
| dépendant du runtime | `--self-contained false` (par défaut) |
| single file | `-p:PublishSingleFile=true` |
| trimming | `-p:PublishTrimmed=true` |
| R2R | `-p:PublishReadyToRun=true` |
| Native AOT | `-p:PublishAot=true` |

---

## 5.7 Structurer l’output et la config

### `appsettings.json`

- FDD/SCD : fichiers copiés dans le dossier publish.
- Single file : selon options, peut embarquer/extraire.

### Environnements

- `DOTNET_ENVIRONMENT=Production|Staging|Development`

### Secrets

- En dev : `dotnet user-secrets`
- En prod : variables d’environnement, Key Vault/Secrets Manager

---

## 5.8 Déployer sur Linux (ex: systemd)

### Publier

```bash
dotnet publish -c Release -r linux-x64 --self-contained true
```

### Unité systemd (exemple)

```ini
[Unit]
Description=MyApp
After=network.target

[Service]
WorkingDirectory=/opt/myapp
ExecStart=/opt/myapp/MyApp
Restart=always
RestartSec=10
User=myapp
Environment=DOTNET_ENVIRONMENT=Production

[Install]
WantedBy=multi-user.target
```

---

## 5.9 Déploiement conteneur (aperçu)

### Dockerfile (FDD)

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:9.0 AS runtime
WORKDIR /app
COPY ./publish/ ./
ENTRYPOINT ["dotnet", "MyWebApp.dll"]
```

### Construire

```bash
dotnet publish -c Release -o publish

docker build -t mywebapp:1.0.0 .
```

> Pour du SCD en conteneur, vous pouvez partir d’une image `runtime-deps` et copier l’exécutable self-contained.

---

# Ateliers / exercices

## Exercice 1 — Créer un package NuGet propre

1. Créez une lib `Company.Math`.
2. Ajoutez metadata NuGet, README, license.
3. Multi-target : `netstandard2.0;net9.0`.
4. Générez un `.nupkg` et inspectez son contenu.

Critères :
- `PackageReadmeFile` visible
- Dépendances correctes par TFM

---

## Exercice 2 — Versioning maîtrisé

1. Fixez `AssemblyVersion` à `1.0.0.0`.
2. Publiez `Version=1.2.0` puis `1.2.1`.
3. Vérifiez dans les assemblies : FileVersion/Informational.

---

## Exercice 3 — Strong-name

1. Générez une clé `.snk`.
2. Signez `Company.Math`.
3. Vérifiez la signature.
4. Discutez : quels impacts pour les consumers ?

---

## Exercice 4 — Publier en self-contained + single-file

1. Créez une console app.
2. Publiez en `win-x64` SCD + single file.
3. Comparez tailles/temps de démarrage.

---

## Exercice 5 — Native AOT

1. Activez `PublishAot`.
2. Publiez en `linux-x64` ou `win-x64`.
3. Identifiez un exemple cassé par reflection et corrigez (source generation / annotations).

---

# Annexes

## A) Commandes CLI utiles

```bash
# restore/build/test
dotnet restore
dotnet build -c Release
dotnet test -c Release

# pack
dotnet pack -c Release

# publish
dotnet publish -c Release
```

## B) Propriétés MSBuild fréquentes

```text
Version
PackageVersion
AssemblyVersion
FileVersion
InformationalVersion
PublishSingleFile
PublishTrimmed
PublishReadyToRun
PublishAot
RuntimeIdentifier
SelfContained
```

## C) Recommandations “production-ready”

- Mettre en place une CI qui : build/test/pack/publish
- Signer (nuget + assemblies) si requis (scenario enterprise)
- Générer SBOM si nécessaire (exigences sécurité)
- Documenter les options de publish pour chaque cible (Windows/Linux)

---

## Fin

Cette session fournit une base solide pour industrialiser la distribution (NuGet) et la livraison (publish/deploy) de composants et applications .NET.
