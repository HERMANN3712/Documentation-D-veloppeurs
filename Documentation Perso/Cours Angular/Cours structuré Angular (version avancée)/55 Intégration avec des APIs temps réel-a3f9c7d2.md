# Formation Angular — Intégration avec des APIs temps réel

**Référence**: 55 — *Intégration avec des APIs temps réel*  
**Public**: Développeurs Angular (débutant++ à confirmé)  
**Prérequis**: TypeScript, bases Angular (components, services, DI), notions d’RxJS (Observable, Subject), HTTPClient  
**Durée conseillée**: 1 à 2 jours (7–14h)  
**Objectif**: Consommer des flux temps réel (WebSocket, Server-Sent Events, SignalR) et construire une UI réactive robuste (reconnexion, backpressure, gestion d’état, tests).

---

## Plan global

1. **Introduction au temps réel dans une SPA Angular**
   - Cas d’usage, contraintes, comparaison WebSocket vs SSE vs SignalR
2. **Architecture côté Angular pour le temps réel**
   - Services, Observables chauds/froids, partage de flux, séparation domaine/transport
3. **WebSocket en Angular (bas niveau + RxJS)**
   - `WebSocketSubject`, protocoles, sérialisation, multiplexing, reconnexion
4. **Server-Sent Events (SSE) en Angular**
   - `EventSource`, consommation RxJS, gestion de reconnection, problèmes CORS
5. **SignalR côté client dans Angular**
   - HubConnection, groupes, streaming, retries, intégration RxJS
6. **Stratégies avancées**
   - Reconnexion robuste, heartbeat, idempotence, ordering, backpressure, buffering
7. **Mise à jour réactive de l’UI**
   - ChangeDetection, `async` pipe, `OnPush`, signaux (optionnel), gestion d’état
8. **Sécurité, performance et observabilité**
   - Auth (JWT, cookies), rotation de token, logs, métriques, diagnostics
9. **Tests (unitaires + intégration) et outillage**
   - Mocks, marbles RxJS, tests E2E, simulateurs de flux
10. **Atelier fil rouge**
   - Construire un mini dashboard temps réel (notifications + ticker + présence)

---

## 1) Introduction au temps réel dans une SPA Angular

### 1.1 Pourquoi du temps réel ?
- **Notifications** (in-app / toast) : messages, alertes, jobs finis.
- **Collaboratif** : chat, co-édition, présence (“X est en ligne”).
- **Données live** : bourse/crypto, IoT, monitoring, logs.
- **Expérience utilisateur** : réduction du polling et latence perçue.

### 1.2 Modèles de communication

| Technologie | Sens | Transport | Quand l’utiliser | Limites |
|---|---:|---|---|---|
| **Polling** | serveur → client (simulé) | HTTP | simple, compat legacy | latence, charge, gaspillage |
| **Long polling** | serveur → client | HTTP | compat proxy, quasi temps réel | complexité, overhead |
| **SSE** | serveur → client | HTTP (flux) | événements unidirectionnels (news feed, logs) | pas client→serveur (hors HTTP), headers limités |
| **WebSocket** | bidirectionnel | WS | chat, actions temps réel, collaboration | infra proxy/load balancer, maintien de connexion |
| **SignalR** | souvent bidirectionnel | abstrait (WS/SSE/LP) | .NET et besoin de fallback + hubs | côté serveur plutôt .NET, dépendance lib |

### 1.3 Enjeux majeurs
- **Robustesse réseau** (mobile, wifi instable) : reconnexion, reprise.
- **Backpressure** : flux trop rapide → UI saturée.
- **Concurrence** : ordering, duplication, idempotence.
- **Sécurité** : auth, token refresh, CORS.
- **Observabilité** : logs, métriques, debug.

---

## 2) Architecture Angular pour le temps réel

### 2.1 Principes
- Encapsuler le transport dans un **service** (DI Angular).
- Exposer des **Observables** (lecture) et des méthodes d’action (écriture).
- Ne pas “pousser” des effets dans les composants : les composants consomment des flux.
- Mutualiser la connexion via **`shareReplay`** / **multicast**.

### 2.2 Schéma recommandé

```
Component -> Facade/Store -> RealtimeService(transport) -> Backend
                 ^
                 | (RxJS streams)
```

