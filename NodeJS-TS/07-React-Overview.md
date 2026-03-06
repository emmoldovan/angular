# 07 - React Overview (Bonus)

> Nu e focusul principal al interviului, dar cunoașterea bazelor React este un avantaj.
> **Tu ai background Angular — aceasta e CEA MAI MARE O AVANTAJ.**
> Conceptele sunt identice, sintaxa diferă. Fiecare concept React are un echivalent exact în Angular.

---

## Angular → React: mapping complet

| Angular | React | Notă |
|---------|-------|------|
| Component class / standalone | Function component | React NU are clase (din 2020+) |
| `@Input()` / `input()` signal | Props | React: props se pasează ca atribute JSX |
| `@Output()` / EventEmitter | Callback prop (`onClick`, `onChange`) | React: funcții pasate ca props |
| `ngOnInit()` | `useEffect(() => {}, [])` | Array gol = rulează o singură dată |
| `ngOnDestroy()` | `return () => {}` din `useEffect` | Cleanup function = ngOnDestroy |
| `signal()` | `useState()` | Ambele: state reactiv local |
| `computed()` | `useMemo()` | Valoare derivată, se recalculează |
| `effect()` | `useEffect()` | Side effect la schimbare de state |
| `BehaviorSubject` | `useReducer` | State complex cu acțiuni |
| Service singleton (`providedIn: 'root'`) | Context API + custom hook | React nu are DI, simulează cu Context |
| NgRx SignalStore / BehaviorSubject service | Zustand store | State global partajat |
| `HttpClient` + RxJS | React Query (TanStack Query) | Server state: loading, error, cache |
| `ng-content` | `children` prop | Content projection |
| `*ngFor` + `trackBy` | `.map()` + `key` prop | Liste reactiove cu identity tracking |
| `*ngIf` | `&&` sau ternary `? :` în JSX | Rendering condiționat |
| `OnPush` change detection | `React.memo` | Skip re-render dacă inputs n-au schimbat |
| `@ViewChild` | `useRef` | Referință stabilă la DOM/component |
| `Pipe` (async, date, currency) | Hook custom sau funcție utilitară | React NU are pipe system |
| `RouterModule` / `<router-outlet>` | `react-router-dom` / `<Routes>` | Routing declarativ |
| `ActivatedRoute` / `paramMap` | `useParams()`, `useSearchParams()` | Parametri de rută |
| `Router.navigate()` | `useNavigate()` | Navigare programatică |
| Lazy loading module (`loadChildren`) | `React.lazy()` + `Suspense` | Code splitting |
| `ErrorHandler` service global | Error Boundary component | Prinde erori în arbore |
| `ngZone.runOutsideAngular()` | *(no direct equivalent)* | React nu are Zone.js |
| Angular Universal SSR | Next.js SSR / React Server Components | SSR în React se face via framework |
| `APP_INITIALIZER` | Suspense cu async server component | Data loading înainte de render |

---

## useState — echivalentul signal()

```tsx
// REACT — useState
import { useState } from 'react';

function Counter() {
  const [count, setCount] = useState(0);

  // ⚠️ React: ÎNTOTDEAUNA folosești setter, nu mutezi direct
  // count++  ← GREȘIT, nu declanșează re-render
  const increment = () => setCount(prev => prev + 1); // ← funcție, nu valoare (async-safe)

  return <button onClick={increment}>Count: {count}</button>;
}
```

```typescript
// ANGULAR — signal() (Angular 17+)
import { Component, signal } from '@angular/core';

@Component({
  selector: 'app-counter',
  template: `<button (click)="increment()">Count: {{ count() }}</button>`,
})
export class CounterComponent {
  count = signal(0); // ← signal, nu BehaviorSubject

  increment() {
    this.count.update(prev => prev + 1); // ← update cu funcție — identic cu setState(prev => prev + 1)
    // SAU: this.count.set(this.count() + 1);  ← set cu valoare nouă
  }
}
```

> **Diferența cheie:** React `useState` returnează `[value, setter]`. Angular `signal()` returnează un obiect cu `.set()` și `.update()`. Ambele sunt reactive — schimbarea valorii declanșează re-render/change detection.

