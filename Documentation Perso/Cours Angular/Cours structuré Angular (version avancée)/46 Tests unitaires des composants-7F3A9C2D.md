# Formation Angular — Tests unitaires des composants (Jasmine/Karma ou Jest)

- **Référence** : Angular (composants), TestBed, fixtures, DebugElement
- **Public** : développeurs Angular (débutant à intermédiaire en tests)
- **Prérequis** : TypeScript, bases Angular (components, @Input/@Output), CLI
- **Durée suggérée** : 1 journée (6–7h) ou 2×3h
- **Objectif principal** : écrire des tests unitaires de composants qui vérifient **le comportement observable** (DOM, interactions, inputs/outputs, comportement local) plutôt que l’implémentation interne.

---

## 0. Introduction : pourquoi tester un composant ?

### 0.1. Ce que couvre un test unitaire de composant
Un **test unitaire de composant Angular** vise à vérifier :

1. **Le rendu** : ce qui apparaît (ou non) dans le DOM selon l’état.
2. **Les interactions utilisateur** : clic, saisie, sélection, raccourcis, événements.
3. **Les `@Input()`** : une donnée entrante modifie l’UI et/ou la logique.
4. **Les `@Output()`** : un événement sortant est émis au bon moment et avec la bonne valeur.
5. **Le comportement métier local** : règles simples encapsulées dans le composant (ex: validation, calcul d’affichage, gestion d’état).

Ce que ce n’est pas :
- Des tests d’intégration E2E (navigation, backend réel, etc.).
- Des tests de services complexes (ça se fait, mais c’est une autre séance).

### 0.2. Principe clé : tester le comportement observable
**Bon réflexe** : tester **ce que l’utilisateur et les composants parents peuvent observer** :
- le DOM,
- les événements émis,
- les appels à un service (quand le composant orchestre une action),
- les changements accessibles.

Éviter autant que possible :
- tester des méthodes privées,
- tester l’ordre interne d’exécution,
- tester des détails d’implémentation (ex: `component.someFlag = true`), sauf si cela se reflète dans le comportement.

---

## 1. Environnement de test Angular

### 1.1. Test runner et frameworks
Angular CLI installe typiquement :
- **Jasmine + Karma** (historiquement par défaut)
- ou **Jest** (de plus en plus courant)

Le contenu ci-dessous reste très proche entre les deux :
- `describe/it/expect`
- spies (`spyOn`, `jest.fn()` / `jest.spyOn`)
- assertions.

### 1.2. Anatomie d’un test de composant
Un test typique comprend :

- **Arrange** : configuration du module de test, création du component
- **Act** : simuler une action (input, clic, changement de valeur)
- **Assert** : vérifier rendu/sorties.

### 1.3. Les outils Angular : `TestBed`, `ComponentFixture`, `DebugElement`
- `TestBed` : crée un mini-module Angular pour le test
- `ComponentFixture<T>` : encapsule le composant + son template et donne accès aux APIs
- `fixture.detectChanges()` : lance la détection de changements et déclenche `ngOnInit` (au premier appel)
- `fixture.nativeElement` : accès DOM brut
- `fixture.debugElement` : accès Angular DebugElement (pratique pour query + triggerEventHandler)

Exemple minimal :

```ts
import { TestBed } from '@angular/core/testing';
import { ComponentFixture } from '@angular/core/testing';
import { MyComponent } from './my.component';

describe('MyComponent', () => {
  let fixture: ComponentFixture<MyComponent>;
  let component: MyComponent;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [],
      declarations: [MyComponent],
      providers: [],
    }).compileComponents();

    fixture = TestBed.createComponent(MyComponent);
    component = fixture.componentInstance;
  });

  it('should create', () => {
    fixture.detectChanges();
    expect(component).toBeTruthy();
  });
});
```

---

## 2. Le composant support (fil rouge)

On va utiliser un composant de démonstration, suffisamment riche pour couvrir : rendu, input, output, interaction, règles locales.

### 2.1. Spécifications fonctionnelles
Composant : `CounterCardComponent`

- Affiche un titre (input)
- Affiche un compteur
- Bouton `+` incrémente
- Bouton `-` décrémente (mais jamais sous `min`)
- Bouton `Reset` remet à la valeur initiale
- Émet `countChange` à chaque modification
- Désactive `-` si `count === min`

### 2.2. Implémentation (exemple)