- **RealtimeService** : connect/disconnect, low-level events, reconnexion.
- **Facade/Store** : transforme en state (liste, map, counters), applique throttle/buffer.
- **Component** : `vm$ | async` (UI)

### 2.3 Observables chauds vs froids
- **Froid**: démarre à l’abonnement (ex: `http.get()`)
- **Chaud**: émet indépendamment des abonnés (ex: WebSocket/SSE)

Dans le temps réel : on gère presque toujours des **flux chauds**, à partager proprement.

---

## 3) WebSocket en Angular (bas niveau + RxJS)

### 3.1 Choix d’implémentation
- API WebSocket native (`new WebSocket(url)`) + wrapper RxJS.
- Ou **RxJS `webSocket`** (recommandé) : `WebSocketSubject`.

Installation (si nécessaire) : RxJS est déjà inclus avec Angular.

### 3.2 `WebSocketSubject` — base

```ts
// realtime-websocket.service.ts
import { Injectable, NgZone } from '@angular/core';
import { Observable, Subject, defer, timer } from 'rxjs';
import { webSocket, WebSocketSubject } from 'rxjs/webSocket';
import {
  catchError,
  filter,
  map,
  retry,
  share,
  switchMap,
  takeUntil,
  tap,
} from 'rxjs/operators';

export interface WsMessage {
  type: string;
  payload: unknown;
}

@Injectable({ providedIn: 'root' })
export class RealtimeWebSocketService {
  private readonly disconnect$ = new Subject<void>();
  private socket?: WebSocketSubject<unknown>;

  constructor(private zone: NgZone) {}

  connect(url: string): Observable<WsMessage> {
    // `defer` pour créer une nouvelle socket au moment de l'abonnement
    const source$ = defer(() => {
      this.socket = webSocket({
        url,
        // Serializer/deserializer (optionnel)
        serializer: (value) => JSON.stringify(value),
        deserializer: (e) => JSON.parse(e.data),
        openObserver: {
          next: () => console.log('[WS] open'),
        },
        closeObserver: {
          next: (evt) => console.log('[WS] close', evt.code, evt.reason),
        },
      });

      return this.socket as unknown as Observable<WsMessage>;
    });

    // `retry` pour reconnexion simple (voir approche avancée plus bas)
    return source$.pipe(
      takeUntil(this.disconnect$),
      retry({
        delay: (_err, retryCount) => timer(Math.min(1000 * retryCount, 10_000)),
      }),
      catchError((err) => {
        console.error('[WS] fatal', err);
        throw err;
      }),
      share()
    );
  }

  send(msg: unknown) {
    this.socket?.next(msg);
  }

  disconnect() {
    this.disconnect$.next();
    this.socket?.complete();
    this.socket = undefined;
  }
}
```

Points clés :
- `share()` pour éviter plusieurs connexions si plusieurs abonnements.
- `takeUntil(disconnect$)` pour couper proprement.
- `retry` basique (améliorable).

### 3.3 Protocole applicatif (types)
Définir un contrat de messages côté client (TypeScript) :

```ts
type ServerEvent =
  | { type: 'price:update'; payload: { symbol: string; value: number; ts: number } }
  | { type: 'chat:message'; payload: { from: string; text: string; ts: number } }
  | { type: 'presence:update'; payload: { userId: string; online: boolean } };

type ClientCommand =
  | { type: 'subscribe'; payload: { topic: string } }
  | { type: 'unsubscribe'; payload: { topic: string } }
  | { type: 'chat:send'; payload: { text: string } };
```

### 3.4 Multiplexing / topics
Idée : un seul socket, plusieurs canaux.

```ts
import { filter, map } from 'rxjs/operators';

prices$(events$: Observable<ServerEvent>, symbol: string) {
  return events$.pipe(
    filter((e) => e.type === 'price:update'),
    map((e) => e.payload),
    filter((p) => p.symbol === symbol)
  );
}
```

### 3.5 Reconnexion avancée (backoff + jitter + état)
Objectif :
- Reconnecter avec **backoff exponentiel** + **jitter** (éviter thundering herd)
- Exposer un **status$** (`connecting/open/closed/error`)

```ts
export type ConnectionStatus =
  | { state: 'idle' }
  | { state: 'connecting' }
  | { state: 'open' }
  | { state: 'retrying'; attempt: number; inMs: number }
  | { state: 'closed' }
  | { state: 'error'; error: unknown };

function backoff(attempt: number, baseMs = 500, maxMs = 15_000) {
  const exp = Math.min(maxMs, baseMs * 2 ** (attempt - 1));
  const jitter = Math.floor(Math.random() * 250);
  return exp + jitter;
}
```

