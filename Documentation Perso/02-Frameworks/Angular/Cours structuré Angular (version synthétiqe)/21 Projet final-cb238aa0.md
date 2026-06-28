# Formation Angular – Projet final : application complète (routing, API, formulaires, auth, gestion d’état)

**Public** : développeurs ayant les bases Angular (components, modules/standalone, services, RxJS de base)

**Objectif** : réaliser une application Angular complète de bout en bout, structurée et maintenable : navigation (routing), consommation d’API, formulaires réactifs, authentification (JWT), gestion d’état (store léger ou NgRx), tests et bonnes pratiques.

**Pré-requis** :
- Node.js LTS, npm/pnpm
- Angular CLI
- Connaissances TypeScript et RxJS (Observable, pipe, map, switchMap)

**Durée suggérée** : 2 à 4 jours (selon profondeur et options avancées)

---

## 0) Fil rouge du projet

Nous allons construire une application type **“TaskFlow”** (gestion de tâches / projets) :
- Inscription / connexion
- Liste de projets, sélection d’un projet
- CRUD de tâches (liste, création, modification, suppression)
- Filtrage, pagination
- Formulaires avec validation
- Authentification via JWT (token + refresh optionnel)
- Interception HTTP, guards, resolvers (option)
- Gestion d’état (store) : session + projets/tâches
- UI responsive simple

### 0.1) Architecture cible
- **Core** : singletons, interceptors, guards, services transverses
- **Shared** : composants/pipes/directives réutilisables
- **Features** : domaines métiers (auth, projects, tasks)
- **Env** : configuration via `environment.ts`

> Les exemples utilisent Angular moderne (standalone components + functional providers), mais l’approche est transposable en modules.

---

## 1) Initialisation du projet

### 1.1) Création
```bash
ng new taskflow --routing --style=scss
cd taskflow
```

### 1.2) Dépendances utiles
```bash
npm i @angular/material
npm i jwt-decode
```

Option gestion d’état avec NgRx :
```bash
npm i @ngrx/store @ngrx/effects @ngrx/store-devtools
```

### 1.3) Structure de dossiers (proposée)
```
src/
  app/
    core/
      guards/
      interceptors/
      services/
      models/
    shared/
      ui/
      pipes/
      directives/
    features/
      auth/
      projects/
      tasks/
    app.routes.ts
    app.component.ts
  environments/
```

### 1.4) Environnements
`src/environments/environment.ts`
```ts
export const environment = {
  production: false,
  apiUrl: 'http://localhost:3000/api'
};
```

---

## 2) Routing : navigation, lazy-loading, guards

### 2.1) Routes principales
Objectifs :
- pages publiques : `/login`, `/register`
- pages protégées : `/projects`, `/projects/:id/tasks`

`app/app.routes.ts`
```ts
import { Routes } from '@angular/router';
import { authGuard } from './core/guards/auth.guard';

export const routes: Routes = [
  {
    path: 'login',
    loadComponent: () => import('./features/auth/pages/login.page').then(m => m.LoginPage)
  },
  {
    path: 'register',
    loadComponent: () => import('./features/auth/pages/register.page').then(m => m.RegisterPage)
  },
  {
    path: 'projects',
    canActivate: [authGuard],
    loadChildren: () => import('./features/projects/projects.routes').then(r => r.PROJECT_ROUTES)
  },
  { path: '', pathMatch: 'full', redirectTo: 'projects' },
  { path: '**', loadComponent: () => import('./shared/ui/not-found.page').then(m => m.NotFoundPage) }
];
```

### 2.2) Lazy routes d’une feature
`features/projects/projects.routes.ts`
```ts
import { Routes } from '@angular/router';

export const PROJECT_ROUTES: Routes = [
  {
    path: '',
    loadComponent: () => import('./pages/projects.page').then(m => m.ProjectsPage)
  },
  {
    path: ':id/tasks',
    loadComponent: () => import('../tasks/pages/tasks.page').then(m => m.TasksPage)
  }
];
```

### 2.3) Guard d’authentification
Principe : si pas de token valide, redirection login.

`core/guards/auth.guard.ts`
```ts
import { CanActivateFn, Router } from '@angular/router';
import { inject } from '@angular/core';
import { SessionService } from '../services/session.service';

export const authGuard: CanActivateFn = () => {
  const session = inject(SessionService);
  const router = inject(Router);

  if (session.isAuthenticated()) return true;
  return router.parseUrl('/login');
};
```

