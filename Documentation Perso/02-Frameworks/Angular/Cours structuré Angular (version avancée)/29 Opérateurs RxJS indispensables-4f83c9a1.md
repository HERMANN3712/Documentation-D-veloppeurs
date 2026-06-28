# Formation — Opérateurs RxJS indispensables (Angular)

**Public cible** : développeurs Angular (intermédiaires) souhaitant maîtriser les opérateurs RxJS les plus utilisés et éviter les bugs subtils.

**Pré-requis** : bases TypeScript, Observables (subscribe/pipe), HttpClient, formulaires/réactivité Angular.

**Durée suggérée** : 1 journée (6–7h) ou 2 demi-journées.

**Objectifs pédagogiques**
- Choisir le bon opérateur de transformation et d’aplatissement (`map`, `switchMap`, `mergeMap`, `concatMap`, `exhaustMap`).
- Gérer les effets de bord et la mise en cache (`tap`, `shareReplay`).
- Filtrer et contrôler le débit d’événements (`filter`, `debounceTime`, `distinctUntilChanged`).
- Gérer les erreurs de façon robuste (`catchError`).
- Éviter les fuites mémoire et annuler proprement (`takeUntil`).

---

## Plan de la formation

1. **Rappels RxJS essentiels et modèle mental**
   - Observable, Observer, Subscription
   - Cold vs Hot, multicasting
   - `pipe`, opérateurs purs vs effets de bord

2. **Opérateurs de transformation et de filtrage**
   - `map`, `filter`, `distinctUntilChanged`

3. **Opérateurs de contrôle de flux (temps et rafales)**
   - `debounceTime`

4. **Aplatissement (flattening) et concurrence**
   - `switchMap`, `mergeMap`, `concatMap`, `exhaustMap`
   - Comparaison, pièges, cas métier Angular

5. **Effets de bord et cache**
   - `tap`
   - `shareReplay`

6. **Gestion d’erreurs**
   - `catchError` (où le placer, comment ne pas casser le stream)

7. **Cycle de vie Angular et désabonnement**
   - `takeUntil` (et variantes modernes)

8. **Ateliers guidés (Angular)**
   - Recherche live + annulation + debounce
   - Enchaînement d’appels HTTP dépendants
   - Upload bouton (anti double-clic)
   - Mise en cache d’un endpoint et partage entre composants

9. **Checklist de choix d’opérateur + anti-patterns**

---

## 1) Rappels RxJS essentiels et modèle mental

### 1.1 Observable : un flux dans le temps
Un **Observable** représente une suite de valeurs pouvant arriver **dans le temps** :
- événements UI (input, click)
- réponses HTTP
- valeurs calculées
- combinaisons de multiples sources

En Angular, les Observables sont omniprésents (HttpClient, Reactive Forms, Router, NgRx…).

### 1.2 `pipe` et opérateurs
On transforme un flux via :

```ts
source$.pipe(
  operatorA(),
  operatorB(),
  operatorC(),
);
```

- **Opérateurs purs** : transforment les valeurs sans effet externe (`map`, `filter`, `distinctUntilChanged`).
- **Opérateurs d’effets de bord** : font “quelque chose” (log, spinner, metrics) sans modifier la donnée (`tap`).

### 1.3 Cold vs Hot (très concret)
- **Cold** : chaque `subscribe()` déclenche la production (ex: `http.get()` est cold : 2 abonnements ⇒ 2 requêtes).
- **Hot** : la source produit indépendamment des abonnements (ex: `fromEvent`, Subject, websockets).

`shareReplay` sert souvent à **partager** un cold observable, pour éviter les duplications.

---

## 2) Transformation et filtrage

### 2.1 `map` — transformer la valeur
**But** : projeter chaque émission vers une autre forme.

```ts
import { map } from 'rxjs/operators';

const userName$ = user$.pipe(
  map(u => `${u.firstName} ${u.lastName}`)
);
```