Implémentation indicative :

```ts
import { BehaviorSubject, Observable, Subject, defer, timer } from 'rxjs';
import { retryWhen, scan, switchMap, tap, takeUntil, share } from 'rxjs/operators';

private readonly statusSubject = new BehaviorSubject<ConnectionStatus>({ state: 'idle' });
status$ = this.statusSubject.asObservable();

connect(url: string): Observable<ServerEvent> {
  const stop$ = this.disconnect$;

  return defer(() => {
    this.statusSubject.next({ state: 'connecting' });

    this.socket = webSocket<ServerEvent>({
      url,
      openObserver: { next: () => this.statusSubject.next({ state: 'open' }) },
      closeObserver: { next: () => this.statusSubject.next({ state: 'closed' }) },
    });

    return this.socket;
  }).pipe(
    takeUntil(stop$),
    retryWhen((errors) =>
      errors.pipe(
        scan((attempt) => attempt + 1, 0),
        switchMap((attempt) => {
          const inMs = backoff(attempt);
          this.statusSubject.next({ state: 'retrying', attempt, inMs });
          return timer(inMs);
        })
      )
    ),
    share()
  );
}
```

### 3.6 Heartbeat (ping/pong)
- Si le serveur ne ferme pas proprement, une connexion peut rester “zombie”.
- On envoie un `ping` périodique et on attend un `pong`, sinon on force reconnect.

Approche : timer + `timeout` sur le flux de pong.

---

## 4) Server-Sent Events (SSE) en Angular

### 4.1 Rappels SSE
- Basé sur HTTP, `Content-Type: text/event-stream`
- Le navigateur gère la reconnexion **automatique** (avec `retry:` côté serveur)
- Unidirectionnel (serveur → client)

### 4.2 Wrapper RxJS autour de `EventSource`

```ts
// realtime-sse.service.ts
import { Injectable } from '@angular/core';
import { Observable } from 'rxjs';

export interface SseEvent<T> {
  event?: string;
  data: T;
}

@Injectable({ providedIn: 'root' })
export class RealtimeSseService {
  connect<T>(url: string, withCredentials = false): Observable<SseEvent<T>> {
    return new Observable((subscriber) => {
      const es = new EventSource(url, { withCredentials });

      es.onmessage = (msg) => {
        try {
          subscriber.next({ data: JSON.parse(msg.data) as T });
        } catch {
          // data non JSON
          subscriber.next({ data: msg.data as unknown as T });
        }
      };

      es.onerror = (err) => {
        // EventSource gère souvent la reconnexion, mais on peut notifier
        // Attention: selon navigateur, onerror peut être fréquent.
        console.warn('[SSE] error', err);
      };

      return () => es.close();
    });
  }
}
```

### 4.3 Points d’attention
- **Authentification** :
  - les headers custom ne sont pas supportés directement par EventSource.
  - préférer cookies (same-site) ou token dans l’URL (à éviter) ou passer par un proxy.
- **CORS** : config serveur `Access-Control-Allow-Origin`, `...-Credentials`.
- **Retry & Last-Event-ID** :
  - serveur peut envoyer `id:` pour reprise.
  - le client renvoie `Last-Event-ID` lors de reconnexion.

---

## 5) SignalR dans Angular

### 5.1 Pourquoi SignalR ?
- Abstrait le transport : WebSockets si possible, sinon SSE, sinon long polling.
- Modèle **Hub** : méthodes serveur, groupes, présence.
- Auto-reconnect configurable.

### 5.2 Installation

```bash
npm i @microsoft/signalr
```

### 5.3 Connexion et écoute