---

## 3) Accès API : HttpClient, services, typage, erreurs

### 3.1) Modèles de données
`core/models/task.model.ts`
```ts
export interface Task {
  id: string;
  projectId: string;
  title: string;
  done: boolean;
  dueDate?: string; // ISO
}
```

`core/models/project.model.ts`
```ts
export interface Project {
  id: string;
  name: string;
  createdAt: string;
}
```

### 3.2) Service API (pattern repository)
`features/tasks/services/tasks-api.service.ts`
```ts
import { HttpClient, HttpParams } from '@angular/common/http';
import { Injectable } from '@angular/core';
import { environment } from '../../../environments/environment';
import { Observable } from 'rxjs';
import { Task } from '../../../app/core/models/task.model';

@Injectable({ providedIn: 'root' })
export class TasksApiService {
  private baseUrl = `${environment.apiUrl}/tasks`;
  constructor(private http: HttpClient) {}

  list(projectId: string, query?: { done?: boolean; q?: string }): Observable<Task[]> {
    let params = new HttpParams().set('projectId', projectId);
    if (query?.done !== undefined) params = params.set('done', String(query.done));
    if (query?.q) params = params.set('q', query.q);
    return this.http.get<Task[]>(this.baseUrl, { params });
  }

  create(payload: Partial<Task>): Observable<Task> {
    return this.http.post<Task>(this.baseUrl, payload);
  }

  update(id: string, patch: Partial<Task>): Observable<Task> {
    return this.http.patch<Task>(`${this.baseUrl}/${id}`, patch);
  }

  remove(id: string): Observable<void> {
    return this.http.delete<void>(`${this.baseUrl}/${id}`);
  }
}
```

### 3.3) Gestion centralisée des erreurs
Créer un service de notification + un interceptor d’erreur (voir section interceptors).

---

## 4) Authentification : login, token, interceptor, refresh (option)

### 4.1) Contrat API (exemple)
- `POST /auth/login` → `{ accessToken, user }`
- `POST /auth/register`
- `POST /auth/refresh` (option)

`core/models/user.model.ts`
```ts
export interface User {
  id: string;
  email: string;
  name: string;
}

export interface AuthResponse {
  accessToken: string;
  user: User;
}
```

### 4.2) SessionService
Responsable de :
- stocker token/user en mémoire + localStorage
- exposer `isAuthenticated()`
- fournir observable d’état

`core/services/session.service.ts`
```ts
import { Injectable } from '@angular/core';
import { BehaviorSubject } from 'rxjs';
import { User } from '../models/user.model';

const TOKEN_KEY = 'tf_token';
const USER_KEY = 'tf_user';

@Injectable({ providedIn: 'root' })
export class SessionService {
  private tokenSubject = new BehaviorSubject<string | null>(localStorage.getItem(TOKEN_KEY));
  private userSubject = new BehaviorSubject<User | null>(
    localStorage.getItem(USER_KEY) ? JSON.parse(localStorage.getItem(USER_KEY)!) : null
  );

  token$ = this.tokenSubject.asObservable();
  user$ = this.userSubject.asObservable();

  setSession(token: string, user: User) {
    localStorage.setItem(TOKEN_KEY, token);
    localStorage.setItem(USER_KEY, JSON.stringify(user));
    this.tokenSubject.next(token);
    this.userSubject.next(user);
  }

  clear() {
    localStorage.removeItem(TOKEN_KEY);
    localStorage.removeItem(USER_KEY);
    this.tokenSubject.next(null);
    this.userSubject.next(null);
  }

  getToken(): string | null {
    return this.tokenSubject.value;
  }

  isAuthenticated(): boolean {
    return !!this.getToken();
  }
}
```

### 4.3) AuthApiService
`features/auth/services/auth-api.service.ts`
```ts
import { HttpClient } from '@angular/common/http';
import { Injectable } from '@angular/core';
import { environment } from '../../../environments/environment';
import { Observable } from 'rxjs';
import { AuthResponse } from '../../../app/core/models/user.model';

@Injectable({ providedIn: 'root' })
export class AuthApiService {
  private baseUrl = `${environment.apiUrl}/auth`;
  constructor(private http: HttpClient) {}

  login(payload: { email: string; password: string }): Observable<AuthResponse> {
    return this.http.post<AuthResponse>(`${this.baseUrl}/login`, payload);
  }

  register(payload: { name: string; email: string; password: string }): Observable<AuthResponse> {
    return this.http.post<AuthResponse>(`${this.baseUrl}/register`, payload);
  }
}
```