---

## useEffect — echivalentul ngOnInit + ngOnDestroy

```tsx
// REACT — useEffect
import { useState, useEffect } from 'react';

function UserProfile({ userId }: { userId: string }) {
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    let cancelled = false; // previne state update pe component demontat

    async function fetchUser() {
      try {
        setLoading(true);
        const data = await api.getUser(userId);
        if (!cancelled) setUser(data); // verifici dacă componenta mai e montată
      } finally {
        if (!cancelled) setLoading(false);
      }
    }

    fetchUser();

    // ⬇️ Return function = cleanup = ngOnDestroy
    return () => { cancelled = true; };
  }, [userId]); // ← dependency array: re-rulează când userId se schimbă (ca ngOnChanges)
  //             [userId] = OnChanges pentru userId
  //             []       = ngOnInit (o singură dată)
  //             (absent) = rulează după FIECARE render

  if (loading) return <div>Loading...</div>;
  if (!user) return <div>User not found</div>;
  return <div>{user.name}</div>;
}
```

```typescript
// ANGULAR — echivalent complet
import { Component, OnInit, OnDestroy, Input, inject } from '@angular/core';
import { Subject } from 'rxjs';
import { takeUntil, switchMap } from 'rxjs/operators';
import { UserService } from './user.service';

@Component({
  selector: 'app-user-profile',
  template: `
    @if (loading()) { <div>Loading...</div> }
    @else if (user()) { <div>{{ user()!.name }}</div> }
    @else { <div>User not found</div> }
  `,
})
export class UserProfileComponent implements OnInit, OnDestroy {
  @Input() userId!: string;

  private userService = inject(UserService);
  private destroy$ = new Subject<void>(); // echivalentul lui "cancelled"

  user = signal<User | null>(null);
  loading = signal(true);

  ngOnInit() { // ← useEffect cu []
    this.userService.getUser(this.userId)
      .pipe(takeUntil(this.destroy$)) // ← auto-cleanup la destroy
      .subscribe({
        next: (u) => { this.user.set(u); this.loading.set(false); },
        error: () => this.loading.set(false),
      });
  }

  ngOnDestroy() { // ← return function din useEffect
    this.destroy$.next();
    this.destroy$.complete();
  }
}

// ✅ Modern Angular 16+ — DestroyRef (mai elegant)
import { DestroyRef, inject } from '@angular/core';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';

export class UserProfileComponent implements OnInit {
  private destroyRef = inject(DestroyRef);

  ngOnInit() {
    this.userService.getUser(this.userId)
      .pipe(takeUntilDestroyed(this.destroyRef)) // ← auto-cleanup, fără Subject manual
      .subscribe(u => this.user.set(u));
  }
}
```

> **Dependency array în React = ngOnChanges în Angular.** Dacă adaugi `userId` în dependency array, `useEffect` re-rulează ori de câte ori `userId` se schimbă — identic cu `ngOnChanges` pe `@Input() userId`.

---

## useMemo / useCallback — echivalentul computed()

```tsx
// REACT
import { useMemo, useCallback } from 'react';

function ProductList({ products, searchTerm, onSelect }: Props) {
  // useMemo = computed() — valoare derivată, se recalculează doar când deps se schimbă
  const filteredProducts = useMemo(() => {
    console.log('Recalculare filtru'); // vezi când rulează
    return products.filter(p =>
      p.name.toLowerCase().includes(searchTerm.toLowerCase())
    );
  }, [products, searchTerm]); // recalculează DOAR când products sau searchTerm se schimbă

  // useCallback = funcție stabilă — nu se recreează la fiecare render
  // Fără useCallback: handleSelect e o NOUĂ funcție la fiecare render
  // → Child se re-renderează inutil dacă e React.memo-izat
  const handleSelect = useCallback((product: Product) => {
    onSelect(product.id);
  }, [onSelect]); // recreează dacă onSelect se schimbă

  return (
    <ul>
      {filteredProducts.map(p => (
        <ListItem key={p.id} item={p} onSelect={handleSelect} />
      ))}
    </ul>
  );
}
```