```ts
import { Component, EventEmitter, Input, Output } from '@angular/core';

@Component({
  selector: 'app-counter-card',
  template: `
    <section class="card">
      <h2 data-testid="title">{{ title }}</h2>

      <p data-testid="count">{{ count }}</p>

      <button data-testid="dec" (click)="decrement()" [disabled]="count <= min">-</button>
      <button data-testid="inc" (click)="increment()">+</button>
      <button data-testid="reset" (click)="reset()">Reset</button>

      <small data-testid="range">min: {{ min }}</small>
    </section>
  `,
})
export class CounterCardComponent {
  @Input() title = 'Counter';
  @Input() initialCount = 0;
  @Input() min = 0;

  @Output() countChange = new EventEmitter<number>();

  count = 0;

  ngOnInit() {
    this.count = this.initialCount;
  }

  increment() {
    this.count++;
    this.countChange.emit(this.count);
  }

  decrement() {
    if (this.count <= this.min) return;
    this.count--;
    this.countChange.emit(this.count);
  }

  reset() {
    this.count = this.initialCount;
    this.countChange.emit(this.count);
  }
}
```

> Remarque : on n’essaie pas de tester chaque ligne, mais le comportement observable.

---

## 3. Tester le rendu (DOM)

### 3.1. Vérifier qu’un texte apparaît
On teste le DOM après `detectChanges()`.

```ts
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { CounterCardComponent } from './counter-card.component';

describe('CounterCardComponent — rendu', () => {
  let fixture: ComponentFixture<CounterCardComponent>;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      declarations: [CounterCardComponent],
    }).compileComponents();

    fixture = TestBed.createComponent(CounterCardComponent);
  });

  it('affiche le titre et le compteur', () => {
    // Arrange
    fixture.componentInstance.title = 'Mon compteur';
    fixture.componentInstance.initialCount = 5;

    // Act
    fixture.detectChanges();

    // Assert
    const el: HTMLElement = fixture.nativeElement;
    expect(el.querySelector('[data-testid="title"]')?.textContent).toContain('Mon compteur');
    expect(el.querySelector('[data-testid="count"]')?.textContent).toContain('5');
  });
});
```

### 3.2. Bonnes pratiques DOM
- Utiliser des sélecteurs stables : `data-testid` > classes CSS susceptibles de changer.
- Vérifier ce que **voit l’utilisateur** (texte, attributs, visuel) et pas l’état interne.

---

## 4. Tester les interactions utilisateur (clic, saisie)

### 4.1. Simuler un clic

```ts
it('incrémente le compteur via le bouton +', () => {
  fixture.componentInstance.initialCount = 1;
  fixture.detectChanges();

  const el: HTMLElement = fixture.nativeElement;
  const incButton = el.querySelector('[data-testid="inc"]') as HTMLButtonElement;

  incButton.click();
  fixture.detectChanges();

  expect(el.querySelector('[data-testid="count"]')?.textContent).toContain('2');
});
```

### 4.2. Vérifier désactivation d’un bouton

```ts
it('désactive le bouton - quand count <= min', () => {
  fixture.componentInstance.initialCount = 0;
  fixture.componentInstance.min = 0;
  fixture.detectChanges();

  const el: HTMLElement = fixture.nativeElement;
  const decButton = el.querySelector('[data-testid="dec"]') as HTMLButtonElement;

  expect(decButton.disabled).toBeTrue();
});
```

### 4.3. Interactions et stabilité
- Après interaction, appeler `fixture.detectChanges()`.
- Si async (observables, timers), préférer `fakeAsync/tick` ou `waitForAsync/whenStable`.

---

## 5. Tester les `@Input()`

Les inputs changent le rendu et/ou la logique. On veut tester :
- « quand le parent fournit X, le composant affiche Y »

### 5.1. Input influençant le rendu

```ts
it('affiche min dans le template', () => {
  fixture.componentInstance.min = 10;
  fixture.detectChanges();

  const el: HTMLElement = fixture.nativeElement;
  expect(el.querySelector('[data-testid="range"]')?.textContent).toContain('10');
});
```

### 5.2. Input influençant l’état initial

```ts
it('initialise count depuis initialCount au premier detectChanges', () => {
  fixture.componentInstance.initialCount = 7;
  fixture.detectChanges();

  const el: HTMLElement = fixture.nativeElement;
  expect(el.querySelector('[data-testid="count"]')?.textContent).toContain('7');
});
```

> Si votre composant réagit aux changements d’input via `ngOnChanges`, vous pouvez mettre à jour l’input et rappeler `fixture.detectChanges()` pour vérifier le nouveau rendu.

---

## 6. Tester les `@Output()`

Un output est une API publique du composant : le test doit vérifier **l’événement** et sa **valeur**.

### 6.1. Spy sur `EventEmitter.emit`

