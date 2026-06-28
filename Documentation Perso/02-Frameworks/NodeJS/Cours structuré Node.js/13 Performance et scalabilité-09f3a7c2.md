# Formation 13 — Performance et scalabilité (Node.js)

- **Public** : développeurs Node.js (niveau intermédiaire à avancé)
- **Durée conseillée** : 1 jour (7h) ou 2 demi-journées
- **Pré-requis** : JavaScript/TypeScript, HTTP, notions de promesses/async-await, bases d’Express/Fastify.
- **Objectifs pédagogiques** :
  - Comprendre *pourquoi* Node.js tient beaucoup de connexions (I/O non bloquantes, event loop).
  - Identifier et mesurer les sources de latence et de consommation CPU/mémoire.
  - Mettre en place des optimisations pragmatiques (caching, pooling, compression, backpressure).
  - Déployer à plusieurs cœurs et à plusieurs instances (clustering, load balancing, stateless, sessions).
  - Savoir quand et comment scaler (vertical/horizontal), et éviter les anti-patterns.

---

## Plan de la formation

1. **Fondamentaux de performance Node.js**
   1. Modèle non bloquant et event loop
   2. CPU vs I/O : comprendre la vraie contrainte
   3. Mesures : latence, throughput, percentiles, saturation
2. **Mesurer avant d’optimiser**
   1. Observabilité : logs, métriques, traces
   2. Profiling CPU et mémoire
   3. Tests de charge et méthodologie
3. **Optimisations côté serveur (Node.js/HTTP)**
   1. Gestion efficace des connexions
   2. Réduction du travail par requête
   3. Streaming, backpressure, gestion des gros payloads
   4. Compression, HTTP caching, keep-alive
4. **Caching (levier majeur)**
   1. Caches : in-memory, Redis, CDN, HTTP
   2. Stratégies : TTL, invalidation, cache stampede
   3. Patterns : cache-aside, write-through, stale-while-revalidate
5. **Scalabilité sur un hôte : clustering et workers**
   1. `cluster` : principes, limites, sticky sessions
   2. `worker_threads` : CPU-bound
   3. Process manager (PM2/systemd) et auto-restart
6. **Scalabilité horizontale et architecture**
   1. Stateless, sessions distribuées
   2. Load balancer, autoscaling
   3. Files de messages (queue) pour lisser la charge
7. **Ateliers pratiques et checklists**
   1. Diagnostic d’une API lente
   2. Ajout de cache Redis
   3. Passage en cluster + mesures avant/après

---

## 1) Fondamentaux de performance Node.js

### 1.1 Le modèle non bloquant : pourquoi Node gère beaucoup de connexions
Node.js repose sur :

- **Une boucle d’événements (event loop)** : orchestre l’exécution du code JavaScript.
- **Des opérations I/O asynchrones** (réseau, disque, DNS, etc.) : déléguées au système (libuv, kernel).
- **Un thread JS principal** : exécute le code utilisateur.

Conséquence :
- Tant que votre code **ne bloque pas** le thread principal, Node peut **gérer des milliers de sockets** en parallèle.
- La scalabilité “naturelle” de Node vient de cette capacité à **attendre en parallèle** des I/O.

#### Ce qui bloque réellement Node
- **CPU intensif** dans le thread principal (hashing lourd, compression synchro, parsing énorme JSON en bloc, boucles).
- **Appels synchrones** (`fs.readFileSync`, crypto sync, etc.).
- **Garbage Collection** : pauses si allocations excessives.

### 1.2 CPU vs I/O : identifier le goulot d’étranglement
- **I/O-bound** : l’application passe son temps à attendre DB, HTTP, disque.
  - Optimisations typiques : caching, pooling, index DB, batching, réduction des requêtes.
- **CPU-bound** : l’application calcule (transformation, chiffrement, rendu).
  - Optimisations typiques : worker threads, offload (services dédiés), algos plus efficaces.

### 1.3 Métriques essentielles
- **Latence** : temps d’une requête.
  - Ne pas se limiter à la moyenne : suivre **p95/p99**.
- **Throughput** : requêtes par seconde (RPS).
- **Saturation** : CPU, mémoire, file d’attente, pool DB saturé.
- **Erreur** : taux 4xx/5xx, timeouts.

---

## 2) Mesurer avant d’optimiser

### 2.1 Observabilité : logs, métriques, traces
#### Logs
- Corrélation (request-id)
- Structure (JSON) plutôt que texte libre
- Éviter les logs verbeux en production (coût CPU/I/O)

#### Métriques
- Temps de réponse (histogrammes)
- Utilisation event loop
- Connexions actives, throughput
- DB : pool, temps de requête, taux d’erreur

Outils possibles : Prometheus/Grafana, OpenTelemetry.