**Bon usage Angular** : adapter la réponse API au modèle UI.

```ts
this.products$ = this.http.get<ApiProduct[]>('/api/products').pipe(
  map(products => products.map(p => ({
    id: p.id,
    label: p.name,
    priceCents: p.price_cents,
  })))
);
```

**À ne pas confondre** : `map` ne gère pas l’asynchronisme. Si vous retournez un Observable dans un `map`, vous créez un `Observable<Observable<T>>` : c’est généralement un signe qu’il faut un opérateur d’aplatissement (`switchMap`, `mergeMap`, etc.).

---

### 2.2 `filter` — garder uniquement ce qui vous intéresse
**But** : laisser passer les émissions qui vérifient une condition.

```ts
import { filter } from 'rxjs/operators';

const nonNullUser$ = user$.pipe(
  filter((u): u is User => u != null)
);
```

**Cas Angular classique** : attendre une valeur réellement exploitable.

```ts
this.route.params.pipe(
  map(p => p['id'] as string | undefined),
  filter((id): id is string => !!id)
);
```

---

### 2.3 `distinctUntilChanged` — éviter les doublons consécutifs
**But** : ne laisser passer une valeur que si elle est différente de la précédente.

```ts
import { distinctUntilChanged } from 'rxjs/operators';

const query$ = searchControl.valueChanges.pipe(
  distinctUntilChanged()
);
```

**Attention** :
- Par défaut, comparaison avec `===`.
- Pour des objets, fournissez un comparateur.

```ts
distinctUntilChanged((a, b) => a.id === b.id)
```

**Bénéfice** : moins de re-rendu, moins de requêtes, moins de logique répétée.

---

## 3) Contrôle de flux

### 3.1 `debounceTime` — attendre la “stabilité”
**But** : n’émettre qu’après une période de silence. Idéal pour la saisie utilisateur.

```ts
import { debounceTime } from 'rxjs/operators';

const query$ = searchControl.valueChanges.pipe(
  debounceTime(300),
  distinctUntilChanged()
);
```

**Quand l’utiliser** :
- barre de recherche
- autocomplétion
- validation asynchrone

**Quand ne pas l’utiliser** :
- interactions où chaque événement compte (ex: tracking précis)
- flux “temps réel” où le délai est indésirable

---

## 4) Aplatissement et concurrence (le cœur des bugs subtils)

Ces opérateurs servent quand vous avez :
- un flux A (ex: input, click, route param)
- et que pour chaque valeur de A, vous lancez un **Observable interne** (souvent un appel HTTP)

### 4.1 Problématique : Observable de Observables

```ts
const result$ = query$.pipe(
  map(q => this.http.get(`/api/search?q=${q}`))
);
// type: Observable<Observable<SearchResult>>
```

On veut **aplatir** pour obtenir `Observable<SearchResult>`.

---

### 4.2 `switchMap` — basculer sur le dernier et annuler le précédent
**But** : à chaque nouvelle valeur, on se désabonne de l’Observable interne précédent.

```ts
import { switchMap } from 'rxjs/operators';

const results$ = query$.pipe(
  debounceTime(300),
  distinctUntilChanged(),
  switchMap(q => this.http.get<SearchResult[]>('/api/search', { params: { q } }))
);
```

**Idéal pour** :
- recherche live (on ne veut que le dernier résultat)
- changements de route (charger les données de la page courante)
- auto-complete

**Piège** :
- l’annulation concerne l’abonnement RxJS ; côté réseau, le navigateur peut ou non annuler la requête selon les APIs/implémentations, mais **le résultat est ignoré**.

---

### 4.3 `mergeMap` — exécuter en parallèle
**But** : lancer tous les Observables internes et émettre au fil de l’eau.

```ts
import { mergeMap } from 'rxjs/operators';

const details$ = ids$.pipe(
  mergeMap(id => this.http.get<Product>(`/api/products/${id}`))
);
```