```typescript
// ANGULAR — computed() și metode stabile
import { Component, Input, computed, signal } from '@angular/core';

@Component({
  selector: 'app-product-list',
  template: `
    <ul>
      @for (product of filteredProducts(); track product.id) {
        <app-list-item [item]="product" (selected)="handleSelect($event)" />
      }
    </ul>
  `,
})
export class ProductListComponent {
  products = input<Product[]>([]);     // @Input() ca signal
  searchTerm = input<string>('');

  // computed() = useMemo — se recalculează automat când inputs se schimbă
  filteredProducts = computed(() => {
    const term = this.searchTerm().toLowerCase();
    return this.products().filter(p => p.name.toLowerCase().includes(term));
  });

  // ⚠️ Metode Angular sunt STABILE prin definiție — nu ai nevoie de useCallback
  // handleSelect NU se recreează la fiecare change detection run
  handleSelect(productId: string) {
    this.onSelect.emit(productId); // sau inject un service
  }
}
```

> **useCallback în React nu are echivalent în Angular** pentru că metodele de clasă sunt stabile prin natură. Problema din React (funcție nouă la fiecare render → child se re-renderează) nu există în Angular.

---

## useRef — echivalentul @ViewChild

```tsx
// REACT — useRef pentru referință DOM și valori stabile
import { useRef, useState } from 'react';

function StopWatch() {
  const [time, setTime] = useState(0);

  // useRef pentru timer ID — NU cauzează re-render la schimbare
  // Dacă ai pune intervalId în useState, fiecare set → re-render inutil
  const intervalRef = useRef<ReturnType<typeof setInterval>>();

  // useRef pentru referință la DOM element
  const inputRef = useRef<HTMLInputElement>(null);

  const start = () => {
    intervalRef.current = setInterval(() => setTime(prev => prev + 1), 1000);
    inputRef.current?.focus(); // acces direct la DOM
  };

  const stop = () => clearInterval(intervalRef.current);

  return (
    <div>
      <p>{time}s</p>
      <input ref={inputRef} /> {/* atașezi ref la DOM element */}
      <button onClick={start}>Start</button>
      <button onClick={stop}>Stop</button>
    </div>
  );
}
```

```typescript
// ANGULAR — @ViewChild și variabile private
import { Component, ViewChild, ElementRef, OnDestroy } from '@angular/core';

@Component({
  selector: 'app-stopwatch',
  template: `
    <p>{{ time }}s</p>
    <input #inputEl />  <!-- template reference variable -->
    <button (click)="start()">Start</button>
    <button (click)="stop()">Stop</button>
  `,
})
export class StopWatchComponent implements OnDestroy {
  @ViewChild('inputEl') inputRef!: ElementRef<HTMLInputElement>; // referință DOM

  time = 0;
  private intervalId?: ReturnType<typeof setInterval>; // variabilă privată, nu state

  start() {
    this.intervalId = setInterval(() => this.time++, 1000);
    this.inputRef.nativeElement.focus();
  }

  stop() { clearInterval(this.intervalId); }

  ngOnDestroy() { clearInterval(this.intervalId); } // cleanup!
}
```

---

## Custom Hooks — echivalentul serviciilor Angular

```tsx
// REACT — custom hook = logică reutilizabilă
// Convenție: începe cu "use"

function useLocalStorage<T>(key: string, initialValue: T) {
  const [value, setValue] = useState<T>(() => {
    try {
      const item = localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch {
      return initialValue;
    }
  });

  const setStoredValue = useCallback((newValue: T | ((prev: T) => T)) => {
    setValue(prev => {
      const resolved = typeof newValue === 'function'
        ? (newValue as (prev: T) => T)(prev)
        : newValue;
      localStorage.setItem(key, JSON.stringify(resolved));
      return resolved;
    });
  }, [key]);

  return [value, setStoredValue] as const;
}

// useApi — fetch cu loading/error state
function useApi<T>(url: string) {
  const [state, setState] = useState<{
    data: T | null; loading: boolean; error: string | null;
  }>({ data: null, loading: true, error: null });

  useEffect(() => {
    const controller = new AbortController();
    fetch(url, { signal: controller.signal })
      .then(res => { if (!res.ok) throw new Error(`HTTP ${res.status}`); return res.json(); })
      .then(data => setState({ data, loading: false, error: null }))
      .catch(err => {
        if (err.name !== 'AbortError')
          setState({ data: null, loading: false, error: err.message });
      });
    return () => controller.abort();
  }, [url]);

  return state;
}
```

