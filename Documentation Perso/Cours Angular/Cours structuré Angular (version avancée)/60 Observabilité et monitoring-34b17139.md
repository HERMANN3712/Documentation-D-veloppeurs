# Formation Angular — Observabilité et monitoring (Frontend)

**Référence :** 60  
**Public :** développeurs Angular (débutant++ à confirmé), leads, formateurs  
**Pré‑requis :** TypeScript, Angular, notions HTTP, RxJS ; bases de Git/CI/CD appréciées  
**Durée indicative :** 1 à 2 jours (adaptable)  
**Objectif général :** être capable d’instrumenter, mesurer, diagnostiquer et améliorer une application Angular en production grâce à l’observabilité (logs/metrics/traces) et au monitoring (alerting, dashboards, SLO).

---

## Table des matières
1. [Introduction : pourquoi l’observabilité en frontend ?](#1-introduction--pourquoi-lobservabilité-en-frontend)
2. [Concepts clés : monitoring vs observabilité](#2-concepts-clés--monitoring-vs-observabilité)
3. [Ce qu’on doit suivre côté frontend](#3-ce-quon-doit-suivre-côté-frontend)
4. [Instrumentation : logs, événements, erreurs, performance](#4-instrumentation--logs-événements-erreurs-performance)
5. [Angular : capture d’erreurs et gestion globale](#5-angular--capture-derreurs-et-gestion-globale)
6. [Web Performance (RUM) : métriques, Web Vitals et timings](#6-web-performance-rum--métriques-web-vitals-et-timings)
7. [APM, tracing et corrélation frontend ↔ backend](#7-apm-tracing-et-corrélation-frontend--backend)
8. [Dashboards, alertes, SLO et bonnes pratiques opérationnelles](#8-dashboards-alertes-slo-et-bonnes-pratiques-opérationnelles)
9. [Outils du marché et critères de choix](#9-outils-du-marché-et-critères-de-choix)
10. [Ateliers pratiques (TP)](#10-ateliers-pratiques-tp)
11. [Checklist de mise en production](#11-checklist-de-mise-en-production)
12. [Annexes : snippets Angular prêts à l’emploi](#12-annexes--snippets-angular-prêts-à-lemploi)

---

## 1) Introduction : pourquoi l’observabilité en frontend ?

### 1.1 Le contexte
Une application **frontend** (Angular) est exécutée sur des appareils, réseaux et navigateurs variés. Les incidents de production peuvent venir :
- d’un **bug** (erreur JavaScript/TypeScript, template, runtime),
- d’un souci de **performance** (chargement trop long, UI jank),
- d’une **API** instable (timeouts, erreurs 5xx, latence),
- d’un problème de **navigation** (route qui boucle, guard mal géré),
- d’un événement métier inattendu (abandon panier, paiement KO, taux d’échec).

### 1.2 Objectifs du monitoring frontend
Le monitoring d’une application frontend consiste à suivre :
- **erreurs** (JS errors, unhandled promise rejection, zones Angular),
- **performances** (temps de chargement, LCP/CLS/INP, ressources),
- **navigation** (parcours, pages les plus consultées, routes),
- **taux d’échec** (API calls, formulaires, actions critiques),
- **temps de chargement et disponibilité perçue**, 
- **événements métier** (conversion, recherche, ajout panier, validation).

Grâce à des outils de **logging** et d’**APM** (Application Performance Monitoring), on diagnostique mieux les incidents, on réduit le MTTR (Mean Time To Recovery) et on améliore l’expérience utilisateur.

---

## 2) Concepts clés : monitoring vs observabilité

### 2.1 Définitions
- **Monitoring** : on surveille des signaux connus avec des seuils/alertes (ex. taux d’erreur > 2% sur 5 minutes). Répond plutôt à : *« est‑ce que ça va ? »*
- **Observabilité** : capacité à comprendre un système via ses sorties (logs, métriques, traces). Répond à : *« pourquoi ça ne va pas ? »*

### 2.2 Les 3 piliers (adaptés au frontend)
1. **Logs** : événements textuels/structurés (info, warn, error) — côté navigateur.
2. **Metrics** : séries temporelles (LCP moyen, taux d’erreurs, p95 latence API, CLS). 
3. **Traces** : suivi d’un « parcours » ou d’une transaction, corrélation d’appels (navigation → API → rendu).

> En frontend, on parle souvent de **RUM** (Real User Monitoring) et de **session replay** (selon RGPD/consentement) comme outils complémentaires.

### 2.3 Anti‑patterns fréquents
- Logger sans structure (messages incomplets, pas de contexte).
- Trop de logs (bruit, coûts élevés, difficulté d’analyse).
- Pas de corrélation (impossible de relier une erreur à un parcours utilisateur ou à un backend).
- Pas de gouvernance (RGPD, données sensibles, rétention).

---

## 3) Ce qu’on doit suivre côté frontend

### 3.1 Erreurs
**Sources** :
- Exceptions JS (runtime)
- Erreurs Angular (change detection, templates)
- Erreurs réseau (HTTP)
- Promesses non gérées

**Dimensions utiles** :
- route/page
- version applicative
- navigateur / OS / device
- locale / pays (attention aux données perso)
- feature flag activé
- corrélation avec un `traceId` backend

### 3.2 Performances
- **Navigation timing** (TTFB, DOMContentLoaded, load)
- **Resource timing** (assets lourds)
- **Web Vitals** : LCP, CLS, INP (ou FID selon ancien modèle)
- **Angular / JS** : temps d’exécution, chunking, lazy loading
- **API** : latence p50/p95/p99, erreurs, retries

### 3.3 Navigation et parcours
- pages vues, routes, temps passé
- entonnoirs (funnel) : ex. inscription → activation → 1re action
- taux de rebond / sorties

### 3.4 Événements métier (business observability)
Exemples :
- `signup_started`, `signup_success`, `signup_failed`
- `checkout_start`, `payment_failed`, `order_confirmed`
- `search_performed`, `no_results`

**Bonnes pratiques** :
- nommage cohérent (snake_case / dot.notation)
- schéma événementiel versionné
- propriétés utiles mais minimales

---

## 4) Instrumentation : logs, événements, erreurs, performance

### 4.1 Logging : philosophie et structure
On préfère des logs **structurés** (JSON) avec :
- `level` (debug/info/warn/error)
- `message`
- `timestamp`
- `context` (route, userAgent, appVersion)
- `correlationId` / `traceId`
- `tags` (feature, module)

Exemple de log structuré :
```json
{
  "level": "error",
  "message": "HTTP 500 on /api/orders",
  "route": "/checkout",
  "appVersion": "1.12.3",
  "traceId": "4f0c...",
  "http": {"method":"POST","status":500,"durationMs": 820}
}
```

### 4.2 Gestion des données sensibles (RGPD)
- Ne pas logger : email, téléphone, adresse, tokens, numéro de carte.
- Masquer/anonymiser : identifiants, IDs.
- Respecter le consentement (cookies/trackers) si applicable.
- Définir la rétention (7/30/90 jours) selon besoin.

### 4.3 Échantillonnage (sampling)
- Erreurs : souvent **100%** (à discuter selon volume)
- Performance : échantillonnage (ex. 10% des sessions)
- Events métier : selon criticité (ex. 100% pour paiement, 10% pour pages vues)

### 4.4 Stratégie d’envoi (client → collecteur)
- Envoi asynchrone, batché
- Retry raisonnable (éviter boucles infinies)
- Backpressure : couper si trop d’erreurs
- Utiliser `navigator.sendBeacon` pour événements de fin de session si possible

---

## 5) Angular : capture d’erreurs et gestion globale

### 5.1 `ErrorHandler` global
Angular permet de centraliser les erreurs via `ErrorHandler`.

Objectifs :
- normaliser (format + contexte)
- filtrer (erreurs connues, bruit)
- envoyer vers l’outil (Sentry, Datadog, Elastic, Azure AppInsights, etc.)

### 5.2 Interception HTTP (`HttpInterceptor`)
Capturer :
- statut HTTP
- durée
- endpoint (sans query sensibles)
- corrélation (headers `traceparent`, `x-request-id`)

**Mesure de durée** : timestamp avant/après, ou via opérateurs RxJS.

### 5.3 Détection des erreurs hors Angular
- `window.onerror`
- `unhandledrejection`

> Important : certaines erreurs ne passent pas par `ErrorHandler`.

### 5.4 Logging applicatif vs console
- En dev : `console.*` acceptable.
- En prod : éviter d’inonder la console, préférer un logger configurable.

### 5.5 Router instrumentation
Instrumenter :
- `NavigationStart`, `NavigationEnd`, `NavigationError`, `NavigationCancel`
- temps de navigation
- route finale (après guards/redirect)

---

## 6) Web Performance (RUM) : métriques, Web Vitals et timings

### 6.1 Le modèle RUM
RUM = mesures **réelles** côté utilisateurs, par opposition à des tests synthétiques (Lighthouse CI).

Avantages :
- reflète la réalité (device low-end, 3G, extensions)
- permet de segmenter par navigateur/pays/route

Limites :
- bruit statistique
- contraintes RGPD

### 6.2 Web Vitals
- **LCP** (Largest Contentful Paint) : vitesse de chargement perçue
- **CLS** (Cumulative Layout Shift) : stabilité visuelle
- **INP** (Interaction to Next Paint) : réactivité globale

Actions typiques :
- LCP : optimiser images, server/TTFB, preloading, critical CSS
- CLS : réserver l’espace (width/height), éviter injections tardives
- INP : réduire long tasks, chunking, defer, web workers

### 6.3 Timings utiles (navigation/resources)
- `performance.getEntriesByType('navigation')`
- `performance.getEntriesByType('resource')`
- Long tasks via `PerformanceObserver`

### 6.4 Angular et performance
- Lazy loading des modules et standalone components
- Preloading strategy (sélective)
- `ChangeDetectionStrategy.OnPush`
- TrackBy sur `*ngFor`
- RxJS : éviter subscriptions non nettoyées

---

## 7) APM, tracing et corrélation frontend ↔ backend

### 7.1 Pourquoi corréler ?
Sans corrélation, on voit :
- côté frontend : *« /checkout lent »*
- côté backend : *« DB spikes »*

Avec corrélation, on relie :
- session/réquest frontend → trace backend → requête DB.

### 7.2 IDs de corrélation
- **W3C Trace Context** : header `traceparent`
- `x-request-id` (custom)
- `sessionId`/`deviceId` (anonymisé)

### 7.3 Stratégie concrète
1. Générer/récupérer un `traceId` côté frontend.
2. Le passer dans les headers HTTP.
3. Le propager côté backend (middleware).
4. L’exposer dans les logs backend + traces.

### 7.4 Tracing côté frontend
Selon l’outil :
- auto-instrumentation des routes & XHR/fetch
- spans custom (ex. « render product list », « finalize checkout »)

---

## 8) Dashboards, alertes, SLO et bonnes pratiques opérationnelles

### 8.1 Dashboards : ce qu’on affiche
- **Santé** : taux d’erreurs JS, erreurs HTTP, crash-free sessions
- **Perf** : LCP p75, INP p75, CLS p75 par route
- **API** : latence p95 par endpoint, erreurs 4xx/5xx
- **Navigation** : top routes, funnel checkout, abandon
- **Business** : conversion, paiement success rate

### 8.2 Alerting (éviter le bruit)
- Alertes sur **taux** et **tendances** (pas sur occurrences isolées)
- Multi‑fenêtres : ex. 5min + 1h (détection + confirmation)
- Seuils basés sur historique
- Routing vers les bonnes équipes

### 8.3 SLI/SLO (frontend)
Exemples de **SLI** :
- % sessions sans erreurs JS (crash-free)
- LCP p75 < 2.5s sur page Home
- INP p75 < 200ms sur pages clés
- % requêtes API < 1s (p95)

Exemple **SLO** :
- « 99.5% des sessions sur /checkout sont crash‑free sur 30 jours »

### 8.4 Release monitoring
- suivre `appVersion`
- comparer avant/après déploiement
- canary / progressive rollout

---

## 9) Outils du marché et critères de choix

### 9.1 Familles d’outils
- **Error tracking** : Sentry, Rollbar
- **APM/RUM** : Datadog RUM, New Relic, Dynatrace
- **Observabilité open-source** : OpenTelemetry + Grafana/Tempo/Loki/Prometheus
- **Logging/analytics** : Elastic, Splunk
- **Cloud** : Azure Application Insights, Google Cloud Operations

### 9.2 Critères
- Angular support (sourcemaps, stack traces)
- RUM + session replay (optionnel)
- coût (volume events, sampling)
- RGPD (hébergement EU, DPA)
- intégration CI/CD (release tags)
- corrélation backend (OTel, trace context)

---

## 10) Ateliers pratiques (TP)

> Ces TP sont décrits de manière **outil‑agnostique**. Vous pouvez les adapter à Sentry/Datadog/OpenTelemetry.

### TP1 — Mettre en place un logger structuré Angular
**Objectif** : centraliser `info/warn/error` et enrichir avec contexte.

Étapes :
1. Créer un `LoggingService`.
2. Ajouter `appVersion` (via `environment`).
3. Ajouter route courante (via `Router`).
4. Envoyer les logs vers un endpoint `/observability/logs` (mock).

**Critères de validation** :
- tous les logs ont un format stable
- le service peut être désactivé/configuré selon environnement

### TP2 — Capturer les erreurs globales
a) via `ErrorHandler` Angular  
b) via `window.onerror` et `unhandledrejection`

**Validation** : déclencher une erreur dans un composant et vérifier la remontée.

### TP3 — Mesurer les performances RUM (Web Vitals)
**Objectif** : collecter LCP/CLS/INP et les associer à une route.

**Validation** : afficher les valeurs dans la console + envoyer au collecteur.

### TP4 — Instrumenter HTTP
- mesurer `durationMs`
- tagger par endpoint (sanitisé)
- logguer erreurs `>=500`

**Validation** : tableau de statistiques p50/p95 (local) et remontée.

### TP5 — Router monitoring & funnel
- capturer `NavigationEnd`
- envoyer `page_view` avec `route` + `referrer`
- exécuter un mini‑funnel (ex. `/cart` → `/checkout` → `/confirmation`)

---

## 11) Checklist de mise en production

### Instrumentation
- [ ] `ErrorHandler` global actif
- [ ] capture `unhandledrejection` + `window.onerror`
- [ ] HTTP interceptor : latence + erreurs + corrélation
- [ ] Router events : page views + timings
- [ ] Web Vitals collectés (sampling configurable)

### Qualité des données
- [ ] événements et logs **structurés**
- [ ] version applicative taggée
- [ ] sourcemaps uploadées vers l’outil (si mis en place)
- [ ] PII scrub/masking

### Opérations
- [ ] dashboards prêts (perf/erreurs/API/business)
- [ ] alertes configurées, testées, non bruyantes
- [ ] SLO définis et revus régulièrement
- [ ] runbooks (quoi faire en cas d’alerte)

---

## 12) Annexes : snippets Angular prêts à l’emploi

> Les snippets ci‑dessous sont des bases génériques. Adaptez l’export (HTTP endpoint, SDK outil, OpenTelemetry) à votre contexte.

### 12.1 Modèles de données
```ts
export type LogLevel = 'debug' | 'info' | 'warn' | 'error';

export interface ObservabilityContext {
  appVersion: string;
  route?: string;
  userAgent?: string;
  traceId?: string;
  tags?: Record<string, string>;
}

export interface LogEvent {
  timestamp: string;
  level: LogLevel;
  message: string;
  context: ObservabilityContext;
  data?: unknown;
}
```

### 12.2 LoggingService
```ts
import { Injectable } from '@angular/core';
import { Router } from '@angular/router';
import { environment } from '../environments/environment';

@Injectable({ providedIn: 'root' })
export class LoggingService {
  constructor(private router: Router) {}

  private baseContext() {
    return {
      appVersion: environment.appVersion,
      route: this.router.url,
      userAgent: navigator.userAgent,
    };
  }

  log(level: 'debug'|'info'|'warn'|'error', message: string, data?: unknown) {
    const evt = {
      timestamp: new Date().toISOString(),
      level,
      message,
      context: this.baseContext(),
      data,
    };

    // Dev console
    if (!environment.production) {
      // eslint-disable-next-line no-console
      console[level === 'debug' ? 'log' : level](evt);
      return;
    }

    // TODO: envoyer vers votre collecteur/SDK
    // fetch('/observability/logs', { method: 'POST', body: JSON.stringify(evt) });
  }

  info(message: string, data?: unknown) { this.log('info', message, data); }
  warn(message: string, data?: unknown) { this.log('warn', message, data); }
  error(message: string, data?: unknown) { this.log('error', message, data); }
  debug(message: string, data?: unknown) { this.log('debug', message, data); }
}
```

### 12.3 GlobalErrorHandler
```ts
import { ErrorHandler, Injectable, NgZone } from '@angular/core';
import { LoggingService } from './logging.service';

@Injectable()
export class GlobalErrorHandler extends ErrorHandler {
  constructor(private logger: LoggingService, private zone: NgZone) {
    super();
  }

  override handleError(error: unknown): void {
    // Toujours appeler la logique par défaut (utile en dev)
    super.handleError(error);

    // Normalisation
    const errObj = this.normalizeError(error);

    // Sortir du cycle Angular si nécessaire (éviter effets de bord)
    this.zone.runOutsideAngular(() => {
      this.logger.error('Unhandled Angular error', errObj);
    });
  }

  private normalizeError(error: unknown) {
    if (error instanceof Error) {
      return {
        name: error.name,
        message: error.message,
        stack: error.stack,
      };
    }
    return { value: error };
  }
}
```

**Enregistrement** :
```ts
import { NgModule, ErrorHandler } from '@angular/core';
import { GlobalErrorHandler } from './observability/global-error-handler';

@NgModule({
  providers: [{ provide: ErrorHandler, useClass: GlobalErrorHandler }],
})
export class AppModule {}
```

### 12.4 Capture `unhandledrejection` et `window.onerror`
```ts
import { Injectable } from '@angular/core';
import { LoggingService } from './logging.service';

@Injectable({ providedIn: 'root' })
export class BrowserErrorCaptureService {
  private initialized = false;

  constructor(private logger: LoggingService) {}

  init() {
    if (this.initialized) return;
    this.initialized = true;

    window.addEventListener('error', (event) => {
      this.logger.error('window.error', {
        message: event.message,
        filename: event.filename,
        lineno: event.lineno,
        colno: event.colno,
        error: event.error ? { name: event.error.name, message: event.error.message, stack: event.error.stack } : undefined,
      });
    });

    window.addEventListener('unhandledrejection', (event) => {
      this.logger.error('unhandledrejection', {
        reason: event.reason instanceof Error
          ? { name: event.reason.name, message: event.reason.message, stack: event.reason.stack }
          : event.reason,
      });
    });
  }
}
```

App bootstrap (ex. `AppComponent`):
```ts
constructor(capture: BrowserErrorCaptureService) {
  capture.init();
}
```

### 12.5 HTTP Interceptor (latence + erreurs)
```ts
import { Injectable } from '@angular/core';
import { HttpEvent, HttpHandler, HttpInterceptor, HttpRequest, HttpErrorResponse } from '@angular/common/http';
import { Observable, catchError, finalize, throwError } from 'rxjs';
import { LoggingService } from './logging.service';

@Injectable()
export class ObservabilityHttpInterceptor implements HttpInterceptor {
  constructor(private logger: LoggingService) {}

  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    const start = performance.now();

    // Éviter de logger des query params sensibles
    const url = req.url.split('?')[0];

    return next.handle(req).pipe(
      catchError((err: unknown) => {
        if (err instanceof HttpErrorResponse) {
          this.logger.error('HTTP error', {
            method: req.method,
            url,
            status: err.status,
            statusText: err.statusText,
          });
        }
        return throwError(() => err);
      }),
      finalize(() => {
        const durationMs = Math.round(performance.now() - start);
        this.logger.info('HTTP request', {
          method: req.method,
          url,
          durationMs,
        });
      })
    );
  }
}
```

### 12.6 Router instrumentation (page_view + timing)
```ts
import { Injectable } from '@angular/core';
import { Router, NavigationStart, NavigationEnd, NavigationCancel, NavigationError } from '@angular/router';
import { filter } from 'rxjs';
import { LoggingService } from './logging.service';

@Injectable({ providedIn: 'root' })
export class RouterMonitoringService {
  private navStartTs = 0;

  constructor(private router: Router, private logger: LoggingService) {}

  init() {
    this.router.events.pipe(filter(e => e instanceof NavigationStart)).subscribe(() => {
      this.navStartTs = performance.now();
    });

    this.router.events.pipe(filter(e => e instanceof NavigationEnd)).subscribe((e) => {
      const durationMs = Math.round(performance.now() - this.navStartTs);
      const end = e as NavigationEnd;
      this.logger.info('page_view', {
        url: end.urlAfterRedirects,
        durationMs,
      });
    });

    this.router.events.pipe(filter(e => e instanceof NavigationCancel)).subscribe((e) => {
      this.logger.warn('navigation_cancel', { reason: (e as NavigationCancel).reason });
    });

    this.router.events.pipe(filter(e => e instanceof NavigationError)).subscribe((e) => {
      this.logger.error('navigation_error', { error: (e as NavigationError).error });
    });
  }
}
```

### 12.7 Web Vitals (base)
Installer (optionnel) : `npm i web-vitals`

```ts
import { Injectable } from '@angular/core';
import { onLCP, onCLS, onINP } from 'web-vitals';
import { LoggingService } from './logging.service';

@Injectable({ providedIn: 'root' })
export class WebVitalsService {
  constructor(private logger: LoggingService) {}

  init(sampleRate = 0.1) {
    if (Math.random() > sampleRate) return;

    onLCP((metric) => this.logger.info('web_vital_lcp', metric));
    onCLS((metric) => this.logger.info('web_vital_cls', metric));
    onINP((metric) => this.logger.info('web_vital_inp', metric));
  }
}
```

### 12.8 Conseils sourcemaps (Angular)
- Générer les sourcemaps (attention sécurité) :
  - soit upload vers l’outil (Sentry/Datadog/etc.)
  - soit conserver, mais protéger l’accès (selon politique)
- Tagguer `release`/`appVersion` pour décoder les stacks.

---

## Résultat attendu en fin de formation
À l’issue, le participant sait :
- définir une stratégie de monitoring (erreurs, performances, navigation, business)
- instrumenter Angular (ErrorHandler, Interceptor, Router)
- mesurer et interpréter Web Vitals
- corréler frontend et backend via trace/correlation IDs
- construire dashboards + alerting/SLO opérationnels

---

*Document prêt à être utilisé comme support de cours et base de TP. Adaptable selon l’outil retenu (Sentry/Datadog/OpenTelemetry/Grafana/Elastic).*