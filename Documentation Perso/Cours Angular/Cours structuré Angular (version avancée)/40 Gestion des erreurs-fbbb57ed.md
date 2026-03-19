# Formation Angular — Gestion des erreurs (avancée)

## Informations générales
- **Public visé** : Développeurs Angular (intermédiaire → avancé), formateurs, équipes produit.
- **Pré-requis** : TypeScript, RxJS (opérateurs de base), HttpClient, DI Angular.
- **Durée indicative** : 1 jour (6–7h) ou 2 × 3h30.
- **Objectifs pédagogiques**
  - Structurer une stratégie de gestion d’erreurs, du navigateur au serveur.
  - Distinguer **erreurs techniques** vs **erreurs métier** et les traiter différemment.
  - Centraliser la capture et la journalisation (logs) via un **ErrorHandler personnalisé**.
  - Gérer les erreurs **HTTP** (codes, retry, fallback, UX contrôlée).
  - Mettre en place une **remontée utilisateur** maîtrisée (toasts, pages d’erreur, formulaires).
  - Concevoir une chaîne de **logs centralisés** (console → backend → observabilité).

---

## Plan de la formation
1. **Panorama et principes**
   1. Pourquoi l’erreur est un “flux” à part entière
   2. Catégoriser : technique vs métier vs validation
   3. Où gérer quoi ? (component, service, interceptor, global)
2. **Erreurs HTTP : approche robuste**
   1. Modèle d’erreur backend (problem details, codes fonctionnels)
   2. `HttpInterceptor` pour normaliser, enrichir, tracer
   3. Stratégies RxJS : `catchError`, `retry`, `retryWhen`, `timeout`, `finalize`
3. **Gestion globale : ErrorHandler personnalisé**
   1. Limites de la gestion locale
   2. Implémentation d’un `GlobalErrorHandler`
   3. Intégration avec Angular (DI) et avec la zone (Zone.js)
4. **Logs centralisés et corrélation**
   1. Logger applicatif (niveaux, contextes)
   2. Correlation ID / Request ID
   3. Envoi serveur (batching, throttling)
5. **Remontée utilisateur maîtrisée**
   1. UX : messages utiles, pas d’exposition de données sensibles
   2. Erreurs globales (page “Oops”) vs locales (champ, toast)
   3. Internationalisation et messages mappés
6. **Erreurs métier : contrat et mapping**
   1. Codes métier, règles, conflits
   2. Traduction en UI (exemples concrets)
   3. Cas fréquents : 409, 422, 400 avec payload métier
7. **Atelier fil rouge**
   1. Mettre en place : interceptor + error handler + service de notification + logger
   2. Scénarios : 401/403, 404, 500, offline, timeouts, validation, erreur métier
8. **Checklist & bonnes pratiques**
   1. Observabilité, testabilité, sécurité
   2. Gouvernance des messages d’erreur

---

# 1) Panorama et principes

## 1.1 Pourquoi l’erreur est un “flux” à part entière
Dans une SPA Angular, les erreurs se propagent par plusieurs canaux :
- **Flux HTTP** (HttpClient/RxJS)
- **Erreurs synchrones** (exceptions JS/TS)
- **Erreurs asynchrones** (promises, timers, event handlers)
- **Erreurs template** (bindings)

Objectif : éviter que chaque composant réinvente la roue → **centraliser** et **standardiser**.

## 1.2 Catégoriser : technique vs métier vs validation
**Erreurs techniques**
- Ex. : 500, réseau, timeout, JSON invalide, bug front
- Traitement : logs + fallback UX + page d’erreur si nécessaire

**Erreurs métier**
- Ex. : “solde insuffisant”, “commande déjà validée”, “quota dépassé”
- Traitement : message clair, relié à l’action utilisateur, souvent **non logué** en erreur (niveau `warn` ou `info` selon cas)

**Validation (client/serveur)**
- Ex. : champ requis, format email, contrainte de longueur
- Traitement : feedback au niveau du formulaire, messages précis