```typescript
// ANGULAR — Service (echivalent exact al custom hook)
// Serviciile Angular sunt SUPERIOARE custom hooks: pot fi singleton, injectate oriunde,
// nu sunt legate de lifecycle-ul unui component specific

@Injectable({ providedIn: 'root' }) // singleton — ca un hook partajat global
export class LocalStorageService {
  get<T>(key: string, defaultValue: T): T {
    try {
      const item = localStorage.getItem(key);
      return item ? JSON.parse(item) : defaultValue;
    } catch { return defaultValue; }
  }

  set<T>(key: string, value: T): void {
    localStorage.setItem(key, JSON.stringify(value));
  }
}

// Pentru state reactiv cu localStorage:
@Injectable({ providedIn: 'root' })
export class ThemeService {
  private _theme = signal<'light' | 'dark'>(
    (localStorage.getItem('theme') as 'light' | 'dark') ?? 'light'
  );

  readonly theme = this._theme.asReadonly(); // expui read-only

  setTheme(theme: 'light' | 'dark') {
    this._theme.set(theme);
    localStorage.setItem('theme', theme);
  }
}

// Echivalent pentru useApi — HttpClient service
@Injectable({ providedIn: 'root' })
export class ApiService {
  private http = inject(HttpClient);

  getUser(id: string) {
    return this.http.get<User>(`/api/users/${id}`).pipe(
      catchError(err => throwError(() => new Error(`HTTP ${err.status}`)))
    );
  }
}

// În component — inject service, nu apelezi hook
export class UserPageComponent {
  private api = inject(ApiService);
  private destroyRef = inject(DestroyRef);

  user = signal<User | null>(null);
  loading = signal(true);
  error = signal<string | null>(null);

  ngOnInit() {
    this.api.getUser(this.id)
      .pipe(takeUntilDestroyed(this.destroyRef))
      .subscribe({
        next: u => { this.user.set(u); this.loading.set(false); },
        error: e => { this.error.set(e.message); this.loading.set(false); },
      });
  }
}
```

> **Cheia diferenței:** React custom hooks sunt "funcții speciale" — pot folosi alte hooks, dar sunt legate de componenta în care sunt apelate. Angular services sunt clase singleton cu DI — complet independente de componente, pot fi injectate oriunde, pot fi testate izolat.

---

## State Management: useReducer / Context = BehaviorSubject Service / NgRx

```tsx
// REACT — useReducer + Context pentru global state fără library
type Action =
  | { type: 'ADD_ITEM'; payload: Product }
  | { type: 'REMOVE_ITEM'; payload: string }
  | { type: 'CLEAR' };

function cartReducer(state: CartState, action: Action): CartState {
  switch (action.type) {
    case 'ADD_ITEM': return { ...state, items: [...state.items, action.payload] };
    case 'REMOVE_ITEM': return { ...state, items: state.items.filter(i => i.id !== action.payload) };
    case 'CLEAR': return { items: [], total: 0 };
    default: return state;
  }
}

const CartContext = React.createContext<{ state: CartState; dispatch: Dispatch<Action> } | null>(null);

function CartProvider({ children }: { children: ReactNode }) {
  const [state, dispatch] = useReducer(cartReducer, { items: [], total: 0 });
  return <CartContext.Provider value={{ state, dispatch }}>{children}</CartContext.Provider>;
}

function useCart() {
  const ctx = useContext(CartContext);
  if (!ctx) throw new Error('useCart must be used within CartProvider');
  return ctx;
}

// Folosire:
function CartButton() {
  const { state, dispatch } = useCart();
  return (
    <button onClick={() => dispatch({ type: 'ADD_ITEM', payload: someProduct })}>
      Cart ({state.items.length})
    </button>
  );
}
```