### 2.2 Profiling CPU : trouver le code qui brûle du temps
- **Node --prof** / **Chrome DevTools** (profiler)
- **0x**, **clinic.js** (Doctor/Flame/Heap)

Objectif : confirmer *où* la CPU part (JSON parse, regex, sérialisation, libs).

### 2.3 Profiling mémoire : fuites et pression GC
- Heap snapshots
- Allocation timeline
- Symptômes : RSS qui grimpe, GC fréquent, latence p99 qui se dégrade.

### 2.4 Tests de charge
- Outils : autocannon, k6, wrk
- Méthode :
  1. Définir un scénario réaliste
  2. Mesurer baseline
  3. Appliquer une optimisation
  4. Re-mesurer (mêmes conditions)

Exemple `autocannon` :

```bash
npx autocannon -c 100 -d 30 http://localhost:3000/api/products
```

---

## 3) Optimisations côté serveur (Node.js/HTTP)

### 3.1 Keep-Alive et réutilisation des connexions
- Activer **HTTP keep-alive** côté clients (agents HTTP).
- Côté serveur, garder des timeouts cohérents.

Pour les appels sortants (Node -> API) :

```js
import http from 'node:http';
import fetch from 'node-fetch';

const agent = new http.Agent({ keepAlive: true, maxSockets: 100 });

await fetch('http://service.local/data', { agent });
```

### 3.2 Réduire le travail par requête
- **Valider** sans surcoût : schémas (zod/joi) oui, mais attention à la complexité.
- Éviter les conversions inutiles (ex: refaire un mapping complet si pas nécessaire).
- Pagination et champs sélectionnés (éviter de renvoyer “tout”).

### 3.3 Streaming + backpressure
Pour gros fichiers/exports, éviter de charger en mémoire.

```js
import fs from 'node:fs';
import { pipeline } from 'node:stream/promises';

app.get('/download', async (req, res) => {
  res.setHeader('Content-Type', 'application/octet-stream');
  await pipeline(
    fs.createReadStream('./bigfile.bin'),
    res
  );
});
```

**Backpressure** : Node régule le débit via les streams. C’est clé pour la scalabilité.

### 3.4 Compression : utile mais coût CPU
- gzip/brotli réduisent le réseau mais consomment du CPU.
- Activer pour les formats compressibles (JSON, HTML, CSS).
- Désactiver/éviter pour les binaires (JPEG, MP4 déjà compressés).

### 3.5 Timeouts, limites et protection
La performance inclut la *résilience* :
- timeouts côté app et côté proxies
- limites de taille body
- rate limiting
- circuit breaker sur les dépendances

---

## 4) Caching (levier majeur)

### 4.1 Où cacher ?
1. **HTTP/CDN** (edge) : assets, pages cacheables.
2. **Reverse proxy** (Nginx/Varnish) : cache de réponses.
3. **Cache applicatif** :
   - in-memory (rapide, mais non partagé)
   - Redis/Memcached (partagé)
4. **Cache DB** : index, query cache (selon DB).

### 4.2 Stratégies principales
- **TTL** : expiration simple.
- **Invalidation** : plus complexe mais cohérente.
- **Stale-while-revalidate** : servir une donnée légèrement vieille et rafraîchir en arrière-plan.

### 4.3 Patterns
#### Cache-aside (le plus courant)
- Lecture : tenter cache -> sinon DB -> stocker cache.
- Écriture : écrire DB puis invalider/mettre à jour cache.

Pseudo-code :

```js
async function getProduct(id) {
  const key = `product:${id}`;
  const cached = await redis.get(key);
  if (cached) return JSON.parse(cached);

  const product = await db.products.findById(id);
  await redis.set(key, JSON.stringify(product), 'EX', 60);
  return product;
}
```

#### Cache stampede (thundering herd)
Si 1000 requêtes arrivent quand le cache expire, elles vont toutes frapper la DB.

Solutions :
- **verrou distribué** (lock)
- **jitter** sur TTL
- **stale-while-revalidate**

Exemple : TTL avec jitter

```js
const ttl = 60 + Math.floor(Math.random() * 10);
await redis.set(key, value, 'EX', ttl);
```

### 4.4 Cache HTTP (ETag / Cache-Control)
Pour des ressources GET cacheables :
- `Cache-Control: public, max-age=60`
- `ETag` + `If-None-Match` -> réponse 304

Cela réduit la charge serveur **et** réseau.

---

## 5) Scalabilité sur un hôte : clustering et workers

### 5.1 Clustering : utiliser tous les cœurs
Node (un process) n’utilise qu’un cœur pour le JS. Le clustering permet de lancer **N workers**.

Avec le module `cluster` :