## 1.3 Où gérer quoi ?
| Niveau | Rôle | Exemple |
|---|---|---|
| Composant | UX locale (afficher/masquer, état), petits fallback | afficher un message sous un champ |
| Service métier | Transformer erreurs en types métier, normaliser | mapping d’erreurs d’API |
| Interceptor HTTP | Centraliser auth, retry, timeouts, normalisation | traiter 401/403, ajouter headers |
| ErrorHandler global | Capturer exceptions non gérées, logs | crash inattendu UI |
| Backend/Observabilité | Stocker, corréler, alerter | Sentry/ELK/OpenTelemetry |

---

# 2) Erreurs HTTP : approche robuste

## 2.1 Modèle d’erreur backend recommandé
Idéalement, le backend renvoie un format stable. Exemple inspiré RFC7807 (*Problem Details*):

```json
{
  "type": "https://api.exemple.com/problems/business",
  "title": "Business rule violated",
  "status": 409,
  "detail": "The order is already confirmed",
  "instance": "/orders/123",
  "code": "ORDER_ALREADY_CONFIRMED",
  "traceId": "4f2c1a...",
  "errors": {
    "fieldName": ["Message..."]
  }
}
```

Points clés :
- `status` = HTTP
- `code` = code métier stable (pour mapping UI)
- `traceId` = corrélation logs
- `errors` = erreurs de validation par champ

## 2.2 Normaliser avec un HttpInterceptor
Objectifs d’un interceptor d’erreur :
- Ajouter un **Correlation ID**
- Normaliser l’erreur en un type applicatif (ex : `AppError`)
- Distinguer technique/métier
- Déclencher actions globales (logout, redirection)

### 2.2.1 Définir des types d’erreurs

```ts
// app-error.model.ts
export type ErrorKind = 'TECHNICAL' | 'BUSINESS' | 'VALIDATION' | 'AUTH' | 'NOT_FOUND' | 'UNKNOWN';

export interface AppError {
  kind: ErrorKind;
  message: string;          // message “safe” pour l’utilisateur (ou clé i18n)
  technicalMessage?: string; // détails pour logs (jamais affichés tels quels)
  status?: number;
  code?: string;            // code métier
  traceId?: string;
  details?: unknown;        // payload brut
  url?: string;
}
```

### 2.2.2 Implémenter l’interceptor

```ts
// error-http.interceptor.ts
import { Injectable } from '@angular/core';
import {
  HttpEvent, HttpInterceptor, HttpHandler, HttpRequest, HttpErrorResponse
} from '@angular/common/http';
import { Observable, throwError } from 'rxjs';
import { catchError } from 'rxjs/operators';
import { AppError } from './app-error.model';

@Injectable()
export class ErrorHttpInterceptor implements HttpInterceptor {
  intercept(req: HttpRequest<unknown>, next: HttpHandler): Observable<HttpEvent<unknown>> {
    const correlationId = crypto.randomUUID?.() ?? String(Date.now());

    const cloned = req.clone({
      setHeaders: {
        'X-Correlation-Id': correlationId,
      }
    });

    return next.handle(cloned).pipe(
      catchError((err: unknown) => {
        // HttpClient encapsule les erreurs HTTP dans HttpErrorResponse
        if (err instanceof HttpErrorResponse) {
          const appError = this.mapHttpError(err);
          // On propage un AppError (standard interne)
          return throwError(() => appError);
        }
        // Erreur non-HTTP
        const unknown: AppError = {
          kind: 'UNKNOWN',
          message: 'Une erreur inattendue est survenue.',
          technicalMessage: err instanceof Error ? err.message : String(err),
          details: err
        };
        return throwError(() => unknown);
      })
    );
  }

  private mapHttpError(err: HttpErrorResponse): AppError {
    const status = err.status;
    const url = err.url ?? undefined;

    // Exemple de lecture d’un payload type “Problem Details”
    const payload = err.error as any;
    const traceId: string | undefined = payload?.traceId;
    const code: string | undefined = payload?.code;

    if (status === 0) {
      return {
        kind: 'TECHNICAL',
        message: 'Impossible de contacter le serveur. Vérifiez votre connexion.',
        technicalMessage: err.message,
        status,
        traceId,
        code,
        details: payload,
        url,
      };
    }

    if (status === 401 || status === 403) {
      return {
        kind: 'AUTH',
        message: status === 401 ? 'Votre session a expiré. Veuillez vous reconnecter.' : 'Accès non autorisé.',
        technicalMessage: payload?.detail ?? err.message,
        status,
        traceId,
        details: payload,
        url,
      };
    }

    if (status === 404) {
      return {
        kind: 'NOT_FOUND',
        message: 'Ressource introuvable.',
        technicalMessage: payload?.detail ?? err.message,
        status,
        traceId,
        details: payload,
        url,
      };
    }

    // Validation (ex: 400/422)
    if (status === 400 || status === 422) {
      const hasFieldErrors = !!payload?.errors;
      return {
        kind: hasFieldErrors ? 'VALIDATION' : 'BUSINESS',
        message: hasFieldErrors ? 'Certains champs sont invalides.' : (payload?.title ?? 'Requête invalide.'),
        technicalMessage: payload?.detail ?? err.message,
        status,
        traceId,
        code,
        details: payload,
        url,
      };
    }

    // Conflit métier typique
    if (status === 409) {
      return {
        kind: 'BUSINESS',
        message: this.mapBusinessMessage(code) ?? 'Action impossible à cause d’un conflit.',
        technicalMessage: payload?.detail ?? err.message,
        status,
        traceId,
        code,
        details: payload,
        url,
      };
    }

    // Fallback technique
    return {
      kind: 'TECHNICAL',
      message: 'Une erreur technique est survenue. Réessayez plus tard.',
      technicalMessage: payload?.detail ?? err.message,
      status,
      traceId,
      code,
      details: payload,
      url,
    };
  }

  private mapBusinessMessage(code?: string): string | null {
    switch (code) {
      case 'ORDER_ALREADY_CONFIRMED':
        return 'Cette commande est déjà confirmée.';
      case 'INSUFFICIENT_FUNDS':
        return 'Solde insuffisant pour réaliser l’opération.';
      default:
        return null;
    }
  }
}
```