### 4.4) Interceptor JWT
Ajoute `Authorization: Bearer <token>`.

`core/interceptors/auth-token.interceptor.ts`
```ts
import { HttpInterceptorFn } from '@angular/common/http';
import { inject } from '@angular/core';
import { SessionService } from '../services/session.service';

export const authTokenInterceptor: HttpInterceptorFn = (req, next) => {
  const session = inject(SessionService);
  const token = session.getToken();

  if (!token) return next(req);

  const authReq = req.clone({
    setHeaders: { Authorization: `Bearer ${token}` }
  });
  return next(authReq);
};
```

Dans `main.ts` (ou app.config) :
```ts
import { provideHttpClient, withInterceptors } from '@angular/common/http';
import { authTokenInterceptor } from './app/core/interceptors/auth-token.interceptor';

// ...
provideHttpClient(withInterceptors([authTokenInterceptor]));
```

### 4.5) Interceptor d’erreurs (401, 403…)
`core/interceptors/http-error.interceptor.ts`
```ts
import { HttpErrorResponse, HttpInterceptorFn } from '@angular/common/http';
import { inject } from '@angular/core';
import { catchError, throwError } from 'rxjs';
import { Router } from '@angular/router';
import { SessionService } from '../services/session.service';

export const httpErrorInterceptor: HttpInterceptorFn = (req, next) => {
  const router = inject(Router);
  const session = inject(SessionService);

  return next(req).pipe(
    catchError((err: unknown) => {
      if (err instanceof HttpErrorResponse) {
        if (err.status === 401) {
          session.clear();
          router.navigateByUrl('/login');
        }
      }
      return throwError(() => err);
    })
  );
};
```

---

## 5) Formulaires réactifs : validation, UX, composants

### 5.1) LoginPage
Objectifs :
- `FormGroup` + validateurs
- feedback UI (touched/dirty)
- appel API, gestion loading, navigation

`features/auth/pages/login.page.ts`
```ts
import { Component, inject } from '@angular/core';
import { FormBuilder, Validators, ReactiveFormsModule } from '@angular/forms';
import { Router } from '@angular/router';
import { CommonModule } from '@angular/common';
import { AuthApiService } from '../services/auth-api.service';
import { SessionService } from '../../../app/core/services/session.service';

@Component({
  standalone: true,
  imports: [CommonModule, ReactiveFormsModule],
  template: `
  <h1>Connexion</h1>

  <form [formGroup]="form" (ngSubmit)="submit()">
    <label>Email</label>
    <input type="email" formControlName="email" />
    <small *ngIf="form.controls.email.touched && form.controls.email.invalid">
      Email invalide
    </small>

    <label>Mot de passe</label>
    <input type="password" formControlName="password" />
    <small *ngIf="form.controls.password.touched && form.controls.password.invalid">
      8 caractères minimum
    </small>

    <button type="submit" [disabled]="form.invalid || loading">
      {{ loading ? 'Connexion…' : 'Se connecter' }}
    </button>
  </form>
  `
})
export class LoginPage {
  private fb = inject(FormBuilder);
  private authApi = inject(AuthApiService);
  private session = inject(SessionService);
  private router = inject(Router);

  loading = false;

  form = this.fb.group({
    email: ['', [Validators.required, Validators.email]],
    password: ['', [Validators.required, Validators.minLength(8)]]
  });

  submit() {
    if (this.form.invalid) {
      this.form.markAllAsTouched();
      return;
    }

    this.loading = true;
    this.authApi.login(this.form.getRawValue() as any).subscribe({
      next: (res) => {
        this.session.setSession(res.accessToken, res.user);
        this.router.navigateByUrl('/projects');
      },
      error: () => {
        this.loading = false;
      },
      complete: () => {
        this.loading = false;
      }
    });
  }
}
```

### 5.2) Formulaire de tâche (composant réutilisable)
But : isoler le formulaire et le rendre composable (create/edit).

