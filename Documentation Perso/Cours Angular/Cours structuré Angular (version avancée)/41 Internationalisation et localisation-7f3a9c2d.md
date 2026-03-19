# Formation Angular — Internationalisation (i18n) & Localisation (l10n)

**Référence cours :** 41  
**Public :** développeurs Angular (niveau intermédiaire) et équipes produit/QA  
**Prérequis :** TypeScript, Angular CLI, notions de pipes, modules/standalone, tests  
**Durée suggérée :** 1 jour (7h) ou 2 demi-journées  

---

## Objectifs pédagogiques
À l’issue de la formation, vous saurez :

- Expliquer la différence entre **internationalisation (i18n)** et **localisation (l10n)**.
- Mettre en place l’i18n **Angular natif** (extraction, XLIFF, build multi-locales).
- Gérer formats **dates/nombres/devises** via `LOCALE_ID`, ICU et pipes.
- Définir une **stratégie projet** (workflow trad, conventions, maintenance, QA).
- Comparer et choisir entre **i18n natif** et une solution **runtime** (ex. Transloco/ngx-translate).
- Anticiper les pièges : pluralisation, RTL, routing, SEO, performance, tests.

---

## Plan de la formation

1. **Fondamentaux i18n/l10n**
   - Concepts, périmètre, contraintes produit
   - Notions : locale, formatage, ICU, RTL

2. **Stratégie projet et choix d’architecture**
   - Build-time vs runtime
   - Organisation des contenus traduisibles
   - Workflow traduction / validation / QA

3. **Angular i18n natif : marquage des messages**
   - Attributs `i18n`, `i18n-*`, descriptions, meaning
   - ICU : pluriels, sélections (genre, statut)

4. **Extraction et fichiers de traduction (XLIFF)**
   - `ng extract-i18n`
   - XLIFF 1.2/2.0, IDs, contextualisation
   - Intégration avec TMS (Crowdin, Lokalise…)

5. **Compilation multi-locales, déploiement & routing**
   - `angular.json`, `localize`, configurations
   - Bases href, stratégies d’URL
   - Impacts sur SEO et cache CDN

6. **Localisation des formats : dates, nombres, devises**
   - `LOCALE_ID`, `registerLocaleData`
   - Pipes, `Intl`, unités

7. **Traduction dynamique (runtime) : alternatives & patterns**
   - Présentation (Transloco / ngx-translate)
   - Lazy loading des langues
   - Interpolation, pluralisation, tests

8. **Qualité, tests et maintenance**
   - Tests unitaires/e2e multi-langues
   - Régression visuelle, pseudo-localisation
   - Gouvernance des clés et nettoyage

9. **Atelier fil rouge**
   - Mise en place complète sur une mini-app
   - Recette i18n : extraction → traduction → build → QA

---

## 1) Fondamentaux i18n / l10n

### Définitions
- **Internationalisation (i18n)** : rendre l’application capable de supporter **plusieurs langues/régions** (structure, extraction de messages, mécanismes de substitution).
- **Localisation (l10n)** : adapter concrètement pour une locale donnée :
  - Traduction des textes
  - Formats de **date/heure**, **nombre**, **devise**
  - Unités, tri, séparateurs décimaux
  - Sens de lecture (LTR/RTL)

### Notion de locale
Une **locale** combine langue + région (et parfois variante). Exemples :
- `fr` (français générique)
- `fr-FR` (France)
- `fr-CA` (Canada)
- `en-US`, `en-GB`

### Risques fonctionnels typiques
- Chaînes concaténées (ordre des mots différent selon la langue)
- Pluralisation mal gérée (0, 1, n)
- Placeholders non cohérents (ex : "{count}" vs "{total}")
- UI qui casse (textes plus longs, boutons, tableaux)
- Incohérence de formats (ex : `1,234.56` vs `1 234,56`)

---

## 2) Stratégie projet et choix d’architecture

### 2.1 Build-time (Angular i18n natif) vs Runtime (bibliothèques)

#### Angular i18n natif (build-time)
**Principe :** les traductions sont **appliquées à la compilation**. Une version buildée par locale (ou génération multi-locales).

**Avantages**
- Très performant côté client
- SEO facilité avec URLs par langue (si vous servez des bundles distincts)
- Support solide d’ICU

**Inconvénients**
- Rebuild/redeploy à chaque mise à jour de traduction
- Moins flexible si vous voulez changer de langue à chaud sans reload

#### Runtime (Transloco, ngx-translate…)
**Principe :** les traductions sont chargées à l’exécution (JSON, API…)

**Avantages**
- Changement de langue à chaud
- Traductions mises à jour sans rebuild (si chargées via API)

**Inconvénients**
- Surcoût runtime + gestion du chargement
- SEO plus complexe (selon SSR / approche)

### 2.2 Recommandations de stratégie
- **Produit vitrine/SEO** et pages publiques : privilégier **build-time** + routing par locale.
- **Application métier interne** avec besoin de switch instantané : runtime souvent plus simple.
- Équipe trad + TMS : prévoir **extraction** automatisée et validation.

