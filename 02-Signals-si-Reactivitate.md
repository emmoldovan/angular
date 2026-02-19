# Signals si Reactivitate in Angular

## Cuprins

1. [Angular Signals - Fundamente](#1-angular-signals---fundamente)
2. [LinkedSignal](#2-linkedsignal)
3. [Resource API](#3-resource-api)
4. [Migrare de la RxJS la Signals](#4-migrare-de-la-rxjs-la-signals)
5. [RxJS - Operatori esentiali](#5-rxjs---operatori-esentiali)
6. [Patterns reactive](#6-patterns-reactive)
7. [Cand sa folosesti Signals vs Observables](#7-cand-sa-folosesti-signals-vs-observables)
8. [toSignal() si toObservable() - Bridging](#8-tosignal-si-toobservable---bridging)
9. [Stabilizare Signals in Angular 20-21](#9-stabilizare-signals-în-angular-20-21)
10. [Intrebari frecvente de interviu](#10-intrebari-frecvente-de-interviu)

---

## 1. Angular Signals - Fundamente

### Ce sunt Signals?

Signals sunt **primitive reactive sincrone** introduse in Angular 16 (developer preview) si stabilizate in Angular 17. Ele reprezinta o valoare care poate notifica consumatorii cand se schimba.

Diferenta fundamentala fata de RxJS: Signals sunt **pull-based** (consumatorul citeste valoarea cand are nevoie), in timp ce Observables sunt **push-based** (emitorul trimite valori catre abonati).

**De ce Signals?**
- Change detection mai granular (la nivel de component, nu tree)
- API mai simplu pentru state management sincron
- Elimina nevoia de `zone.js` (zoneless Angular)
- Interoperabilitate cu RxJS prin `toSignal()` / `toObservable()`

### signal() - WritableSignal

`signal()` creaza un **WritableSignal** - o valoare reactiva pe care o poti citi si modifica.

```typescript
import { signal, WritableSignal } from '@angular/core';

// Creare
const count: WritableSignal<number> = signal(0);
const name = signal('Angular'); // type inference: WritableSignal<string>

// Citire - apelezi ca o functie (getter)
console.log(count()); // 0
console.log(name());  // 'Angular'

// Scriere - 3 metode
count.set(5);                    // seteaza direct
count.update(val => val + 1);    // update bazat pe valoarea curenta
name.set('Angular 19');

// Exemplu complet intr-un component
@Component({
  selector: 'app-counter',
  template: `
    <p>Count: {{ count() }}</p>
    <button (click)="increment()">+1</button>
    <button (click)="decrement()">-1</button>
    <button (click)="reset()">Reset</button>
  `
})
export class CounterComponent {
  count = signal(0);

  increment() {
    this.count.update(c => c + 1);
  }

  decrement() {
    this.count.update(c => c - 1);
  }

  reset() {
    this.count.set(0);
  }
}
```

### WritableSignal vs Signal (readonly)

Distinctia este importanta din punct de vedere al encapsularii:

```typescript
import { signal, Signal, WritableSignal } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class UserService {
  // Privat - WritableSignal, doar service-ul il poate modifica
  private readonly _currentUser = signal<User | null>(null);

  // Public - Signal (readonly), consumatorii pot doar citi
  readonly currentUser: Signal<User | null> = this._currentUser.asReadonly();

  // Alternativ, tipul Signal<T> nu expune .set() / .update()
  // Consumatorul nu poate face: userService.currentUser.set(...)

  login(user: User): void {
    this._currentUser.set(user);
  }

  logout(): void {
    this._currentUser.set(null);
  }
}

// In component
@Component({
  template: `
    @if (userService.currentUser(); as user) {
      <p>Welcome, {{ user.name }}</p>
    } @else {
      <p>Please log in</p>
    }
  `
})
export class HeaderComponent {
  userService = inject(UserService);
  // userService.currentUser.set(...) -> EROARE TypeScript!
  // Tipul Signal<T> nu are metoda .set()
}
```

**Regula de baza:** Expune `Signal<T>` (readonly) in afara clasei, pastreaza `WritableSignal<T>` privat. Aceasta e aceeasi filosofie ca si `private BehaviorSubject` + `public Observable` din pattern-ul RxJS.

### computed() - Signals derivate

`computed()` creaza un **signal readonly** care se recalculeaza automat cand dependentele se schimba. Este echivalentul `combineLatest` + `map` din RxJS, dar sincron.

```typescript
import { signal, computed } from '@angular/core';

const firstName = signal('John');
const lastName = signal('Doe');

// Se recalculeaza automat cand firstName SAU lastName se schimba
const fullName = computed(() => `${firstName()} ${lastName()}`);

console.log(fullName()); // 'John Doe'
firstName.set('Jane');
console.log(fullName()); // 'Jane Doe'
```

**Exemplu avansat - filtrare si derivare:**

```typescript
@Component({
  selector: 'app-product-list',
  template: `
    <input [ngModel]="searchTerm()" (ngModelChange)="searchTerm.set($event)" />
    <select [ngModel]="selectedCategory()" (ngModelChange)="selectedCategory.set($event)">
      <option value="">All</option>
      @for (cat of categories(); track cat) {
        <option [value]="cat">{{ cat }}</option>
      }
    </select>

    <p>Showing {{ filteredProducts().length }} of {{ products().length }} products</p>

    @for (product of filteredProducts(); track product.id) {
      <app-product-card [product]="product" />
    }
  `
})
export class ProductListComponent {
  private productService = inject(ProductService);

  products = this.productService.products;       // Signal<Product[]>
  searchTerm = signal('');
  selectedCategory = signal('');

  // Computed: categorii unice extrase din produse
  categories = computed(() => {
    const cats = this.products().map(p => p.category);
    return [...new Set(cats)].sort();
  });

  // Computed: produse filtrate (depinde de products, searchTerm, selectedCategory)
  filteredProducts = computed(() => {
    let result = this.products();
    const search = this.searchTerm().toLowerCase();
    const category = this.selectedCategory();

    if (search) {
      result = result.filter(p =>
        p.name.toLowerCase().includes(search) ||
        p.description.toLowerCase().includes(search)
      );
    }

    if (category) {
      result = result.filter(p => p.category === category);
    }

    return result;
  });

  // Computed: statistici derivate
  totalValue = computed(() =>
    this.filteredProducts().reduce((sum, p) => sum + p.price, 0)
  );
}
```

**Proprietati importante ale `computed()`:**
- **Lazy** - se recalculeaza doar cand este citit si dependentele s-au schimbat
- **Memoized** - daca dependentele nu s-au schimbat, returneaza valoarea cached
- **Readonly** - nu poti face `.set()` pe un computed signal
- **Glitch-free** - Angular garanteaza consistenta (nu vei vedea stari intermediare)

### effect() - Side effects reactive

`effect()` executa cod cand signal-urile citite in interiorul sau se schimba. Se foloseste pentru **side effects** (logging, localStorage, API calls).

```typescript
import { signal, effect } from '@angular/core';

@Component({
  selector: 'app-settings',
  template: `
    <label>
      Theme:
      <select [ngModel]="theme()" (ngModelChange)="theme.set($event)">
        <option value="light">Light</option>
        <option value="dark">Dark</option>
      </select>
    </label>
    <label>
      Language:
      <select [ngModel]="language()" (ngModelChange)="language.set($event)">
        <option value="en">English</option>
        <option value="ro">Romanian</option>
      </select>
    </label>
  `
})
export class SettingsComponent {
  theme = signal(localStorage.getItem('theme') ?? 'light');
  language = signal(localStorage.getItem('language') ?? 'en');

  // Effect: sincronizeaza cu localStorage
  private persistTheme = effect(() => {
    const currentTheme = this.theme(); // dependenta urmarita automat
    localStorage.setItem('theme', currentTheme);
    document.body.setAttribute('data-theme', currentTheme);
  });

  private persistLanguage = effect(() => {
    localStorage.setItem('language', this.language());
  });

  // Effect cu cleanup
  private pollingEffect = effect((onCleanup) => {
    const lang = this.language();
    const intervalId = setInterval(() => {
      console.log(`Checking updates for language: ${lang}`);
    }, 30000);

    // Cleanup se apeleaza INAINTE de re-executie si la destroy
    onCleanup(() => {
      clearInterval(intervalId);
    });
  });
}
```

**Reguli importante pentru `effect()`:**

```typescript
// GRESIT - nu modifica signals in effect (produce infinite loop potential)
effect(() => {
  if (this.count() > 10) {
    this.count.set(10); // EROARE la runtime (in dev mode)!
  }
});

// CORECT - daca trebuie neaparat, foloseste allowSignalWrites (rar necesar)
effect(() => {
  if (this.count() > 10) {
    this.count.set(10);
  }
}, { allowSignalWrites: true });

// MAI CORECT - foloseste computed() pentru derivare
const clampedCount = computed(() => Math.min(this.count(), 10));
```

**Cand sa folosesti `effect()`:**
- Sincronizare cu localStorage / sessionStorage
- Logging / analytics
- Manipulare DOM care nu poate fi facuta declarativ
- Integrare cu librarii non-Angular (charts, maps)
- **NU** pentru derivare de state (foloseste `computed()`)

### Signals si Change Detection

Signals transforma modul in care Angular face change detection:

```typescript
// INAINTE (Zone.js + Default Change Detection)
// Orice event async (click, setTimeout, HTTP) -> Zone.js notifica Angular
// -> Angular verifica TOTI componentii din arbore (top-down)

// CU SIGNALS (Angular 17+)
// Signal se schimba -> doar componentii care CITESC acel signal sunt marcati dirty
// -> Angular verifica DOAR acei componenti

@Component({
  selector: 'app-parent',
  // Cu OnPush + Signals, Angular stie EXACT ce s-a schimbat
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <app-header [userName]="userName()" />
    <app-content [items]="items()" />
    <app-footer />
  `
})
export class ParentComponent {
  userName = signal('John');
  items = signal<Item[]>([]);

  // Cand userName.set('Jane') -> doar app-header este verificat
  // app-content si app-footer NU sunt verificate
}
```

**Zoneless Angular (experimental din Angular 18, stabil in Angular 19):**

```typescript
// bootstrap fara Zone.js
bootstrapApplication(AppComponent, {
  providers: [
    provideExperimentalZonelessChangeDetection() // Angular 18
    // provideZonelessChangeDetection() // Angular 19+
  ]
});

// In zoneless mode:
// - Change detection este COMPLET bazat pe signals
// - setTimeout, setInterval, fetch NU triggeruiesc CD automat
// - Trebuie sa folosesti signals pentru ORICE state din template
```

### Equality function customizata

```typescript
// By default, signals folosesc Object.is() pentru comparatie
// Poti customiza:

interface User {
  id: number;
  name: string;
  lastSeen: Date;
}

const currentUser = signal<User>(
  { id: 1, name: 'John', lastSeen: new Date() },
  {
    // Nu re-notifica daca doar lastSeen s-a schimbat
    equal: (a, b) => a.id === b.id && a.name === b.name
  }
);

// Util si pentru computed:
const sortedItems = computed(
  () => [...this.items()].sort((a, b) => a.name.localeCompare(b.name)),
  {
    equal: (a, b) => JSON.stringify(a) === JSON.stringify(b)
  }
);
```

---

## 2. LinkedSignal

### Ce este LinkedSignal?

`linkedSignal()` (introdus in Angular 19) creaza un **WritableSignal care este legat de o sursa**. Cand sursa se schimba, linkedSignal-ul se **reseteaza automat** la o valoare derivata din noua sursa.

Gandeste-te la el ca un `computed()` care **poate fi si scris manual**, dar care se reseteaza cand sursa se schimba.

### Problema pe care o rezolva

```typescript
// FARA linkedSignal - trebuie sa gestionezi manual resetarea
@Component({
  template: `
    <select (change)="selectedCategory.set($event.target.value)">
      @for (cat of categories(); track cat) {
        <option [value]="cat">{{ cat }}</option>
      }
    </select>

    <select (change)="selectedProduct.set($event.target.value)">
      @for (prod of productsInCategory(); track prod.id) {
        <option [value]="prod.id">{{ prod.name }}</option>
      }
    </select>
  `
})
export class ProductSelectorComponent {
  categories = signal(['Electronics', 'Books', 'Clothing']);
  selectedCategory = signal('Electronics');
  selectedProduct = signal<string | null>(null);

  // PROBLEMA: cand schimbi categoria, selectedProduct ramane pe valoarea veche!
  // Trebuie sa adaugi un effect manual:
  private resetProduct = effect(() => {
    this.selectedCategory(); // urmareste categoria
    this.selectedProduct.set(null); // reseteaza produsul
  }, { allowSignalWrites: true });
}
```

### Solutia cu linkedSignal

```typescript
import { signal, computed, linkedSignal } from '@angular/core';

@Component({
  template: `
    <select (change)="selectedCategory.set($event.target.value)">
      @for (cat of categories(); track cat) {
        <option [value]="cat">{{ cat }}</option>
      }
    </select>

    <select (change)="selectedProduct.set($event.target.value)">
      @for (prod of productsInCategory(); track prod.id) {
        <option [value]="prod.id">{{ prod.name }}</option>
      }
    </select>
    <p>Selected: {{ selectedProduct() }}</p>
  `
})
export class ProductSelectorComponent {
  categories = signal(['Electronics', 'Books', 'Clothing']);
  selectedCategory = signal('Electronics');

  allProducts = signal<Product[]>([
    { id: '1', name: 'Laptop', category: 'Electronics' },
    { id: '2', name: 'Phone', category: 'Electronics' },
    { id: '3', name: 'Novel', category: 'Books' },
  ]);

  productsInCategory = computed(() =>
    this.allProducts().filter(p => p.category === this.selectedCategory())
  );

  // linkedSignal: se reseteaza la primul produs din categorie
  // cand selectedCategory se schimba
  selectedProduct = linkedSignal<string, string | null>({
    source: this.selectedCategory,
    computation: () => {
      const products = this.productsInCategory();
      return products.length > 0 ? products[0].id : null;
    }
  });

  // selectedProduct POATE fi scris manual (e.g., user selecteaza alt produs)
  // DAR se reseteaza automat cand categoria se schimba
}
```

### Forma scurta (shorthand)

```typescript
// Cand computation-ul e simplu, poti folosi forma scurta:
const source = signal(0);

// Forma scurta - linkedSignal primeste direct o computation function
const doubled = linkedSignal(() => source() * 2);
// doubled() === 0
source.set(5);
// doubled() === 10
doubled.set(42);
// doubled() === 42
source.set(3);
// doubled() === 6 (s-a resetat!)
```

### Exemplu practic: Pagination

```typescript
@Component({
  selector: 'app-data-table',
  template: `
    <input [ngModel]="searchTerm()" (ngModelChange)="searchTerm.set($event)" />

    @for (item of paginatedItems(); track item.id) {
      <tr>...</tr>
    }

    <div class="pagination">
      <button (click)="currentPage.set(currentPage() - 1)"
              [disabled]="currentPage() <= 1">
        Previous
      </button>
      <span>Page {{ currentPage() }} of {{ totalPages() }}</span>
      <button (click)="currentPage.set(currentPage() + 1)"
              [disabled]="currentPage() >= totalPages()">
        Next
      </button>
    </div>
  `
})
export class DataTableComponent {
  items = signal<Item[]>([]);
  searchTerm = signal('');
  pageSize = signal(10);

  filteredItems = computed(() => {
    const term = this.searchTerm().toLowerCase();
    if (!term) return this.items();
    return this.items().filter(i => i.name.toLowerCase().includes(term));
  });

  totalPages = computed(() =>
    Math.ceil(this.filteredItems().length / this.pageSize())
  );

  // Cand filteredItems se schimba (cautare noua), reseteaza la pagina 1
  // Dar utilizatorul POATE naviga manual intre pagini
  currentPage = linkedSignal({
    source: this.filteredItems,
    computation: () => 1 // mereu reseteaza la pagina 1
  });

  paginatedItems = computed(() => {
    const start = (this.currentPage() - 1) * this.pageSize();
    return this.filteredItems().slice(start, start + this.pageSize());
  });
}
```

### linkedSignal vs computed vs effect

| Caracteristica | `computed()` | `linkedSignal()` | `effect()` |
|---------------|-------------|------------------|------------|
| **Readonly** | Da | Nu (writable) | N/A |
| **Se reseteaza** | Mereu recalculat | Cand sursa se schimba | N/A |
| **Scriere manuala** | Nu | Da | N/A |
| **Use case** | Derivare pura | State dependent cu override | Side effects |

---

## 3. Resource API

### Ce este Resource API?

`resource()` (Angular 19) este un API declarativ pentru **incarcarea datelor asincrone** bazata pe signals. Gestioneaza automat loading states, erori, si annularea request-urilor.

### resource() - varianta nativa (cu Promise)

```typescript
import { signal, resource, ResourceStatus } from '@angular/core';

@Component({
  selector: 'app-user-profile',
  template: `
    @switch (userResource.status()) {
      @case (ResourceStatus.Idle) {
        <p>Select a user</p>
      }
      @case (ResourceStatus.Loading) {
        <app-spinner />
      }
      @case (ResourceStatus.Resolved) {
        <div class="profile">
          <h2>{{ userResource.value()?.name }}</h2>
          <p>{{ userResource.value()?.email }}</p>
        </div>
      }
      @case (ResourceStatus.Error) {
        <div class="error">
          <p>Failed to load user</p>
          <button (click)="userResource.reload()">Retry</button>
        </div>
      }
    }
  `
})
export class UserProfileComponent {
  protected ResourceStatus = ResourceStatus;

  userId = signal<number | null>(null);

  userResource = resource({
    // request: derivata din signals - cand userId se schimba,
    // resource-ul se re-fetches automat
    request: () => ({ id: this.userId() }),

    // loader: functia async care incarca datele
    loader: async ({ request, abortSignal }) => {
      if (request.id === null) {
        return undefined;
      }

      const response = await fetch(`/api/users/${request.id}`, {
        signal: abortSignal // IMPORTANT: permite anularea!
      });

      if (!response.ok) {
        throw new Error(`HTTP ${response.status}`);
      }

      return response.json() as Promise<User>;
    }
  });

  selectUser(id: number) {
    this.userId.set(id);
    // Resource-ul se reincarca AUTOMAT - nu trebuie sa apelezi nimic
  }
}
```

### rxResource() - varianta RxJS

```typescript
import { signal } from '@angular/core';
import { rxResource } from '@angular/core/rxjs-interop';
import { HttpClient } from '@angular/common/http';

@Component({
  selector: 'app-product-detail',
  template: `
    @if (productResource.isLoading()) {
      <app-skeleton-loader />
    }

    @if (productResource.value(); as product) {
      <h1>{{ product.name }}</h1>
      <p>{{ product.price | currency }}</p>
    }

    @if (productResource.error(); as error) {
      <app-error-banner
        [message]="error.message"
        (retry)="productResource.reload()" />
    }
  `
})
export class ProductDetailComponent {
  private http = inject(HttpClient);

  productId = signal<string>('');

  productResource = rxResource({
    request: () => this.productId(),
    loader: ({ request: id }) => {
      if (!id) return EMPTY;
      return this.http.get<Product>(`/api/products/${id}`).pipe(
        retry({ count: 2, delay: 1000 })
      );
    }
  });
}
```

### Proprietatile Resource

```typescript
const userResource = resource({ /* ... */ });

// Citire
userResource.value();    // T | undefined - datele incarcate
userResource.status();   // ResourceStatus enum
userResource.error();    // unknown - eroarea (daca exista)
userResource.isLoading(); // boolean - convenience getter

// Actiuni
userResource.reload();   // Re-executa loader-ul cu acelasi request

// Status-uri posibile:
// ResourceStatus.Idle     - inca nu a inceput (request e undefined)
// ResourceStatus.Loading  - in curs de incarcare
// ResourceStatus.Reloading - reincarcare (avea deja date)
// ResourceStatus.Resolved - succes
// ResourceStatus.Error    - eroare
// ResourceStatus.Local    - valoare setata local (fara fetch)

// Scriere locala (WritableResource)
userResource.set(localUser);        // seteaza valoare fara fetch
userResource.update(u => ({...u, name: 'Updated'}));
```

### Exemplu complet: Lista cu cautare si paginare

```typescript
@Component({
  selector: 'app-search-page',
  template: `
    <div class="search-bar">
      <input
        placeholder="Search products..."
        [ngModel]="searchQuery()"
        (ngModelChange)="searchQuery.set($event)" />

      <select [ngModel]="sortBy()" (ngModelChange)="sortBy.set($event)">
        <option value="name">Name</option>
        <option value="price">Price</option>
        <option value="rating">Rating</option>
      </select>
    </div>

    @if (searchResults.isLoading()) {
      <div class="results-skeleton">
        @for (i of [1,2,3,4,5]; track i) {
          <app-card-skeleton />
        }
      </div>
    }

    @if (searchResults.value(); as results) {
      <p class="count">{{ results.total }} results found</p>

      @for (product of results.items; track product.id) {
        <app-product-card [product]="product" />
      } @empty {
        <p>No products match your search.</p>
      }

      <app-paginator
        [currentPage]="page()"
        [totalPages]="results.totalPages"
        (pageChange)="page.set($event)" />
    }

    @if (searchResults.error()) {
      <app-error-state (retry)="searchResults.reload()" />
    }
  `
})
export class SearchPageComponent {
  searchQuery = signal('');
  sortBy = signal('name');
  page = linkedSignal({
    source: this.searchQuery,
    computation: () => 1  // reset la pagina 1 cand se schimba cautarea
  });

  private http = inject(HttpClient);

  // Debounce pe search - folosim un computed cu trick
  // (in practica, pentru debounce real pe signal, bridge la RxJS)
  searchResults = rxResource({
    request: () => ({
      query: this.searchQuery(),
      sort: this.sortBy(),
      page: this.page()
    }),
    loader: ({ request }) =>
      this.http.get<PaginatedResponse<Product>>('/api/products/search', {
        params: {
          q: request.query,
          sort: request.sort,
          page: request.page.toString(),
          limit: '20'
        }
      })
  });
}
```

### Resource vs manual approach

```typescript
// INAINTE - manual cu RxJS (mult boilerplate)
@Component({})
export class OldWayComponent implements OnInit, OnDestroy {
  private destroy$ = new Subject<void>();

  user: User | null = null;
  loading = false;
  error: string | null = null;

  @Input() set userId(id: number) {
    this._userId$.next(id);
  }
  private _userId$ = new Subject<number>();

  ngOnInit() {
    this._userId$.pipe(
      tap(() => {
        this.loading = true;
        this.error = null;
      }),
      switchMap(id =>
        this.http.get<User>(`/api/users/${id}`).pipe(
          catchError(err => {
            this.error = err.message;
            return EMPTY;
          }),
          finalize(() => this.loading = false)
        )
      ),
      takeUntil(this.destroy$)
    ).subscribe(user => this.user = user);
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}

// ACUM - cu resource() (declarativ, automat)
@Component({})
export class NewWayComponent {
  userId = input.required<number>();

  userResource = rxResource({
    request: () => this.userId(),
    loader: ({ request: id }) => this.http.get<User>(`/api/users/${id}`)
  });
  // Loading, error, cleanup - TOTUL e gestionat automat
}
```

---

## 4. Migrare de la RxJS la Signals

### Cand sa migrezi

**Migreaza la Signals:**
- State sincron al componentului (form values, UI toggles, counters)
- State derivat simplu (computed values, filtered lists)
- State partajat intre componente (prin services)
- Input/Output al componentelor (signal inputs)
- Cand vrei sa adopti zoneless change detection

**NU migra la Signals:**
- Streams de events complexe (mouse moves, WebSocket messages)
- Cand ai nevoie de operatori de timp (debounce, throttle, delay)
- Fluxuri cu retry/backoff logic
- Race conditions management (switchMap, exhaustMap)
- Cand RxJS-ul existent functioneaza bine si e testat

### Abordare pas cu pas

**Faza 1: Coexistenta (safe, non-breaking)**

```typescript
// Service - adauga signals LANGA observables existente
@Injectable({ providedIn: 'root' })
export class AuthService {
  // EXISTENT - pastrat pentru compatibilitate
  private userSubject = new BehaviorSubject<User | null>(null);
  user$ = this.userSubject.asObservable();

  // NOU - signal derivat din observable
  user = toSignal(this.user$, { initialValue: null });

  // NOU - computed signal
  isAuthenticated = computed(() => this.user() !== null);

  // Metodele existente raman neschimbate
  login(credentials: Credentials): Observable<User> {
    return this.http.post<User>('/api/login', credentials).pipe(
      tap(user => this.userSubject.next(user))
    );
  }
}
```

**Faza 2: Noi features cu Signals (forward-looking)**

```typescript
// Componente noi folosesc signals direct
@Component({
  template: `
    @if (auth.isAuthenticated()) {
      <app-dashboard />
    } @else {
      <app-login />
    }
  `
})
export class AppComponent {
  auth = inject(AuthService);
  // Foloseste signal-ul, nu observable-ul
}
```

**Faza 3: Migrare graduala a componentelor existente**

```typescript
// INAINTE
@Component({
  template: `
    <div *ngIf="user$ | async as user">{{ user.name }}</div>
    <div *ngIf="loading$ | async">Loading...</div>
  `
})
export class ProfileComponent implements OnInit, OnDestroy {
  user$!: Observable<User>;
  loading$ = new BehaviorSubject(false);
  private destroy$ = new Subject<void>();

  ngOnInit() {
    this.user$ = this.route.params.pipe(
      map(p => p['id']),
      tap(() => this.loading$.next(true)),
      switchMap(id => this.userService.getUser(id)),
      tap(() => this.loading$.next(false)),
      takeUntil(this.destroy$)
    );
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}

// DUPA
@Component({
  template: `
    @if (userResource.isLoading()) {
      <app-spinner />
    }
    @if (userResource.value(); as user) {
      {{ user.name }}
    }
  `
})
export class ProfileComponent {
  private route = inject(ActivatedRoute);
  private userService = inject(UserService);

  private userId = toSignal(
    this.route.params.pipe(map(p => p['id'])),
    { initialValue: '' }
  );

  userResource = rxResource({
    request: () => this.userId(),
    loader: ({ request: id }) =>
      id ? this.userService.getUser(id) : EMPTY
  });
  // Fara OnDestroy, fara takeUntil, fara manual loading state
}
```

**Faza 4: Eliminarea BehaviorSubject-urilor (cand e posibil)**

```typescript
// INAINTE
@Injectable({ providedIn: 'root' })
export class CartService {
  private itemsSubject = new BehaviorSubject<CartItem[]>([]);
  items$ = this.itemsSubject.asObservable();
  totalItems$ = this.items$.pipe(map(items => items.length));
  totalPrice$ = this.items$.pipe(
    map(items => items.reduce((sum, i) => sum + i.price * i.quantity, 0))
  );

  addItem(item: CartItem) {
    const current = this.itemsSubject.value;
    this.itemsSubject.next([...current, item]);
  }
}

// DUPA
@Injectable({ providedIn: 'root' })
export class CartService {
  private _items = signal<CartItem[]>([]);

  readonly items = this._items.asReadonly();
  readonly totalItems = computed(() => this._items().length);
  readonly totalPrice = computed(() =>
    this._items().reduce((sum, i) => sum + i.price * i.quantity, 0)
  );

  // Bridge pentru consumatori legacy
  readonly items$ = toObservable(this._items);

  addItem(item: CartItem) {
    this._items.update(current => [...current, item]);
  }
}
```

### Checklist de migrare

| Pas | Actiune | Risc |
|-----|---------|------|
| 1 | Adauga `toSignal()` wrappers in services existente | Scazut |
| 2 | Inlocuieste `async` pipe cu signal reads in templates | Scazut |
| 3 | Converteste `@Input` la `input()` signal inputs | Mediu |
| 4 | Inlocuieste `BehaviorSubject` cu `signal()` in services de state | Mediu |
| 5 | Adopta `resource()` pentru data fetching | Mediu |
| 6 | Elimina `Zone.js` si treci la zoneless | Ridicat |

---

## 5. RxJS - Operatori esentiali

### Higher-Order Mapping Operators

Acestia sunt operatorii care **transforma un Observable in alt Observable** (flatMap pattern). Diferenta dintre ei e **strategia de gestionare a concurentei**.

#### switchMap - Anuleaza precedentul

**Comportament:** Cand vine o noua valoare, **anuleaza** subscription-ul anterior si creaza unul nou.

**Foloseste pentru:** Search/autocomplete, navigare, orice unde doar ultimul request conteaza.

```typescript
// EXEMPLU CLASIC: Search autocomplete
@Component({
  selector: 'app-search',
  template: `
    <input [formControl]="searchControl" placeholder="Search..." />
    <ul>
      @for (result of results$ | async; track result.id) {
        <li>{{ result.name }}</li>
      }
    </ul>
  `
})
export class SearchComponent {
  searchControl = new FormControl('');

  results$ = this.searchControl.valueChanges.pipe(
    debounceTime(300),          // asteapta 300ms de la ultima tastare
    distinctUntilChanged(),      // nu re-cauta daca valoarea e aceeasi
    filter(term => term.length >= 2), // minim 2 caractere
    switchMap(term =>            // ANULEAZA request-ul anterior!
      this.searchService.search(term).pipe(
        catchError(() => of([]))  // la eroare, returneaza lista goala
      )
    )
  );

  constructor(private searchService: SearchService) {}
}

// DE CE switchMap?
// Utilizatorul tasteaza: "ang" -> request 1 pleaca
// Apoi tasteaza: "angular" -> request 1 este ANULAT, request 2 pleaca
// Fara switchMap: request 1 ar putea reveni DUPA request 2 -> date gresite!
```

```typescript
// EXEMPLU: Route params -> data loading
@Component({})
export class UserDetailComponent {
  user$ = this.route.params.pipe(
    map(params => params['id']),
    switchMap(id => this.userService.getUser(id))
    // Cand navigam de la /users/1 la /users/2,
    // request-ul pentru user 1 este anulat automat
  );
}
```

#### mergeMap - Executie paralela

**Comportament:** Cand vine o noua valoare, creaza un nou subscription **fara sa-l anuleze** pe cel anterior. Toate ruleaza **in paralel**.

**Foloseste pentru:** Fire-and-forget, operatii independente care nu se influenteaza.

```typescript
// EXEMPLU: Upload multiple fisiere simultan
@Component({
  template: `
    <input type="file" multiple (change)="onFilesSelected($event)" />
    <div>{{ uploadProgress$ | async }}% complete</div>
  `
})
export class FileUploadComponent {
  private files$ = new Subject<File>();

  uploads$ = this.files$.pipe(
    mergeMap(
      file => this.uploadService.upload(file).pipe(
        catchError(err => {
          console.error(`Failed to upload ${file.name}:`, err);
          return EMPTY;
        })
      ),
      3 // concurrency limit! Maximum 3 uploads simultan
    )
  );

  onFilesSelected(event: Event) {
    const files = (event.target as HTMLInputElement).files;
    if (files) {
      Array.from(files).forEach(f => this.files$.next(f));
    }
  }
}

// EXEMPLU: Notificari - fiecare se proceseaza independent
notifications$.pipe(
  mergeMap(notification =>
    this.notificationService.process(notification)
  )
).subscribe();
```

**Atentie:** `mergeMap` fara concurrency limit poate cauza memory leaks sau overwhelm server-ul! Foloseste al doilea parametru pentru limit.

#### concatMap - Executie secventiala

**Comportament:** Cand vine o noua valoare, **asteapta** sa se termine observable-ul curent, apoi proceseaza urmatoarea valoare. Garanteaza **ordinea**.

**Foloseste pentru:** Operatii unde ordinea conteaza (queue, tranzactii secventiale).

```typescript
// EXEMPLU: Queue de save-uri secventiale
@Component({
  template: `
    <textarea [formControl]="content" (input)="save()"></textarea>
    @if (saving) { <span>Saving...</span> }
  `
})
export class DocumentEditorComponent implements OnInit, OnDestroy {
  content = new FormControl('');
  saving = false;
  private save$ = new Subject<string>();
  private destroy$ = new Subject<void>();

  ngOnInit() {
    this.save$.pipe(
      concatMap(text => {
        this.saving = true;
        return this.documentService.save(text).pipe(
          finalize(() => this.saving = false)
        );
      }),
      takeUntil(this.destroy$)
    ).subscribe();
  }

  save() {
    this.save$.next(this.content.value!);
    // Daca save-ul anterior e inca in curs,
    // noul save asteapta sa se termine primul
    // NICIODATA nu pierde un save si ordinea e garantata
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}

// EXEMPLU: Operatii cu dependente secventiale
createOrder$.pipe(
  concatMap(order => this.orderService.create(order)),
  concatMap(createdOrder => this.paymentService.charge(createdOrder.id)),
  concatMap(payment => this.notificationService.send(payment.receipt))
).subscribe();
```

#### exhaustMap - Ignora pana se termina

**Comportament:** Cand vine o noua valoare si exista un subscription activ, **ignora** noua valoare. Proceseaza urmatoarea abia dupa ce cel curent se termina.

**Foloseste pentru:** Form submit, login, orice actiune care NU trebuie duplicata.

```typescript
// EXEMPLU CLASIC: Form submit - previne double-submit
@Component({
  template: `
    <form (ngSubmit)="submit$.next()">
      <!-- form fields -->
      <button type="submit">Place Order</button>
    </form>
  `
})
export class CheckoutComponent implements OnInit, OnDestroy {
  submit$ = new Subject<void>();
  private destroy$ = new Subject<void>();

  ngOnInit() {
    this.submit$.pipe(
      exhaustMap(() =>
        this.orderService.placeOrder(this.form.value).pipe(
          tap({
            next: (order) => this.router.navigate(['/orders', order.id]),
            error: (err) => this.showError(err.message)
          }),
          catchError(() => EMPTY) // previne unsubscribe pe eroare
        )
      ),
      takeUntil(this.destroy$)
    ).subscribe();
  }

  // De ce exhaustMap si nu switchMap?
  // switchMap ar ANULA comanda anterioara - PERICULOS!
  // exhaustMap IGNORA click-urile suplimentare pana comanda se finalizeaza

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}

// EXEMPLU: Login - nu trimite multiple request-uri
loginButton$.pipe(
  exhaustMap(() =>
    this.authService.login(credentials).pipe(
      catchError(err => {
        this.errorMessage = err.message;
        return EMPTY;
      })
    )
  )
).subscribe();

// EXEMPLU: Refresh token
tokenRefresh$.pipe(
  exhaustMap(() => this.authService.refreshToken())
).subscribe();
```

### Tabel comparativ: Higher-Order Mapping

| Operator | Anterior activ? | Noua valoare | Metafora |
|----------|----------------|--------------|----------|
| `switchMap` | Anuleaza | Proceseaza | Schimba canalul TV |
| `mergeMap` | Pastreaza | Proceseaza paralel | Mai multi bucatari in bucatarie |
| `concatMap` | Pastreaza | Asteapta la coada | Coada la magazin |
| `exhaustMap` | Pastreaza | Ignora | "Ocupat, reveniti" |

### Operatori de combinare

#### combineLatest

Emite cand **oricare** sursa emite, combinand **cele mai recente** valori de la toate sursele. Necesita ca **fiecare sursa sa fi emis cel putin o data**.

```typescript
// EXEMPLU: Filtrare cu multiple criterii
@Component({
  template: `
    <input [formControl]="search" placeholder="Search" />
    <select [formControl]="category">
      <option value="">All</option>
      <option value="electronics">Electronics</option>
      <option value="books">Books</option>
    </select>
    <select [formControl]="sortBy">
      <option value="name">Name</option>
      <option value="price">Price</option>
    </select>

    @for (product of filteredProducts$ | async; track product.id) {
      <app-product-card [product]="product" />
    }
  `
})
export class ProductFilterComponent {
  search = new FormControl('');
  category = new FormControl('');
  sortBy = new FormControl('name');

  filteredProducts$ = combineLatest([
    this.search.valueChanges.pipe(startWith('')),
    this.category.valueChanges.pipe(startWith('')),
    this.sortBy.valueChanges.pipe(startWith('name'))
  ]).pipe(
    debounceTime(200), // debounce combinat
    switchMap(([search, category, sort]) =>
      this.productService.search({ search, category, sort })
    )
  );
}

// EXEMPLU: Dashboard cu date din surse multiple
dashboardData$ = combineLatest({
  user: this.userService.currentUser$,
  notifications: this.notificationService.unread$,
  stats: this.statsService.today$
}).pipe(
  // Emite un obiect cu toate cele 3 valori
  // Se actualizeaza cand ORICARE se schimba
);
```

#### forkJoin

Emite **o singura data**, cand **toate** observables-urile se **completeaza**. E ca `Promise.all()`.

```typescript
// EXEMPLU: Incarcare initiala - asteapta toate datele
@Component({})
export class DashboardComponent implements OnInit {
  loading = true;
  dashboardData!: DashboardData;

  ngOnInit() {
    forkJoin({
      user: this.userService.getCurrentUser(),
      orders: this.orderService.getRecent(5),
      stats: this.statsService.getToday(),
      notifications: this.notificationService.getUnread()
    }).pipe(
      finalize(() => this.loading = false)
    ).subscribe({
      next: (data) => {
        // data = { user: User, orders: Order[], stats: Stats, notifications: Notification[] }
        this.dashboardData = data;
      },
      error: (err) => {
        // ATENTIE: daca ORICARE esueaza, TOTUL esueaza
        this.showError('Failed to load dashboard');
      }
    });
  }
}

// ATENTIE cu forkJoin:
// - Daca un observable nu se completeaza niciodata, forkJoin NU emite niciodata
// - Daca un observable emite eroare, TOATE sunt anulate
// - Nu il folosi cu observables infinite (Subject-uri, intervals)
```

#### withLatestFrom

Cand sursa principala emite, combina cu **cea mai recenta valoare** de la celelalte surse. Diferenta fata de `combineLatest`: doar sursa principala triggeruieste emisia.

```typescript
// EXEMPLU: Save cu context curent
saveButton$.pipe(
  withLatestFrom(
    this.authService.currentUser$,
    this.formData$
  ),
  exhaustMap(([_, user, formData]) =>
    this.api.save({
      ...formData,
      updatedBy: user.id,
      updatedAt: new Date()
    })
  )
).subscribe();

// EXEMPLU: Actiune cu configurare curenta
refreshButton$.pipe(
  withLatestFrom(this.settingsService.config$),
  switchMap(([_, config]) =>
    this.dataService.fetch(config.endpoint)
  )
).subscribe();
```

### Operatori de filtrare si timing

#### debounceTime

Asteapta un interval fara noi valori, apoi emite ultima valoare.

```typescript
// Search input - asteapta 300ms dupa ce utilizatorul se opreste din tastat
searchInput$.pipe(
  debounceTime(300),
  distinctUntilChanged(),
  switchMap(term => this.search(term))
);

// Auto-save - salveaza la 2 secunde dupa ultima modificare
formChanges$.pipe(
  debounceTime(2000),
  switchMap(value => this.save(value))
);
```

#### distinctUntilChanged

Emite doar cand valoarea s-a schimbat fata de precedenta.

```typescript
// Default - foloseste === pentru comparatie
searchTerm$.pipe(
  distinctUntilChanged() // "abc" -> "abc" = ignorat
);

// Custom comparator - util pentru obiecte
selectedFilter$.pipe(
  distinctUntilChanged((prev, curr) =>
    prev.category === curr.category && prev.sortBy === curr.sortBy
  )
);
```

#### takeUntil

**Completare automata** - se dezaboneaza cand Observable-ul furnizat emite. **Pattern-ul standard** pentru cleanup in componente Angular.

```typescript
@Component({})
export class MyComponent implements OnDestroy {
  private destroy$ = new Subject<void>();

  ngOnInit() {
    this.someService.data$.pipe(
      takeUntil(this.destroy$)
    ).subscribe(data => this.processData(data));

    this.anotherService.events$.pipe(
      takeUntil(this.destroy$)
    ).subscribe(event => this.handleEvent(event));
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}

// MODERN: cu DestroyRef (Angular 16+)
@Component({})
export class ModernComponent {
  private destroyRef = inject(DestroyRef);

  ngOnInit() {
    this.someService.data$.pipe(
      takeUntilDestroyed(this.destroyRef)
    ).subscribe(data => this.processData(data));
  }
}

// SAU si mai simplu - takeUntilDestroyed() in constructor/field initializer
@Component({})
export class SimplestComponent {
  data$ = this.someService.data$.pipe(
    takeUntilDestroyed() // fara argument, preia automat din injection context
  );
}
```

### Error handling

#### catchError

Intercepteaza erori si returneaza un Observable alternativ.

```typescript
// EXEMPLU: Fallback la eroare
this.http.get<User[]>('/api/users').pipe(
  catchError(error => {
    console.error('API failed, using cached data:', error);
    return of(this.cachedUsers); // fallback
  })
);

// EXEMPLU: Re-throw cu transformare
this.http.get('/api/data').pipe(
  catchError(error => {
    if (error.status === 401) {
      this.router.navigate(['/login']);
      return EMPTY; // nu emite nimic
    }
    // Re-throw alte erori
    return throwError(() => new AppError('Data fetch failed', error));
  })
);

// PATTERN: catchError IN switchMap (nu afara!)
searchTerm$.pipe(
  switchMap(term =>
    this.api.search(term).pipe(
      catchError(() => of([])) // eroarea nu distruge stream-ul outer
    )
  )
  // GRESIT: catchError AICI ar distruge intregul stream la prima eroare
);
```

#### retry / retryWhen

```typescript
// Retry simplu
this.http.get('/api/data').pipe(
  retry({ count: 3, delay: 1000 }) // 3 incercari cu 1s delay
);

// Exponential backoff
this.http.get('/api/data').pipe(
  retry({
    count: 3,
    delay: (error, retryCount) => {
      const delay = Math.pow(2, retryCount) * 1000; // 2s, 4s, 8s
      console.log(`Retry #${retryCount} in ${delay}ms`);
      return timer(delay);
    }
  })
);

// Retry doar pe anumite erori
this.http.get('/api/data').pipe(
  retry({
    count: 3,
    delay: (error) => {
      if (error.status === 404 || error.status === 403) {
        return throwError(() => error); // nu retry pe 404/403
      }
      return timer(1000);
    }
  })
);
```

#### finalize

Executa cod la **completare** sau **eroare** (echivalentul `finally` din try/catch).

```typescript
// EXEMPLU: Loading state
loadData() {
  this.loading = true;
  this.http.get<Data[]>('/api/data').pipe(
    finalize(() => {
      this.loading = false; // se executa MEREU, succes sau eroare
    })
  ).subscribe({
    next: data => this.data = data,
    error: err => this.error = err.message
  });
}
```

---

## 6. Patterns reactive

### Subject vs BehaviorSubject vs ReplaySubject vs AsyncSubject

Un **Subject** este simultan **Observable** (poti face subscribe) si **Observer** (poti face `.next()`, `.error()`, `.complete()`). Este un multicast event bus.

#### Subject

```typescript
import { Subject } from 'rxjs';

// Subject simplu - NU retine valori, NU are valoare initiala
const subject = new Subject<string>();

// Subscriber A - se aboneaza INAINTE de emit
subject.subscribe(val => console.log('A:', val));

subject.next('Hello');  // A: Hello
subject.next('World');  // A: World

// Subscriber B - se aboneaza DUPA emit
subject.subscribe(val => console.log('B:', val));

subject.next('!');      // A: !   B: !
// B nu a primit 'Hello' si 'World' - s-a abonat prea tarziu

// USE CASE: Event bus, actiuni (click$, submit$, refresh$)
@Injectable({ providedIn: 'root' })
export class EventBusService {
  private actions$ = new Subject<Action>();

  dispatch(action: Action) {
    this.actions$.next(action);
  }

  on(type: string): Observable<Action> {
    return this.actions$.pipe(
      filter(action => action.type === type)
    );
  }
}
```

#### BehaviorSubject

```typescript
import { BehaviorSubject } from 'rxjs';

// BehaviorSubject - NECESITA valoare initiala, RETINE ultima valoare
const subject = new BehaviorSubject<number>(0); // valoare initiala: 0

// Poti accesa valoarea curenta SINCRON
console.log(subject.value); // 0
console.log(subject.getValue()); // 0 (alternativ)

subject.subscribe(val => console.log('A:', val)); // A: 0 (primeste imediat!)

subject.next(1); // A: 1
subject.next(2); // A: 2

subject.subscribe(val => console.log('B:', val)); // B: 2 (primeste ultima valoare!)

subject.next(3); // A: 3   B: 3

// USE CASE: State management, current value store
@Injectable({ providedIn: 'root' })
export class AuthService {
  private currentUserSubject = new BehaviorSubject<User | null>(null);

  // Expune ca Observable readonly
  currentUser$ = this.currentUserSubject.asObservable();

  // Getter sincron pentru valoarea curenta
  get currentUser(): User | null {
    return this.currentUserSubject.value;
  }

  login(user: User): void {
    this.currentUserSubject.next(user);
  }

  logout(): void {
    this.currentUserSubject.next(null);
  }
}
```

#### ReplaySubject

```typescript
import { ReplaySubject } from 'rxjs';

// ReplaySubject - retine ultimele N valori si le trimite la noi subscriberi
const subject = new ReplaySubject<string>(3); // buffer size: 3

subject.next('A');
subject.next('B');
subject.next('C');
subject.next('D');

// Noul subscriber primeste ultimele 3 valori: B, C, D
subject.subscribe(val => console.log('Late:', val));
// Late: B
// Late: C
// Late: D

// Cu window de timp: retine valori din ultimele 2 secunde
const timedSubject = new ReplaySubject<string>(Infinity, 2000);

// USE CASE: Mesaje chat (ultimele N mesaje), log-uri, audit trail
@Injectable({ providedIn: 'root' })
export class NotificationService {
  // Retine ultimele 50 de notificari
  private notifications$ = new ReplaySubject<Notification>(50);

  push(notification: Notification): void {
    this.notifications$.next(notification);
  }

  // Un component care se afiseaza mai tarziu primeste istoricul
  getNotifications(): Observable<Notification> {
    return this.notifications$.asObservable();
  }
}
```

#### AsyncSubject

```typescript
import { AsyncSubject } from 'rxjs';

// AsyncSubject - emite DOAR ultima valoare si DOAR la complete
const subject = new AsyncSubject<string>();

subject.subscribe(val => console.log('A:', val));

subject.next('X');
subject.next('Y');
subject.next('Z');
// Nimic inca! Subscriber A nu a primit nimic.

subject.complete();
// A: Z (DOAR ultima valoare, DOAR la complete)

// Noul subscriber primeste tot Z
subject.subscribe(val => console.log('B:', val));
// B: Z

// USE CASE: Rezultatul final al unei operatii, cache one-time values
// In practica se foloseste rar - Observable.last() sau shareReplay(1) sunt
// de obicei mai potrivite.
```

### Tabel comparativ

| Tip | Valoare initiala | Retine | Emite la subscribe | Use case |
|-----|-----------------|--------|-------------------|----------|
| `Subject` | Nu | Nimic | Nimic | Events, actions |
| `BehaviorSubject` | Da (obligatorie) | Ultima valoare | Ultima valoare | Current state |
| `ReplaySubject` | Nu | Ultimele N | Ultimele N | History, cache |
| `AsyncSubject` | Nu | Ultima valoare | La complete | Final result |

### Cand sa folosesti fiecare

```
Ai nevoie de valoarea curenta SINCRON?
  DA -> BehaviorSubject (sau Signal!)
  NU -> continua

Noii subscriberi trebuie sa primeasca valori trecute?
  DA -> Cate?
    Una -> BehaviorSubject
    N -> ReplaySubject(N)
    Toate -> ReplaySubject() fara limit
  NU -> Subject

Trebuie doar rezultatul final?
  DA -> AsyncSubject (sau lastValueFrom)
  NU -> Subject
```

---

## 7. Cand sa folosesti Signals vs Observables

### Matrice de decizie

| Scenariu | Signal | Observable | De ce |
|----------|--------|-----------|-------|
| UI state (show/hide, toggle) | **Da** | Nu | Sincron, simplu |
| Form values | **Da** | Partial | Signals pentru valori, RxJS pentru validare async |
| Date din HTTP | **Da** (resource) | **Da** | Resource API simplifica, dar RxJS e la fel de valid |
| Search autocomplete | Nu | **Da** | Necesita debounce, switchMap |
| WebSocket stream | Nu | **Da** | Stream infinit, push-based |
| State in service | **Da** | Legacy | Signals inlocuiesc BehaviorSubject |
| Animatii bazate pe timp | Nu | **Da** | timer, interval, animationFrameScheduler |
| Retry / backoff logic | Nu | **Da** | Necesita operatori de retry |
| Derivare sincron de state | **Da** | Nu | computed() e ideal |
| Race conditions | Nu | **Da** | switchMap/exhaustMap |
| Event bus (pub/sub) | Nu | **Da** | Subject pattern |
| Route params -> data | **Da** | **Da** | resource() cu toSignal(route.params) sau switchMap |

### Regula practica

```
State SINCRON al aplicatiei -> Signals
  (variabile, toggle-uri, selected items, form values, computed derivates)

Streams ASINCRONE cu logica complexa -> Observables
  (events in timp, retry, debounce, race conditions, WebSocket)

Date din server -> Resource API (bridge-ul ideal)
  (combina signals cu async loading)
```

### Ghid pe nivel de component

```typescript
// COMPONENT: majoritar Signals
@Component({
  template: `
    <input [ngModel]="search()" (ngModelChange)="search.set($event)" />
    {{ filteredCount() }} results

    @for (item of filtered(); track item.id) { ... }
  `
})
export class ListComponent {
  items = input.required<Item[]>();  // signal input
  search = signal('');
  filtered = computed(() => /* ... */);
  filteredCount = computed(() => this.filtered().length);
}

// SERVICE: poate fi mix
@Injectable({ providedIn: 'root' })
export class DataService {
  // State -> Signals
  private _items = signal<Item[]>([]);
  readonly items = this._items.asReadonly();

  // Complex async flows -> RxJS (internal)
  loadItems(): Observable<void> {
    return this.http.get<Item[]>('/api/items').pipe(
      retry(3),
      tap(items => this._items.set(items)),
      map(() => void 0),
      catchError(err => {
        this.errorService.report(err);
        return throwError(() => err);
      })
    );
  }
}
```

---

## 8. toSignal() si toObservable() - Bridging

### toSignal() - Observable to Signal

Converteste un Observable intr-un Signal readonly. **Subscription-ul se gestioneaza automat** (se dezaboneaza la destroy).

```typescript
import { toSignal } from '@angular/core/rxjs-interop';

@Component({
  template: `
    <p>Current route: {{ currentRoute() }}</p>
    <p>Window width: {{ windowWidth() }}</p>
  `
})
export class MyComponent {
  private route = inject(ActivatedRoute);

  // VARIANTA 1: Cu initialValue (tipul include initialValue)
  currentRoute = toSignal(
    this.route.params.pipe(map(p => p['id'])),
    { initialValue: '' } // Signal<string>
  );

  // VARIANTA 2: Fara initialValue (tipul include undefined)
  currentRoute2 = toSignal(
    this.route.params.pipe(map(p => p['id']))
  );
  // Tipul: Signal<string | undefined>
  // Trebuie sa gestionezi undefined in template!

  // VARIANTA 3: requireSync - pentru BehaviorSubject sau ReplaySubject(1)
  // care emit imediat (sincron)
  windowWidth = toSignal(
    fromEvent(window, 'resize').pipe(
      map(() => window.innerWidth),
      startWith(window.innerWidth) // emit sincron la subscribe
    ),
    { requireSync: true } // Signal<number> - fara undefined!
  );
}
```

**Gotchas importante:**

```typescript
// GOTCHA 1: toSignal() face subscribe IMEDIAT si tine subscription-ul
// activ pana la destroy. Nu il folosi pentru Observable-uri one-shot
// daca nu vrei sa le tii vii.

// GOTCHA 2: Trebuie apelat in injection context
// (constructor, field initializer, sau cu injector explicit)

// GRESIT - in afara injection context:
ngOnInit() {
  this.data = toSignal(this.service.getData()); // EROARE!
}

// CORECT - cu injector explicit:
private injector = inject(Injector);

ngOnInit() {
  this.data = toSignal(this.service.getData(), {
    injector: this.injector,
    initialValue: null
  });
}

// CORECT - in field initializer (injection context):
data = toSignal(this.service.getData$, { initialValue: null });

// GOTCHA 3: Ce se intampla la error?
// toSignal() ARUNCERA eroarea la urmatoarea citire!
errorSignal = toSignal(
  this.http.get('/api/data').pipe(
    catchError(() => of(null)) // TRATEAZA erorile inainte de toSignal!
  ),
  { initialValue: null }
);

// GOTCHA 4: manualCleanup - NU se dezaboneaza automat
const persistent = toSignal(source$, {
  manualCleanup: true // ATENTIE: trebuie gestionat manual!
});
```

### toObservable() - Signal to Observable

Converteste un Signal intr-un Observable. Emite cand valoarea signal-ului se schimba.

```typescript
import { toObservable } from '@angular/core/rxjs-interop';

@Component({})
export class SearchComponent {
  searchTerm = signal('');

  // Converteste signal -> observable pentru a folosi operatori RxJS
  results$ = toObservable(this.searchTerm).pipe(
    debounceTime(300),
    distinctUntilChanged(),
    filter(term => term.length >= 2),
    switchMap(term => this.searchService.search(term)),
    catchError(() => of([]))
  );

  // Apoi poti converti inapoi la signal daca vrei
  results = toSignal(this.results$, { initialValue: [] as SearchResult[] });
}
```

**Pattern avansat: Signal -> RxJS -> Signal pipeline:**

```typescript
@Component({})
export class AutocompleteComponent {
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
  results = toSignal(this.results$, { initialValue: [] });

  // Template foloseste signals:
  // {{ results().length }} results
  // @for (r of results(); track r.id) { ... }

  // Acest pattern combina cel mai bun din ambele lumi:
  // - Signals pentru template binding (simplu, no async pipe)
  // - RxJS pentru logica de timp si cancellation
}
```

**Gotchas toObservable():**

```typescript
// GOTCHA 1: Emite ASINCRON (in microtask), nu sincron
const count = signal(0);
const count$ = toObservable(count);

count$.subscribe(val => console.log(val));
count.set(1);
count.set(2);
count.set(3);
// Console: 3 (nu 0, 1, 2, 3!)
// toObservable coaleste schimbarile si emite doar ultima valoare
// in urmatorul microtask

// GOTCHA 2: Necesita injection context (ca si toSignal)
// Trebuie apelat in constructor sau field initializer

// GOTCHA 3: Prima emisie
// toObservable emite valoarea curenta a signal-ului la subscribe
// (similar cu BehaviorSubject)
```

### Tabel: toSignal() options

| Optiune | Tip | Default | Efect |
|---------|-----|---------|-------|
| `initialValue` | `T` | - | Valoare inainte de prima emisie; elimina `undefined` din tip |
| `requireSync` | `boolean` | `false` | Asigura ca Observable-ul emite sincron; eroare daca nu |
| `injector` | `Injector` | auto | Permite apelarea in afara injection context |
| `manualCleanup` | `boolean` | `false` | Nu se dezaboneaza la destroy |
| `equal` | `(a, b) => boolean` | `Object.is` | Custom equality pentru notificari |

---

## 9. Stabilizare Signals in Angular 20-21 (cu exemple inainte/dupa)

### Angular 20 - Signals API devine stabil

#### effect() - de la developer preview la stabil

```typescript
// INAINTE (Angular 17-19): effect() era in developer preview
// Putea avea breaking changes intre versiuni minore
// Unii developeri evitau sa-l foloseasca in productie
import { signal, effect } from '@angular/core';

@Component({})
export class OldComponent {
  theme = signal('light');

  // In Angular 17-18, effect() avea API putin diferit
  // si era marcat ca @developerPreview
  themeEffect = effect(() => {
    document.body.setAttribute('data-theme', this.theme());
  });
}

// Alternativa "safe" folosita in loc de effect():
@Component({})
export class SafeOldComponent implements OnChanges {
  @Input() theme = 'light';

  ngOnChanges() {
    document.body.setAttribute('data-theme', this.theme);
  }
}

// DUPA (Angular 20): effect() stabil - safe pentru productie
@Component({})
export class NewComponent {
  theme = signal('light');

  // API-ul e garantat stabil, nu se va schimba
  themeEffect = effect(() => {
    document.body.setAttribute('data-theme', this.theme());
  });
}
```

#### linkedSignal() - de la developer preview la stabil

```typescript
// INAINTE (Angular 18 si mai devreme): effect() cu allowSignalWrites (hack)
@Component({})
export class OldComponent {
  category = signal('Electronics');
  selectedItem = signal<string | null>(null);

  // Pattern problematic: effect() care scrie signals
  private resetEffect = effect(() => {
    this.category(); // track
    this.selectedItem.set(null); // NECESITA allowSignalWrites
  }, { allowSignalWrites: true });
  // Probleme: difficult to reason about, ordering issues
}

// DUPA (Angular 20, stabil): linkedSignal rezolva elegant
@Component({})
export class NewComponent {
  category = signal('Electronics');

  // Se reseteaza automat la null cand category se schimba
  // DAR poate fi scris manual de user (e WritableSignal)
  selectedItem = linkedSignal({
    source: this.category,
    computation: () => null
  });
}
```

#### Zoneless change detection - de la experimental la developer preview

```typescript
// INAINTE (Angular 18): experimental, API instabil
bootstrapApplication(AppComponent, {
  providers: [
    provideExperimentalZonelessChangeDetection() // Angular 18
    // API-ul putea sa se schimbe oricand
  ]
});

// DUPA (Angular 20): developer preview, API stabil
bootstrapApplication(AppComponent, {
  providers: [
    provideZonelessChangeDetection() // Angular 20
    // Fara "Experimental" in nume, API stabilizat
    // Inca DP dar garanteaza backwards compatibility
  ]
});
```

### Angular 21 - Zoneless devine default

```typescript
// INAINTE (Angular 19-20): Zone.js era default, zoneless era opt-in
// angular.json includea zone.js in polyfills
// {
//   "polyfills": ["zone.js"]
// }

bootstrapApplication(AppComponent, {
  providers: [
    provideRouter(routes),
    // Zone.js monitoriza AUTOMAT:
    // - setTimeout/setInterval
    // - Promise.then
    // - addEventListener
    // -> rula change detection la FIECARE event async
  ]
});

// DUPA (Angular 21): zoneless default la ng new
// angular.json NU mai include zone.js
bootstrapApplication(AppComponent, {
  providers: [
    provideRouter(routes),
    // Zoneless e default! Change detection se bazeaza pe:
    // - Signals care se schimba
    // - Events din template (click, input, etc.)
    // - markForCheck() explicit
    //
    // setTimeout(() => this.name = 'X', 1000) -> NU triggeruieste CD!
    // setTimeout(() => this.name.set('X'), 1000) -> OK, signal notifica Angular
  ]
});
```

### Signal Forms (Angular 21 - Experimental)
Signal Forms sunt cel mai important feature nou legat de signals. Mai jos, comparatie completa inainte/dupa:

#### Definire formular si validari

```typescript
// ===== INAINTE: Reactive Forms (Angular 14-20) =====
import { FormBuilder, Validators, AbstractControl, ValidationErrors } from '@angular/forms';

@Injectable()
export class OldFormComponent {
  private fb = inject(FormBuilder);

  // Form group cu validari in arrays
  profileForm = this.fb.group({
    name: ['', [Validators.required]],
    email: ['', [Validators.required, Validators.email]],
    age: [0, [Validators.required, Validators.min(18)]],
    password: ['', [Validators.required, Validators.minLength(8)]],
    confirmPassword: ['', [Validators.required]]
  }, {
    // Cross-field validator ca functie separata
    validators: [this.passwordMatchValidator]
  });

  // Cross-field validator - verbose
  passwordMatchValidator(control: AbstractControl): ValidationErrors | null {
    const password = control.get('password')?.value;
    const confirm = control.get('confirmPassword')?.value;
    return password === confirm ? null : { mismatch: true };
  }
}

// ===== DUPA: Signal Forms (Angular 21) =====
import { form, required, email, minLength, validate, customError } from '@angular/forms/signals';

@Component({})
export class NewFormComponent {
  private model = signal({ name: '', email: '', age: 0, password: '', confirmPassword: '' });

  profileForm = form(this.model, (f) => {
    required(f.name);
    required(f.email);
    email(f.email);
    required(f.password);
    minLength(f.password, 8);
    required(f.confirmPassword);

    // Cross-field validator - inline, cu auto-tracking
    validate(f.confirmPassword, ({ value, valueOf }) => {
      if (value() !== valueOf(f.password)) {
        return customError({ kind: 'mismatch', message: 'Parolele nu coincid' });
      }
      return undefined;
      // Se re-evalueaza AUTOMAT cand password SAU confirmPassword se schimba
      // (Reactive Forms necesita manual updateValueAndValidity())
    });
  });
}
```

#### Template binding

```html
<!-- ===== INAINTE: Reactive Forms ===== -->
<form [formGroup]="profileForm" (ngSubmit)="onSubmit()">
  <input formControlName="name" />
  @if (profileForm.get('name')?.errors?.['required'] &&
       profileForm.get('name')?.touched) {
    <span class="error">Numele e obligatoriu</span>
  }

  <input formControlName="email" />
  @if (profileForm.get('email')?.hasError('email') &&
       profileForm.get('email')?.dirty) {
    <span class="error">Email invalid</span>
  }

  <button type="submit"
          [disabled]="profileForm.invalid || submitting">
    {{ submitting ? 'Se trimite...' : 'Salveaza' }}
  </button>
</form>
<!-- Nota: trebuia gestionat manual `submitting` state cu boolean -->

<!-- ===== DUPA: Signal Forms ===== -->
<input [formField]="profileForm.name">
@if (profileForm.name().errors().length && profileForm.name().touched()) {
  <span class="error">Numele e obligatoriu</span>
}

<input [formField]="profileForm.email">
@if (profileForm.email().errors().length && profileForm.email().dirty()) {
  <span class="error">Email invalid</span>
}

<button (click)="onSubmit()"
        [disabled]="profileForm().submitting()">
  {{ profileForm().submitting() ? 'Se trimite...' : 'Salveaza' }}
</button>
<!-- Nota: submitting state gestionat automat de submit() -->
```

#### Acces la stare si submit

```typescript
// ===== INAINTE: Reactive Forms =====
// Acces la valori si stare
const name = this.profileForm.get('name')?.value;       // any!
const isValid = this.profileForm.get('name')?.valid;     // boolean | undefined
const errors = this.profileForm.get('name')?.errors;     // ValidationErrors | null

// Submit manual
submitting = false;
async onSubmit() {
  if (this.profileForm.invalid) return;
  this.submitting = true;
  try {
    await this.api.save(this.profileForm.value);
    // profileForm.value poate contine null pentru optionale
    // Tipul: Partial<{name: string | null, email: string | null, ...}>
  } finally {
    this.submitting = false;
  }
}

// ===== DUPA: Signal Forms =====
// Acces la valori si stare - fully typed
const name = this.profileForm.name().value();     // string (typed!)
const isValid = this.profileForm.name().valid();   // boolean
const errors = this.profileForm.name().errors();   // Error[] (typed!)

// Submit gestionat automat
async onSubmit() {
  await submit(this.profileForm, async (form) => {
    // Se executa DOAR daca valid
    // form().submitting() === true automat
    // Previne double-submit automat
    await this.api.save(form().value());
    // form().value() -> tipul exact al modelului, fara null
  });
}
```

#### Schema reutilizabila (nou in Signal Forms)

```typescript
// ===== INAINTE: Reactive Forms - validari refolosite cu functii ===
function createAddressFormGroup(fb: FormBuilder): FormGroup {
  return fb.group({
    street: ['', Validators.required],
    city: ['', Validators.required],
    zip: ['', [Validators.required, Validators.pattern(/^\d{6}$/)]],
  });
}

// Utilizare
orderForm = this.fb.group({
  billing: createAddressFormGroup(this.fb),
  shipping: createAddressFormGroup(this.fb),
});
// Problema: nu e type-safe, returneaza FormGroup generic

// ===== DUPA: Signal Forms - schema declarativa =====
const addressSchema = schema<Address>((a) => {
  required(a.street);
  required(a.city);
  required(a.zip);
  validate(a.zip, ({ value }) => {
    if (!/^\d{6}$/.test(value())) {
      return customError({ kind: 'pattern', message: 'Cod postal invalid' });
    }
    return undefined;
  });
});

// Utilizare - type-safe, compozabila
const orderModel = signal({ billing: emptyAddress(), shipping: emptyAddress() });
const orderForm = form(orderModel, (f) => {
  apply(f.billing, addressSchema);
  apply(f.shipping, addressSchema);
});
// orderForm.billing.city().value() -> string (fully typed!)
```

**Diferente cheie Signal Forms vs Reactive Forms:**

| Aspect | Signal Forms | Reactive Forms |
|--------|-------------|----------------|
| Sursa stare | Signal model extern | Form-ul detine starea |
| Type safety | Completa, fara null | Necesita casting |
| Navigare | Dot notation tipizata | `get()` returneaza AbstractControl |
| Custom controls | `FormValueControl` | `ControlValueAccessor` |
| Validatori | Auto-track dependente | Manual `updateValueAndValidity()` |
| Directiva | O singura: `[formField]` | Multiple: formControl, formGroup, etc |
| Submit | `submit()` gestioneaza totul | Manual: loading, validation, prevent double |
| Reactivitate | Nativa cu signals/computed | valueChanges Observable |

---

## 10. Intrebari frecvente de interviu

### Q1: Care este diferenta fundamentala dintre Signals si Observables?

**Signals** sunt **primitive reactive sincrone, pull-based** - contin mereu o valoare curenta pe care o citesti apeland signal-ul ca functie. Sunt ideale pentru state management sincron si template binding.

**Observables** sunt **stream-uri asincrone, push-based** - emit valori in timp, pot fi infinite, si ofera un ecosistem bogat de operatori pentru transformari complexe (debounce, retry, race conditions).

In practica: Signals pentru **state**, Observables pentru **events si fluxuri asincrone complexe**.

---

### Q2: De ce a introdus Angular Signals cand avea deja RxJS?

Trei motive principale:

1. **Eliminarea Zone.js** - Zone.js patch-uieste TOATE API-urile async ale browser-ului pentru change detection. Signals permit Angular sa stie EXACT ce s-a schimbat, fara monkey-patching global. Asta deschide calea catre **zoneless Angular**.

2. **Simplitate** - RxJS are o curba de invatare abrupta. Pentru state management simplu (un counter, un toggle, o lista filtrata), Signals sunt dramatic mai simple.

3. **Performance granulara** - Cu Signals, Angular poate face change detection la nivel de **binding individual**, nu la nivel de component tree. Stie PRECIS care binding din care template s-a schimbat.

---

### Q3: Cand nu ar trebui sa folosesti Signals?

- Cand ai nevoie de **operatori de timp** (debounceTime, throttleTime, delay, interval)
- Cand gestionezi **race conditions** (switchMap pentru search, exhaustMap pentru submit)
- Cand lucrezi cu **streams infinite** (WebSocket, Server-Sent Events)
- Cand ai **retry logic cu backoff** exponential
- Cand ai **logica complexa de combinare** cu timing (buffer, window, sample)

In aceste cazuri, foloseste RxJS si bridge-ul `toSignal()` / `toObservable()` la granita cu template-ul.

---

### Q4: Explica diferenta dintre switchMap, mergeMap, concatMap si exhaustMap.

**switchMap**: Anuleaza subscription-ul anterior cand vine o noua valoare. Foloseste pentru search/autocomplete - doar ultimul request conteaza.

**mergeMap**: Ruleaza toate subscription-urile in paralel. Foloseste pentru fire-and-forget operations - upload multiple fisiere simultan.

**concatMap**: Asteapta sa se termine subscription-ul curent, apoi proceseaza urmatorul. Foloseste cand ordinea conteaza - queue de save-uri.

**exhaustMap**: Ignora noile valori cat timp subscription-ul curent e activ. Foloseste pentru form submit - previne double-submit.

---

### Q5: Ce este linkedSignal si cand il folosesti?

`linkedSignal()` (Angular 19) creaza un **WritableSignal care se reseteaza automat** cand o sursa signal se schimba. E ca un `computed()` care poate fi si scris manual.

**Use case tipic:** Selectie dependenta - ai un dropdown de categorii si unul de produse. Cand schimbi categoria, produsul selectat trebuie resetat la primul din noua categorie. Dar utilizatorul poate selecta manual orice produs. `linkedSignal` rezolva acest pattern elegant, fara `effect()` cu `allowSignalWrites`.

---

### Q6: Cum functioneaza Resource API si ce problema rezolva?

`resource()` si `rxResource()` sunt API-uri declarative pentru data fetching. Rezolva boilerplate-ul de: loading state manual, error handling, cancellation la schimbare de parametri, si cleanup la destroy.

Un resource are un `request` (derivat din signals) si un `loader` (functia async). Cand signals din request se schimba, loader-ul se re-executa automat, request-ul anterior este anulat, si resource-ul expune `value()`, `status()`, `error()`, `isLoading()`.

Inlocuieste pattern-ul clasic de `BehaviorSubject loading + switchMap + takeUntil + finalize`.

---

### Q7: Care este pattern-ul corect de cleanup pentru subscriptions in Angular?

**Modern (recomandat):** `takeUntilDestroyed()` din `@angular/core/rxjs-interop`:

```typescript
data$ = this.service.data$.pipe(takeUntilDestroyed());
```

**Classic:** `takeUntil` cu `Subject`:

```typescript
private destroy$ = new Subject<void>();
ngOnDestroy() { this.destroy$.next(); this.destroy$.complete(); }
// .pipe(takeUntil(this.destroy$))
```

**Cel mai bine:** Foloseste Signals si `resource()` - nu mai ai subscriptions de gestionat. Sau `async` pipe in template - se dezaboneaza automat.

---

### Q8: Cum ai migra o aplicatie Angular mare de la RxJS la Signals?

**Gradual, nu big-bang.** Abordare in 4 faze:

1. **Coexistenta** - Adauga `toSignal()` wrappers in services existente. Zero breaking changes.
2. **Noi features** - Scrie componentele noi cu Signals. Foloseste `resource()` pentru data fetching.
3. **Migrare componente** - Inlocuieste `async` pipe cu signal reads. Converteste `@Input` la `input()`.
4. **Migrare services** - Inlocuieste `BehaviorSubject` cu `signal()`. Pastreaza RxJS pentru fluxuri complexe.

**NU migra:** streams cu operatori de timp, WebSocket handling, complex retry logic. Foloseste bridge-ul `toSignal()` / `toObservable()` la granita.

---

### Q9: Ce inseamna "glitch-free" in contextul Signals?

"Glitch-free" inseamna ca **nu vei vedea niciodata stari intermediare inconsistente**. Daca ai `firstName`, `lastName` si `fullName = computed(() => firstName() + ' ' + lastName())`, si schimbi ambele in acelasi tick sincron, `fullName` va reflecta ambele schimbari simultan. Nu va exista un moment in care `fullName` are noul `firstName` cu vechiul `lastName`.

Angular realizeaza asta prin **lazy evaluation** - computed signals nu se recalculeaza imediat la schimbare, ci doar cand sunt citite. Cand Angular face change detection, citeste toate signals-urile, si la acel moment toate dependentele sunt deja actualizate.

---

### Q10: Cum ai implementa un global state management cu Signals?

```typescript
// SignalStore pattern (inspirat din NgRx SignalStore)
@Injectable({ providedIn: 'root' })
export class AppStore {
  // Private writable state
  private _user = signal<User | null>(null);
  private _cart = signal<CartItem[]>([]);
  private _theme = signal<'light' | 'dark'>('light');

  // Public readonly signals
  readonly user = this._user.asReadonly();
  readonly cart = this._cart.asReadonly();
  readonly theme = this._theme.asReadonly();

  // Computed derivations
  readonly isLoggedIn = computed(() => this._user() !== null);
  readonly cartTotal = computed(() =>
    this._cart().reduce((sum, i) => sum + i.price * i.qty, 0)
  );
  readonly cartCount = computed(() => this._cart().length);

  // Actions (metode publice)
  login(user: User) { this._user.set(user); }
  logout() { this._user.set(null); this._cart.set([]); }
  addToCart(item: CartItem) {
    this._cart.update(cart => [...cart, item]);
  }
}
```

Acest pattern ofera **encapsulare** (doar store-ul modifica state), **derivari reactive** (computed), si **simplitate** fata de NgRx clasic cu actions/reducers/effects.

---

*Ultimul update: Februarie 2026*