`features/tasks/components/task-form.component.ts`
```ts
import { Component, EventEmitter, Input, Output, inject } from '@angular/core';
import { CommonModule } from '@angular/common';
import { FormBuilder, ReactiveFormsModule, Validators } from '@angular/forms';
import { Task } from '../../../app/core/models/task.model';

@Component({
  standalone: true,
  imports: [CommonModule, ReactiveFormsModule],
  selector: 'tf-task-form',
  template: `
  <form [formGroup]="form" (ngSubmit)="save()">
    <label>Titre</label>
    <input formControlName="title" />
    <small *ngIf="form.controls.title.touched && form.controls.title.invalid">
      Le titre est requis (3 caractères min.)
    </small>

    <label>
      <input type="checkbox" formControlName="done" /> Terminée
    </label>

    <label>Échéance</label>
    <input type="date" formControlName="dueDate" />

    <button type="submit" [disabled]="form.invalid">Enregistrer</button>
  </form>
  `
})
export class TaskFormComponent {
  private fb = inject(FormBuilder);

  @Input() set value(v: Task | null) {
    if (!v) return;
    this.form.patchValue({
      title: v.title,
      done: v.done,
      dueDate: v.dueDate ? v.dueDate.substring(0, 10) : ''
    });
  }

  @Output() submitted = new EventEmitter<{ title: string; done: boolean; dueDate?: string | null }>();

  form = this.fb.group({
    title: ['', [Validators.required, Validators.minLength(3)]],
    done: [false, [Validators.required]],
    dueDate: ['']
  });

  save() {
    if (this.form.invalid) {
      this.form.markAllAsTouched();
      return;
    }
    this.submitted.emit(this.form.getRawValue() as any);
  }
}
```

---

## 6) Feature Projects : page liste + sélection

### 6.1) ProjectsApiService (exemple)
`features/projects/services/projects-api.service.ts`
```ts
import { HttpClient } from '@angular/common/http';
import { Injectable } from '@angular/core';
import { environment } from '../../../environments/environment';
import { Observable } from 'rxjs';
import { Project } from '../../../app/core/models/project.model';

@Injectable({ providedIn: 'root' })
export class ProjectsApiService {
  private baseUrl = `${environment.apiUrl}/projects`;
  constructor(private http: HttpClient) {}

  list(): Observable<Project[]> {
    return this.http.get<Project[]>(this.baseUrl);
  }

  create(payload: { name: string }): Observable<Project> {
    return this.http.post<Project>(this.baseUrl, payload);
  }
}
```

### 6.2) ProjectsPage
- charge la liste
- navigue vers les tâches du projet

`features/projects/pages/projects.page.ts`
```ts
import { Component, inject } from '@angular/core';
import { CommonModule } from '@angular/common';
import { RouterModule, Router } from '@angular/router';
import { ProjectsApiService } from '../services/projects-api.service';
import { Project } from '../../../app/core/models/project.model';

@Component({
  standalone: true,
  imports: [CommonModule, RouterModule],
  template: `
  <h1>Projets</h1>

  <button (click)="create()">Nouveau projet</button>

  <ul>
    <li *ngFor="let p of projects" (click)="open(p)">
      <strong>{{ p.name }}</strong>
      <small>— {{ p.createdAt | date:'short' }}</small>
    </li>
  </ul>
  `
})
export class ProjectsPage {
  private api = inject(ProjectsApiService);
  private router = inject(Router);

  projects: Project[] = [];

  ngOnInit() {
    this.api.list().subscribe(p => (this.projects = p));
  }

  open(p: Project) {
    this.router.navigate(['/projects', p.id, 'tasks']);
  }

  create() {
    const name = prompt('Nom du projet ?');
    if (!name) return;
    this.api.create({ name }).subscribe(() => this.ngOnInit());
  }
}
```

> En production, remplacer `prompt()` par un dialogue (Angular Material) ou un composant de formulaire.

---

## 7) Feature Tasks : CRUD + query params + UI

### 7.1) TasksPage
Fonctionnalités :
- récupère `projectId` depuis l’URL
- charge les tâches
- filtre (terminées / recherche)
- crée/modifie/supprime

