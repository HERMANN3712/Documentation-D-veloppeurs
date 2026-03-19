# Formation Angular (Avancé)

## Content projection avancée

### Public visé
Développeurs Angular intermédiaires/confirmés souhaitant construire des composants hautement réutilisables (design system, librairie UI, composants « headless ») grâce à une projection de contenu maîtrisée.

### Pré-requis
- Connaissances solides Angular (composants, modules, directives, binding, lifecycle, RxJS de base)
- Compréhension de `ng-content` simple (projection basique)
- Notions de templates (`ng-template`), directives structurelles (`*ngIf`, `*ngFor`) et `TemplateRef`

### Objectifs pédagogiques
À l’issue de la formation, vous saurez :
- Construire des composants avec **projection multi-slots** (sélecteurs multiples)
- Mettre en place la **projection de templates** (TemplateRef / ng-template) pour des composants « headless »
- Exploiter `@ContentChild(ren)`, `ngAfterContentInit`, `ngAfterContentChecked` et `QueryList` pour orchestrer du contenu projeté
- Créer des APIs de composants réutilisables : slots typés, directives d’annotation (marker directives), fallback, contraintes
- Composer des composants avancés : `Tabs`, `Card`, `Modal`, `Table` extensible, etc.
- Anticiper les pièges (styles, encapsulation, accessibilité, performance)

### Durée recommandée
1 journée (6–7h) ou 2 demi-journées.

---

## Plan de la formation

1. **Rappels & modèle mental de la projection Angular**
   - `ng-content` et la compilation
   - DOM logique vs DOM rendu
   - Limites de `ng-content` (pas de re-projection, pas de transformation structurelle directe)

2. **Projection multi-slots (sélecteurs multiples)**
   - Slots nommés via sélecteurs CSS
   - Sélecteurs d’attributs et de composants
   - Priorités et « first match wins »
   - Gestion d’un slot par défaut

3. **Fallback content & UX robuste**
   - Contenu par défaut
   - Slots optionnels vs requis
   - Détection côté composant (ContentChild) pour afficher un fallback conditionnel

4. **Projection de templates : libérer le composant (Headless UI)**
   - `ng-template`, `TemplateRef`, `ngTemplateOutlet`
   - Passer du contexte (implicit, variables)
   - Templates optionnels et sensés (header/footer/body)

5. **API de slots via directives “marker”**
   - Directives de marquage (`appDialogTitle`, `appCardActions`)
   - Récupération en `@ContentChild` / `@ContentChildren`
   - Typage, contraintes, hiérarchies

6. **Orchestration avancée du contenu projeté**
   - `QueryList`, écoute des changements
   - Lifecycles : `ngAfterContentInit` / `ngAfterContentChecked`
   - Composition : `ContentChildren` + directives + templates

7. **Cas pratiques structurants**
   - `Card` multi-slots + actions
   - `Tabs` avec items projetés
   - `Modal/Dialog` avec templates (title/body/footer)
   - `DataList/Table` extensible : colonnes déclaratives par projection

8. **Qualité : styles, accessibilité, performance, tests**
   - Encapsulation, `:host`, `::ng-deep` (quand éviter)
   - A11y (roles, aria, focus management)
   - Performance (change detection, projection coûteuse)
   - Tests unitaires (TestBed) et harness/CDK

---

# 1) Rappels & modèle mental

## 1.1 Qu’est-ce que la projection de contenu ?
La projection permet à un composant **conteneur** (comme vous) d’accepter un contenu fourni par son **consommateur** (l’utilisateur du composant), puis de l’insérer à l’endroit voulu grâce à `ng-content`.

Exemple simple :

```html
<!-- consumer -->
<app-panel>
  <p>Je suis projeté dans le panel</p>
</app-panel>
```

```ts
@Component({
  selector: 'app-panel',
  template: `
    <section class="panel">
      <ng-content></ng-content>
    </section>
  `
})
export class PanelComponent {}
```

## 1.2 Modèle mental : compilation & matching
- Angular **compile** le template du consommateur.
- Lors du rendu du composant conteneur, Angular effectue un **matching** : quels nœuds du consommateur vont dans quel `ng-content`.
- Le matching se fait grâce à des **sélecteurs** CSS-like (`select`) sur chaque `ng-content`.

## 1.3 Limitations clés
- `ng-content` n’est pas un `*ngFor` : vous ne « transformez » pas la structure du contenu projeté directement.
- Il n’y a pas de **re-projection** naïve (projeter dans un composant A qui projette dans B) sans précautions.
- Pour des comportements riches (ex : “je veux un template pour chaque ligne”), on bascule souvent vers **TemplateRef** + `ngTemplateOutlet`.