**Idéal pour** :
- préchargement de plusieurs ressources indépendantes
- traitements parallèles
- fan-out (un événement déclenche plusieurs appels)

**Risques** :
- surcharge serveur / rate-limit
- ordering non garanti (les réponses arrivent dans n’importe quel ordre)

**Astuce** : limiter la concurrence avec le paramètre `concurrent`.

```ts
mergeMap(id => this.http.get(`/api/p/${id}`), 5) // max 5 en parallèle
```

---

### 4.4 `concatMap` — exécuter séquentiellement (file d’attente)
**But** : lancer les Observables internes **un par un**, dans l’ordre.

```ts
import { concatMap } from 'rxjs/operators';

const saveAll$ = formSubmits$.pipe(
  concatMap(payload => this.http.post('/api/save', payload))
);
```

**Idéal pour** :
- opérations qui doivent respecter l’ordre
- éviter les conflits d’écriture
- workflows type “queue”

**Coût** : peut devenir lent si la file grandit.

---

### 4.5 `exhaustMap` — ignorer tant que c’est en cours
**But** : si un Observable interne est actif, ignorer les nouveaux déclenchements.

```ts
import { exhaustMap } from 'rxjs/operators';

const login$ = loginClicks$.pipe(
  exhaustMap(() => this.http.post('/api/login', creds))
);
```

**Idéal pour** :
- éviter le double-clic (login, checkout, paiement)
- actions “non réentrantes”

**Piège** :
- l’utilisateur peut changer d’avis pendant l’action : les nouveaux clics sont ignorés.

---

### 4.6 Tableau de choix rapide

| Opérateur | Concurrence | Annule précédent ? | Conserve l’ordre ? | Cas d’usage typique |
|---|---:|---:|---:|---|
| `switchMap` | 1 (dernier) | Oui | N/A | recherche, route-changes |
| `mergeMap` | N (parallèle) | Non | Non | paralléliser, fan-out |
| `concatMap` | 1 (queue) | Non | Oui | sauvegardes séquentielles |
| `exhaustMap` | 1 (ignore) | Ignore | N/A | anti double-submit |

---

## 5) Effets de bord et cache

### 5.1 `tap` — observer sans transformer
**But** : exécuter une action sans modifier les valeurs.

```ts
import { tap } from 'rxjs/operators';

this.user$ = this.http.get<User>('/api/me').pipe(
  tap(() => this.loading = true),
  // (mieux: mettre loading=true avant le call, et false en finalize)
);
```

**Bonnes pratiques** :
- `tap` pour logs, metrics, instrumentation
- éviter d’y faire de la logique métier qui devrait être dans `map`/`switchMap`

**Pattern spinner propre** (avec `finalize`, pas demandé mais utile) :

```ts
import { finalize, tap } from 'rxjs/operators';

this.loading = true;
this.user$ = this.http.get<User>('/api/me').pipe(
  tap(() => console.log('request started')),
  finalize(() => this.loading = false)
);
```

---

### 5.2 `shareReplay` — partager et rejouer la dernière valeur
**But** :
- partager une même exécution (éviter `http.get` multiple)
- fournir la dernière valeur aux nouveaux abonnés

```ts
import { shareReplay } from 'rxjs/operators';

this.config$ = this.http.get<AppConfig>('/api/config').pipe(
  shareReplay({ bufferSize: 1, refCount: true })
);
```

**Pourquoi `refCount: true` ?**
- Permet de fermer la source quand plus personne n’est abonné.

**Pièges fréquents** :
- `shareReplay(1)` sans `refCount` peut garder en mémoire et maintenir la source active selon les cas.
- En cas d’erreur HTTP, le résultat (erreur) peut être “rejoué” aux futurs abonnés : il faut parfois gérer l’erreur avant.

Exemple robuste avec gestion d’erreur :