### 2.3 Bonnes pratiques de gouvernance
- Décider tôt :
  - locales supportées
  - stratégie d’URL (`/fr/`, sous-domaine, query param)
  - format des fichiers (XLIFF vs JSON)
- Définir une convention :
  - descriptions systématiques
  - variables/patterns interdits (concaténation)
  - règles de reviews (PR i18n)

---

## 3) Angular i18n natif : marquage des messages

> Angular propose `@angular/localize` et un marquage dans les templates via `i18n`.

### 3.1 Préparation
Selon version Angular, vous aurez souvent :

```bash
ng add @angular/localize
```

Cela ajoute la dépendance et prépare le projet à l’i18n.

### 3.2 Marquer un texte simple

```html
<h1 i18n>Bienvenue dans l’application</h1>
```

Angular extraira le message et permettra de le traduire.

### 3.3 Ajouter une description et un meaning
Utile pour les traducteurs et pour distinguer des chaînes identiques ayant des sens différents.

```html
<!-- meaning|description -->
<button i18n="action|Bouton principal pour valider le formulaire">Valider</button>
```

### 3.4 Traduire un attribut

```html
<input
  i18n-placeholder="input|Placeholder du champ email"
  placeholder="Votre email"
/>
```

Ou plusieurs attributs : `i18n-title`, `i18n-aria-label`, etc.

### 3.5 Gérer du texte avec interpolation

```html
<p i18n="profile|Message d’accueil utilisateur">Bonjour {{ userName }} !</p>
```

**Règle :** éviter la concaténation dans le TS. Préférer un message complet dans le template.

---

## 4) ICU : pluriels et sélections

ICU est essentiel pour une traduction « correcte ».

### 4.1 Pluralisation

```html
<p i18n>
  {itemCount, plural,
    =0 {Aucun article}
    =1 {1 article}
    other {{{ itemCount }} articles}
  }
</p>
```

### 4.2 Sélection (ex : statut)

```html
<p i18n>
  {status, select,
    draft {Brouillon}
    published {Publié}
    archived {Archivé}
    other {Statut inconnu}
  }
</p>
```

### 4.3 Sélection + plural (cas fréquent)

```html
<p i18n>
  {gender, select,
    male {{photoCount, plural, =0 {Il n’a aucune photo} =1 {Il a 1 photo} other {Il a # photos}}}
    female {{photoCount, plural, =0 {Elle n’a aucune photo} =1 {Elle a 1 photo} other {Elle a # photos}}}
    other {{photoCount, plural, =0 {Aucune photo} =1 {1 photo} other {# photos}}}
  }
</p>
```

---

## 5) Extraction & fichiers de traduction (XLIFF)

### 5.1 Extraction des messages

```bash
ng extract-i18n --format=xlf --output-path=src/locale
```

Vous obtiendrez un fichier source (souvent `messages.xlf`).

### 5.2 Comprendre XLIFF (simplifié)
Dans XLIFF, chaque unité a une source et une cible :

```xml
<trans-unit id="..." datatype="html">
  <source>Bienvenue dans l’application</source>
  <target>Welcome to the application</target>
</trans-unit>
```

### 5.3 Bonnes pratiques pour l’extraction
- Toujours fournir **description** et **meaning** si ambigu
- Éviter de modifier la structure de messages pour limiter les diffs
- Ne pas éditer manuellement `id` sauf stratégie maîtrisée

### 5.4 Intégration TMS (Translation Management System)
Workflow recommandé :
1. CI : `ng extract-i18n`
2. Upload `messages.xlf` vers TMS
3. Traduction/validation
4. Download des `messages.<locale>.xlf`
5. PR automatique ou commit dans un dépôt dédié

---

## 6) Compilation multi-locales, build & déploiement

### 6.1 Déclarer les locales (approche conceptuelle)
Dans `angular.json`, on configure `i18n` et les fichiers de traduction par locale.

Exemple (structure indicative) :

```json
{
  "i18n": {
    "sourceLocale": "fr",
    "locales": {
      "en": "src/locale/messages.en.xlf",
      "de": "src/locale/messages.de.xlf"
    }
  }
}
```

Selon versions Angular, la configuration exacte peut se trouver dans la section build du projet.

### 6.2 Construire toutes les locales

```bash
ng build --localize
```

Résultat : plusieurs répertoires de sortie, un par locale.

### 6.3 Déploiement
Approches :
- Déployer `dist/<app>/fr/`, `dist/<app>/en/` sur un même host
- Ou serveurs/sous-domaines distincts

Points d’attention :
- `baseHref` par locale
- redirections serveur (ex : `/` → `/fr/` selon `Accept-Language`)
- cache CDN (varier selon chemin)

### 6.4 Routing par locale
Options courantes :
- Prefix path : `/fr/...`, `/en/...` (le plus simple)
- Sous-domaines : `fr.example.com`
- Paramètre `?lang=fr` (moins SEO, pratique interne)