---

# 2) Projection multi-slots (sélecteurs multiples)

## 2.1 Principe
Un composant peut déclarer **plusieurs emplacements** (slots). Chaque slot est un `ng-content` avec un sélecteur.

### Exemple : composant `Card`

Objectif : permettre au consommateur de fournir :
- un header
- un body
- des actions
- et un slot par défaut

```ts
@Component({
  selector: 'app-card',
  template: `
    <article class="card">
      <header class="card__header">
        <ng-content select="[cardTitle]"></ng-content>
      </header>

      <section class="card__body">
        <ng-content></ng-content>
      </section>

      <footer class="card__actions">
        <ng-content select="[cardActions]"></ng-content>
      </footer>
    </article>
  `,
})
export class CardComponent {}
```

Utilisation :

```html
<app-card>
  <h3 cardTitle>Profil</h3>

  <p>Contenu principal projeté dans le slot par défaut.</p>

  <div cardActions>
    <button type="button">Annuler</button>
    <button type="button">Sauvegarder</button>
  </div>
</app-card>
```

## 2.2 Types de sélecteurs
- **Attribut** : `select="[cardTitle]"`
- **Élément** : `select="app-card-title"` (si vous exposez un composant dédié)
- **Classe** : `select=".title"` (moins « API-driven », plutôt éviter)

### Recommandation (librairie UI)
- Préférer **attributs** ou **directives marker**, car cela définit une API stable et lisible.

## 2.3 Règles de matching (important)
- Le premier `ng-content` dont le `select` match un nœud **capture** ce nœud.
- Si plusieurs `ng-content` pourraient matcher, l’**ordre** de déclaration dans le template du composant est déterminant.

---

# 3) Fallback content & UX robuste

## 3.1 Fallback statique dans un slot
Vous pouvez fournir un contenu par défaut à l’intérieur de `ng-content`.

```html
<header class="card__header">
  <ng-content select="[cardTitle]">
    <h3 class="card__title">Titre par défaut</h3>
  </ng-content>
</header>
```

## 3.2 Fallback conditionnel (quand le slot est vide)
Parfois, vous voulez ajuster la structure si le slot est absent (ex : ne pas afficher le header).

Approche : utiliser une directive marker + `@ContentChild`.

### Directive marker

```ts
@Directive({ selector: '[cardTitle]' })
export class CardTitleMarkerDirective {}
```

### Composant

```ts
@Component({
  selector: 'app-card',
  template: `
    <article class="card">
      <header *ngIf="hasTitle" class="card__header">
        <ng-content select="[cardTitle]"></ng-content>
      </header>

      <section class="card__body">
        <ng-content></ng-content>
      </section>
    </article>
  `
})
export class CardComponent implements AfterContentInit {
  @ContentChild(CardTitleMarkerDirective) titleMarker?: CardTitleMarkerDirective;
  hasTitle = false;

  ngAfterContentInit(): void {
    this.hasTitle = !!this.titleMarker;
  }
}
```

Points à noter :
- `ngAfterContentInit` est le moment où la projection est « prête ».
- Si le contenu projeté change dynamiquement, vous devrez observer les changements (voir section 6).

---

# 4) Projection de templates (Headless UI)

La projection via `ng-content` projette des **nœuds** (éléments). Dans une approche plus avancée, vous projetez des **templates** afin que le composant conteneur puisse **décider quand et avec quel contexte** les rendre.

## 4.1 `ng-template` et `TemplateRef`

Côté consommateur :

```html
<app-dialog>
  <ng-template dialogTitle>Suppression</ng-template>

  <ng-template dialogBody let-item>
    Voulez-vous supprimer l’élément : <strong>{{ item.name }}</strong> ?
  </ng-template>

  <ng-template dialogFooter let-close="close">
    <button type="button" (click)="close()">Annuler</button>
    <button type="button" class="danger">Supprimer</button>
  </ng-template>
</app-dialog>
```

Ici, `dialogTitle`, `dialogBody`, `dialogFooter` seront des directives marker appliquées sur des `ng-template`.

## 4.2 Directives de template

```ts
@Directive({ selector: 'ng-template[dialogTitle]' })
export class DialogTitleTplDirective {
  constructor(public tpl: TemplateRef<unknown>) {}
}

@Directive({ selector: 'ng-template[dialogBody]' })
export class DialogBodyTplDirective<T = unknown> {
  constructor(public tpl: TemplateRef<T>) {}
}

@Directive({ selector: 'ng-template[dialogFooter]' })
export class DialogFooterTplDirective<C = unknown> {
  constructor(public tpl: TemplateRef<C>) {}
}
```