```ts
this.config$ = this.http.get<AppConfig>('/api/config').pipe(
  catchError(err => {
    // fallback ou rethrow selon le besoin
    return of({ featureX: false } as AppConfig);
  }),
  shareReplay({ bufferSize: 1, refCount: true })
);
```

---

## 6) Gestion d’erreurs

### 6.1 `catchError` — intercepter et retourner un Observable
**But** : remplacer une erreur par un autre flux (fallback, retry, valeur par défaut, ou re-propagation).

```ts
import { catchError } from 'rxjs/operators';
import { of, throwError } from 'rxjs';

this.user$ = this.http.get<User>('/api/me').pipe(
  catchError(err => {
    if (err.status === 401) return of(null);
    return throwError(() => err);
  })
);
```

**Point crucial** : `catchError` **termine** le flux en erreur s’il rethrow, sinon il **remplace** par le flux retourné.

### 6.2 Où placer `catchError` ?
- **Au plus proche** de la source qui peut échouer (souvent le HTTP) pour un fallback local.
- **Plus haut** si vous voulez une stratégie globale.

Exemple avec `switchMap` :

```ts
results$ = query$.pipe(
  switchMap(q => this.http.get<SearchResult[]>('/api/search', { params: { q } }).pipe(
    catchError(() => of([]))
  ))
);
```

Ici, l’erreur d’une requête **n’éteint pas** le flux de recherche : on renvoie `[]` et on continue à écouter les prochaines saisies.

---

## 7) Désabonnement et cycle de vie

### 7.1 `takeUntil` — couper le flux quand un signal arrive
**But** : stopper un observable (et prévenir les fuites) quand un autre observable émet (souvent `destroy$`).

```ts
import { Subject } from 'rxjs';
import { takeUntil } from 'rxjs/operators';

export class DemoComponent {
  private destroy$ = new Subject<void>();

  ngOnInit() {
    this.route.params.pipe(
      takeUntil(this.destroy$)
    ).subscribe();
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

**Quand l’utiliser en Angular** :
- subscriptions manuelles (`subscribe()` dans le composant)
- streams longs : websockets, fromEvent, interval

**Note moderne** : Angular propose aussi `takeUntilDestroyed()` via `@angular/core/rxjs-interop` (Angular 16+), mais `takeUntil` reste fondamental, notamment hors Angular ou pour comprendre le pattern.

---

## 8) Ateliers guidés (exercices)

> Chaque atelier inclut un énoncé, une solution type et les points d’attention.

### Atelier 1 — Recherche live (debounce + switchMap)
**Objectif** : éviter le spam réseau et ignorer les réponses obsolètes.

**Énoncé** : un champ de recherche déclenche une requête `/api/search?q=...`.

**Solution** :

```ts
results$ = this.searchCtrl.valueChanges.pipe(
  map(v => (v ?? '').trim()),
  debounceTime(300),
  distinctUntilChanged(),
  filter(q => q.length >= 2),
  switchMap(q => this.http.get<Item[]>('/api/search', { params: { q } }).pipe(
    catchError(() => of([]))
  )),
  shareReplay({ bufferSize: 1, refCount: true })
);
```

**Points clés** :
- `switchMap` annule l’ancienne recherche
- `catchError` à l’intérieur évite d’éteindre le flux
- `shareReplay` partage le résultat si plusieurs async pipes

---

### Atelier 2 — Enchaîner 2 appels dépendants (switchMap)
**Objectif** : récupérer un utilisateur puis ses commandes.

```ts
orders$ = this.http.get<User>('/api/me').pipe(
  switchMap(user => this.http.get<Order[]>(`/api/users/${user.id}/orders`)),
  catchError(err => {
    // stratégie selon le produit
    return of([] as Order[]);
  })
);
```

---

### Atelier 3 — Traitement en parallèle (mergeMap) avec limite
**Objectif** : pour une liste d’IDs, charger les détails avec concurrence max = 3.

```ts
details$ = from(ids).pipe(
  mergeMap(id => this.http.get<Detail>(`/api/details/${id}`), 3),
);
```

---

### Atelier 4 — Sauvegarde séquentielle (concatMap)
**Objectif** : l’utilisateur clique “Enregistrer” plusieurs fois, on veut une file d’attente.

```ts
saveResult$ = this.saveClicks$.pipe(
  map(() => this.form.getRawValue()),
  concatMap(payload => this.http.post('/api/save', payload).pipe(
    catchError(err => of({ ok: false, err }))
  ))
);
```

---

### Atelier 5 — Anti double-submit (exhaustMap)
**Objectif** : éviter plusieurs requêtes concurrentes sur un checkout.

```ts
checkout$ = this.checkoutClicks$.pipe(
  exhaustMap(() => this.http.post('/api/checkout', this.cart))
);
```

---

### Atelier 6 — Cache applicatif léger (shareReplay)
**Objectif** : `/api/config` appelé par plusieurs composants.

Dans un service singleton :

```ts
@Injectable({ providedIn: 'root' })
export class ConfigService {
  readonly config$ = this.http.get<AppConfig>('/api/config').pipe(
    shareReplay({ bufferSize: 1, refCount: true })
  );