Pour build-time i18n, le prefix path est souvent naturel, car on sert des bundles par locale.

---

## 7) Localisation des formats : dates, nombres, devises

### 7.1 `LOCALE_ID` et locale data
Angular formate via les **locale data**. Pour certaines locales, il faut les enregistrer.

```ts
import { LOCALE_ID } from '@angular/core';
import localeFr from '@angular/common/locales/fr';
import { registerLocaleData } from '@angular/common';

registerLocaleData(localeFr);

bootstrapApplication(AppComponent, {
  providers: [{ provide: LOCALE_ID, useValue: 'fr-FR' }]
});
```

> En build-time i18n, Angular gère généralement le `LOCALE_ID` par locale compilée.

### 7.2 Pipes de base

```html
<p>Date : {{ today | date:'longDate' }}</p>
<p>Nombre : {{ amount | number:'1.2-2' }}</p>
<p>Devise : {{ price | currency:'EUR':'symbol':'1.2-2' }}</p>
```

Le rendu dépend de la locale (virgules, espaces, ordre jour/mois…).

### 7.3 Formatage via `Intl`
Pour des besoins avancés (unités, listes…), `Intl` est utile :

```ts
const formatted = new Intl.NumberFormat('de-DE', {
  style: 'currency', currency: 'EUR'
}).format(1234.56);
```

### 7.4 Attention au fuseau horaire
- Décider : affichage en **local user timezone** ou timezone métier.
- Angular `date` pipe accepte un paramètre timezone.

---

## 8) Traduction dynamique (runtime) : alternatives & patterns

Même si le focus est Angular natif, il est important de connaître les alternatives.

### 8.1 Quand préférer une solution runtime
- App avec **switch de langue sans rechargement**
- Traductions pilotées par back-office
- Micro-frontends où chaque MF gère ses ressources

### 8.2 Patterns recommandés (runtime)
- Lazy-load des fichiers par langue (réduire le bundle initial)
- Mettre en cache les traductions
- Centraliser une abstraction `I18nService` (découpler du vendor)

### 8.3 Pièges runtime
- Flash de contenu non traduit au chargement
- Interpolations non typées
- Pluralisation incomplète si la lib ne gère pas ICU complet

---

## 9) Qualité : tests, QA et maintenance

### 9.1 Pseudo-localisation
Technique de QA : remplacer les chaînes par une version allongée et « bruitée » pour détecter :
- débordements
- chaînes non traduites

Exemple :
- `Valider` → `[[ Vâlîdér !!! ]]`

### 9.2 Checklist QA i18n
- Toutes les pages sont traduites, y compris erreurs/empty states
- Formats date/nombre/devise conformes
- ICU correct (0/1/n)
- RTL si langue supportée (ar, he) :
  - direction CSS
  - icônes/chevrons inversés si nécessaire

### 9.3 Tests unitaires
- Tests de rendu par locale (snapshot/DOM assertions)
- Vérifier les pipes selon `LOCALE_ID`

### 9.4 Tests e2e
- Exécuter un subset e2e par langue
- Valider navigation/routing selon URL de locale

### 9.5 Maintenance continue
- Nettoyage des messages orphelins (non utilisés)
- Politique de versionnement des fichiers trad
- Automatiser l’extraction et l’import via CI

---

## Atelier fil rouge (recommandé)

### Énoncé
Vous disposez d’une mini-app "Product Catalog" :
- Accueil avec un titre, un CTA
- Liste de produits avec prix
- Détail produit avec stock (pluralisation)

Objectif : supporter `fr-FR` (source), `en-US` et `de-DE`.

### Étapes
1. Installer/activer `@angular/localize`
2. Marquer les messages dans templates (`i18n`, `i18n-attr`)
3. Ajouter un message ICU pour stock (0/1/n)
4. Extraire les messages (`ng extract-i18n`)
5. Produire `messages.en.xlf` et `messages.de.xlf`
6. Configurer `angular.json` pour locales
7. Builder en multi-locales (`ng build --localize`)
8. Vérifier les formats (currency/date) par locale

### Critères de réussite
- Trois builds fonctionnels
- Aucun texte resté en langue source dans les builds `en` et `de`
- Prices affichés avec le bon séparateur et symbole

---

## Annexes

### A) Règles d’or i18n
- Ne pas concaténer des morceaux de phrases
- Toujours contextualiser les messages ambigus
- Utiliser ICU pour pluriels/sélections
- Tester avec une langue « longue » (de) et une pseudo-locale

### B) Glossaire
- **i18n** : internationalisation
- **l10n** : localisation
- **ICU** : syntaxe de messages (plural/select)
- **XLIFF** : format standard d’échange de traductions
- **TMS** : outil de gestion de traduction (Crowdin, Lokalise…)

### C) Livrables fournis aux participants
- Checklists i18n
- Exemples de messages ICU
- Modèle de workflow CI pour extraction/import