```ts
// realtime-signalr.service.ts
import { Injectable } from '@angular/core';
import * as signalR from '@microsoft/signalr';
import { BehaviorSubject, Observable } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class RealtimeSignalrService {
  private connection?: signalR.HubConnection;
  private statusSubject = new BehaviorSubject<'idle' | 'connecting' | 'connected' | 'disconnected'>('idle');
  status$ = this.statusSubject.asObservable();

  async start(url: string, accessTokenFactory?: () => string | Promise<string>) {
    this.statusSubject.next('connecting');

    this.connection = new signalR.HubConnectionBuilder()
      .withUrl(url, {
        accessTokenFactory,
        transport: signalR.HttpTransportType.WebSockets, // optionnel, laisser auto si besoin
      })
      .withAutomaticReconnect([0, 1000, 5000, 10_000])
      .configureLogging(signalR.LogLevel.Information)
      .build();

    this.connection.onreconnecting(() => this.statusSubject.next('connecting'));
    this.connection.onreconnected(() => this.statusSubject.next('connected'));
    this.connection.onclose(() => this.statusSubject.next('disconnected'));

    await this.connection.start();
    this.statusSubject.next('connected');
  }

  stop() {
    return this.connection?.stop();
  }

  on<T>(eventName: string): Observable<T> {
    return new Observable<T>((subscriber) => {
      const handler = (data: T) => subscriber.next(data);
      this.connection?.on(eventName, handler);

      return () => {
        this.connection?.off(eventName, handler);
      };
    });
  }

  invoke<T>(methodName: string, ...args: unknown[]) {
    return this.connection!.invoke<T>(methodName, ...args);
  }
}
```

### 5.4 Intégration RxJS : exposition d’un “bus” d’événements
- `on('PriceUpdated')` → Observable
- `invoke('Subscribe', topic)` pour s’abonner côté serveur

### 5.5 Streaming SignalR
SignalR supporte le streaming (IAsyncEnumerable/Channel côté serveur). Côté client :

```ts
const stream = this.connection!.stream<number>('Counter');
stream.subscribe({ next: v => console.log(v), complete: () => console.log('done') });
```

---

## 6) Stratégies avancées : reconnexion, diffusion RxJS, backpressure

### 6.1 Reconnexion fiable
Checklist :
- Backoff exponentiel + jitter
- **Limite de tentatives** / bascule en mode dégradé
- Indicateur UI “hors ligne / reconnexion” via `status$`
- Re-subscribe aux topics après reconnexion

### 6.2 Reprise (resume) avec cursor / last event id
- Le serveur doit fournir un **id** monotone (`eventId`, `offset`, `ts`)
- À la reconnexion, le client envoie “reprendre depuis id X”

Approche générique :
- stocker `lastSeenId`
- lors de reconnexion : envoyer `{type:'resume', payload:{from:lastSeenId}}`

### 6.3 Diffusion en RxJS (hot stream) et partage
- Un flux socket doit être **partagé** : `share()` ou `shareReplay({bufferSize:1, refCount:true})`
- Attention à `shareReplay` sans `refCount` (risque de fuite / connexion persistante)

### 6.4 Backpressure (flux trop rapide)
Outils RxJS :
- `throttleTime` : limiter la fréquence
- `auditTime` : conserver la dernière valeur sur une fenêtre
- `bufferTime` / `bufferCount` : batcher
- `sampleTime` : échantillonnage

Exemple : ticker de prix à haute fréquence

```ts
const priceUi$ = priceRaw$.pipe(
  auditTime(250) // max 4 updates/seconde dans l’UI
);
```

### 6.5 Déduplication et ordering
- Utiliser un `eventId`
- Dédupliquer via `scan` + Set/Map (attention mémoire), ou conserver fenêtre glissante.

---

## 7) Mise à jour réactive de l’interface

### 7.1 OnPush + async pipe
- Préférer `ChangeDetectionStrategy.OnPush`
- Laisser Angular gérer l’abonnement via `| async`

```ts
@Component({
  selector: 'app-dashboard',
  template: `
    <section *ngIf="vm$ | async as vm">
      <p>Status: {{ vm.status }}</p>
      <ul>
        <li *ngFor="let n of vm.notifications">{{ n.text }}</li>
      </ul>
    </section>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class DashboardComponent {
  vm$ = this.facade.vm$;
  constructor(private facade: DashboardFacade) {}
}
```

### 7.2 Facade : composition d’un ViewModel (VM)

```ts
export interface Notification { id: string; text: string; ts: number; }

@Injectable({ providedIn: 'root' })
export class DashboardFacade {
  private notificationsSubject = new BehaviorSubject<Notification[]>([]);
  notifications$ = this.notificationsSubject.asObservable();

  constructor(private ws: RealtimeWebSocketService) {
    const events$ = this.ws.connect('wss://example.com/realtime');

    events$.pipe(
      filter(e => e.type === 'notify'),
      map(e => e.payload as Notification)
    ).subscribe(n => {
      const current = this.notificationsSubject.value;
      this.notificationsSubject.next([n, ...current].slice(0, 50));
    });
  }