```typescript
// ANGULAR — Signal-based service (echivalent modern, Angular 17+)
// Mult mai simplu decât Context + useReducer — fără Provider, fără boilerplate

@Injectable({ providedIn: 'root' })
export class CartService {
  // State intern ca signals
  private _items = signal<CartItem[]>([]);

  // Expui computed values — echivalentul selectori Zustand
  readonly items = this._items.asReadonly();
  readonly total = computed(() =>
    this._items().reduce((sum, i) => sum + i.price * i.quantity, 0)
  );
  readonly count = computed(() => this._items().length);

  // Actions (echivalentul dispatch)
  addItem(product: Product) {
    this._items.update(items => {
      const existing = items.find(i => i.id === product.id);
      if (existing) return items.map(i => i.id === product.id ? { ...i, quantity: i.quantity + 1 } : i);
      return [...items, { ...product, quantity: 1 }];
    });
  }

  removeItem(id: string) {
    this._items.update(items => items.filter(i => i.id !== id));
  }

  clear() { this._items.set([]); }
}

// Folosire — fără Provider, inject direct:
@Component({
  template: `<button (click)="addToCart()">Cart ({{ cart.count() }})</button>`,
})
export class CartButtonComponent {
  cart = inject(CartService); // ← direct inject, fără context wrapper

  addToCart() { this.cart.addItem(someProduct); }
}
```

> **Angular e mai simplu pentru global state.** Nu ai nevoie de Provider wrapper sau Context.Provider. Serviciul Angular e singleton automat — injectezi și e gata.

---

## Performance: React.memo = OnPush + ChangeDetectorRef

```tsx
// REACT — React.memo previne re-render dacă props nu s-au schimbat (shallow compare)
const ExpensiveList = React.memo(function ExpensiveList({ items }: { items: Item[] }) {
  return <ul>{items.map(item => <li key={item.id}>{item.name}</li>)}</ul>;
});

// React.lazy + Suspense — code splitting (lazy loading)
const Dashboard = React.lazy(() => import('./Dashboard'));
const Analytics = React.lazy(() => import('./Analytics'));

function App() {
  return (
    <Suspense fallback={<LoadingSpinner />}>
      <Routes>
        <Route path="/dashboard" element={<Dashboard />} />
        <Route path="/analytics" element={<Analytics />} />
      </Routes>
    </Suspense>
  );
}

// Key-uri în liste — ajută React să identifice ce s-a schimbat
// ❌ index ca key — dacă lista e reordonată/filtrată, React refolosește DOM greșit
{items.map((item, index) => <Item key={index} item={item} />)}

// ✅ ID stabil
{items.map(item => <Item key={item.id} item={item} />)}
```

```typescript
// ANGULAR — OnPush + trackBy (echivalente directe)

// OnPush = React.memo — Angular nu re-renderizează componenta dacă inputs (ca referință) nu s-au schimbat
@Component({
  selector: 'app-expensive-list',
  template: `
    <ul>
      @for (item of items; track item.id) {  <!-- track = key în React -->
        <li>{{ item.name }}</li>
      }
    </ul>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush, // ← React.memo echivalent
})
export class ExpensiveListComponent {
  @Input() items: Item[] = [];
}

// Lazy loading Angular — în routing (echivalent React.lazy + Suspense)
const routes: Routes = [
  {
    path: 'dashboard',
    loadComponent: () => import('./dashboard/dashboard.component')
      .then(m => m.DashboardComponent),
    // SAU pentru module-based:
    // loadChildren: () => import('./dashboard/dashboard.module').then(m => m.DashboardModule),
  },
  {
    path: 'analytics',
    loadComponent: () => import('./analytics/analytics.component')
      .then(m => m.AnalyticsComponent),
  },
];
// Angular afișează automat loading state în router-outlet în timp ce se încarcă
```

> **`trackBy` în Angular = `key` în React.** Ambele îi spun framework-ului care item din listă e care, astfel încât la re-render/change detection să refolosească DOM-ul existent în loc să-l distrugă și recreeze.
>
> **`@for (item of items; track item.id)` este sintaxa Angular 17+.** Echivalentul exact al `{items.map(item => <Item key={item.id} />)}`.

---

## TypeScript în React vs Angular

```tsx
// REACT — Props cu TypeScript
interface ButtonProps {
  label: string;
  onClick: () => void;
  variant?: 'primary' | 'secondary' | 'danger';
  disabled?: boolean;
  children?: React.ReactNode; // echivalentul ng-content
}