```ts
it('émet countChange quand on incrémente', () => {
  fixture.componentInstance.initialCount = 0;
  fixture.detectChanges();

  const component = fixture.componentInstance;
  const spy = spyOn(component.countChange, 'emit');

  const el: HTMLElement = fixture.nativeElement;
  (el.querySelector('[data-testid="inc"]') as HTMLButtonElement).click();

  expect(spy).toHaveBeenCalledWith(1);
});
```

### 6.2. Tester plusieurs émissions

```ts
it('émet à chaque modification', () => {
  fixture.componentInstance.initialCount = 1;
  fixture.detectChanges();

  const component = fixture.componentInstance;
  const spy = spyOn(component.countChange, 'emit');

  const el: HTMLElement = fixture.nativeElement;
  (el.querySelector('[data-testid="inc"]') as HTMLButtonElement).click(); // 2
  (el.querySelector('[data-testid="inc"]') as HTMLButtonElement).click(); // 3

  expect(spy.calls.allArgs()).toEqual([[2], [3]]);
});
```

---

## 7. Tester le comportement métier local (règles)

Ici la règle : ne pas descendre sous `min`.

### 7.1. Vérifier l’absence de changement observable

```ts
it('ne décrémente pas sous min et n’émet pas', () => {
  fixture.componentInstance.initialCount = 0;
  fixture.componentInstance.min = 0;
  fixture.detectChanges();

  const component = fixture.componentInstance;
  const spy = spyOn(component.countChange, 'emit');

  const el: HTMLElement = fixture.nativeElement;
  (el.querySelector('[data-testid="dec"]') as HTMLButtonElement).click();
  fixture.detectChanges();

  expect(el.querySelector('[data-testid="count"]')?.textContent).toContain('0');
  expect(spy).not.toHaveBeenCalled();
});
```

### 7.2. Tester `reset` comme cas métier

```ts
it('reset remet la valeur initiale et émet', () => {
  fixture.componentInstance.initialCount = 5;
  fixture.detectChanges();

  const component = fixture.componentInstance;
  const spy = spyOn(component.countChange, 'emit');

  const el: HTMLElement = fixture.nativeElement;
  (el.querySelector('[data-testid="inc"]') as HTMLButtonElement).click();
  (el.querySelector('[data-testid="inc"]') as HTMLButtonElement).click();

  (el.querySelector('[data-testid="reset"]') as HTMLButtonElement).click();
  fixture.detectChanges();

  expect(el.querySelector('[data-testid="count"]')?.textContent).toContain('5');
  expect(spy).toHaveBeenCalledWith(5);
});
```

---

## 8. Composants avec dépendances : stubs, doubles et spies

Un composant réel dépend souvent :
- de services (HTTP, state)
- de router
- de pipes
- de composants enfants

### 8.1. Mock de service : stratégie
Objectif : isoler le composant.
- Fournir un **fake** (objet simple) via `providers`.
- Vérifier les appels (spy) et les effets observables.

Exemple : composant `UserBadgeComponent` qui charge un user via `UserService`.

#### Service
```ts
export interface User {
  id: string;
  name: string;
}

export abstract class UserService {
  abstract getUser(id: string): Observable<User>;
}
```

#### Composant
```ts
@Component({
  selector: 'app-user-badge',
  template: `
    <ng-container *ngIf="user$ | async as user">
      <span data-testid="name">{{ user.name }}</span>
    </ng-container>
  `,
})
export class UserBadgeComponent {
  @Input() userId!: string;
  user$!: Observable<User>;

  constructor(private userService: UserService) {}

  ngOnInit() {
    this.user$ = this.userService.getUser(this.userId);
  }
}
```

#### Test avec stub
```ts
import { of } from 'rxjs';

describe('UserBadgeComponent', () => {
  it('affiche le nom venant du service', async () => {
    const userServiceStub = {
      getUser: (id: string) => of({ id, name: 'Ada' }),
    };

    await TestBed.configureTestingModule({
      declarations: [UserBadgeComponent],
      providers: [{ provide: UserService, useValue: userServiceStub }],
    }).compileComponents();

    const fixture = TestBed.createComponent(UserBadgeComponent);
    fixture.componentInstance.userId = '42';
    fixture.detectChanges();

    const el: HTMLElement = fixture.nativeElement;
    expect(el.querySelector('[data-testid="name"]')?.textContent).toContain('Ada');
  });
});
```

### 8.2. Vérifier l’appel au service (comportement d’orchestration)