## 4.3 Composant `Dialog` avec `ngTemplateOutlet`

```ts
type DialogBodyContext<T> = { $implicit: T };
type DialogFooterContext = { close: () => void };

@Component({
  selector: 'app-dialog',
  template: `
    <div class="backdrop" (click)="close()"></div>

    <div class="dialog" role="dialog" aria-modal="true">
      <header class="dialog__header">
        <ng-container *ngIf="titleTpl; else defaultTitle"
          [ngTemplateOutlet]="titleTpl.tpl">
        </ng-container>
        <ng-template #defaultTitle><h2>Dialogue</h2></ng-template>
      </header>

      <section class="dialog__body">
        <ng-container *ngIf="bodyTpl; else defaultBody"
          [ngTemplateOutlet]="bodyTpl.tpl"
          [ngTemplateOutletContext]="bodyContext">
        </ng-container>
        <ng-template #defaultBody>Contenu par défaut</ng-template>
      </section>

      <footer class="dialog__footer">
        <ng-container *ngIf="footerTpl"
          [ngTemplateOutlet]="footerTpl.tpl"
          [ngTemplateOutletContext]="footerContext">
        </ng-container>
      </footer>
    </div>
  `
})
export class DialogComponent<T = unknown> {
  @Input() item!: T;

  @ContentChild(DialogTitleTplDirective) titleTpl?: DialogTitleTplDirective;
  @ContentChild(DialogBodyTplDirective) bodyTpl?: DialogBodyTplDirective<DialogBodyContext<T>>;
  @ContentChild(DialogFooterTplDirective) footerTpl?: DialogFooterTplDirective<DialogFooterContext>;

  get bodyContext(): DialogBodyContext<T> {
    return { $implicit: this.item };
  }

  get footerContext(): DialogFooterContext {
    return { close: () => this.close() };
  }

  close(): void {
    // logiques de fermeture: Output, service, etc.
  }
}
```

Bénéfice :
- Le composant contrôle le **moment** et les **données** du rendu.
- On peut construire des composants hautement réutilisables « headless ».

---

# 5) API de slots via directives “marker”

## 5.1 Pourquoi des directives marker ?
- API stable (évite les classes CSS sujettes à refactor)
- Possibilité d’ajouter du comportement ultérieurement (validation, injection, a11y)
- Permet une collecte facile via `@ContentChild(ren)`

### Exemple : `Tabs` déclaratives

Objectif :

```html
<app-tabs>
  <app-tab title="Général">Contenu A</app-tab>
  <app-tab title="Sécurité">Contenu B</app-tab>
</app-tabs>
```

Le composant `Tabs` détecte les `TabComponent` projetés.

```ts
@Component({
  selector: 'app-tab',
  template: `<ng-content></ng-content>`
})
export class TabComponent {
  @Input({ required: true }) title!: string;
  active = false;
}

@Component({
  selector: 'app-tabs',
  template: `
    <div class="tabs">
      <button
        *ngFor="let tab of tabs; let i = index"
        type="button"
        class="tab"
        [class.active]="tab.active"
        (click)="activate(i)">
        {{ tab.title }}
      </button>

      <section class="tabs__content">
        <ng-container *ngFor="let tab of tabs">
          <div *ngIf="tab.active" class="panel">
            <!-- On projette le contenu du tab actif via son template -->
            <ng-container [ngTemplateOutlet]="tabTpl" [ngTemplateOutletContext]="{ $implicit: tab }"></ng-container>
          </div>
        </ng-container>

        <ng-template #tabTpl let-tab>
          <!-- Ici on réutilise un ng-content? Non : le contenu est dans l’instance du composant Tab -->
          <!-- Approche simple : rendre TabComponent dans la vue via *ngIf et son template interne -->
          <ng-container *ngIf="tab.active">
            <!-- On rend directement le composant tab projeté -->
            <ng-container *ngComponentOutlet="tab.constructor"></ng-container>
          </ng-container>
        </ng-template>
      </section>
    </div>
  `
})
export class TabsComponent implements AfterContentInit {
  @ContentChildren(TabComponent) tabs!: QueryList<TabComponent>;

  ngAfterContentInit(): void {
    const first = this.tabs.first;
    if (first) first.active = true;
  }

  activate(index: number): void {
    this.tabs.forEach((t, i) => (t.active = i === index));
  }
}
```

> Note : L’exemple ci-dessus illustre l’intention ; en pratique, pour des tabs on rend généralement les `<app-tab>` projetés avec `*ngIf` (en gardant leur template) plutôt que `ngComponentOutlet`. Une implémentation plus réaliste est donnée en section 7.

