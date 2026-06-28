# 16) Repères versions & stratégie (Java)

> Public cible : développeurs/tech leads/architectes et équipes DevOps qui doivent **choisir une version de Java**, comprendre le **cadencement des sorties**, et définir une **stratégie de migration** (notamment en production).

---

## Objectifs pédagogiques

À l’issue de cette formation, vous saurez :

- Expliquer le **cycle de release Java** (features vs LTS) et le notionnel de support.
- Identifier les **versions LTS Oracle** pertinentes aujourd’hui et à horizon 2027.
- Définir une **stratégie de versionning en production** : stabilité, sécurité, cadence.
- Mettre en place un **plan de migration régulier** (outillage, tests, CI/CD, rollback).
- Documenter une **politique d’entreprise** : JDK, distribution, maintenance, conformité.

---

## Pré-requis

- Connaissances de base Java (ou au moins écosystème JVM) et pratique de la livraison en production.
- Notions de CI/CD appréciées.

---

## Durée & format

- **Durée recommandée : 1h à 2h** (selon la profondeur sur les ateliers).
- Format : exposé + discussion + mini-ateliers (checklists et plan de migration).

---

## Plan (sommaire)

1. **Pourquoi “repères versions” ?** (risque, sécurité, compatibilité)
2. **Cadence de release Java** : releases semestrielles, LTS
3. **Repères LTS (Oracle)** : 8, 11, 17, 21, 25 ; prochaine : 29 (sept. 2027)
4. **Stratégie de production** : viser LTS, planifier des migrations régulières
5. **Choisir une version cible** : critères (stabilité, perfs, conformité, librairies)
6. **Plan de migration** : démarche, outillage, tests, déploiement progressif
7. **Gouvernance** : politique interne, calendrier, responsabilités
8. **Synthèse & checklists**

---

# 1. Pourquoi “repères versions” ?

Java évolue rapidement : nouvelles APIs, améliorations de performance, changements de comportement (GC, modules, TLS/certificats, etc.). Ne pas maîtriser les versions entraîne :

- **Risque sécurité** : vulnérabilités corrigées uniquement dans les versions supportées.
- **Risque conformité** : exigences SOC2/ISO, audits, SBOM, “supported software only”.
- **Risque opérationnel** : dépendances (frameworks, drivers) qui cessent de supporter des versions anciennes.
- **Coût de migration** : plus on attend, plus le saut est grand (outillage, code, pipelines).

> Repère clé : une stratégie version permet de **réduire le coût total** (migrations plus petites, régulières) et d’augmenter la **résilience** (patching sécurité, compatibilité).

---

# 2. Cadence de release Java : feature releases vs LTS

## 2.1 Feature releases

Depuis Java 9, l’écosystème suit un rythme **environ semestriel**.

- Les versions “feature” livrent des nouveautés.
- Elles ont souvent une durée de support plus courte (selon la distribution).
- Utile pour :
  - tester tôt des fonctionnalités
  - anticiper les futurs changements
  - accélérer l’innovation sur des produits internes

## 2.2 LTS (Long-Term Support)

Les versions **LTS** sont celles autour desquelles les organisations construisent souvent leur “socle” :

- **Support et correctifs sur la durée** (selon l’éditeur/distribution et le contrat).
- Plus d’adoption par les frameworks/outils (Spring, Tomcat, Gradle, etc.).
- Meilleur candidat pour la **production**.

> Point important : “LTS” ne veut pas dire “sans migration”. Cela signifie plutôt : **fenêtre plus confortable** et **stabilité accrue**.

---

# 3. Repères LTS côté Oracle

## 3.1 LTS Oracle : liste actuelle

Référentiel demandé (Oracle) :

- **Java 8** — LTS
- **Java 11** — LTS
- **Java 17** — LTS
- **Java 21** — LTS
- **Java 25** — LTS

**Prochaine LTS planifiée :**

- **Java 29** — planifiée **septembre 2027**

## 3.2 Ce que cela implique

- En production, le standard “entreprise” est typiquement : **rester sur une LTS**.
- L’organisation planifie un saut régulier, généralement :
  - **LTS → LTS** (ex : 17 → 21 → 25 …)
  - ou LTS → LTS en sautant une LTS (moins conseillé, mais possible) selon contraintes.

## 3.3 Visualisation (repères)

```text
LTS Oracle : 8 ---- 11 ---- 17 ---- 21 ---- 25 ---- 29
                          ^            ^      ^      ^
                        (prod)       (prod) (prod) (cible)

Prochaine LTS : 29 (sept. 2027)
```

> Remarque : les dates exactes de fin de support et modalités varient selon l’éditeur et les contrats, mais la **roadmap Oracle** constitue un repère de planification.

---

# 4. Stratégie de production : viser LTS, planifier des migrations régulières

## 4.1 Principe directeur

- **Prod = stabilité + support sécurité** → viser une **version LTS**.
- **Migrations régulières** → éviter les “big bang” (coûteux et risqués).

## 4.2 Stratégie recommandée (standard entreprise)

### Option A — “LTS-to-LTS” en continu (recommandée)

- Cibler une LTS pour la prod.
- Préparer la migration vers la suivante **dès la publication** (ou après un délai de stabilisation interne).
- Exemple :
  - socle actuel Java 17
  - planification Java 21
  - puis Java 25

**Avantages**
- Sauts plus petits (moins de ruptures)
- Continuous compliance
- Compatibilité plus simple avec frameworks

**Inconvénients**
- Demande une discipline : budget, backlog, créneaux release