  constructor(private http: HttpClient) {}
}
```

---

## 9) Checklist et anti-patterns

### 9.1 Checklist de choix rapide
- Je veux **uniquement le dernier** résultat (input/route) → `switchMap`
- Je veux **tout exécuter en parallèle** → `mergeMap` (avec `concurrent` si besoin)
- Je veux **respecter l’ordre** et éviter la concurrence → `concatMap`
- Je veux **ignorer** les demandes tant qu’une est en cours → `exhaustMap`

### 9.2 Anti-patterns fréquents
1. **Nested subscribe**
   ```ts
   a$.subscribe(a => {
     b$(a).subscribe(...)
   })
   ```
   Remplacer par `switchMap/mergeMap/...`.

2. **Oublier `catchError` dans un stream UI**
   Une seule erreur peut tuer le flux (ex: search input). Traiter l’erreur à l’intérieur.

3. **`shareReplay` mal configuré**
   Attention au cache d’erreur et aux comportements de subscription. Préférer :
   ```ts
   shareReplay({ bufferSize: 1, refCount: true })
   ```

4. **Comparaison naïve avec `distinctUntilChanged` sur objets**
   Utiliser un comparateur ou `map` vers une primitive stable.

5. **Choisir `mergeMap` par défaut**
   Souvent source de bugs (ordre, concurrence). Choisir intentionnellement.

---

## Annexes — Mini mémo

### Opérateurs couverts
- Transformation : `map`
- Aplatissement : `switchMap`, `mergeMap`, `concatMap`, `exhaustMap`
- Side effects : `tap`
- Erreurs : `catchError`
- Timing & filtration : `debounceTime`, `distinctUntilChanged`, `filter`
- Cache & partage : `shareReplay`
- Lifecycle : `takeUntil`

### Exemples de signatures mentales
- `map: T -> R`
- `switchMap: T -> Observable<R> (garde le dernier)`
- `mergeMap: T -> Observable<R> (parallèle)`
- `concatMap: T -> Observable<R> (queue)`
- `exhaustMap: T -> Observable<R> (ignore si busy)`

---

## Propositions de slides (si besoin)
- Flux et opérateurs : "je lis de gauche à droite"
- Les 4 flattening operators : diagrammes temporels
- Table de décision (case studies Angular)

---

### Fin
Ce cours a été conçu pour vous donner un modèle mental clair : **le choix de l’opérateur influence le comportement métier, les performances et l’absence de bugs subtils**. Pour chaque scénario Angular, sachez répondre : *faut-il annuler, paralléliser, séquencer ou ignorer ?*.