---

# 6) Orchestration avancée : QueryList, lifecycles, changements dynamiques

## 6.1 `@ContentChildren` et `QueryList`
`@ContentChildren` récupère toutes les occurrences projetées correspondant à un type (composant/directive).

```ts
@ContentChildren(TabComponent) tabs!: QueryList<TabComponent>;
```

## 6.2 Écouter les changements
Si le consommateur ajoute/retire des éléments projetés (ex : `*ngIf`, `*ngFor`), le contenu projeté **évolue**.

```ts
import { Subscription } from 'rxjs';

export class TabsComponent implements AfterContentInit, OnDestroy {
  private sub = new Subscription();

  ngAfterContentInit(): void {
    this.sub.add(
      this.tabs.changes.subscribe(() => {
        // Recalculer l’onglet actif, relire les titres, etc.
        if (!this.tabs.some(t => t.active) && this.tabs.first) {
          this.tabs.first.active = true;
        }
      })
    );
  }

  ngOnDestroy(): void {
    this.sub.unsubscribe();
  }
}
```

## 6.3 `ngAfterContentChecked`
À utiliser avec parcimonie : appelé très souvent. Préférez des observables et des signaux (si vous utilisez Angular récent) ou des recalculs ciblés.

---

# 7) Cas pratiques (composants de librairie)

## 7.1 Card multi-slots (version « propre » via directives marker)

### Directives

```ts
@Directive({ selector: '[cardTitle]' })
export class CardTitleDirective {}

@Directive({ selector: '[cardActions]' })
export class CardActionsDirective {}
```

### Composant

```ts
@Component({
  selector: 'app-card',
  template: `
    <article class="card">
      <header *ngIf="hasTitle" class="card__header">
        <ng-content select="[cardTitle]"></ng-content>
      </header>

      <section class="card__body">
        <ng-content></ng-content>
      </section>

      <footer *ngIf="hasActions" class="card__actions">
        <ng-content select="[cardActions]"></ng-content>
      </footer>
    </article>
  `
})
export class CardComponent implements AfterContentInit {
  @ContentChild(CardTitleDirective) title?: CardTitleDirective;
  @ContentChild(CardActionsDirective) actions?: CardActionsDirective;

  hasTitle = false;
  hasActions = false;

  ngAfterContentInit(): void {
    this.hasTitle = !!this.title;
    this.hasActions = !!this.actions;
  }
}
```

### Usage

```html
<app-card>
  <h3 cardTitle>Facturation</h3>

  <p>Votre abonnement est actif.</p>

  <div cardActions>
    <button type="button">Gérer</button>
  </div>
</app-card>
```

## 7.2 Tabs réalistes via projection de composants

### `TabComponent`

```ts
@Component({
  selector: 'app-tab',
  template: `
    <div *ngIf="active" class="tab-panel" role="tabpanel">
      <ng-content></ng-content>
    </div>
  `
})
export class TabComponent {
  @Input({ required: true }) title!: string;
  active = false;
}
```

### `TabsComponent`

```ts
@Component({
  selector: 'app-tabs',
  template: `
    <div class="tabs" role="tablist">
      <button
        *ngFor="let tab of tabs.toArray(); let i = index"
        type="button"
        role="tab"
        [attr.aria-selected]="tab.active"
        (click)="activate(i)">
        {{ tab.title }}
      </button>

      <ng-content></ng-content>
    </div>
  `
})
export class TabsComponent implements AfterContentInit {
  @ContentChildren(TabComponent) tabs!: QueryList<TabComponent>;

  ngAfterContentInit(): void {
    if (this.tabs.first) this.tabs.first.active = true;
  }

  activate(index: number): void {
    this.tabs.forEach((t, i) => (t.active = i === index));
  }
}
```

Usage :

```html
<app-tabs>
  <app-tab title="Général">
    <p>Paramètres généraux...</p>
  </app-tab>

  <app-tab title="Sécurité">
    <p>2FA, mots de passe, etc.</p>
  </app-tab>
</app-tabs>
```

Ce pattern est très utilisé : **composant conteneur** + **enfants projetés** détectés via `ContentChildren`.

## 7.3 Dialog headless via templates

Cas où l’on veut :
- un composant de dialogue qui gère focus, overlay, fermeture
- mais dont le contenu est totalement customisable

Voir section 4 pour l’implémentation.

## 7.4 Table extensible : colonnes déclaratives (pattern bibliothèque)

Objectif :