### 2.2.3 Enregistrement de l’interceptor

```ts
// app.config.ts (Angular Standalone)
import { provideHttpClient, withInterceptors } from '@angular/common/http';
import { ErrorHttpInterceptor } from './error-http.interceptor';

export const appConfig = {
  providers: [
    provideHttpClient(
      withInterceptors([
        (req, next) => new ErrorHttpInterceptor().intercept(req, next) // ou via DI (préférable)
      ])
    )
  ]
};
```

> Remarque : en vrai projet, on fournira l’interceptor via DI (classe injectée) et non via `new`.

## 2.3 Stratégies RxJS recommandées

### 2.3.1 `catchError` : capter et transformer

```ts
this.api.getOrder(id).pipe(
  catchError((err: AppError) => {
    // fallback local possible
    if (err.kind === 'NOT_FOUND') {
      return of(null);
    }
    return throwError(() => err);
  })
)
```

### 2.3.2 `retry` / `retryWhen` : uniquement sur erreurs transitoires
Éviter de retry sur 4xx (souvent non transitoires).

```ts
import { retry, timer } from 'rxjs';

this.api.loadDashboard().pipe(
  retry({
    count: 2,
    delay: (_error, retryCount) => timer(300 * retryCount)
  })
)
```

### 2.3.3 `timeout` : contrôler les attentes

```ts
import { timeout } from 'rxjs/operators';

this.api.search(q).pipe(
  timeout({ first: 8000 })
)
```

### 2.3.4 `finalize` : garantir la sortie d’un état “loading”

```ts
this.loading = true;
this.api.getData().pipe(
  finalize(() => this.loading = false)
)
.subscribe();
```

---

# 3) Gestion globale : ErrorHandler personnalisé

## 3.1 Pourquoi un ErrorHandler global ?
Même avec de bons `catchError`, il restera :
- erreurs inattendues UI
- erreurs dans des callbacks
- erreurs de template (certaines)
- erreurs non interceptées

Le but :
- capturer
- logger de façon centralisée
- afficher une UI maîtrisée (sans divulguer)

## 3.2 Implémentation