### Option B — “Sauter une LTS” (déconseillée sauf contraintes)

- Exemple : rester en 17, sauter 21, aller en 25.

**Avantages**
- Moins de migrations (en apparence)

**Inconvénients**
- Migration plus risquée et plus longue
- Plus de “breaking changes” à absorber d’un coup
- Dépendances possiblement incompatibles

## 4.3 Politique de “stabilité”

- **Version JDK figée** par application (pinning) : ex. `temurin-21.0.x`.
- **Mises à jour patch** régulières (CPU/PSU) via pipelines.
- Environnements (dev/recette/prod) alignés pour éviter l’effet “works on my machine”.

---

# 5. Choisir une version cible : critères de décision

## 5.1 Critères techniques

- **Compatibilité frameworks** : Spring Boot, Quarkus, Micronaut, Jakarta EE, etc.
- **Compatibilité outillage** : Maven/Gradle, plugins, agents APM, testcontainers, etc.
- **Runtime features** : GC (G1/ZGC), performance, footprint, etc.
- **Build toolchain** : version du compilateur, `--release`, target bytecode.

## 5.2 Critères sécurité & conformité

- **Support éditeur/distribution** (SLA, patching)
- **Cadence de patch** et process de CVE
- Exigences audit (logiciel supporté, SBOM, signature artifacts)

## 5.3 Critères organisationnels

- Capacité à :
  - tester (automatisation)
  - déployer (progressif)
  - monitorer (observabilité)
- Niveau de maturité DevOps
- Taille du parc applicatif (mono vs multi repos / microservices)

---

# 6. Plan de migration (approche pratico-pratique)

Objectif : migrer de manière reproductible, observable et réversible.

## 6.1 Démarche en 7 étapes

1. **Inventaire**
   - Version Java actuelle (runtime et compilation)
   - dépendances clés (framework, drivers, libs natives)
   - modes d’exécution (container, VM, serverless)

2. **Choix de la cible**
   - typiquement la **dernière LTS** compatible avec l’écosystème

3. **Mise à niveau toolchain**
   - CI : images JDK, caches, scanners, build agents
   - Version Maven/Gradle/Plugins
   - Paramétrage compilation : `--release` (recommandé)

4. **Montée de version en “branch d’upgrade”**
   - PR dédiée, itérations courtes
   - corrections de compilation + tests

5. **Validation**
   - **tests unitaires** + **tests d’intégration** + tests end-to-end
   - tests perf (si critique)
   - validation sécurité (SCA, SBOM)

6. **Déploiement progressif**
   - canary / blue-green selon dispo
   - rollback prêt (image précédente)

7. **Stabilisation & industrialisation**
   - documentation, standardisation
   - planifier la migration suivante (calendrier)

## 6.2 Checklists de migration (actionnables)

### Checklist “Build & CI”

- [ ] La CI dispose des JDK nécessaires (actuel + cible)
- [ ] Les builds sont reproductibles (lockfiles si applicables)
- [ ] Le paramètre `--release` est utilisé (évite des surprises de compat)
- [ ] Les tests d’intégration tournent sur la version cible

### Checklist “Runtime/Prod”

- [ ] Image Docker / base OS validée avec le JDK cible
- [ ] Observabilité : logs, metrics, tracing, APM compat.
- [ ] Paramètres JVM revus (mémoire, GC, options obsolètes)
- [ ] Plan de rollback documenté et testé

### Checklist “Dépendances”

- [ ] Framework principal compatible (ex. Spring Boot)
- [ ] Drivers critiques (JDBC, Kafka, etc.) validés
- [ ] Libs natives (compression, crypto, netty native…) testées

---

# 7. Gouvernance : définir une politique interne

## 7.1 Politique de versions (exemple)

- Production : **uniquement LTS**
- Maintenance :
  - patching mensuel/trim. (selon politique sécurité)
  - interdiction des versions non supportées
- Calendrier :
  - migration vers une nouvelle LTS **dans les X mois** suivant son adoption par l’écosystème interne

## 7.2 Rôles

- **Architecture/Platform** : définit la version socle & templates.
- **Équipes produit** : appliquent la migration dans leur cadence.
- **Sécurité** : exigences de patching, exceptions, audits.
- **DevOps** : images, agents, pipelines, observabilité.

## 7.3 Communication & pilotage

- Tableau de bord : applications par version Java
- Indicateurs :
  - % du parc sur la LTS cible
  - nombre d’exceptions
  - délai moyen de patch

---

# 8. Mini-ateliers (optionnels)

## Atelier 1 — Définir votre stratégie LTS

1. Version actuelle prod : ____
2. LTS cible : ____
3. Date cible de migration : ____
4. Risques principaux : ____
5. Plan de mitigation (tests/outillage) : ____

## Atelier 2 — Plan de migration par lots

- Lot 1 : apps low-risk (internal tools)
- Lot 2 : apps business standard
- Lot 3 : apps critiques (SLA fort)

Objectif : apprendre sur les lots 1/2 avant les lots 3.

---

# Synthèse

- Les **repères LTS Oracle** : **8, 11, 17, 21, 25** ; prochaine **29 (sept. 2027)**.
- Stratégie recommandée : **Production sur LTS** + **migrations régulières** (LTS-to-LTS).
- Industrialiser : inventaire, upgrade CI/toolchain, tests, déploiement progressif, gouvernance.

---

# Sources

- **Oracle Java SE Support Roadmap**
- **Oracle Downloads** (Java)