  vm$ = combineLatest([
    this.ws.status$,
    this.notifications$
  ]).pipe(
    map(([status, notifications]) => ({
      status: status.state,
      notifications
    }))
  );
}
```

### 7.3 Notes sur Angular Signals (optionnel)
- Si votre codebase utilise **Signals**, vous pouvez convertir un Observable en signal via `toSignal()`.
- Garder néanmoins RxJS pour la partie “streaming/backpressure”.

---

## 8) Sécurité, performance, observabilité

### 8.1 Auth
- **WebSocket** : souvent token dans querystring ou header via protocole applicatif.
- **SignalR** : `accessTokenFactory` recommandé.
- **SSE** : privilégier cookies/withCredentials.

Prévoir :
- rotation/refresh token → reconnect si token expiré
- fermeture serveur en cas d’accès non autorisé

### 8.2 Performance
- limiter les mises à jour UI (audit/throttle)
- éviter d’accumuler des listes sans bornes
- sérialisation efficace, payload minimal

### 8.3 Observabilité
- journaliser : open/close, retry attempt, latence (si ping/pong)
- exposer un panneau debug (dev-only)
- côté serveur : métriques connexions, messages/sec, erreurs

---

## 9) Tests

### 9.1 Unit tests : services et transformation RxJS
- Tester la logique de réduction (scan), throttling, etc.
- RxJS marbles (`TestScheduler`) pour valider le backpressure.

### 9.2 Tests d’intégration
- Simuler un serveur WS (Node `ws`) ou mock SignalR
- Vérifier reconnexion, resubscribe, UI

### 9.3 E2E
- Avec Playwright/Cypress : simuler perte réseau, vérifier état “reconnecting”.

---

## 10) Atelier fil rouge (pas à pas)

### Objectif
Construire un mini **dashboard temps réel** avec :
- statut connexion
- notifications
- ticker de prix (sample/audit)
- présence (online/offline)

### Étapes
1. Créer `RealtimeWebSocketService` (connect/send/status)
2. Définir types `ServerEvent` / `ClientCommand`
3. Créer `DashboardFacade` :
   - `events$` partagé
   - `notifications$`, `prices$`, `presence$`
4. Appliquer backpressure sur `prices$`
5. UI `OnPush` avec `vm$ | async`
6. Ajout reprise (resume) : stocker dernier `eventId`
7. Tests : marbles sur `prices$` et déduplication

### Exemple : backpressure + agrégation de prix

```ts
const rawPrices$ = events$.pipe(
  filter((e): e is Extract<ServerEvent, {type:'price:update'}> => e.type === 'price:update'),
  map(e => e.payload)
);

const pricesBySymbol$ = rawPrices$.pipe(
  // batch 200ms
  bufferTime(200),
  filter(batch => batch.length > 0),
  map(batch => {
    const mapBy = new Map<string, number>();
    for (const p of batch) mapBy.set(p.symbol, p.value);
    return mapBy;
  }),
  scan((acc, patch) => {
    patch.forEach((v, k) => acc.set(k, v));
    return new Map(acc);
  }, new Map<string, number>())
);
```

---

## Annexes

### A) Check-list de production
- [ ] Connexion unique partagée (pas de multi-sockets)
- [ ] Politique de reconnexion documentée (backoff, limites)
- [ ] Resume (cursor/last id) si nécessaire
- [ ] Backpressure appliqué (UI stable)
- [ ] Nettoyage des subscriptions (async pipe / takeUntil)
- [ ] Observabilité (logs, metrics)
- [ ] Sécurité (auth + refresh)

### B) Pièges fréquents
- Utiliser `shareReplay` sans `refCount` → socket jamais libéré
- Mettre la logique de flux dans les composants → duplication et fuites
- Mettre à jour l’UI à 60+ events/sec → perf dégradée
- Oublier la gestion de reconnexion + resubscribe

---

## Conclusion
Cette formation vous donne une approche complète pour intégrer des APIs temps réel dans Angular via **WebSocket**, **SSE** et **SignalR**, tout en appliquant les bonnes pratiques **RxJS** (partage de flux, backpressure, réduction/agrégation), et en garantissant une UI **réactive** et **robuste** en production.