```ts
// global-error-handler.ts
import { ErrorHandler, Injectable, Injector, NgZone } from '@angular/core';
import { LoggerService } from './logger.service';
import { NotificationService } from './notification.service';

@Injectable()
export class GlobalErrorHandler implements ErrorHandler {
  constructor(
    private injector: Injector,
    private zone: NgZone
  ) {}

  handleError(error: unknown): void {
    // Récupération lazy pour éviter cycles DI
    const logger = this.injector.get(LoggerService);
    const notifier = this.injector.get(NotificationService);

    const normalized = this.normalize(error);

    // Log centralisé
    logger.error('Unhandled error', normalized);

    // Remontée UX (dans la zone Angular)
    this.zone.run(() => {
      notifier.showError(normalized.userMessage);
    });

    // Optionnel : laisser Angular écrire en console en dev
    // console.error(error);
  }

  private normalize(error: unknown): { userMessage: string; technical: unknown } {
    if (error instanceof Error) {
      return {
        userMessage: 'Une erreur inattendue est survenue.',
        technical: { name: error.name, message: error.message, stack: error.stack }
      };
    }
    return {
      userMessage: 'Une erreur inattendue est survenue.',
      technical: error
    };
  }
}
```

## 3.3 Enregistrement

```ts
import { ErrorHandler } from '@angular/core';
import { GlobalErrorHandler } from './global-error-handler';

export const appConfig = {
  providers: [
    { provide: ErrorHandler, useClass: GlobalErrorHandler }
  ]
};
```

> Bonnes pratiques : ne pas afficher directement `error.message` à l’utilisateur. Utiliser un message “safe” et loguer les détails côté observabilité.

---

# 4) Logs centralisés et corrélation

## 4.1 Construire un Logger applicatif
Objectifs :
- niveaux : `debug`, `info`, `warn`, `error`
- contexte : utilisateur, route, feature, traceId
- sortie : console en dev, backend en prod

```ts
// logger.service.ts
import { Injectable } from '@angular/core';

export type LogLevel = 'debug' | 'info' | 'warn' | 'error';

@Injectable({ providedIn: 'root' })
export class LoggerService {
  debug(message: string, context?: unknown) { this.log('debug', message, context); }
  info(message: string, context?: unknown)  { this.log('info', message, context); }
  warn(message: string, context?: unknown)  { this.log('warn', message, context); }
  error(message: string, context?: unknown) { this.log('error', message, context); }

  private log(level: LogLevel, message: string, context?: unknown) {
    // Simplifié : console. En prod, envoyer à une API / outil.
    const payload = { ts: new Date().toISOString(), level, message, context };
    // eslint-disable-next-line no-console
    console[level === 'debug' ? 'log' : level](payload);
  }
}
```

## 4.2 Corrélation (traceId)
- Générer un `X-Correlation-Id` côté front
- Le backend le renvoie (header ou payload error)
- Utiliser ce même id dans les logs front + logs back

## 4.3 Envoi serveur (à industrialiser)
Approches possibles :
- **Batching** (envoyer par lots)
- **Throttling** (limiter volume)
- **Filtrage** (ne pas envoyer les erreurs métier comme “error”)
- **Protection RGPD** (pas de données personnelles dans les logs)

---

# 5) Remontée utilisateur maîtrisée

## 5.1 Principes UX
- Dire **quoi faire** : “Réessayez”, “Reconnectez-vous”, “Contactez le support avec le code X”.
- Ne pas exposer : stacktrace, SQL, payloads sensibles.
- Conserver un **ton** cohérent.

## 5.2 Service de notifications

```ts
// notification.service.ts
import { Injectable } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class NotificationService {
  showError(message: string) {
    // MatSnackBar / Toast custom / modal
    alert(message); // placeholder pédagogique
  }
  showInfo(message: string) {
    alert(message);
  }
}
```

## 5.3 Stratégie d’affichage
- **Global** : crash inattendu → page d’erreur, bandeau global
- **Local** : validation → erreurs au niveau des champs
- **Action** : erreurs métier → message contextualisé (toast) + garder l’utilisateur sur l’écran

---

# 6) Erreurs métier : contrat et mapping

## 6.1 Exemple : mapping par code
Objectif : n’afficher qu’un message contrôlé, localisable.