```html
<app-table [data]="users">
  <ng-template appColumn="name" let-row>
    <strong>{{ row.name }}</strong>
  </ng-template>

  <ng-template appColumn="email" let-row>
    <a [href]="'mailto:' + row.email">{{ row.email }}</a>
  </ng-template>
</app-table>
```

### Directive colonne

```ts
@Directive({ selector: 'ng-template[appColumn]' })
export class ColumnDirective<T = unknown> {
  @Input('appColumn') key!: string;
  constructor(public tpl: TemplateRef<{ $implicit: T }>) {}
}
```

### Composant table

```ts
@Component({
  selector: 'app-table',
  template: `
    <table class="table">
      <thead>
        <tr>
          <th *ngFor="let col of columns">{{ col.key }}</th>
        </tr>
      </thead>

      <tbody>
        <tr *ngFor="let row of data">
          <td *ngFor="let col of columns">
            <ng-container
              [ngTemplateOutlet]="col.tpl"
              [ngTemplateOutletContext]="{ $implicit: row }">
            </ng-container>
          </td>
        </tr>
      </tbody>
    </table>
  `
})
export class TableComponent<T = unknown> implements AfterContentInit {
  @Input() data: T[] = [];

  @ContentChildren(ColumnDirective) columnList!: QueryList<ColumnDirective<T>>;
  columns: Array<ColumnDirective<T>> = [];

  ngAfterContentInit(): void {
    this.columns = this.columnList.toArray();

    this.columnList.changes.subscribe(() => {
      this.columns = this.columnList.toArray();
    });
  }
}
```

Ce pattern est un classique des bibliothèques Angular (tables, listes, menus, etc.).

---

# 8) Styles, accessibilité, performance, tests

## 8.1 Styles & encapsulation
### Points sensibles
- Le contenu projeté appartient au template du consommateur : les styles du composant hôte ne ciblent pas toujours ce contenu comme vous l’imaginez.
- Avec l’encapsulation par défaut (Emulated), les styles du composant sont scoppés.

### Bonnes pratiques
- Styliser via des wrappers internes (ex : `.card__body`) plutôt que cibler directement les éléments projetés.
- Prévoir des classes utilitaires/API (ex : `host classes`, inputs `variant`, `size`).

## 8.2 Accessibilité (A11y)
- `Tabs` : rôles `tablist`, `tab`, `tabpanel`, gestion `aria-selected`, navigation clavier
- `Dialog` : `role="dialog"`, `aria-modal="true"`, focus trap, fermeture Esc

> Pour un dialog robuste, pensez au CDK (`@angular/cdk/dialog` / `Overlay`) si votre projet le permet.

## 8.3 Performance
- Éviter `ngAfterContentChecked` pour faire des recalculs lourds.
- Préférer OnPush (ou signaux) + calculs dérivés.
- Sur de gros tableaux : `trackBy`, virtual scroll (CDK) si nécessaire.

## 8.4 Tests
### Tests de slots
- Vérifier que chaque slot reçoit le bon contenu.
- Vérifier les fallbacks (slot manquant).

### Exemple (pseudo-code)
- Créer un composant hôte de test qui consomme votre `Card`.
- Interroger le DOM rendu (`fixture.nativeElement`) et vérifier la présence dans les bons conteneurs.

---

# Exercices (avec corrections attendues)

## Exercice 1 — Card multi-slots
**Énoncé** : créer `app-card` avec slots title/actions + fallback.
- Si pas de title => pas de header.
- Si pas d’actions => pas de footer.

**Attendu** : utilisation de directives marker + `ContentChild`.

## Exercice 2 — Dialog template-based
**Énoncé** : créer `app-dialog` avec `ng-template` pour title/body/footer.
- Body doit recevoir un objet `item` en `$implicit`.
- Footer doit recevoir une fonction `close`.

**Attendu** : directives `ng-template[...]` + `TemplateRef` + `ngTemplateOutletContext`.

## Exercice 3 — Table à colonnes projetées
**Énoncé** : créer `app-table` avec colonnes déclarées par projection.
- N colonnes, affichage dynamique.
- Support des changements (colonnes ajoutées via `*ngIf`).

**Attendu** : `ContentChildren(ColumnDirective)` + écoute `changes`.

---

# Conclusion
La projection de contenu avancée transforme vos composants Angular en **briques réutilisables** :
- `ng-content` multi-slots pour une API simple et expressive
- templates projetés (`TemplateRef`) pour des composants « headless » et puissants
- directives marker + `ContentChild(ren)` pour une API stable, testable et évolutive

En combinant ces patterns, vous construisez des bibliothèques UI robustes, élégantes et maintenables.
