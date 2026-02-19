# 5. Performanta si Optimizare in Angular

> Ghid complet pentru Angular Principal Engineer -- acoperind Change Detection,
> Zoneless Angular, Lazy Loading, @defer, Bundle Optimization, SSR, Hydration,
> Virtual Scrolling, Web Workers si metrici de performanta.

---

## Cuprins

1. [Change Detection](#1-change-detection)
2. [Zoneless Angular (fara Zone.js)](#2-zoneless-angular-fara-zonejs)
3. [Lazy Loading](#3-lazy-loading)
4. [@defer Blocks (Angular 17+)](#4-defer-blocks-angular-17)
5. [Bundle Size Optimization si Tree Shaking](#5-bundle-size-optimization-si-tree-shaking)
6. [trackBy in @for / ngFor](#6-trackby-in-for--ngfor)
7. [Virtual Scrolling](#7-virtual-scrolling)
8. [Web Workers](#8-web-workers)
9. [SSR (Server-Side Rendering) si Hydration](#9-ssr-server-side-rendering-si-hydration)
10. [Metrici de Performanta](#10-metrici-de-performanta)
11. [Chrome DevTools si Angular DevTools](#11-chrome-devtools-si-angular-devtools)
12. [Noutati Performanta Angular 20-21](#noutati-performanta-angular-20-21)
13. [Intrebari frecvente de interviu](#intrebari-frecvente-de-interviu)

---

## 1. Change Detection

Change Detection (CD) este mecanismul prin care Angular sincronizeaza starea
aplicatiei (modelul) cu DOM-ul (view-ul). Intelegerea profunda a acestui mecanism
este esentiala pentru orice Principal Engineer, deoarece controlul CD este principala
parghie de optimizare a performantei in Angular.

### 1.1 Strategia Default

Cu strategia **Default** (`ChangeDetectionStrategy.Default`), Angular verifica
**intregul arbore de componente** de sus in jos la fiecare ciclu de Change Detection.
Orice eveniment async (click, HTTP response, setTimeout) triggereaza un ciclu complet.

```typescript
@Component({
  selector: 'app-parent',
  // changeDetection: ChangeDetectionStrategy.Default  <-- implicit
  template: `
    <h1>{{ title }}</h1>
    <app-child [data]="items"></app-child>
    <app-sibling></app-sibling>
  `
})
export class ParentComponent {
  title = 'Dashboard';
  items = [1, 2, 3];

  addItem() {
    // Chiar daca doar ParentComponent se modifica,
    // Angular va verifica si ChildComponent si SiblingComponent.
    this.items.push(4);
  }
}
```

**Problema:** Intr-o aplicatie cu sute de componente, verificarea intregului arbore
la fiecare click de mouse este extrem de costisitoare.

### 1.2 Strategia OnPush

Cu **OnPush** (`ChangeDetectionStrategy.OnPush`), Angular verifica o componenta
**doar** in urmatoarele situatii:

1. **Referinta unui `@Input()` se schimba** (nu mutatia interna a obiectului)
2. **Un eveniment DOM** este emis din interiorul componentei (click, keyup etc.)
3. **`async` pipe** primeste o noua valoare dintr-un Observable/Promise
4. **Un signal** citit in template isi schimba valoarea
5. **`markForCheck()`** este apelat manual pe `ChangeDetectorRef`

```typescript
@Component({
  selector: 'app-user-card',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <div class="card">
      <h3>{{ user().name }}</h3>
      <p>Email: {{ user().email }}</p>
      <p>Status: {{ status$ | async }}</p>
      <button (click)="refresh()">Refresh</button>
    </div>
  `
})
export class UserCardComponent {
  // Signal input -- schimbarea referintei triggereaza CD
  user = input.required<User>();

  // Observable cu async pipe -- noua emisie triggereaza CD
  status$ = inject(UserService).getStatus(this.user().id);

  refresh() {
    // Eveniment DOM din interiorul componentei -- triggereaza CD
    console.log('Refreshing...');
  }
}
```

**Exemplu: De ce mutatia nu functioneaza cu OnPush**

```typescript
@Component({
  selector: 'app-list',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <ul>
      @for (item of items(); track item.id) {
        <li>{{ item.name }}</li>
      }
    </ul>
  `
})
export class ListComponent {
  items = input.required<Item[]>();
}

// In parinte:
@Component({
  template: `<app-list [items]="myItems" />`
})
export class ParentComponent {
  myItems: Item[] = [{ id: 1, name: 'A' }, { id: 2, name: 'B' }];

  addItemWrong() {
    // GRESIT: mutatia array-ului NU schimba referinta
    // OnPush NU va detecta aceasta schimbare!
    this.myItems.push({ id: 3, name: 'C' });
  }

  addItemCorrect() {
    // CORECT: cream o noua referinta prin spread
    this.myItems = [...this.myItems, { id: 3, name: 'C' }];
  }
}
```

### 1.3 Cum Zone.js triggereaza Change Detection

**Zone.js** este o biblioteca care face monkey-patching la TOATE API-urile asincrone
ale browser-ului (setTimeout, Promise, addEventListener, XMLHttpRequest, fetch etc.).
Angular creeaza o zona speciala numita **NgZone**. Orice operatie asincrona care
ruleaza in aceasta zona triggereaza automat `ApplicationRef.tick()`, care porneste
un ciclu complet de Change Detection.

```
                    Eveniment async (click, HTTP, timer)
                                |
                                v
                     Zone.js intercepteaza
                                |
                                v
                     NgZone.onMicrotaskEmpty
                                |
                                v
                     ApplicationRef.tick()
                                |
                                v
                  Change Detection (root -> leaves)
```

```typescript
@Component({
  selector: 'app-zone-demo',
  template: `<p>Counter: {{ counter }}</p>`
})
export class ZoneDemoComponent {
  counter = 0;
  private ngZone = inject(NgZone);

  startTimer() {
    // Ruleaza IN zona Angular -- triggereaza CD la fiecare iteratie
    setInterval(() => {
      this.counter++;
    }, 1000);
  }

  startTimerOptimized() {
    // Ruleaza IN AFARA zonei Angular -- NU triggereaza CD
    this.ngZone.runOutsideAngular(() => {
      setInterval(() => {
        this.counter++;
        // Cand vrem sa actualizam UI-ul, revenim in zona:
        if (this.counter % 10 === 0) {
          this.ngZone.run(() => {
            // Acum Angular va face Change Detection
          });
        }
      }, 100);
    });
  }
}
```

### 1.4 ChangeDetectorRef API

`ChangeDetectorRef` ofera control granular asupra Change Detection la nivel de
componenta individuala.

```typescript
@Component({
  selector: 'app-manual-cd',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <p>Data: {{ data }}</p>
    <p>Websocket messages: {{ messageCount }}</p>
  `
})
export class ManualCdComponent implements OnInit, OnDestroy {
  data = '';
  messageCount = 0;

  private cdr = inject(ChangeDetectorRef);
  private wsService = inject(WebSocketService);
  private destroy$ = new Subject<void>();

  ngOnInit() {
    // --- detectChanges() ---
    // Triggereaza CD sincron, imediat, DOAR pentru aceasta componenta
    // si copiii sai. Util cand stii exact cand s-a schimbat ceva.
    this.wsService.messages$
      .pipe(takeUntil(this.destroy$))
      .subscribe(msg => {
        this.data = msg.payload;
        this.cdr.detectChanges(); // Forteaza verificare ACUM
      });

    // --- markForCheck() ---
    // Marcheaza componenta si TOTI stramosii ca "dirty".
    // CD va rula la urmatorul ciclu (nu imediat).
    // Preferat in cele mai multe cazuri -- mai sigur decat detectChanges().
    this.wsService.counter$
      .pipe(takeUntil(this.destroy$))
      .subscribe(count => {
        this.messageCount = count;
        this.cdr.markForCheck(); // Marcheaza pentru urmatorul ciclu
      });
  }

  // --- detach() si reattach() ---
  // Scoate componenta complet din arborele de CD.
  // Util pentru componente care se actualizeaza extrem de rar.
  pauseUpdates() {
    this.cdr.detach();
    // Componenta NU va mai fi verificata de Angular deloc.
    // Nici OnPush triggers nu vor functiona.
  }

  resumeUpdates() {
    this.cdr.reattach();
    this.cdr.markForCheck(); // Asiguram ca se verifica imediat
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

**Diferenta critica: `detectChanges()` vs `markForCheck()`**

| Aspect | `detectChanges()` | `markForCheck()` |
|--------|-------------------|------------------|
| Executie | Sincrona, imediat | Asincrona, urmatorul ciclu |
| Scop | Componenta + copii | Marcheaza pana la root |
| Risc | Poate cauza `ExpressionChangedAfterItHasBeenChecked` | Sigur, respecta fluxul normal |
| Cand | Cand ai nevoie de update imediat (ex: test) | In 90% din cazuri -- recomandarea oficiala |

### 1.5 Exemplu complet: OnPush cu imutabilitate

```typescript
// Interfata imutabila
interface TodoItem {
  readonly id: number;
  readonly text: string;
  readonly completed: boolean;
}

@Component({
  selector: 'app-todo-list',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  imports: [TodoItemComponent],
  template: `
    <h2>Todos ({{ todos().length }})</h2>
    @for (todo of todos(); track todo.id) {
      <app-todo-item
        [todo]="todo"
        (toggle)="onToggle($event)"
        (remove)="onRemove($event)"
      />
    }
  `
})
export class TodoListComponent {
  todos = input.required<TodoItem[]>();
  toggle = output<number>();
  remove = output<number>();

  onToggle(id: number) {
    this.toggle.emit(id);
  }

  onRemove(id: number) {
    this.remove.emit(id);
  }
}

@Component({
  selector: 'app-todo-item',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <div [class.completed]="todo().completed">
      <span>{{ todo().text }}</span>
      <button (click)="toggle.emit(todo().id)">Toggle</button>
      <button (click)="remove.emit(todo().id)">Remove</button>
    </div>
  `
})
export class TodoItemComponent {
  todo = input.required<TodoItem>();
  toggle = output<number>();
  remove = output<number>();
  // OnPush + signal input = Angular verifica DOAR cand todo() primeste
  // o noua referinta. Celelalte todo-uri din lista nu sunt re-verificate.
}

// In container (smart component):
@Component({
  selector: 'app-todo-container',
  standalone: true,
  imports: [TodoListComponent],
  template: `
    <app-todo-list
      [todos]="todos()"
      (toggle)="toggleTodo($event)"
      (remove)="removeTodo($event)"
    />
  `
})
export class TodoContainerComponent {
  private store = inject(TodoStore);
  todos = this.store.todos; // signal din store

  toggleTodo(id: number) {
    // Store-ul creeaza un nou array (imutabil) =>
    // referinta se schimba => OnPush detecteaza
    this.store.toggle(id);
  }

  removeTodo(id: number) {
    this.store.remove(id);
  }
}
```

---

## 2. Zoneless Angular (fara Zone.js)

### 2.1 Conceptul Zoneless

Incepand cu Angular 18+, exista suport experimental (stabil din Angular 19) pentru
a rula aplicatii **fara Zone.js**. In loc sa se bazeze pe monkey-patching-ul
API-urilor async, Angular foloseste **signals** ca mecanism primar de notificare
a schimbarilor.

```
            Zone-based                          Zoneless
     ========================           ========================
     Event -> Zone.js patch             Event -> Signal update
          -> NgZone                          -> Notification
          -> tick()                          -> Schedule CD
          -> Full tree check                 -> Targeted CD
```

### 2.2 Activarea Zoneless Change Detection

```typescript
// main.ts
import { bootstrapApplication } from '@angular/platform-browser';
import { provideExperimentalZonelessChangeDetection } from '@angular/core';
import { AppComponent } from './app/app.component';

bootstrapApplication(AppComponent, {
  providers: [
    // Inlocuieste complet provideZoneChangeDetection()
    provideExperimentalZonelessChangeDetection(),
    // ... alti provideri
  ]
});
```

```json
// angular.json -- eliminam Zone.js din polyfills
{
  "projects": {
    "my-app": {
      "architect": {
        "build": {
          "options": {
            "polyfills": [
              // "zone.js"  <-- ELIMINAT
            ]
          }
        }
      }
    }
  }
}
```

### 2.3 Cum signals conduc Change Detection fara Zone.js

In modul zoneless, Angular programeaza Change Detection cand:

1. Un **signal** citit in template isi schimba valoarea
2. Un **input** al componentei se schimba
3. Un **eveniment DOM** este emis din template
4. **`async` pipe** primeste o valoare noua
5. **`markForCheck()`** este apelat manual
6. **`ApplicationRef.tick()`** este apelat direct

```typescript
@Component({
  selector: 'app-counter',
  standalone: true,
  // Cu zoneless, OnPush nu mai este strict necesar,
  // dar este recomandat pentru claritate.
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <p>Count: {{ count() }}</p>
    <p>Double: {{ doubleCount() }}</p>
    <button (click)="increment()">+1</button>
    <button (click)="decrement()">-1</button>
  `
})
export class CounterComponent {
  // Signal -- Angular stie exact CAND si UNDE s-a schimbat
  count = signal(0);

  // Computed signal -- se recalculeaza automat
  doubleCount = computed(() => this.count() * 2);

  increment() {
    // Actualizarea signal-ului notifica Angular sa programeze CD
    // DOAR pentru componentele care citesc acest signal
    this.count.update(c => c + 1);
  }

  decrement() {
    this.count.update(c => c - 1);
  }
}
```

### 2.4 Migrare de la Zone-based la Zoneless

**Pasul 1: Adoptarea signals in componente existente**

```typescript
// INAINTE -- zone-based, proprietati simple
@Component({
  template: `<p>{{ userName }}</p>`
})
export class ProfileComponent implements OnInit {
  userName = '';

  constructor(private userService: UserService) {}

  ngOnInit() {
    this.userService.getUser().subscribe(user => {
      this.userName = user.name;
      // Zone.js detecteaza subscribe-ul si triggereaza CD
    });
  }
}

// DUPA -- signal-based, pregatit pentru zoneless
@Component({
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `<p>{{ userName() }}</p>`
})
export class ProfileComponent {
  private userService = inject(UserService);

  // Varianta 1: toSignal (converteste Observable -> Signal)
  private user = toSignal(this.userService.getUser());
  userName = computed(() => this.user()?.name ?? '');

  // Varianta 2: Signal writable + effect
  // userName = signal('');
  // constructor() {
  //   this.userService.getUser().subscribe(user => {
  //     this.userName.set(user.name);
  //   });
  // }
}
```

**Pasul 2: Eliminarea dependentelor de NgZone**

```typescript
// INAINTE
export class MapComponent {
  constructor(private ngZone: NgZone) {}

  initMap() {
    this.ngZone.runOutsideAngular(() => {
      // Cod third-party
      this.map = new ThirdPartyMap();
      this.map.onReady(() => {
        this.ngZone.run(() => {
          this.ready = true;
        });
      });
    });
  }
}

// DUPA -- cu zoneless nu mai exista NgZone
export class MapComponent {
  ready = signal(false);

  initMap() {
    // Third-party codul nu mai triggereaza CD inutil
    // deoarece nu exista Zone.js care sa intercepteze
    this.map = new ThirdPartyMap();
    this.map.onReady(() => {
      // Signal-ul notifica Angular direct
      this.ready.set(true);
    });
  }
}
```

**Pasul 3: Verificarea codului care depinde de timing-ul Zone.js**

```typescript
// ATENTIE: setTimeout/setInterval NU mai triggereaza CD automat!
// INAINTE (zone-based) -- functiona automat
setInterval(() => {
  this.clock = new Date(); // Zone.js detecta si triggera CD
}, 1000);

// DUPA (zoneless) -- trebuie signal
clock = signal(new Date());

constructor() {
  setInterval(() => {
    this.clock.set(new Date()); // Signal notifica Angular
  }, 1000);
}
```

### 2.5 Beneficiile Zoneless

| Beneficiu | Detalii |
|-----------|---------|
| **Bundle mai mic** | ~15-20 KB mai putin (Zone.js eliminat din bundle) |
| **Performanta superioara** | Fara monkey-patching la API-uri native; CD targetat |
| **Debugging simplificat** | Stack trace-uri curate, fara cadre Zone.js |
| **Compatibilitate** | Mai buna cu Web Components, micro-frontends, third-party libs |
| **Predictibilitate** | CD ruleaza DOAR cand signals se schimba, nu la orice event async |
| **SSR mai performant** | Fara overhead Zone.js pe server |

---

## 3. Lazy Loading

Lazy loading amana incarcarea modulelor/componentelor pana cand sunt necesare,
reducand dramatic bundle-ul initial si timpul de incarcare.

### 3.1 Lazy Loading la nivel de ruta

**Cu `loadComponent` (standalone components -- recomandat)**

```typescript
// app.routes.ts
export const routes: Routes = [
  {
    path: '',
    loadComponent: () => import('./home/home.component')
      .then(m => m.HomeComponent),
  },
  {
    path: 'dashboard',
    loadComponent: () => import('./dashboard/dashboard.component')
      .then(m => m.DashboardComponent),
    canActivate: [authGuard],
  },
  {
    path: 'admin',
    loadComponent: () => import('./admin/admin-layout.component')
      .then(m => m.AdminLayoutComponent),
    // Rute copil lazy-loaded individual
    loadChildren: () => import('./admin/admin.routes')
      .then(m => m.ADMIN_ROUTES),
  },
  {
    path: 'reports',
    // Lazy loading cu un fisier de rute separat
    loadChildren: () => import('./reports/reports.routes')
      .then(m => m.REPORT_ROUTES),
  }
];

// admin/admin.routes.ts
export const ADMIN_ROUTES: Routes = [
  {
    path: '',
    // Fiecare ruta copil este un chunk separat
    loadComponent: () => import('./admin-dashboard.component')
      .then(m => m.AdminDashboardComponent),
  },
  {
    path: 'users',
    loadComponent: () => import('./users/users.component')
      .then(m => m.UsersComponent),
  },
  {
    path: 'settings',
    loadComponent: () => import('./settings/settings.component')
      .then(m => m.SettingsComponent),
  }
];
```

**Cu `loadChildren` si NgModule (legacy, dar inca valid)**

```typescript
{
  path: 'legacy-feature',
  loadChildren: () => import('./legacy-feature/legacy-feature.module')
    .then(m => m.LegacyFeatureModule),
}
```

### 3.2 Dynamic Import Syntax

Fiecare `import()` creaza un **chunk** separat in procesul de build. Webpack / esbuild
analizeaza aceste expresii si produce fisiere JS separate care sunt incarcate on-demand.

```typescript
// Rezultat build:
// main.js          -- 150 KB (app shell + rute eager)
// chunk-ABCD.js    -- 45 KB  (dashboard)
// chunk-EFGH.js    -- 30 KB  (admin)
// chunk-IJKL.js    -- 20 KB  (reports)
```

### 3.3 Strategii de Preloading

Preloading incarca modulele lazy in background DUPA ce aplicatia initiala s-a
incarcat, astfel incat navigarea ulterioara este instantanee.

**PreloadAllModules -- preincarca TOTUL**

```typescript
// app.config.ts
import { provideRouter, withPreloading, PreloadAllModules } from '@angular/router';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(
      routes,
      withPreloading(PreloadAllModules)
    ),
  ]
};
```

**Strategie custom -- preincarca selectiv**

```typescript
import { PreloadingStrategy, Route } from '@angular/router';
import { Observable, of, timer } from 'rxjs';
import { mergeMap } from 'rxjs/operators';

@Injectable({ providedIn: 'root' })
export class SelectivePreloadStrategy implements PreloadingStrategy {
  preload(route: Route, load: () => Observable<any>): Observable<any> {
    // Preincarca DOAR rutele marcate cu data.preload = true
    if (route.data?.['preload']) {
      // Optional: delay pentru a nu bloca incarcarea initiala
      const delay = route.data?.['preloadDelay'] ?? 2000;
      return timer(delay).pipe(mergeMap(() => load()));
    }
    return of(null);
  }
}

// Utilizare in rute:
export const routes: Routes = [
  {
    path: 'dashboard',
    loadComponent: () => import('./dashboard/dashboard.component')
      .then(m => m.DashboardComponent),
    data: { preload: true, preloadDelay: 1000 }, // Se preincarca dupa 1s
  },
  {
    path: 'admin',
    loadComponent: () => import('./admin/admin.component')
      .then(m => m.AdminComponent),
    data: { preload: false }, // NU se preincarca
  },
];

// Configurare:
provideRouter(routes, withPreloading(SelectivePreloadStrategy))
```

**Strategie bazata pe conexiune (Network-Aware)**

```typescript
@Injectable({ providedIn: 'root' })
export class NetworkAwarePreloadStrategy implements PreloadingStrategy {
  preload(route: Route, load: () => Observable<any>): Observable<any> {
    // Verificam tipul conexiunii
    const connection = (navigator as any).connection;

    if (connection) {
      // Nu preincarcam pe conexiuni lente sau cu data saver
      if (connection.saveData || connection.effectiveType === '2g') {
        return of(null);
      }
    }

    // Pe conexiuni rapide, preincarcam rutele marcate
    return route.data?.['preload'] ? load() : of(null);
  }
}
```

---

## 4. @defer Blocks (Angular 17+)

`@defer` este un mecanism declarativ de lazy loading la nivel de template. Permite
amanarea incarcarii de componente, directive si pipe-uri pana cand o conditie este
indeplinita, cu suport built-in pentru placeholder, loading si error states.

### 4.1 Sintaxa de baza si triggere

```typescript
@Component({
  selector: 'app-dashboard',
  standalone: true,
  imports: [HeavyChartComponent, DataTableComponent, AnalyticsWidget],
  template: `
    <h1>Dashboard</h1>

    <!-- Trigger: on idle -- se incarca cand browser-ul este idle -->
    @defer (on idle) {
      <app-heavy-chart [data]="chartData()" />
    } @placeholder {
      <div class="chart-skeleton">Se incarca graficul...</div>
    } @loading (minimum 300ms) {
      <div class="spinner">Loading chart...</div>
    } @error {
      <div class="error">Eroare la incarcarea graficului.</div>
    }

    <!-- Trigger: on viewport -- se incarca cand elementul devine vizibil -->
    @defer (on viewport) {
      <app-data-table [rows]="tableData()" />
    } @placeholder {
      <div class="table-skeleton" style="height: 400px">
        Tabelul se va incarca cand devine vizibil...
      </div>
    }

    <!-- Trigger: on interaction -- se incarca la click/focus/input -->
    @defer (on interaction) {
      <app-analytics-widget />
    } @placeholder {
      <button>Incarca Analytics</button>
    }

    <!-- Trigger: on hover -- se incarca la mouse hover -->
    @defer (on hover) {
      <app-tooltip-content />
    } @placeholder {
      <span class="help-icon">?</span>
    }

    <!-- Trigger: on timer -- se incarca dupa un timp specificat -->
    @defer (on timer(5s)) {
      <app-promotional-banner />
    } @placeholder {
      <div class="banner-placeholder"></div>
    }

    <!-- Trigger: when -- se incarca cand conditia devine true -->
    @defer (when isLoggedIn()) {
      <app-user-preferences />
    } @placeholder {
      <p>Logheaza-te pentru a vedea preferintele.</p>
    }
  `
})
export class DashboardComponent {
  chartData = signal<ChartData[]>([]);
  tableData = signal<Row[]>([]);
  isLoggedIn = signal(false);
}
```

### 4.2 Prefetch Triggers

Prefetch separa **descarcarea** codului de **randarea** lui. Codul se descarca
devreme, dar se randeaza doar cand trigger-ul principal este indeplinit.

```typescript
@Component({
  standalone: true,
  imports: [ProductReviewsComponent, RecommendationsComponent],
  template: `
    <!-- Prefetch pe idle, render pe viewport -->
    <!-- Codul se descarca in background, dar componenta se randeaza
         doar cand scroll-ul ajunge la ea -->
    @defer (on viewport; prefetch on idle) {
      <app-product-reviews [productId]="productId()" />
    } @placeholder {
      <div class="reviews-placeholder" style="height: 600px">
        Reviews...
      </div>
    }

    <!-- Prefetch pe hover, render pe interaction (click) -->
    <!-- Utilizatorul face hover => codul se descarca
         Utilizatorul face click => componenta se randeaza instant -->
    @defer (on interaction; prefetch on hover) {
      <app-recommendations />
    } @placeholder {
      <button class="show-recommendations">
        Arata Recomandari
      </button>
    }

    <!-- Prefetch cu timer, render conditionat -->
    @defer (when showDetails(); prefetch on timer(3s)) {
      <app-detailed-analytics />
    } @placeholder {
      <button (click)="showDetails.set(true)">
        Vezi Detalii Analytics
      </button>
    }
  `
})
export class ProductPageComponent {
  productId = input.required<string>();
  showDetails = signal(false);
}
```

### 4.3 @placeholder, @loading, @error -- detalii

```typescript
template: `
  @defer (on viewport) {
    <app-heavy-component />
  }

  // @placeholder -- afisata INAINTE de orice incarcare
  // Accepta 'minimum' -- durata minima de afisare (previne flickering)
  @placeholder (minimum 500ms) {
    <div class="skeleton-loader">
      <div class="skeleton-line"></div>
      <div class="skeleton-line short"></div>
    </div>
  }

  // @loading -- afisata IN TIMPUL incarcarii
  // 'after' -- delay inainte de a arata loading (nu arata spinner daca e rapid)
  // 'minimum' -- durata minima de afisare (previne flickering)
  @loading (after 200ms; minimum 500ms) {
    <div class="loading-indicator">
      <mat-spinner diameter="40" />
      <p>Se incarca...</p>
    </div>
  }

  // @error -- afisata daca incarcarea esueaza (network error etc.)
  @error {
    <div class="error-state">
      <mat-icon>error</mat-icon>
      <p>Nu s-a putut incarca componenta.</p>
      <button (click)="retry()">Reincearca</button>
    </div>
  }
`
```

### 4.4 Exemplu practic: pagina de produs cu @defer

```typescript
@Component({
  selector: 'app-product-page',
  standalone: true,
  imports: [
    ProductHeaderComponent,
    ProductGalleryComponent,
    ProductReviewsComponent,
    RelatedProductsComponent,
    ProductVideoComponent,
    SizeGuideComponent,
  ],
  template: `
    <!-- Componentele critice (above the fold) -- NU sunt defer-uite -->
    <app-product-header [product]="product()" />

    <!-- Galeria -- defer pe idle (probabil vizibila, dar poate fi greoaie) -->
    @defer (on idle; prefetch on idle) {
      <app-product-gallery [images]="product().images" />
    } @placeholder (minimum 200ms) {
      <div class="gallery-skeleton" style="height: 500px; background: #f0f0f0">
      </div>
    }

    <!-- Reviews -- defer pe viewport (sub fold) -->
    @defer (on viewport; prefetch on idle) {
      <app-product-reviews [productId]="product().id" />
    } @placeholder {
      <div class="reviews-skeleton" style="height: 300px">
        <h3>Recenzii</h3>
        <p>Scroll pentru a vedea recenziile...</p>
      </div>
    }

    <!-- Video -- defer pe interactiune (utilizatorul trebuie sa dea click) -->
    @defer (on interaction; prefetch on hover) {
      <app-product-video [videoUrl]="product().videoUrl" />
    } @placeholder {
      <div class="video-placeholder">
        <img [src]="product().videoThumbnail" alt="Video thumbnail" />
        <button class="play-button">Reda Video</button>
      </div>
    }

    <!-- Size Guide -- defer cand utilizatorul da click pe link -->
    @defer (when showSizeGuide(); prefetch on idle) {
      <app-size-guide [category]="product().category" />
    } @placeholder {
      <a (click)="showSizeGuide.set(true)">Ghid de marimi</a>
    }

    <!-- Related Products -- defer pe viewport, la sfarsitul paginii -->
    @defer (on viewport; prefetch on timer(5s)) {
      <app-related-products [categoryId]="product().categoryId" />
    } @placeholder {
      <div class="related-skeleton" style="height: 250px">
        Produse similare...
      </div>
    }
  `
})
export class ProductPageComponent {
  product = input.required<Product>();
  showSizeGuide = signal(false);
}
```

---

## 5. Bundle Size Optimization si Tree Shaking

### 5.1 Angular Budgets in angular.json

Budget-urile definesc limite pentru dimensiunea bundle-ului. Build-ul va emite
**warning** sau **error** cand limitele sunt depasite.

```json
{
  "projects": {
    "my-app": {
      "architect": {
        "build": {
          "configurations": {
            "production": {
              "budgets": [
                {
                  "type": "initial",
                  "maximumWarning": "500kB",
                  "maximumError": "1MB"
                },
                {
                  "type": "anyComponentStyle",
                  "maximumWarning": "4kB",
                  "maximumError": "8kB"
                },
                {
                  "type": "anyScript",
                  "maximumWarning": "100kB",
                  "maximumError": "200kB"
                }
              ]
            }
          }
        }
      }
    }
  }
}
```

Tipuri de budget-uri:
- **`initial`** -- bundle-ul initial (main + polyfills + styles)
- **`allScript`** -- toate script-urile (inclusiv lazy chunks)
- **`anyScript`** -- orice script individual
- **`anyComponentStyle`** -- stilul oricarei componente
- **`bundle`** -- un bundle specific (prin `name`)

### 5.2 Analiza Bundle-ului

```bash
# Source map explorer -- vizualizare treemap
npm install --save-dev source-map-explorer

# Build cu source maps
ng build --source-map

# Analizeaza
npx source-map-explorer dist/my-app/browser/main.js

# Alternativ: webpack-bundle-analyzer (pentru webpack builder)
npm install --save-dev webpack-bundle-analyzer

# esbuild (default din Angular 17+) -- stats.json
ng build --stats-json
npx webpack-bundle-analyzer dist/my-app/browser/stats.json
```

### 5.3 Eliminarea importurilor neutilizate

```typescript
// GRESIT -- importam intregul moment.js (~300 KB!)
import moment from 'moment';
const date = moment().format('YYYY-MM-DD');

// CORECT -- folosim date-fns (tree-shakable) sau Temporal API
import { format } from 'date-fns';
const date = format(new Date(), 'yyyy-MM-dd');

// GRESIT -- importam intregul lodash (~70 KB)
import _ from 'lodash';
const result = _.groupBy(items, 'category');

// CORECT -- import individual (tree-shakable)
import groupBy from 'lodash-es/groupBy';
const result = groupBy(items, 'category');

// GRESIT -- importam intregul RxJS (legacy)
import { Observable, map, filter } from 'rxjs';

// CORECT (de fapt, din RxJS 7+ importurile sunt deja tree-shakable)
// Dar atentie la operatorii neutilizati importati dar nefolositi
import { map, filter } from 'rxjs/operators'; // tree-shakable
```

### 5.4 providedIn: 'root' pentru servicii tree-shakable

```typescript
// RECOMANDAT -- tree-shakable
// Daca serviciul nu este injectat nicaieri, Angular il elimina din bundle
@Injectable({ providedIn: 'root' })
export class AnalyticsService {
  trackEvent(event: string) { /* ... */ }
}

// NE-RECOMANDAT -- NU este tree-shakable
// Serviciul va fi inclus in bundle chiar daca nu este folosit
@NgModule({
  providers: [AnalyticsService] // Mereu inclus in bundle
})
export class CoreModule {}

// OPTIONAL: providedIn la un scop specific
@Injectable({ providedIn: 'platform' }) // Shared intre aplicatii
export class SharedCacheService {}
```

### 5.5 Standalone Components reduc bundle-ul

```typescript
// Cu NgModules -- dependintele intregului modul sunt incluse
@NgModule({
  declarations: [ComponentA, ComponentB, ComponentC],
  imports: [
    CommonModule,      // Include TOTUL din CommonModule
    ReactiveFormsModule, // Include TOTUL, chiar daca folosim doar FormControl
    MatButtonModule,
    MatTableModule,
    MatDialogModule,   // Inclus chiar daca doar ComponentC il foloseste
  ]
})
export class FeatureModule {}

// Cu Standalone -- fiecare componenta importa DOAR ce are nevoie
@Component({
  standalone: true,
  imports: [
    // Import individual -- tree-shakable
    MatButton,           // Doar butonul, nu intregul modul
    DatePipe,            // Doar DatePipe, nu intregul CommonModule
    ReactiveFormsModule, // Inclus doar in aceasta componenta
  ],
  template: `...`
})
export class ComponentAStandalone {}
```

---

## 6. trackBy in @for / ngFor

### 6.1 De ce tracking-ul este important

Cand Angular randeaza o lista si acea lista se schimba (adaugare, stergere,
reordonare), Angular trebuie sa decida ce elemente DOM sa reutilizeze si ce
elemente sa recreeze. Fara tracking, Angular compara elementele **prin referinta**
-- daca primeste un nou array (chiar cu aceleasi date), distruge TOATE elementele
DOM si le recreeaza de la zero.

```
Fara track:
  Lista veche: [A, B, C]  -- 3 elemente DOM
  Lista noua:  [A, B, C, D]  -- distruge 3, creeaza 4 elemente DOM

Cu track by id:
  Lista veche: [A, B, C]  -- 3 elemente DOM
  Lista noua:  [A, B, C, D]  -- reutilizeaza 3, creeaza 1 element DOM
```

### 6.2 Noul @for cu track expression (Angular 17+)

```typescript
@Component({
  selector: 'app-user-list',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <!-- track este OBLIGATORIU in @for (Angular 17+) -->
    <!-- Angular forteaza dezvoltatorul sa gandeasca la tracking -->

    <!-- Track by un camp unic (cel mai comun) -->
    @for (user of users(); track user.id) {
      <app-user-card [user]="user" />
    } @empty {
      <p>Nu exista utilizatori.</p>
    }

    <!-- Track by index (cand elementele nu au identitate unica) -->
    <!-- ATENTIE: Re-render la reordonare! Folositi doar pentru liste statice -->
    @for (color of colors(); track $index) {
      <div [style.background]="color">{{ color }}</div>
    }

    <!-- Track by compus (cand un singur camp nu e suficient) -->
    @for (cell of gridCells(); track cell.row + '-' + cell.col) {
      <app-grid-cell [data]="cell" />
    }

    <!-- Variabile contextuale disponibile in @for -->
    @for (item of items(); track item.id; let i = $index,
          first = $first, last = $last, even = $even, odd = $odd,
          count = $count) {
      <div
        [class.first]="first"
        [class.last]="last"
        [class.even]="even"
        [class.odd]="odd"
      >
        {{ i + 1 }} / {{ count }}: {{ item.name }}
      </div>
    }
  `
})
export class UserListComponent {
  users = input.required<User[]>();
  colors = signal(['red', 'green', 'blue']);
  gridCells = signal<GridCell[]>([]);
  items = signal<Item[]>([]);
}
```

### 6.3 ngFor cu trackBy (legacy, dar inca utilizat)

```typescript
@Component({
  template: `
    <!-- ngFor legacy syntax -->
    <div *ngFor="let user of users; trackBy: trackByUserId">
      {{ user.name }}
    </div>
  `
})
export class LegacyListComponent {
  users: User[] = [];

  // Functia trackBy primeste index si elementul, returneaza identificatorul unic
  trackByUserId(index: number, user: User): number {
    return user.id;
  }
}
```

### 6.4 Impactul asupra performantei cu liste mari

```typescript
@Component({
  standalone: true,
  template: `
    <input (input)="filterUsers($event)" placeholder="Cauta..." />

    <!-- Cu track by id: filtrarea re-randeaza doar elementele modificate -->
    @for (user of filteredUsers(); track user.id) {
      <app-user-row
        [user]="user"
        [highlight]="isHighlighted(user.id)"
      />
    }

    <p>{{ filteredUsers().length }} / {{ allUsers().length }} utilizatori</p>
  `
})
export class LargeListComponent {
  allUsers = signal<User[]>([]);
  searchTerm = signal('');

  filteredUsers = computed(() => {
    const term = this.searchTerm().toLowerCase();
    if (!term) return this.allUsers();
    return this.allUsers().filter(u =>
      u.name.toLowerCase().includes(term)
    );
  });

  filterUsers(event: Event) {
    this.searchTerm.set((event.target as HTMLInputElement).value);
  }

  isHighlighted(id: number): boolean {
    return this.highlightedIds().has(id);
  }

  highlightedIds = signal(new Set<number>());
}
```

**Benchmark aproximativ (1000 de elemente):**

| Operatie | Fara track | Cu track by id |
|----------|-----------|----------------|
| Adaugare 1 element | ~50ms (re-render tot) | ~2ms (adauga 1 DOM node) |
| Stergere 1 element | ~50ms (re-render tot) | ~1ms (sterge 1 DOM node) |
| Reordonare | ~50ms (re-render tot) | ~5ms (muta DOM nodes) |
| Actualizare 1 camp | ~50ms (re-render tot) | ~1ms (update in-place) |

---

## 7. Virtual Scrolling

Virtual scrolling randeaza **doar** elementele vizibile in viewport, mentinand
performanta constanta indiferent de dimensiunea listei. Esential pentru liste
cu mii sau zeci de mii de elemente.

### 7.1 Setup CDK ScrollingModule

```bash
npm install @angular/cdk
```

```typescript
import { ScrollingModule } from '@angular/cdk/scrolling';

@Component({
  selector: 'app-virtual-list',
  standalone: true,
  imports: [ScrollingModule],
  styles: [`
    .viewport {
      height: 600px;
      width: 100%;
    }
    .item {
      height: 72px;
      display: flex;
      align-items: center;
      padding: 0 16px;
      border-bottom: 1px solid #eee;
    }
  `],
  template: `
    <!-- itemSize = inaltimea fixa a fiecarui element in pixeli -->
    <cdk-virtual-scroll-viewport itemSize="72" class="viewport">
      <div *cdkVirtualFor="let item of items; trackBy: trackById"
           class="item">
        <img [src]="item.avatar" width="48" height="48" />
        <div>
          <strong>{{ item.name }}</strong>
          <p>{{ item.email }}</p>
        </div>
      </div>
    </cdk-virtual-scroll-viewport>
  `
})
export class VirtualListComponent {
  // 100,000 de elemente -- fara virtual scrolling, ar fi imposibil
  items = Array.from({ length: 100_000 }, (_, i) => ({
    id: i,
    name: `User ${i}`,
    email: `user${i}@example.com`,
    avatar: `https://i.pravatar.cc/48?img=${i % 70}`
  }));

  trackById(index: number, item: any): number {
    return item.id;
  }
}
```

### 7.2 itemSize si autoSize Strategies

```typescript
// Strategia FIXA (itemSize) -- recomandata pentru performanta maxima
// Fiecare element are EXACT aceeasi inaltime
@Component({
  standalone: true,
  imports: [ScrollingModule],
  template: `
    <cdk-virtual-scroll-viewport itemSize="48" class="viewport">
      <div *cdkVirtualFor="let item of items" style="height: 48px">
        {{ item.name }}
      </div>
    </cdk-virtual-scroll-viewport>
  `
})
export class FixedSizeListComponent {}

// Strategia AUTO-SIZE (experimental) -- pentru elemente cu inaltime variabila
// NOTA: autosize este experimental in CDK
import { ScrollingModule } from '@angular/cdk/scrolling';
import {
  CdkVirtualScrollViewport,
  CdkVirtualForOf
} from '@angular/cdk/scrolling';

@Component({
  standalone: true,
  imports: [ScrollingModule],
  template: `
    <!-- autoSize nu este inca stabil; alternativa: custom VirtualScrollStrategy -->
    <cdk-virtual-scroll-viewport
      autosize
      class="viewport">
      <div *cdkVirtualFor="let message of messages"
           class="message"
           [style.min-height.px]="40">
        <strong>{{ message.sender }}</strong>
        <p>{{ message.text }}</p>
        <!-- Inaltimea variaza in functie de lungimea mesajului -->
      </div>
    </cdk-virtual-scroll-viewport>
  `
})
export class ChatListComponent {
  messages: ChatMessage[] = [];
}
```

### 7.3 Virtual Scrolling cu data source reactive

```typescript
import { CollectionViewer, DataSource } from '@angular/cdk/collections';
import { Observable, BehaviorSubject, Subscription } from 'rxjs';

export class InfiniteScrollDataSource extends DataSource<Item> {
  private pageSize = 50;
  private cachedData: Item[] = [];
  private dataStream = new BehaviorSubject<Item[]>([]);
  private subscription = new Subscription();
  private lastPage = 0;

  connect(collectionViewer: CollectionViewer): Observable<Item[]> {
    this.subscription.add(
      collectionViewer.viewChange.subscribe(range => {
        const neededPage = Math.floor(range.end / this.pageSize);
        if (neededPage > this.lastPage) {
          this.lastPage = neededPage;
          this.loadPage(neededPage);
        }
      })
    );
    return this.dataStream;
  }

  disconnect(): void {
    this.subscription.unsubscribe();
  }

  private async loadPage(page: number): Promise<void> {
    const response = await fetch(
      `/api/items?page=${page}&size=${this.pageSize}`
    );
    const newItems = await response.json();
    this.cachedData = [...this.cachedData, ...newItems];
    this.dataStream.next(this.cachedData);
  }
}

// Utilizare in componenta
@Component({
  standalone: true,
  imports: [ScrollingModule],
  template: `
    <cdk-virtual-scroll-viewport itemSize="72" class="viewport">
      <div *cdkVirtualFor="let item of dataSource" class="item">
        {{ item.name }}
      </div>
    </cdk-virtual-scroll-viewport>
  `
})
export class InfiniteListComponent {
  dataSource = new InfiniteScrollDataSource();
}
```

---

## 8. Web Workers

Web Workers permit executarea de cod JavaScript intr-un thread separat de UI thread,
prevenind blocarea interfetei la calcule intensive.

### 8.1 Generarea unui Web Worker cu Angular CLI

```bash
ng generate web-worker app
# Creeaza: src/app/app.worker.ts
```

### 8.2 Worker-ul si comunicatia prin postMessage

```typescript
// src/app/workers/data-processing.worker.ts
/// <reference lib="webworker" />

// Tipuri partajate intre worker si main thread
interface WorkerRequest {
  type: 'SORT' | 'FILTER' | 'AGGREGATE' | 'CSV_PARSE';
  payload: any;
}

interface WorkerResponse {
  type: string;
  result: any;
  duration: number;
}

addEventListener('message', ({ data }: MessageEvent<WorkerRequest>) => {
  const start = performance.now();

  let result: any;

  switch (data.type) {
    case 'SORT':
      // Sortare complexa -- ar bloca UI-ul pe main thread
      result = heavySort(data.payload.items, data.payload.criteria);
      break;

    case 'FILTER':
      result = complexFilter(data.payload.items, data.payload.conditions);
      break;

    case 'AGGREGATE':
      result = aggregate(data.payload.items, data.payload.groupBy);
      break;

    case 'CSV_PARSE':
      result = parseLargeCSV(data.payload.csvText);
      break;
  }

  const duration = performance.now() - start;

  postMessage({
    type: data.type,
    result,
    duration
  } as WorkerResponse);
});

function heavySort(items: any[], criteria: string[]): any[] {
  return [...items].sort((a, b) => {
    for (const key of criteria) {
      if (a[key] < b[key]) return -1;
      if (a[key] > b[key]) return 1;
    }
    return 0;
  });
}

function complexFilter(items: any[], conditions: any[]): any[] {
  return items.filter(item =>
    conditions.every(cond => {
      switch (cond.operator) {
        case 'eq': return item[cond.field] === cond.value;
        case 'gt': return item[cond.field] > cond.value;
        case 'contains': return item[cond.field]?.includes(cond.value);
        case 'regex': return new RegExp(cond.value).test(item[cond.field]);
        default: return true;
      }
    })
  );
}

function aggregate(items: any[], groupBy: string): Record<string, any[]> {
  return items.reduce((acc, item) => {
    const key = item[groupBy];
    (acc[key] = acc[key] || []).push(item);
    return acc;
  }, {} as Record<string, any[]>);
}

function parseLargeCSV(csvText: string): any[] {
  const lines = csvText.split('\n');
  const headers = lines[0].split(',').map(h => h.trim());
  return lines.slice(1).map(line => {
    const values = line.split(',');
    return headers.reduce((obj, header, i) => {
      obj[header] = values[i]?.trim();
      return obj;
    }, {} as Record<string, string>);
  });
}
```

### 8.3 Serviciu Angular pentru comunicarea cu Worker-ul

```typescript
@Injectable({ providedIn: 'root' })
export class DataProcessingService {
  private worker: Worker | null = null;
  private pendingRequests = new Map<string, Subject<any>>();

  constructor() {
    if (typeof Worker !== 'undefined') {
      this.worker = new Worker(
        new URL('../workers/data-processing.worker', import.meta.url),
        { type: 'module' }
      );

      this.worker.onmessage = ({ data }) => {
        const subject = this.pendingRequests.get(data.type);
        if (subject) {
          subject.next(data.result);
          subject.complete();
          this.pendingRequests.delete(data.type);
        }
        console.log(`Worker [${data.type}] completed in ${data.duration}ms`);
      };

      this.worker.onerror = (error) => {
        console.error('Worker error:', error);
        // Notificam toate request-urile in asteptare
        this.pendingRequests.forEach(subject => {
          subject.error(error);
        });
        this.pendingRequests.clear();
      };
    }
  }

  sortItems(items: any[], criteria: string[]): Observable<any[]> {
    return this.sendRequest('SORT', { items, criteria });
  }

  filterItems(items: any[], conditions: any[]): Observable<any[]> {
    return this.sendRequest('FILTER', { items, conditions });
  }

  aggregateItems(items: any[], groupBy: string): Observable<Record<string, any[]>> {
    return this.sendRequest('AGGREGATE', { items, groupBy });
  }

  parseCSV(csvText: string): Observable<any[]> {
    return this.sendRequest('CSV_PARSE', { csvText });
  }

  private sendRequest(type: string, payload: any): Observable<any> {
    if (!this.worker) {
      // Fallback: executam pe main thread daca Web Workers nu sunt disponibili
      console.warn('Web Workers not available, running on main thread');
      return of(this.processOnMainThread(type, payload));
    }

    const subject = new Subject<any>();
    this.pendingRequests.set(type, subject);
    this.worker.postMessage({ type, payload });
    return subject.asObservable();
  }

  private processOnMainThread(type: string, payload: any): any {
    // Fallback logic -- aceeasi logica ca in worker
    // In productie, importam functiile din fisiere shared
    return payload;
  }

  ngOnDestroy() {
    this.worker?.terminate();
  }
}

// Utilizare in componenta
@Component({
  standalone: true,
  template: `
    <button (click)="processData()" [disabled]="processing()">
      {{ processing() ? 'Se proceseaza...' : 'Proceseaza 1M randuri' }}
    </button>
    <p>Rezultat: {{ resultCount() }} grupuri</p>
  `
})
export class ReportComponent {
  private processingService = inject(DataProcessingService);
  processing = signal(false);
  resultCount = signal(0);

  processData() {
    this.processing.set(true);
    const largeDataset = this.generateData(1_000_000);

    this.processingService.aggregateItems(largeDataset, 'category')
      .subscribe({
        next: (grouped) => {
          this.resultCount.set(Object.keys(grouped).length);
          this.processing.set(false);
          // UI-ul a ramas COMPLET responsiv pe durata procesarii!
        },
        error: () => this.processing.set(false)
      });
  }

  private generateData(count: number) {
    return Array.from({ length: count }, (_, i) => ({
      id: i,
      name: `Item ${i}`,
      category: `Cat-${i % 100}`,
      value: Math.random() * 1000
    }));
  }
}
```

---

## 9. SSR (Server-Side Rendering) si Hydration

### 9.1 Angular SSR (fost Angular Universal)

SSR randeaza aplicatia Angular pe server, trimitand HTML complet catre client.
Acest lucru imbunatateste dramatic SEO, LCP (Largest Contentful Paint) si
experienta pe conexiuni lente.

```bash
# Adaugare SSR la un proiect existent (Angular 17+)
ng add @angular/ssr

# Creeaza:
# - server.ts (Express server)
# - src/app/app.config.server.ts
# - Modifica angular.json pentru SSR build
```

```typescript
// src/app/app.config.ts (client)
import { ApplicationConfig } from '@angular/core';
import { provideRouter } from '@angular/router';
import { provideClientHydration, withEventReplay } from '@angular/platform-browser';
import { provideHttpClient, withFetch } from '@angular/common/http';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes),
    provideHttpClient(withFetch()),
    // Activeaza hydration pe client
    provideClientHydration(
      withEventReplay() // Replay-ul evenimentelor inainte de hydration
    ),
  ]
};

// src/app/app.config.server.ts (server)
import { mergeApplicationConfig, ApplicationConfig } from '@angular/core';
import { provideServerRendering } from '@angular/platform-server';
import { provideServerRoutesConfig } from '@angular/ssr';
import { appConfig } from './app.config';
import { serverRoutes } from './app.routes.server';

const serverConfig: ApplicationConfig = {
  providers: [
    provideServerRendering(),
    provideServerRoutesConfig(serverRoutes),
  ]
};

export const config = mergeApplicationConfig(appConfig, serverConfig);

// src/app/app.routes.server.ts
import { RenderMode, ServerRoute } from '@angular/ssr';

export const serverRoutes: ServerRoute[] = [
  {
    path: '',
    renderMode: RenderMode.Prerender, // Static la build time
  },
  {
    path: 'products/:id',
    renderMode: RenderMode.Server, // Dinamic la request time
  },
  {
    path: 'dashboard/**',
    renderMode: RenderMode.Client, // Doar pe client (no SSR)
  },
];
```

### 9.2 Hydration

**Hydration** este procesul prin care Angular pe client "preia" DOM-ul generat
de server, in loc sa il distruga si recreeze. Acest lucru elimina "flickering"
si imbunatateste performanta.

```
FARA Hydration:
  Server HTML -> Client vede continut -> Angular bootstrap ->
  DISTRUGE DOM -> RECREEAZA DOM -> Interactiv
  (utilizatorul vede "flash" / flickering)

CU Hydration:
  Server HTML -> Client vede continut -> Angular bootstrap ->
  REUTILIZEAZA DOM existent -> Ataseaza event listeners -> Interactiv
  (tranzitie seamless, fara flickering)
```

```typescript
// Componenta care functioneaza corect cu SSR + Hydration
@Component({
  standalone: true,
  template: `
    <div>
      <h1>{{ title() }}</h1>
      <!-- Folosim isPlatformBrowser pentru cod client-only -->
      @if (isBrowser) {
        <app-interactive-chart [data]="chartData()" />
      }
    </div>
  `
})
export class ProductComponent {
  private platformId = inject(PLATFORM_ID);
  isBrowser = isPlatformBrowser(this.platformId);

  title = signal('Product');
  chartData = signal<number[]>([]);

  constructor() {
    // afterNextRender ruleaza DOAR pe client, DUPA hydration
    afterNextRender(() => {
      // Acces sigur la window, document, localStorage etc.
      const savedData = localStorage.getItem('chartData');
      if (savedData) {
        this.chartData.set(JSON.parse(savedData));
      }
    });
  }
}
```

### 9.3 Incremental Hydration (Angular 19)

Incremental Hydration permite hydratarea componentelor **la cerere**, nu pe toate
odata. Componentele care nu sunt vizibile sau nu sunt inca necesare raman ca HTML
static pana cand un trigger le activeaza.

```typescript
@Component({
  standalone: true,
  imports: [CommentsSection, RelatedProducts, LiveChat],
  template: `
    <!-- Componenta principala -- hydratata imediat -->
    <app-product-header [product]="product()" />

    <!-- Hydratare la viewport -- comentariile se hydrateaza
         cand utilizatorul scrolleaza pana la ele -->
    @defer (hydrate on viewport) {
      <app-comments-section [productId]="product().id" />
    } @placeholder {
      <div class="comments-placeholder">Comentarii...</div>
    }

    <!-- Hydratare la interactiune -- produsele similare se hydrateaza
         cand utilizatorul interactioneaza cu ele -->
    @defer (hydrate on interaction) {
      <app-related-products [categoryId]="product().categoryId" />
    } @placeholder {
      <div class="related-placeholder">Produse similare...</div>
    }

    <!-- Hydratare conditionata -->
    @defer (hydrate when isLoggedIn()) {
      <app-live-chat />
    } @placeholder {
      <div>Live chat...</div>
    }

    <!-- Hydratare niciodata -- ramane HTML static pentru totdeauna
         Ideal pentru continut pur informational (SEO) -->
    @defer (hydrate never) {
      <app-static-footer />
    }
  `
})
export class ProductPageComponent {
  product = input.required<Product>();
  isLoggedIn = signal(false);
}
```

### 9.4 Beneficiile SSR

| Beneficiu | Detalii |
|-----------|---------|
| **SEO** | Motoarele de cautare primesc HTML complet, nu o pagina goala |
| **LCP** | Continutul vizibil este disponibil imediat (HTML de la server) |
| **Initial Load** | Utilizatorul vede continut fara sa astepte download JS + bootstrap |
| **Social Sharing** | Open Graph tags sunt disponibile in HTML-ul initial |
| **Accesibilitate** | Continutul este disponibil chiar daca JS esueaza |

### 9.5 Event Replay in timpul Hydration

Event Replay captureaza evenimentele utilizatorului (click, input etc.) care au
loc **inainte** ca Angular sa fie complet hydratat, si le re-emite dupa hydration.

```typescript
// app.config.ts
provideClientHydration(
  withEventReplay() // Activeaza event replay
)

// Ce se intampla:
// 1. Server trimite HTML cu continut vizibil
// 2. Utilizatorul vede un buton "Add to Cart" si da click
// 3. Angular inca se incarca / hydrateaza
// 4. EVENT REPLAY: click-ul este capturat si stocat
// 5. Angular termina hydration
// 6. Click-ul capturat este replayed -- actiunea se executa
// 7. Utilizatorul nu pierde interactiunea!
```

---

## 10. Metrici de Performanta

### 10.1 Core Web Vitals

**LCP (Largest Contentful Paint)**
- Masoara cand cel mai mare element vizibil (imagine, text block) este randat
- Target: **< 2.5 secunde**
- Optimizari Angular: SSR, preload fonts, `NgOptimizedImage`, `@defer`

**FID (First Input Delay) / INP (Interaction to Next Paint)**
- FID (deprecated 2024): timpul pana la procesarea primei interactiuni
- INP (inlocuitor): masoara responsivitatea pe durata intregii sesiuni
- Target INP: **< 200ms**
- Optimizari Angular: OnPush, Web Workers, `runOutsideAngular`, zoneless

**CLS (Cumulative Layout Shift)**
- Masoara cat de mult se "muta" layout-ul neasteptat in timpul incarcarii
- Target: **< 0.1**
- Optimizari Angular: dimensiuni explicite pe imagini/containere, `@placeholder` cu dimensiuni fixe, font-display: swap

### 10.2 Alte metrici importante

**TTFB (Time to First Byte)**
- Timpul de la request pana la primirea primului byte de la server
- Target: **< 800ms**
- Imbunatatit de: SSR eficient, CDN, edge computing, caching

**FCP (First Contentful Paint)**
- Primul moment cand browser-ul randeaza orice continut (text, imagine, SVG)
- Target: **< 1.8 secunde**
- Imbunatatit de: SSR, inline critical CSS, preload resurse

### 10.3 Masurare cu Lighthouse si Web Vitals

```typescript
// Instalare web-vitals library
// npm install web-vitals

// src/app/services/performance-monitoring.service.ts
import { Injectable } from '@angular/core';
import { onLCP, onINP, onCLS, onFCP, onTTFB, type Metric } from 'web-vitals';

@Injectable({ providedIn: 'root' })
export class PerformanceMonitoringService {

  initMonitoring() {
    // Fiecare callback primeste metrica cand este disponibila
    onLCP(this.reportMetric);
    onINP(this.reportMetric);
    onCLS(this.reportMetric);
    onFCP(this.reportMetric);
    onTTFB(this.reportMetric);
  }

  private reportMetric = (metric: Metric) => {
    console.log(`[Web Vital] ${metric.name}: ${metric.value}`);

    // Trimite catre analytics backend
    const body = {
      name: metric.name,
      value: metric.value,
      rating: metric.rating,  // 'good' | 'needs-improvement' | 'poor'
      delta: metric.delta,
      id: metric.id,
      navigationType: metric.navigationType,
      url: window.location.href,
      timestamp: Date.now(),
    };

    // Folosim sendBeacon pentru a nu bloca navigarea
    if (navigator.sendBeacon) {
      navigator.sendBeacon('/api/analytics/web-vitals', JSON.stringify(body));
    } else {
      fetch('/api/analytics/web-vitals', {
        method: 'POST',
        body: JSON.stringify(body),
        keepalive: true,
      });
    }
  };
}

// Initializare in app.component.ts
@Component({
  selector: 'app-root',
  standalone: true,
  template: `<router-outlet />`
})
export class AppComponent {
  constructor() {
    afterNextRender(() => {
      inject(PerformanceMonitoringService).initMonitoring();
    });
  }
}
```

```typescript
// Masurare custom a Change Detection cu Performance API
@Injectable({ providedIn: 'root' })
export class CdProfilingService {
  private appRef = inject(ApplicationRef);

  startProfiling() {
    // Monitorizam fiecare ciclu de CD
    this.appRef.isStable.subscribe(stable => {
      if (stable) {
        performance.mark('cd-cycle-end');
        performance.measure('change-detection', 'cd-cycle-start', 'cd-cycle-end');

        const entries = performance.getEntriesByName('change-detection');
        const lastEntry = entries[entries.length - 1];
        if (lastEntry && lastEntry.duration > 16) {
          // Un ciclu CD > 16ms inseamna un frame pierdut (sub 60fps)
          console.warn(`Slow CD cycle: ${lastEntry.duration.toFixed(2)}ms`);
        }
        performance.clearMarks();
        performance.clearMeasures('change-detection');
      } else {
        performance.mark('cd-cycle-start');
      }
    });
  }
}
```

```bash
# Lighthouse din CLI
npx lighthouse https://my-app.com --output=json --output=html \
  --only-categories=performance

# Chrome DevTools -> Lighthouse tab:
# 1. Deschide DevTools (F12)
# 2. Tab "Lighthouse"
# 3. Selecteaza "Performance"
# 4. Click "Analyze page load"
```

---

## 11. Chrome DevTools si Angular DevTools

### 11.1 Chrome DevTools Performance Profiling

```
Pasi pentru profilare:
1. Deschide Chrome DevTools (F12)
2. Tab "Performance"
3. Click pe butonul Record (sau Ctrl+E)
4. Executa actiunile pe care vrei sa le analizezi (click-uri, navigare etc.)
5. Stop recording

Ce sa cauti:
- "Long Tasks" (barele rosii) -- task-uri > 50ms care blocheaza main thread
- "Scripting" galben -- executie JavaScript (inclusiv Change Detection)
- "Rendering" violet -- layout, paint, composite
- "Frames" -- frame-urile sub 60fps (peste 16.67ms per frame) sunt problematice
- "Main" lane -- fluxul detaliat pe main thread
```

**Identificarea problemelor de CD in Performance tab:**

```
In flame chart (Main lane), cauta:
- ApplicationRef.tick        -- un ciclu complet de CD
  - refreshView              -- verificarea unei componente
    - executeTemplate        -- evaluarea template-ului
    - detectChangesInternal  -- compararea valorilor
  - refreshView              -- urmatoarea componenta
    ...

Daca vezi MULTE apeluri refreshView sub un singur tick,
aplicatia verifica prea multe componente. => Foloseste OnPush.

Daca tick() apare prea des (la fiecare miscare de mouse),
evenimentele triggereaza CD inutil. => runOutsideAngular sau zoneless.
```

### 11.2 Angular DevTools

Angular DevTools este o extensie Chrome/Edge dedicata Angular.

```
Instalare:
- Chrome Web Store -> "Angular DevTools"
- Sau Edge Add-ons -> "Angular DevTools"

Functionalitati:

1. COMPONENT TREE (Tab "Components"):
   - Vizualizeaza ierarhia componentelor
   - Inspectarea proprietatilor (inputs, signals, state)
   - Editarea valorilor in timp real
   - Vizualizarea dependency injection tree

2. PROFILER (Tab "Profiler"):
   - Inregistreaza ciclurile de Change Detection
   - Arata CATE ori fiecare componenta a fost verificata
   - Arata DURATA verificarii fiecarei componente
   - Flame chart specific Angular (nu generic ca Performance tab)
   - Identifica componentele "fierbinti" (verificate prea des)

3. ROUTER TREE:
   - Vizualizeaza configuratia rutelor
   - Starea curenta a router-ului
   - Lazy loaded vs eager loaded
```

### 11.3 Identificarea ciclurilor de CD inutile

```typescript
// Tehnica 1: Getter cu log in template (doar pentru debugging!)
@Component({
  template: `<p>{{ debugData }}</p>`
})
export class DebugComponent {
  private cdCount = 0;

  get debugData(): string {
    this.cdCount++;
    console.log(`CD #${this.cdCount} for DebugComponent`);
    // Daca vezi acest log de zeci de ori pe secunda,
    // componenta este verificata prea des!
    return 'some data';
  }
}

// Tehnica 2: ngDoCheck lifecycle hook
@Component({
  template: `...`
})
export class MonitoredComponent implements DoCheck {
  private checkCount = 0;

  ngDoCheck() {
    this.checkCount++;
    if (this.checkCount % 100 === 0) {
      console.warn(`Component checked ${this.checkCount} times!`);
    }
  }
}

// Tehnica 3: Angular DevTools Profiler
// 1. Deschide Angular DevTools -> Profiler tab
// 2. Click Record
// 3. Interactioneaza cu aplicatia
// 4. Stop
// 5. Analizeaza:
//    - "Change Detection Cycles" -- cate cicluri au avut loc
//    - "Component" column -- care componente au fost verificate
//    - "Time" column -- cat a durat verificarea
//    - ROSU = componente verificate inutil (fara schimbari reale)
```

### 11.4 Checklist de optimizare folosind DevTools

```
DIAGNOSTICARE:
[ ] Ruleaza Lighthouse -> nota Performance > 90
[ ] Performance tab -> nu exista Long Tasks > 50ms in interactiuni obisnuite
[ ] Angular Profiler -> fiecare CD cycle < 16ms
[ ] Angular Profiler -> componentele OnPush NU sunt verificate inutil
[ ] Network tab -> bundle initial < 500 KB (gzipped)
[ ] Network tab -> lazy chunks se incarca la navigare, nu la start
[ ] Coverage tab -> cod neutilizat < 30%

REZOLVARE:
[ ] Componente verificate prea des -> OnPush + signals
[ ] CD cycles prea multe -> runOutsideAngular / zoneless
[ ] Bundle prea mare -> lazy loading + @defer + tree shaking
[ ] Long Tasks -> Web Workers pentru calcule grele
[ ] LCP slab -> SSR + NgOptimizedImage + preload
[ ] CLS mare -> dimensiuni fixe pe placeholder-uri
```

---

## Noutati Performanta Angular 20-21

### Zoneless devine default (Angular 21)
- Angular 21 foloseste zoneless change detection **by default** pentru aplicatii noi
- Provider-ul a fost redenumit: `provideZonelessChangeDetection()` (fara "Experimental" -- Angular 20 l-a promovat la developer preview)
- Schematics de migrare disponibile pentru proiecte existente
- Beneficii concrete: bundle mai mic (~15KB mai putin fara zone.js), performanta mai buna, debugging simplificat
- Signals sunt acum mecanismul principal de notificare a change detection

### Vitest ca default test runner (Angular 21)
- Vitest inlocuieste Karma ca test runner default
- Mai rapid decat Karma si Jest datorita Vite
- Schematic de migrare: `ng g @schematics/angular:refactor-jasmine-vitest`
- Suport nativ pentru ESM, TypeScript, watch mode rapid

### Template Hot Module Replacement (Angular 20)
- HMR pentru templates activat by default
- Modificari in template se reflecta instant fara full reload
- Pastreaza starea componentei la modificari de template

### Host Binding Type Checking (Angular 20)
- Expresiile din `host` metadata sunt acum validate de TypeScript
- Hover tooltips in IDE pentru host bindings
- Diagnostice noi: nullish coalescing invalid, track functions neinvocate, importuri lipsa structural directives

### HttpClient provided by default (Angular 21)
- Nu mai trebuie `provideHttpClient()` in app config
- HttpResponse si HttpErrorResponse au nou `responseType` property (expune tipul Fetch API)

### ng-reflect eliminat (Angular 20)
- Atributele `ng-reflect-*` din development mode nu mai sunt emise
- Debugging se face prin Angular DevTools in loc de DOM attributes

---

## Intrebari frecvente de interviu

### 1. Care este diferenta intre `ChangeDetectionStrategy.Default` si `OnPush`? Cand folosesti fiecare?

**Raspuns:**

Cu **Default**, Angular verifica intregul arbore de componente la fiecare ciclu de Change Detection -- orice eveniment async (click, HTTP response, setTimeout) triggereaza verificarea tuturor componentelor, de la root pana la frunze.

Cu **OnPush**, Angular verifica o componenta doar cand: (1) o referinta de `@Input` se schimba (nu mutatia interna), (2) un eveniment DOM este emis din interiorul componentei, (3) `async` pipe primeste o valoare noua, (4) un signal citit in template se schimba, sau (5) `markForCheck()` este apelat.

**Cand folosesc fiecare:** In practica, la nivel de Principal Engineer, **toate componentele noi ar trebui sa fie OnPush**. Default este acceptabil doar in prototipuri sau componente extrem de simple. OnPush combinat cu signals si imutabilitate ofera cea mai buna performanta si predictibilitate.

---

### 2. Explica diferenta intre `detectChanges()` si `markForCheck()`. Cand folosesti fiecare?

**Raspuns:**

`detectChanges()` executa Change Detection **sincron si imediat** pentru componenta curenta si copiii sai. Este util in teste unitare sau cand ai nevoie absoluta de actualizare imediata a DOM-ului.

`markForCheck()` marcheaza componenta si **toti stramosii** pana la root ca "dirty", iar verificarea efectiva are loc la **urmatorul ciclu de CD** programat de Angular. Este mai sigur deoarece respecta fluxul normal de CD si evita erori precum `ExpressionChangedAfterItHasBeenChecked`.

**Recomandare:** In 90% din cazuri, `markForCheck()` este alegerea corecta. `detectChanges()` trebuie folosit cu precautie, in special in `ngAfterViewInit` sau callback-uri externe.

In practica, cu signals si `async` pipe, raramente mai ai nevoie de apeluri manuale. Ele devin necesare doar cand integrezi biblioteci third-party care modifica starea in afara mecanismelor Angular.

---

### 3. Ce este Zoneless Angular si cum functioneaza?

**Raspuns:**

Zoneless Angular elimina dependenta de **Zone.js** -- biblioteca care face monkey-patching la toate API-urile asincrone ale browser-ului (setTimeout, Promise, fetch, addEventListener etc.). In locul interceptarii globale a evenimentelor async, Angular se bazeaza pe **signals** pentru a sti exact cand si unde s-a schimbat starea.

Se activeaza cu `provideExperimentalZonelessChangeDetection()` in `app.config.ts` si eliminarea `zone.js` din polyfills.

**Beneficii:** Bundle cu ~15-20 KB mai mic, performanta superioara (fara overhead de monkey-patching), stack trace-uri curate, CD targetat (nu intreg arborele), compatibilitate mai buna cu Web Components si micro-frontends.

**Migrare:** Pasul 1 -- adoptarea signals in loc de proprietati simple. Pasul 2 -- eliminarea dependintelor de `NgZone`. Pasul 3 -- verificarea codului care depinde de timing-ul Zone.js (setTimeout/setInterval nu mai triggereaza CD automat).

---

### 4. Cum functioneaza `@defer` si cum difera de lazy loading prin rute?

**Raspuns:**

`@defer` este un mecanism de lazy loading la nivel de **template**, nu de ruta. Permite amanarea incarcarii componentelor, directivelor si pipe-urilor pana cand un trigger specific este indeplinit: `on viewport`, `on interaction`, `on hover`, `on idle`, `on timer(Xs)`, sau `when conditie`.

**Diferenta fata de route lazy loading:**
- Route lazy loading opereaza la nivel de navigare (intreaga pagina sau sectiune)
- `@defer` opereaza la nivel de componenta individuala in cadrul aceluiasi view
- `@defer` suporta `prefetch` triggers separate (descarca devreme, randeaza tarziu)
- `@defer` are `@placeholder`, `@loading`, `@error` ca state management built-in

**Use case tipic:** O pagina de produs unde header-ul si pretul se incarca imediat, dar reviews-urile (sub fold) sunt `@defer (on viewport)`, iar ghidul de marimi este `@defer (on interaction)`.

---

### 5. Cum optimizezi bundle size intr-o aplicatie Angular mare?

**Raspuns:**

Abordez optimizarea in mai multe straturi:

**1. Lazy loading agresiv:** Fiecare feature are propriile rute lazy-loaded cu `loadComponent` / `loadChildren`. Preloading strategy custom bazata pe probabilitatea de navigare.

**2. @defer pentru componente grele:** Grafice, editoare rich-text, harti -- sunt defer-uite cu trigger `on viewport` sau `on interaction`.

**3. Standalone components:** Elimina overhead-ul NgModules. Fiecare componenta importa doar ce are nevoie, permitand tree shaking granular.

**4. Tree-shakable services:** `providedIn: 'root'` permite Angular sa elimine serviciile nefolosite din bundle.

**5. Analiza dependintelor:** Source map explorer identifica librariile mari. Inlocuiesc `moment.js` cu `date-fns`, `lodash` cu `lodash-es` (importuri individuale).

**6. Budgets in angular.json:** Initial bundle < 500 KB warning, < 1 MB error. CI/CD verifica automat.

**7. Audit periodic:** Rulez `npx source-map-explorer` dupa fiecare release si investighez cresteri neasteptate.

---

### 6. Explica SSR si Hydration in Angular. Ce este Incremental Hydration?

**Raspuns:**

**SSR** randeaza aplicatia Angular pe server, trimitand HTML complet catre browser. Utilizatorul vede continut imediat (LCP bun), iar motoarele de cautare pot indexa pagina (SEO).

**Hydration** este procesul prin care Angular pe client "preia" DOM-ul generat de server, in loc sa-l distruga si recreeze. Asta elimina flickering-ul si reduce TTI (Time to Interactive).

**Event Replay** (activat cu `withEventReplay()`) captureaza interactiunile utilizatorului care au loc INAINTE ca Angular sa fie hydratat si le re-emite dupa hydration -- utilizatorul nu pierde niciodata un click.

**Incremental Hydration** (Angular 19) merge si mai departe: nu hydrateaza toate componentele odata. Componentele care nu sunt vizibile sau necesare raman ca HTML static. Hydratarea are loc on-demand: `hydrate on viewport`, `hydrate on interaction`, sau chiar `hydrate never` pentru continut pur static. Reduce dramatic JavaScript-ul executat la incarcarea paginii.

---

### 7. Cum folosesti Virtual Scrolling si cand este necesar?

**Raspuns:**

Virtual Scrolling este necesar cand randezi liste cu **sute sau mii de elemente**. In loc sa creeze DOM nodes pentru toate elementele, CDK `cdk-virtual-scroll-viewport` randeaza DOAR elementele vizibile in viewport, plus un buffer mic.

**Implementare:** Importam `ScrollingModule` din `@angular/cdk/scrolling`, folosim `cdk-virtual-scroll-viewport` cu `itemSize` (inaltime fixa per element), si `*cdkVirtualFor` in loc de `*ngFor`.

**Cand este necesar:**
- Liste cu peste 100-200 de elemente vizibile simultan
- Tabele cu mii de randuri (mai ales cu sorting/filtering)
- Feed-uri infinite (social media, logs, chat history)

**Limitari:**
- `itemSize` fix functioneaza cel mai bine -- inaltimi variabile sunt mai complexe
- Scroll position restoration poate fi dificila
- Accessibility (screen readers) necesita atentie speciala

**Alternativa pentru liste moderate (50-200 items):** Paginarea sau "load more" pot fi suficiente si mai simple de implementat.

---

### 8. Ce metrici Core Web Vitals monitorizezi si cum le optimizezi in Angular?

**Raspuns:**

**LCP (Largest Contentful Paint) < 2.5s:**
- Activez SSR pentru continutul initial
- Folosesc `NgOptimizedImage` cu `priority` pe imaginea principala
- Preload fonturi si CSS critic
- `@defer` pe componentele sub fold

**INP (Interaction to Next Paint) < 200ms:**
- Toate componentele OnPush + signals
- Web Workers pentru calcule grele
- `runOutsideAngular` pentru animatii/timere
- Zoneless Angular elimina overhead CD

**CLS (Cumulative Layout Shift) < 0.1:**
- Dimensiuni explicite pe imagini (`width`, `height`)
- `@placeholder` cu dimensiuni fixe in `@defer`
- `font-display: swap` cu font fallback dimensionat corect
- Spatiu rezervat pentru componente async

**Masurare:** Integrez `web-vitals` library cu `navigator.sendBeacon` catre un analytics backend. Monitorizez percentila 75 (nu media) in dashboard-ul de productie. Lighthouse in CI/CD ca gate (scor > 90). Real User Monitoring (RUM) pe productie.

---

### 9. Cum identifici si rezolvi probleme de performanta legate de Change Detection?

**Raspuns:**

**Identificare:**
1. **Angular DevTools Profiler:** Inregistrez un ciclu de interactiuni, analizez care componente sunt verificate si cat dureaza.
2. **Chrome Performance tab:** Caut Long Tasks si examinez flame chart-ul pentru `ApplicationRef.tick()` si `refreshView` excesive.
3. **`ngDoCheck` counter:** Temporar adaug un contor in componente suspecte pentru a vedea cate verificari au loc.

**Rezolvare, in ordinea impactului:**
1. **OnPush pe toate componentele** -- reduce dramatic numarul de verificari
2. **Signals in loc de proprietati simple** -- Angular stie exact ce s-a schimbat
3. **`async` pipe in loc de subscribe manual** -- gestioneaza automat markForCheck
4. **`@for` cu `track`** -- previne re-crearea DOM-ului la fiecare schimbare de lista
5. **`runOutsideAngular`** pentru event listeners frecvente (scroll, mousemove, resize)
6. **`detach()` pe componente statice** -- scoase complet din arborele CD
7. **Zoneless Angular** -- elimina CD inutile triggerate de orice event async

Cele mai frecvente greseli pe care le identific in echipe: getteri complecsi in template-uri (recalculati la fiecare CD), subscribe in `ngOnInit` fara unsubscribe (memory leak + CD inutile), si lipsa `track` / `trackBy` pe liste.

---

### 10. Descrie o strategie completa de optimizare a performantei pentru o aplicatie Angular enterprise.

**Raspuns:**

Abordez optimizarea pe trei axe: **Build Time**, **Runtime**, si **Perceived Performance**.

**Build Time:**
- Standalone components cu importuri granulare
- Strict mode TypeScript (elimina cod mort)
- Budget-uri in CI/CD (initial < 500 KB, orice chunk < 200 KB)
- Tree-shakable services cu `providedIn: 'root'`
- Analiza periodica cu source-map-explorer

**Runtime:**
- OnPush + signals pe TOATE componentele (policy de echipa)
- Virtual scrolling pentru liste > 200 elemente
- Web Workers pentru procesare date / sortare / filtrare
- `@defer` pe toate componentele non-critice
- Lazy loading pe fiecare feature route cu preloading strategy custom
- Zoneless Angular (migrat incremental, modul cu modul)

**Perceived Performance:**
- SSR cu incremental hydration
- Event replay pentru interactivitate inainte de hydration
- `NgOptimizedImage` cu `priority` pe hero images
- Skeleton loaders in `@placeholder`-uri (cu dimensiuni fixe -- previne CLS)
- Preloading strategy bazata pe probabilitatea de navigare
- Service Worker pentru cache-ul asset-urilor statice

**Monitoring:**
- `web-vitals` library cu RUM in productie
- Dashboard cu percentila 75 per metric per pagina
- Alert-uri cand LCP > 2.5s sau INP > 200ms in productie
- Lighthouse CI ca quality gate in pipeline (fail build daca scor < 85)
- Review-uri de performanta trimestriale cu Angular DevTools Profiler