const Button: React.FC<ButtonProps> = ({ label, onClick, variant = 'primary', children }) => (
  <button className={`btn btn-${variant}`} onClick={onClick}>
    {children ?? label}  {/* children = ng-content */}
  </button>
);

// Generic component
function Select<T>({ options, value, onChange, getLabel, getValue }: SelectProps<T>) {
  return (
    <select value={getValue(value)} onChange={e => {
      const selected = options.find(o => getValue(o) === e.target.value);
      if (selected) onChange(selected);
    }}>
      {options.map(option => (
        <option key={getValue(option)} value={getValue(option)}>{getLabel(option)}</option>
      ))}
    </select>
  );
}

// Event types în React
const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => setValue(e.target.value);
const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => { e.preventDefault(); };
```

```typescript
// ANGULAR — Input/Output cu TypeScript
@Component({
  selector: 'app-button',
  template: `
    <button [class]="'btn btn-' + variant()" (click)="onClick.emit()">
      <ng-content>{{ label() }}</ng-content>  <!-- ng-content = children -->
    </button>
  `,
})
export class ButtonComponent {
  // Input signals (Angular 17+ — preferabil față de @Input() clasic)
  label = input.required<string>();
  variant = input<'primary' | 'secondary' | 'danger'>('primary');
  disabled = input<boolean>(false);

  // Output (callback prop în React)
  onClick = output<void>();
}

// Event types în Angular — template-driven
// Angular folosește $event direct, NU event objects cu tipuri speciale
// (click)="handleClick()" — void
// (input)="handleInput($event)" — $event e InputEvent (inferit din context)
// (ngSubmit)="handleSubmit()" — FormGroup submit
```

---

## Întrebări de interviu — cu context Angular

**Q: Explică diferența dintre `useState` și `useRef`.**

> `useState` declanșează re-render la fiecare schimbare. `useRef` returnează o referință stabilă care NU cauzează re-render.
>
> **Angular parallel:** `signal()` = useState (declanșează change detection). O variabilă privată normală (`private intervalId`) = useRef (nu declanșează nimic). `@ViewChild` = useRef pentru referință DOM.

**Q: Când folosești `useCallback` și `useMemo`?**

> Ambele pentru optimizare. `useCallback` când pasezi o funcție ca prop la un component memo-izat. `useMemo` pentru computații costisitoare. Regula: măsoară mai întâi cu Profiler.
>
> **Angular parallel:** `useMemo` = `computed()`. `useCallback` nu are echivalent în Angular — metodele de clasă sunt stabile prin natură. `React.memo` = `ChangeDetectionStrategy.OnPush`.

**Q: Ce e lifting state up?**

> Când două componente sibling trebuie să partajeze state, muți state-ul în cel mai apropiat ancestor comun. Alternativa la scară: Context API sau Zustand.
>
> **Angular parallel:** Exact aceeași problemă, soluție diferită. Muți state-ul într-un service partajat cu `inject(MyService)` în ambele componente — nu ai nevoie de "lifting". Serviciile Angular sunt mult mai elegante pentru state partajat.

**Q: `useEffect` cleanup — când e important?**

> Când efectul creează resurse: subscripții (WebSocket, EventEmitter), timers, fetch requests (AbortController). Fără cleanup: memory leaks.
>
> **Angular parallel:** `ngOnDestroy()` sau `takeUntilDestroyed(this.destroyRef)` pentru RxJS. Exact aceeași problemă — dacă nu curăți subscripțiile, ai memory leak identic.

---

*[← 06 - Gândire](./06-Gandire-si-Problem-Solving.md) | [08 - Q&A →](./08-Intrebari-si-Raspunsuri.md)*