```ts
it('appelle getUser avec userId', async () => {
  const userServiceStub = {
    getUser: jasmine.createSpy('getUser').and.returnValue(of({ id: '42', name: 'Ada' })),
  };

  await TestBed.configureTestingModule({
    declarations: [UserBadgeComponent],
    providers: [{ provide: UserService, useValue: userServiceStub }],
  }).compileComponents();

  const fixture = TestBed.createComponent(UserBadgeComponent);
  fixture.componentInstance.userId = '42';
  fixture.detectChanges();

  expect(userServiceStub.getUser).toHaveBeenCalledWith('42');
});
```

---

## 9. Tester l’asynchrone (Observables, Promises, timers)

### 9.1. `async` pipe et `whenStable()`
Si un rendu dépend d’une Promise ou d’une microtask, utilisez :

```ts
await fixture.whenStable();
fixture.detectChanges();
```

### 9.2. `fakeAsync` et `tick()`
Pour timers (setTimeout) ou debounces.

```ts
import { fakeAsync, tick } from '@angular/core/testing';

it('met à jour après 300ms', fakeAsync(() => {
  // ... action qui déclenche un timer
  tick(300);
  fixture.detectChanges();
  // ... assert
}));
```

### 9.3. Conseil
- Préférer `fakeAsync/tick` quand vous voulez contrôler le temps.
- Préférer des streams déterministes (ex: `of(...)`) pour des unit tests.

---

## 10. Tester un composant avec enfants : shallow vs deep

### 10.1. Shallow test (recommandé en unit)
But : tester le composant **sans** rendre réellement les enfants.

Techniques :
- Utiliser `NO_ERRORS_SCHEMA` (ignore les éléments inconnus)
- ou déclarer des **stubs** de composants enfant.

Exemple rapide :

```ts
import { NO_ERRORS_SCHEMA } from '@angular/core';

await TestBed.configureTestingModule({
  declarations: [ParentComponent],
  schemas: [NO_ERRORS_SCHEMA],
}).compileComponents();
```

### 10.2. Deep test (plus proche intégration)
Déclarer enfants réels + leurs dépendances : plus fragile, utile ponctuellement.

---

## 11. Stratégie de test : recommandations concrètes

### 11.1. Pyramide de test (rappel)
- Beaucoup de tests unitaires rapides (composants isolés)
- Quelques tests d’intégration (assemblage)
- Peu d’e2e (parcours clés)

### 11.2. Ce qu’on teste en priorité sur un composant
1. **Cas nominal** : rendu initial, interaction principale.
2. **Cas limites** : min/max, champs vides, désactivations.
3. **Contrats d’API** : inputs/outputs.
4. **Erreurs gérées** (si pertinent) : message d’erreur rendu, fallback.

### 11.3. Ce qu’on évite
- Tester la présence d’une méthode
- Tester des propriétés internes sans effet observable
- Trop de snapshots non justifiés (fragiles)

---

## 12. Atelier guidé (exercices)

### Exercice 1 — Rendu conditionnel
Ajouter au composant un message :
- si `count === min` afficher `"Minimum atteint"`.

**À tester** :
- visible quand `count === min`
- absent sinon.

### Exercice 2 — Output métier
Émettre `reachedMin = new EventEmitter<void>()` quand `count` atteint `min` après un decrement.

**À tester** :
- `reachedMin.emit()` est appelé exactement au moment attendu.

### Exercice 3 — Service + erreur
Simuler un service qui renvoie une erreur (ex: `throwError`) et afficher un message.

**À tester** :
- rendu du message d’erreur
- absence d’affichage du nom.

---

## 13. Checklist de validation (avant de merger)

- [ ] Chaque test a une intention claire (« doit … »)
- [ ] Sélecteurs DOM stables (`data-testid`)
- [ ] Peu de couplage à l’implémentation
- [ ] Tests rapides (stubs synchrones)
- [ ] Cas limites couverts
- [ ] Pas de tests redondants

---

## 14. Annexes

### A. Gabarit AAA

```ts
it('doit ...', () => {
  // Arrange

  // Act

  // Assert
});
```

### B. Notes Jest (si migration)
- Remplacer `jasmine.createSpy` par `jest.fn()` ou `jest.spyOn`
- Assertions : `toHaveBeenCalledWith` identiques
- Setup via `jest-preset-angular`

---

## Conclusion
Les tests unitaires de composants Angular doivent valider : **rendu**, **interactions**, **inputs/outputs**, et **règles locales**, en privilégiant toujours la **vérification du comportement observable**. Cette approche rend les tests plus robustes, moins fragiles face aux refactorings et plus utiles pour maintenir la qualité du front.