```ts
// business-errors.ts
export const BUSINESS_MESSAGES: Record<string, string> = {
  ORDER_ALREADY_CONFIRMED: 'Cette commande est déjà confirmée.',
  INSUFFICIENT_FUNDS: 'Solde insuffisant pour réaliser l’opération.',
};
```

Dans l’interceptor (ou service métier), on mappe `code` → message.

## 6.2 Validation : remonter au formulaire
Exemple pour appliquer des erreurs serveur sur un `FormGroup`.

```ts
import { FormGroup } from '@angular/forms';
import { AppError } from './app-error.model';

export function applyServerValidation(form: FormGroup, err: AppError) {
  const payload: any = err.details;
  const fieldErrors = payload?.errors;
  if (!fieldErrors) return;

  Object.keys(fieldErrors).forEach(field => {
    const control = form.get(field);
    if (control) {
      control.setErrors({ server: fieldErrors[field].join(' ') });
    }
  });
}
```

---

# 7) Atelier fil rouge (pas à pas)

## 7.1 Objectif
Mettre en place une chaîne complète :
1. Interceptor de normalisation (`AppError`)
2. Notifications UX
3. Logger centralisé
4. ErrorHandler global

## 7.2 Scénarios à tester
- **Offline / status 0** : message “connexion”
- **401** : notifier et rediriger login
- **403** : accès refusé
- **404** : page ou message ressource introuvable
- **409 + code métier** : message métier
- **422 validation** : erreurs par champs
- **500** : message technique générique + log complet
- **Exception JS** : captée par `GlobalErrorHandler`

## 7.3 Critères de réussite
- Aucun composant n’affiche de stacktrace
- Les messages sont cohérents et i18n-ready
- Les logs contiennent `traceId`/url/status
- Les erreurs métier ne polluent pas les alertes “techniques”

---

# 8) Checklist & bonnes pratiques

## 8.1 Design
- [ ] Un type d’erreur applicatif unique (`AppError`)
- [ ] Interceptor HTTP pour normaliser
- [ ] Gestion globale via `ErrorHandler`
- [ ] Messages user “safe” + message technique loggable

## 8.2 Observabilité
- [ ] Corrélation `traceId`
- [ ] Niveaux de logs (warn vs error)
- [ ] Filtrage RGPD
- [ ] Alerting côté outil (Sentry/ELK/etc.)

## 8.3 Sécurité
- [ ] Ne pas loguer tokens, mots de passe, données sensibles
- [ ] Ne pas afficher les détails techniques

## 8.4 Tests
- [ ] Tests unitaires des mappers d’erreurs
- [ ] Tests d’intégration interceptor (HttpTestingController)

---

## Annexe A — Exemple de test d’interceptor (aperçu)

```ts
import { TestBed } from '@angular/core/testing';
import { HttpClientTestingModule, HttpTestingController } from '@angular/common/http/testing';
import { HTTP_INTERCEPTORS, HttpClient } from '@angular/common/http';
import { ErrorHttpInterceptor } from './error-http.interceptor';

describe('ErrorHttpInterceptor', () => {
  let http: HttpClient;
  let ctrl: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [HttpClientTestingModule],
      providers: [
        { provide: HTTP_INTERCEPTORS, useClass: ErrorHttpInterceptor, multi: true },
      ]
    });

    http = TestBed.inject(HttpClient);
    ctrl = TestBed.inject(HttpTestingController);
  });

  it('should map 404 to NOT_FOUND AppError', (done) => {
    http.get('/api/x').subscribe({
      next: () => done.fail('should error'),
      error: (err) => {
        expect(err.kind).toBe('NOT_FOUND');
        done();
      }
    });

    const req = ctrl.expectOne('/api/x');
    req.flush({ title: 'Not found' }, { status: 404, statusText: 'Not Found' });
  });
});
```

---

## Conclusion
Une gestion d’erreurs avancée dans Angular repose sur :
- **standardiser** les erreurs (HTTP → `AppError`)
- **centraliser** la capture (interceptor + `ErrorHandler`)
- **loguer** de manière exploitable (corrélation, niveaux)
- **communiquer** à l’utilisateur sans fuite d’information et avec un UX cohérent
- **distinguer** technique vs métier pour éviter les mauvais réflexes (ex : “toast d’erreur” systématique).