`features/tasks/pages/tasks.page.ts`
```ts
import { Component, inject } from '@angular/core';
import { CommonModule } from '@angular/common';
import { ActivatedRoute, RouterModule } from '@angular/router';
import { switchMap } from 'rxjs';
import { TasksApiService } from '../services/tasks-api.service';
import { TaskFormComponent } from '../components/task-form.component';
import { Task } from '../../../app/core/models/task.model';

@Component({
  standalone: true,
  imports: [CommonModule, RouterModule, TaskFormComponent],
  template: `
  <a routerLink="/projects">← Retour projets</a>
  <h1>Tâches</h1>

  <section>
    <label>Recherche</label>
    <input [value]="q" (input)="setQuery(($any($event.target)).value)" />

    <label>
      <input type="checkbox" [checked]="doneOnly" (change)="toggleDone(($any($event.target)).checked)" />
      Terminées uniquement
    </label>
  </section>

  <hr />

  <h2>Nouvelle tâche</h2>
  <tf-task-form (submitted)="create($event)"></tf-task-form>

  <hr />

  <ul>
    <li *ngFor="let t of tasks">
      <label>
        <input type="checkbox" [checked]="t.done" (change)="setDone(t, ($any($event.target)).checked)" />
        <strong [style.textDecoration]="t.done ? 'line-through' : 'none'">{{ t.title }}</strong>
      </label>
      <small *ngIf="t.dueDate">(échéance: {{ t.dueDate | date:'shortDate' }})</small>
      <button (click)="remove(t)">Supprimer</button>
    </li>
  </ul>
  `
})
export class TasksPage {
  private route = inject(ActivatedRoute);
  private api = inject(TasksApiService);

  projectId!: string;
  tasks: Task[] = [];

  q = '';
  doneOnly = false;

  ngOnInit() {
    this.route.paramMap
      .pipe(
        switchMap((params) => {
          this.projectId = params.get('id')!;
          return this.api.list(this.projectId);
        })
      )
      .subscribe((tasks) => (this.tasks = tasks));
  }

  private reload() {
    this.api
      .list(this.projectId, { q: this.q || undefined, done: this.doneOnly ? true : undefined })
      .subscribe((tasks) => (this.tasks = tasks));
  }

  setQuery(v: string) {
    this.q = v;
    this.reload();
  }

  toggleDone(checked: boolean) {
    this.doneOnly = checked;
    this.reload();
  }

  create(payload: { title: string; done: boolean; dueDate?: string | null }) {
    this.api
      .create({
        projectId: this.projectId,
        title: payload.title,
        done: payload.done,
        dueDate: payload.dueDate || undefined
      })
      .subscribe(() => this.reload());
  }

  setDone(task: Task, done: boolean) {
    this.api.update(task.id, { done }).subscribe(() => this.reload());
  }

  remove(task: Task) {
    if (!confirm('Supprimer cette tâche ?')) return;
    this.api.remove(task.id).subscribe(() => this.reload());
  }
}
```

---

## 8) Gestion d’état : stratégie et implémentation

Deux approches pédagogiques possibles :
1. **Store léger** (BehaviorSubject) : simple, excellent pour formations courtes.
2. **NgRx** : standard enterprise, plus verbeux.

### 8.1) Store léger (recommandé pour ce projet)
Cas d’usage :
- stocker la liste de projets/tâches
- éviter de recharger après chaque action
- centraliser les side effects

#### 8.1.1) Exemple : tasks.store
`features/tasks/state/tasks.store.ts`
```ts
import { Injectable } from '@angular/core';
import { BehaviorSubject, finalize } from 'rxjs';
import { Task } from '../../../app/core/models/task.model';
import { TasksApiService } from '../services/tasks-api.service';

interface TasksState {
  loading: boolean;
  tasks: Task[];
  error?: string;
}

@Injectable({ providedIn: 'root' })
export class TasksStore {
  private stateSubject = new BehaviorSubject<TasksState>({ loading: false, tasks: [] });
  state$ = this.stateSubject.asObservable();

  constructor(private api: TasksApiService) {}

  get snapshot() {
    return this.stateSubject.value;
  }

  load(projectId: string, query?: { done?: boolean; q?: string }) {
    this.stateSubject.next({ ...this.snapshot, loading: true, error: undefined });
    this.api
      .list(projectId, query)
      .pipe(finalize(() => this.stateSubject.next({ ...this.snapshot, loading: false })))
      .subscribe({
        next: (tasks) => this.stateSubject.next({ ...this.snapshot, tasks }),
        error: () => this.stateSubject.next({ ...this.snapshot, error: 'Erreur de chargement' })
      });
  }
}
```

#### 8.1.2) Consommation dans TasksPage
- abonnement à `state$`
- appels `store.load()`