```js
import cluster from 'node:cluster';
import os from 'node:os';
import http from 'node:http';

const numCPUs = os.cpus().length;

if (cluster.isPrimary) {
  for (let i = 0; i < numCPUs; i++) cluster.fork();

  cluster.on('exit', (worker) => {
    console.error(`Worker ${worker.process.pid} died, restarting...`);
    cluster.fork();
  });
} else {
  const server = http.createServer((req, res) => {
    res.end(`Handled by ${process.pid}`);
  });

  server.listen(3000);
}
```

**À retenir** :
- Chaque worker a sa mémoire : cache in-memory non partagé.
- Les connexions entrantes sont réparties (round-robin selon plateforme/config).

### 5.2 Sticky sessions
Si vous stockez la session en mémoire locale d’un worker, vous aurez besoin de “sticky sessions”, sinon un utilisateur peut changer de worker.

Meilleure approche : **sessions externalisées** (Redis) ou JWT stateless.

### 5.3 `worker_threads` pour CPU-bound
Quand la charge est CPU (ex: hashing, image processing), déléguer à un pool de workers.

```js
import { Worker } from 'node:worker_threads';

function runJob(data) {
  return new Promise((resolve, reject) => {
    const worker = new Worker(new URL('./job-worker.js', import.meta.url), {
      workerData: data,
    });
    worker.on('message', resolve);
    worker.on('error', reject);
  });
}
```

Utiliser une lib de pool (piscina) pour éviter de créer/détruire des workers en boucle.

---

## 6) Scalabilité horizontale et architecture

### 6.1 Vertical vs horizontal
- **Vertical** : machine plus puissante (simple, limite matérielle).
- **Horizontal** : plusieurs instances (résilience, scalabilité).

Node s’adapte très bien à l’horizontal si l’app est **stateless**.

### 6.2 Stateless : principe
- Aucun état utilisateur en mémoire locale (sessions locales, caches critiques).
- État en services partagés : DB, Redis, S3, etc.

### 6.3 Load balancing
- Nginx/HAProxy/ALB (cloud)
- Stratégies : round-robin, least connections
- Terminaison TLS au LB

### 6.4 File de messages (queue) pour lisser
Quand des traitements sont longs :
- Répondre vite (202 Accepted)
- Exécuter en asynchrone (BullMQ/Redis, RabbitMQ, Kafka)

Bénéfices :
- découplage
- backpressure naturel
- meilleure stabilité lors des pics

---

## 7) Ateliers pratiques

### Atelier A — Diagnostiquer une API lente
**But** : identifier si le goulot est CPU, DB, ou réseau.

1. Mesurer p95/p99 et RPS baseline.
2. Activer un profiling CPU (clinic flame) sur une courte fenêtre.
3. Ajouter des métriques simples :
   - durée handler
   - durée requête DB
   - event loop lag

**Livrable** : hypothèse + plan d’action.

### Atelier B — Ajouter un cache Redis (cache-aside)
**But** : réduire le nombre de hits DB.

1. Définir clés + TTL.
2. Protéger contre stampede (jitter + lock ou stale).
3. Mesurer amélioration p95 et charge DB.

### Atelier C — Clustering + mesures
**But** : utiliser tous les cœurs.

1. Passer de 1 process à N workers.
2. Refaire le test de charge.
3. Comparer throughput et p95.

---

## Checklists

### Checklist performance (quick wins)
- [ ] Pas d’I/O sync en prod
- [ ] Keep-alive configuré pour appels sortants
- [ ] Pagination + limitation de payload
- [ ] Streaming pour gros payloads
- [ ] Timeouts partout (HTTP client, DB, LB)
- [ ] Compression activée avec discernement
- [ ] Cache aux bons endroits (HTTP/Redis)

### Checklist scalabilité
- [ ] App stateless (sessions externalisées)
- [ ] Cache partagé si nécessaire (Redis)
- [ ] Clustering/PM2 sur un hôte
- [ ] Instances multiples derrière LB
- [ ] Queue pour tâches longues

---

## Annexes

### A) Mesurer l’event loop lag
Exemple simplifié :

```js
let last = Date.now();
setInterval(() => {
  const now = Date.now();
  const lag = now - last - 1000;
  last = now;
  console.log({ eventLoopLagMs: lag });
}, 1000);
```

### B) Mini guide : quand cluster vs worker_threads ?
- **cluster** : augmenter le débit HTTP en exploitant plusieurs cœurs, surtout I/O-bound.
- **worker_threads** : isoler calculs CPU lourds pour ne pas bloquer l’event loop.

---

## Conclusion
Node.js peut gérer un grand nombre de connexions grâce à son modèle non bloquant. Pour aller plus loin, on combine :
- **mesure** (profiling, métriques)
- **optimisations** (streaming, keep-alive, limitation, compression)
- **caching** (Redis/HTTP/CDN)
- **scalabilité** (clustering, stateless, load balancing, queues)

Ce socle permet de tenir la charge, réduire la latence (p95/p99) et absorber des pics sans dégrader l’expérience utilisateur.