### 8.2) Variante NgRx (option)
Livrables :
- actions `loadTasks`, `loadTasksSuccess`, `loadTasksFailure`
- reducer + selectors
- effects (API calls)
- devtools

> Ajouter NgRx si le contexte entreprise le demande.

---

## 9) UX & qualité : loaders, désabonnements, performance

### 9.1) AsyncPipe et signaux (approche moderne)
- Remplacer les `subscribe()` manuels par `async` dans le template dès que possible.
- Utiliser `takeUntilDestroyed()` (Angular 16+) pour éviter fuites mémoire.

### 9.2) Stratégie de détection
- `ChangeDetectionStrategy.OnPush` sur composants UI.
- Segmenter l’UI en petits composants.

### 9.3) Accessibilité et formulaires
- associer `label`/`for`
- messages d’erreur explicites

---

## 10) Tests : unitaires + intégration légère

### 10.1) Test d’un service API (HttpTestingController)
`tasks-api.service.spec.ts` (extrait)
```ts
import { TestBed } from '@angular/core/testing';
import { provideHttpClient } from '@angular/common/http';
import { provideHttpClientTesting, HttpTestingController } from '@angular/common/http/testing';
import { TasksApiService } from './tasks-api.service';

describe('TasksApiService', () => {
  let service: TasksApiService;
  let httpMock: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [provideHttpClient(), provideHttpClientTesting()]
    });
    service = TestBed.inject(TasksApiService);
    httpMock = TestBed.inject(HttpTestingController);
  });

  it('list() should call GET /tasks', () => {
    service.list('p1').subscribe();

    const req = httpMock.expectOne(r => r.method === 'GET' && r.url.includes('/tasks'));
    expect(req.request.params.get('projectId')).toBe('p1');
    req.flush([]);
  });
});
```

### 10.2) Test composant (form)
- vérifier validation
- vérifier émission `submitted`

---

## 11) Sécurité & bonnes pratiques

- Ne jamais stocker des secrets dans le front
- Token en mémoire si possible (risque XSS vs CSRF selon stratégie)
- Interceptor 401 centralisé
- Limiter la duplication des appels API
- Logger uniquement en dev

---

## 12) Packaging & déploiement

### 12.1) Build
```bash
ng build
```

### 12.2) Variables d’environnement
- `environment.prod.ts` → `apiUrl` prod

### 12.3) Hébergement
- static hosting (Nginx, S3, Vercel…)
- rewrite vers `index.html` pour le routing SPA

Exemple Nginx :
```nginx
location / {
  try_files $uri $uri/ /index.html;
}
```

---

# Ateliers (pas à pas)

## Atelier A — Bootstrapping + routing
**Livrable** : app navigable, pages login/register/projects/tasks.

1. Générer pages (standalone)
2. Configurer routes + lazy loading
3. Page NotFound
4. Navigation de base

## Atelier B — Auth complète
**Livrable** : login stocke token + guard protège `/projects`.

1. Formulaire login
2. SessionService
3. Interceptor token
4. Guard + redirection

## Atelier C — CRUD + formulaires
**Livrable** : création/suppression/édition des tâches.

1. Service API tasks
2. TaskForm réutilisable
3. TasksPage + interactions

## Atelier D — Gestion d’état
**Livrable** : store léger ou NgRx.

1. Définir state
2. Charger et sélectionner
3. Optimiser les rechargements

## Atelier E — Qualité (tests + erreurs)
**Livrable** : interceptors d’erreurs + 2 tests.

1. Interceptor erreurs 401
2. Test d’un service
3. Test d’un formulaire

---

# Check-list de fin de projet

- [ ] Routing lazy + guard OK
- [ ] Auth : token + logout + 401 handling
- [ ] API : services typés, gestion erreurs
- [ ] Formulaires : validations + UX
- [ ] Gestion d’état (au moins session + tâches)
- [ ] Tests minimum (1 service + 1 composant)
- [ ] Build prod et configuration déploiement

---

# Annexes

## A) Idées d’améliorations
- Refresh token + rotation
- Pagination serveur
- Recherche debounce (`debounceTime`)
- Resolver pour précharger
- Skeleton loaders

## B) Conventions de commit (exemple)
- `feat: add tasks store`
- `fix: handle 401 redirect`
- `test: add tasks api tests`

---

**Fin de formation – Projet final Angular**
