# 09 - Next.js & Frontend Avansat

> Next.js SSR/SSG/CSR, state management (Zustand, React Query), debugging React,
> testare componente async. Acoperă toate întrebările frontend din lista interviului.
>
> **Dacă știi Angular, știi deja 80% din concepte — doar sintaxa diferă.**

---

## 1. Next.js — Structura pentru MVP rapid și mentenabil

### App Router (Next.js 13+) — structura recomandată

```
src/
├── app/                        # App Router — routing bazat pe folder structure
│   ├── layout.tsx              # Root layout (html, body, providers) = AppComponent
│   ├── page.tsx                # Home page (/)
│   ├── loading.tsx             # Loading UI (Suspense automat) = skeleton guard
│   ├── error.tsx               # Error boundary = ErrorHandler
│   ├── not-found.tsx           # 404
│   │
│   ├── (marketing)/            # Route group — nu afectează URL-ul
│   │   └── about/page.tsx      # /about
│   │
│   ├── (app)/                  # Route group — layout separat pentru app
│   │   ├── layout.tsx          # Layout cu sidebar, nav = RouterModule cu named outlets
│   │   ├── dashboard/page.tsx  # /dashboard
│   │   └── settings/page.tsx   # /settings
│   │
│   └── api/                    # API Routes (Route Handlers)
│       └── users/route.ts      # GET /api/users, POST /api/users
│
├── components/
│   ├── ui/                     # Componente primitive (Button, Input, Modal)
│   └── features/               # Componente specifice featurelui
│       └── dashboard/
│
├── lib/                        # Utilities, helpers, clients (db, api) = shared/
│   ├── db.ts                   # Prisma client singleton
│   └── auth.ts                 # Auth helpers
│
├── hooks/                      # Custom hooks = services/ în Angular
├── stores/                     # Zustand stores = NgRx / signal services
└── types/                      # TypeScript types = models/
```

> **Angular parallel pentru structură:** folder `features/` în Next.js ≈ lazy-loaded modules în Angular. `app/layout.tsx` ≈ `AppComponent`. `api/users/route.ts` ≈ un BFF simplu (nu ai nevoie de backend separat pentru MVP).

### Principii pentru MVP rapid și mentenabil

```
RAPID (speed):                  MENTENABIL (maintainability):
- shadcn/ui pentru UI ready-made   - Feature-based folder structure
- Zustand (simplu, fără boilerplate)  - Single responsibility pe componente
- Route Handlers ca BFF simplu     - Types în fișiere separate
- Server Components by default     - Env variables validate (zod)
- Vercel pentru deployment 1-click - Error boundaries la fiecare feature
```

**Regula pentru MVP:** Nu construi infrastructure pe care nu o ai nevoie acum. Un Zustand store simplu > Redux + saga. Un Route Handler simplu > microserviciu separat.

---

## 2. SSR vs SSG vs ISR vs CSR — când alegi ce

### Comparație rapidă

| Rendering | Când se generează HTML | Bun pentru | Next.js syntax |
|-----------|----------------------|------------|----------------|
| **SSR** | La fiecare request, pe server | Date personalizate, real-time, auth | `async` Server Component |
| **SSG** | La build time | Date statice, blog, marketing | `generateStaticParams` |
| **ISR** | La build + revalidare periodic | Date care se schimbă rar | `revalidate: 3600` |
| **CSR** | În browser, după hydration | Date interactive, user-specific after load | `'use client'` + useEffect |

### Echivalente Angular

| Next.js | Angular | Notă |
|---------|---------|------|
| Server Component (SSR) | Angular Universal / Angular 17+ SSR | Date pe server înainte de HTML |
| SSG cu generateStaticParams | Angular prerendering (`outputMode: 'static'`) | Build-time HTML generation |
| ISR (revalidate: 3600) | Nu are direct echivalent — CDN + cache headers | ISR e specific Next.js |
| CSR (`'use client'`) | Default Angular SPA | Toate componentele sunt "client" în Angular classic |
| React Server Components | Angular Server Components (experimental în 19+) | Cod care rulează DOAR pe server |

---

### Next.js App Router — Server Components by default

```tsx
// SERVER COMPONENT (default — NU scrii 'use client')
// Rulează pe server, are acces la DB, env vars, fișiere
// NU poate folosi: useState, useEffect, onClick, browser APIs

async function ProductPage({ params }: { params: { id: string } }) {
  // Fetch direct pe server — fără loading state, fără useEffect
  const product = await db.product.findUnique({ where: { id: params.id } });

  if (!product) notFound(); // din next/navigation

  return <ProductDetails product={product} />;
}

// CLIENT COMPONENT — necesit interactivitate
'use client';

function AddToCartButton({ productId }: { productId: string }) {
  const [loading, setLoading] = useState(false);

  async function handleClick() {
    setLoading(true);
    await addToCart(productId);
    setLoading(false);
  }

  return <button onClick={handleClick}>{loading ? 'Adding...' : 'Add to Cart'}</button>;
}

// Pattern corect: Server Component wrappează Client Component
// Server Components pot importa Client Components
// Client Components NU pot importa Server Components directe
async function ProductPage({ params }) {
  const product = await db.product.findUnique(/* ... */);

  return (
    <div>
      <h1>{product.name}</h1>         {/* Server — static */}
      <p>{product.description}</p>    {/* Server — static */}
      <AddToCartButton productId={product.id} />  {/* Client — interactiv */}
    </div>
  );
}
```

**Angular parallel — Angular 17+ SSR:**
```typescript
// Angular Universal / SSR — componentele Angular rulează pe server
// Nu există distincție Server vs Client components în Angular clasic
// Dar poți verifica dacă ești pe server:

import { isPlatformBrowser, isPlatformServer } from '@angular/common';
import { PLATFORM_ID, inject } from '@angular/core';

@Component({ ... })
export class ProductPageComponent {
  private platformId = inject(PLATFORM_ID);

  ngOnInit() {
    if (isPlatformServer(this.platformId)) {
      // Cod care rulează DOAR pe server (SSR)
      // Acces la TransferState pentru a pasa date de pe server la browser
    }
    if (isPlatformBrowser(this.platformId)) {
      // Cod care rulează DOAR în browser (window, localStorage)
    }
  }
}
```

> **Marea diferență:** Next.js face distincția Server/Client explicit cu `'use client'`. Angular Universal rulează tot componenta pe server, apoi rehydratează pe client — mai puțin granular, dar mai simplu de raționat.

---

### Când alegi SSR (Server Components async)

```tsx
// Date personalizate per user (dashboard, profil)
// Date care trebuie să fie fresh la fiecare request
// SEO important + date dinamice

async function Dashboard() {
  const user = await getCurrentUser(); // din cookies/session
  const stats = await getUserStats(user.id); // date real-time
  return <DashboardView stats={stats} user={user} />;
}
```

**Angular equivalent:**
```typescript
// Angular SSR — datele se pot pre-fetch pe server cu TransferState
// Ca să nu se facă același HTTP call de două ori (server + client)
import { TransferState, makeStateKey } from '@angular/core';

const USER_KEY = makeStateKey<User>('currentUser');

@Component({ ... })
export class DashboardComponent implements OnInit {
  private transferState = inject(TransferState);
  private http = inject(HttpClient);

  ngOnInit() {
    if (this.transferState.hasKey(USER_KEY)) {
      // Browser: ia din state transferat de server (fără HTTP call)
      this.user = this.transferState.get(USER_KEY, null);
      this.transferState.remove(USER_KEY);
    } else {
      // Server: fetch real + stochează în TransferState
      this.http.get<User>('/api/user').subscribe(user => {
        this.user = user;
        this.transferState.set(USER_KEY, user); // trimis în HTML
      });
    }
  }
}
```

---

### Când alegi SSG

```tsx
// Blog posts, marketing pages, documentație
// Date care nu se schimbă sau se schimbă rar

export async function generateStaticParams() {
  const posts = await db.post.findMany({ select: { slug: true } });
  return posts.map(post => ({ slug: post.slug }));
}

async function BlogPost({ params }: { params: { slug: string } }) {
  const post = await db.post.findUnique({ where: { slug: params.slug } });
  return <PostContent post={post} />;
}
```

**Angular equivalent:**
```typescript
// angular.json / app.config.ts — prerendering
// Angular 17+ suportă static site generation (SSG)
// Definești rutele care trebuie pre-rendered la build time
export default {
  prerender: {
    routesFile: 'routes.txt', // lista de rute de pre-rendered
    // SAU discoverRoutes: true (Angular încearcă să descopere automat)
  }
};
// routes.txt:
// /blog/post-1
// /blog/post-2
// /about
```

---

### Când alegi ISR (Incremental Static Regeneration)

```tsx
// E-commerce: prețuri, stocuri — se schimbă dar nu în timp real
async function ProductCatalog() {
  const products = await fetchProducts();
  return <ProductGrid products={products} />;
}

export const revalidate = 3600; // revalidează la fiecare oră
```

> ISR nu are echivalent direct în Angular. Cel mai apropiat: **CDN caching cu cache-control headers** pe endpoint-ul Angular (ex: `Cache-Control: max-age=3600, stale-while-revalidate=86400`). Nginx sau Cloudflare servesc HTML/JSON cached, Angular Universal regenerează la expirare.

---

### Când alegi CSR

```tsx
// Date care depind de interacțiunea utilizatorului DUPĂ load
'use client';

function SearchResults() {
  const [query, setQuery] = useState('');
  const { data } = useQuery({
    queryKey: ['search', query],
    queryFn: () => searchProducts(query),
    enabled: query.length > 2,
  });

  return (/* ... */);
}
```

**Angular equivalent — default Angular SPA:**
```typescript
// Angular e CSR by default — tot codul rulează în browser
// Echivalentul useQuery în Angular:

@Component({
  template: `
    <input [(ngModel)]="query" (input)="onSearch()" />
    @if (loading()) { <app-skeleton /> }
    @for (product of results(); track product.id) {
      <app-product-card [product]="product" />
    }
  `,
})
export class SearchComponent {
  private productService = inject(ProductService);
  private destroyRef = inject(DestroyRef);

  query = signal('');
  results = signal<Product[]>([]);
  loading = signal(false);

  onSearch() {
    const q = this.query();
    if (q.length < 3) return;

    this.loading.set(true);
    this.productService.search(q)
      .pipe(
        debounceTime(300),      // nu trimiți la fiecare tastă
        distinctUntilChanged(), // nu trimiți dacă query e același
        takeUntilDestroyed(this.destroyRef),
      )
      .subscribe(products => {
        this.results.set(products);
        this.loading.set(false);
      });
  }
}
```

---

## 3. State Management — Context vs Zustand vs Redux vs React Query

### Când folosești ce (React)

```
UI State (local):           useState / useReducer în component
Shared UI State (simplu):   Context API (theme, locale, modal open/close)
Shared App State (complex): Zustand
Server/Async State:         React Query (TanStack Query) sau SWR
Global Complex + DevTools:  Redux Toolkit (rar necesar în 2026)
```

### Angular echivalent

```
UI State (local):           signal() în component
Shared UI State (simplu):   Service cu signal / BehaviorSubject
Shared App State (complex): NgRx Signals Store sau service cu signals
Server/Async State:         Angular HttpClient + RxJS / TanStack Query Angular
Global Complex + DevTools:  NgRx Store (Redux pattern — ai deja experiență!)
```

---

### Zustand — simplu, fără boilerplate

```typescript
// stores/cart.store.ts
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

interface CartStore {
  items: CartItem[];
  total: number;
  addItem: (item: Omit<CartItem, 'quantity'>) => void;
  removeItem: (id: string) => void;
  clearCart: () => void;
}

export const useCartStore = create<CartStore>()(
  persist(  // persists în localStorage automat — ca LocalStorageService în Angular
    (set, get) => ({
      items: [],
      total: 0,

      addItem: (item) => set((state) => {
        const existing = state.items.find(i => i.id === item.id);
        const items = existing
          ? state.items.map(i => i.id === item.id
              ? { ...i, quantity: i.quantity + 1 }
              : i)
          : [...state.items, { ...item, quantity: 1 }];

        return {
          items,
          total: items.reduce((sum, i) => sum + i.price * i.quantity, 0),
        };
      }),

      removeItem: (id) => set((state) => {
        const items = state.items.filter(i => i.id !== id);
        return { items, total: items.reduce((sum, i) => sum + i.price * i.quantity, 0) };
      }),

      clearCart: () => set({ items: [], total: 0 }),
    }),
    { name: 'cart-storage' } // localStorage key
  )
);

// Folosire în component (fără Provider — direct!)
function CartIcon() {
  const { items, total } = useCartStore();
  return <div>Cart ({items.length}) — ${total}</div>;
}

// Selector pentru performance — re-render doar când count se schimbă
function CartBadge() {
  const count = useCartStore(state => state.items.length);
  return <span>{count}</span>;
}
```

**Angular equivalent — Signal-based Service:**
```typescript
// cart.service.ts — exact aceleași capabilități, syntax Angular
@Injectable({ providedIn: 'root' })
export class CartService {
  private _items = signal<CartItem[]>(
    JSON.parse(localStorage.getItem('cart') ?? '[]') // persist la init
  );

  // Computed values = Zustand selectori
  readonly items = this._items.asReadonly();
  readonly total = computed(() =>
    this._items().reduce((sum, i) => sum + i.price * i.quantity, 0)
  );
  readonly count = computed(() => this._items().length);

  addItem(item: Omit<CartItem, 'quantity'>) {
    this._items.update(items => {
      const existing = items.find(i => i.id === item.id);
      const updated = existing
        ? items.map(i => i.id === item.id ? { ...i, quantity: i.quantity + 1 } : i)
        : [...items, { ...item, quantity: 1 }];
      localStorage.setItem('cart', JSON.stringify(updated)); // persist
      return updated;
    });
  }

  removeItem(id: string) {
    this._items.update(items => {
      const updated = items.filter(i => i.id !== id);
      localStorage.setItem('cart', JSON.stringify(updated));
      return updated;
    });
  }

  clear() {
    this._items.set([]);
    localStorage.removeItem('cart');
  }
}

// Folosire — inject direct, fără Provider
@Component({
  template: `
    <div>Cart ({{ cart.count() }}) — ${{ cart.total() }}</div>
  `,
})
export class CartIconComponent {
  cart = inject(CartService); // ← direct, fără context wrapper
}
```

---

### React Query (TanStack Query) — server state

```typescript
// Server state = date care trăiesc pe server, tu le caching local
// React Query gestionează: loading, error, caching, refetch, optimistic updates

// 1. Setup provider în layout — NU există în Angular (serviciile sunt singleton global)
const queryClient = new QueryClient();
<QueryClientProvider client={queryClient}>{children}</QueryClientProvider>

// 2. Fetch cu useQuery
function UserProfile({ userId }: { userId: string }) {
  const { data: user, isLoading, error } = useQuery({
    queryKey: ['user', userId],      // cache key — schimbă userId → refetch automat
    queryFn: () => fetchUser(userId),
    staleTime: 5 * 60 * 1000,        // consider fresh 5 minute
    retry: 3,
  });

  if (isLoading) return <Skeleton />;
  if (error) return <ErrorMessage error={error} />;
  return <UserCard user={user} />;
}

// 3. Mutații cu useMutation + invalidare cache
const mutation = useMutation({
  mutationFn: (data: UpdateUserDto) => updateUser(userId, data),
  onSuccess: () => {
    queryClient.invalidateQueries({ queryKey: ['user', userId] }); // refetch automat
  },
});
```

**Angular equivalent — HttpClient + RxJS:**
```typescript
// Angular NU are React Query built-in, dar TanStack Query are adapter Angular
// SAU folosești pattern-ul manual cu HttpClient

// Varianta 1: Manual cache cu shareReplay (echivalent staleTime)
@Injectable({ providedIn: 'root' })
export class UserService {
  private http = inject(HttpClient);
  private cache = new Map<string, Observable<User>>();

  getUser(id: string): Observable<User> {
    if (!this.cache.has(id)) {
      // shareReplay(1) = cache ultimul rezultat, nu mai face HTTP call dacă există
      this.cache.set(id,
        this.http.get<User>(`/api/users/${id}`).pipe(
          shareReplay({ bufferSize: 1, refCount: false }),
        )
      );
    }
    return this.cache.get(id)!;
  }

  invalidateUser(id: string) {
    this.cache.delete(id); // echivalentul queryClient.invalidateQueries
  }
}

// Varianta 2: TanStack Query Angular adapter (recomandat în proiecte noi)
// npm install @tanstack/angular-query-experimental
import { injectQuery, injectMutation, QueryClient } from '@tanstack/angular-query-experimental';

@Component({ ... })
export class UserProfileComponent {
  userId = input.required<string>();

  // useQuery = injectQuery
  userQuery = injectQuery(() => ({
    queryKey: ['user', this.userId()],
    queryFn: () => fetch(`/api/users/${this.userId()}`).then(r => r.json()),
    staleTime: 5 * 60 * 1000,
  }));

  // useMutation = injectMutation
  updateMutation = injectMutation(() => ({
    mutationFn: (data: UpdateUserDto) =>
      fetch(`/api/users/${this.userId()}`, { method: 'PATCH', body: JSON.stringify(data) }),
    onSuccess: () => {
      this.queryClient.invalidateQueries({ queryKey: ['user', this.userId()] });
    },
  }));
}
```

> **Dacă intervievatorul întreabă de React Query în Angular context:** TanStack Query are acum un adapter oficial Angular (`@tanstack/angular-query-experimental`). API-ul e aproape identic, dar folosești `injectQuery()` în loc de `useQuery()`.

---

### Context API — când e suficient

```typescript
// React Context = Angular DI, dar mai limitat
// Context în React e bun pentru: theme, locale, auth user

const ThemeContext = createContext<'light' | 'dark'>('light');

function ThemeProvider({ children }: { children: ReactNode }) {
  const [theme, setTheme] = useState<'light' | 'dark'>('light');
  return (
    <ThemeContext.Provider value={theme}>
      {children}
    </ThemeContext.Provider>
  );
}

// Componentele consumatoare trebuie să fie în interiorul Provider-ului
function Header() {
  const theme = useContext(ThemeContext);
  return <header className={theme}>...</header>;
}
```

**Angular equivalent — Service cu DI (mult mai simplu):**
```typescript
// NU ai nevoie de Provider wrapper — serviciul e disponibil global automat
@Injectable({ providedIn: 'root' })
export class ThemeService {
  theme = signal<'light' | 'dark'>('light');

  toggle() {
    this.theme.update(t => t === 'light' ? 'dark' : 'light');
  }
}

// Orice component injectează direct — fără Provider wrapper:
@Component({
  template: `<header [class]="theme.theme()">...</header>`,
})
export class HeaderComponent {
  theme = inject(ThemeService);
}
```

---

## 4. Debugging și optimizarea unui React component lent

### Pas 1: Identifică problema cu React DevTools Profiler

```
React DevTools → Profiler tab → Record → interacționezi → Stop
→ Flame chart arată ce componente au rerenderat și cât timp au luat
→ Caută componente care re-renderează fără motiv (fără props schimbate)
```

**Angular equivalent — Angular DevTools:**
```
Angular DevTools (extensie Chrome) → Profiler tab → Start profiling
→ Change detection runs arată câte CD cycles rulează
→ Caută componente cu OnPush care totuși se re-renderizează des
→ Signals: schimbarea unui signal declanșează CD GRANULAR, nu pe tot arborele
```

### Pas 2: Cauze comune și fix-uri

```tsx
// REACT — cauze re-render inutil

// CAUZA 1: Funcție recreată la fiecare render
// ❌
function Parent() {
  const handleClick = () => doSomething(); // nouă funcție la fiecare render
  return <Child onClick={handleClick} />;
}
// ✅
function Parent() {
  const handleClick = useCallback(() => doSomething(), []); // stabilă
  return <Child onClick={handleClick} />;
}

// CAUZA 2: Obiect/array recreat la fiecare render
// ❌
const config = { color: 'red', size: 'large' }; // nou la fiecare render
// ✅
const CONFIG = { color: 'red', size: 'large' }; // în afara componentei — constant

// CAUZA 3: Child nu e memo-izat
const Child = React.memo(({ name }) => <div>{name}</div>);

// CAUZA 4: Context schimbat frecvent = re-render pe TOȚI consumatorii
// Fix: Zustand cu selectori granulari
const count = useStore(state => state.count); // re-render DOAR când count se schimbă
```

```typescript
// ANGULAR — cauze re-render inutil

// CAUZA 1: Change detection implicit (Default strategy) — CD rulează pe tot arborele
// ✅ Fix: OnPush — CD rulează DOAR când @Input referința se schimbă
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush, // ← React.memo equivalent
})
export class ChildComponent {
  @Input() name!: string;
}

// CAUZA 2: Metode apelate în template (se recalculează la fiecare CD run)
// ❌
// template: {{ getExpensiveValue() }}  — se apelează la fiecare CD!
// ✅ Fix: computed signal sau pipe pur
expensive = computed(() => this.doExpensiveCalculation()); // calculat o dată, cached
// template: {{ expensive() }}

// CAUZA 3: Subscripție fără async pipe (se recalculează)
// ✅ async pipe se unsubscribe automat + declanșează CD corect
// template: {{ user$ | async }}

// CAUZA 4: Signals — granular reactivity (Angular 17+)
// signal() și computed() declanșează CD GRANULAR per component
// Nu reiterează toată aplicația — cel mai eficient
```

### Pas 3: Computații costisitoare și virtualizare

```tsx
// REACT — useMemo pentru filtrare/sortare costisitoare
const filteredProducts = useMemo(() =>
  products
    .filter(p => p.category === selectedCategory)
    .sort((a, b) => a.price - b.price),
  [products, selectedCategory]
);

// Virtualizare pentru liste lungi (1000+ iteme)
import { FixedSizeList } from 'react-window';

function BigList({ items }) {
  return (
    <FixedSizeList height={500} itemCount={items.length} itemSize={50} width="100%">
      {({ index, style }) => <div style={style}>{items[index].name}</div>}
    </FixedSizeList>
  );
}
```

```typescript
// ANGULAR — computed() pentru filtrare costisitoare
filteredProducts = computed(() =>
  this.products()
    .filter(p => p.category === this.selectedCategory())
    .sort((a, b) => a.price - b.price)
  // Recalculat DOAR când products() sau selectedCategory() se schimbă
);

// Virtualizare Angular — @angular/cdk VirtualScrollViewport
// npm install @angular/cdk
@Component({
  template: `
    <cdk-virtual-scroll-viewport itemSize="50" style="height: 500px">
      <div *cdkVirtualFor="let item of items; trackBy: trackById">
        {{ item.name }}
      </div>
    </cdk-virtual-scroll-viewport>
  `,
})
export class BigListComponent {
  items = input<Item[]>([]);
  trackById = (_: number, item: Item) => item.id;
}
```

---

## 5. Testarea componentelor cu async data fetching

```tsx
// REACT — Setup: vitest + React Testing Library + MSW (Mock Service Worker)
// MSW interceptează fetch la nivel de network — realistic!

// 1. Mock server setup
import { setupServer } from 'msw/node';
import { http, HttpResponse } from 'msw';

export const server = setupServer(
  http.get('/api/users/:id', ({ params }) => {
    return HttpResponse.json({ id: params.id, name: 'Test User' });
  }),
);

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers()); // reset după fiecare test!
afterAll(() => server.close());

// 2. Test pentru component cu fetch
function renderWithProviders(ui: React.ReactElement) {
  const queryClient = new QueryClient({
    defaultOptions: { queries: { retry: false } } // nu retry în teste!
  });
  return render(
    <QueryClientProvider client={queryClient}>{ui}</QueryClientProvider>
  );
}

describe('UserProfile', () => {
  it('shows user data after fetch', async () => {
    renderWithProviders(<UserProfile userId="1" />);

    await waitFor(() => {
      expect(screen.getByText('Test User')).toBeInTheDocument();
    });
  });

  it('shows error state when fetch fails', async () => {
    server.use(
      http.get('/api/users/:id', () => HttpResponse.error())
    );
    renderWithProviders(<UserProfile userId="1" />);
    await waitFor(() => {
      expect(screen.getByText(/failed to load/i)).toBeInTheDocument();
    });
  });
});

// 3. User interactions async
it('submits form and shows success', async () => {
  const user = userEvent.setup();
  renderWithProviders(<CreateUserForm />);

  await user.type(screen.getByLabelText('Name'), 'Emanuel');
  await user.click(screen.getByRole('button', { name: /submit/i }));

  await waitFor(() => {
    expect(screen.getByText(/user created/i)).toBeInTheDocument();
  });
});
```

**Angular equivalent — HttpClientTestingModule + Spectator:**
```typescript
// ANGULAR — TestBed + HttpClientTestingModule (built-in) SAU MSW (funcționează la fel)

// Varianta 1: HttpClientTestingModule (standard Angular)
import { TestBed } from '@angular/core/testing';
import { HttpClientTestingModule, HttpTestingController } from '@angular/common/http/testing';

describe('UserProfileComponent', () => {
  let httpMock: HttpTestingController;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [UserProfileComponent, HttpClientTestingModule],
    }).compileComponents();

    httpMock = TestBed.inject(HttpTestingController);
  });

  afterEach(() => httpMock.verify()); // verifică că nu sunt requesturi neprevăzute

  it('shows user data after fetch', async () => {
    const fixture = TestBed.createComponent(UserProfileComponent);
    fixture.componentInstance.userId = '1';
    fixture.detectChanges();

    // Intercept request HTTP (Angular-way de MSW)
    const req = httpMock.expectOne('/api/users/1');
    expect(req.request.method).toBe('GET');
    req.flush({ id: '1', name: 'Test User' }); // simul response

    fixture.detectChanges(); // trigger change detection

    const el = fixture.nativeElement.querySelector('.user-name');
    expect(el.textContent).toContain('Test User');
  });

  it('shows error state', async () => {
    const fixture = TestBed.createComponent(UserProfileComponent);
    fixture.componentInstance.userId = '1';
    fixture.detectChanges();

    const req = httpMock.expectOne('/api/users/1');
    req.error(new ProgressEvent('error'), { status: 500 }); // simul eroare

    fixture.detectChanges();
    expect(fixture.nativeElement.querySelector('.error')).toBeTruthy();
  });
});

// Varianta 2: MSW funcționează și cu Angular (intercept la nivel browser/node)
// Aceleași handlere, același API — avantaj: tests mai realistic
import { setupServer } from 'msw/node';
// ...identic cu React, MSW nu știe de React/Angular
```

> **MSW e framework-agnostic.** Dacă știi MSW din Angular testing, funcționează identic în React testing. Interceptează requesturi la nivel de network, înainte să ajungă la orice framework.

---

## Întrebări de interviu — Frontend

**Q: Cum structurezi un proiect Next.js pentru MVP rapid?**

> Feature-based folder structure. Server Components by default pentru tot ce nu e interactiv. shadcn/ui. Zustand pentru state simplu. React Query pentru server state. Route Handlers ca BFF. Vercel pentru deployment.
>
> **Dacă compari cu Angular:** "La fel cum în Angular fac feature modules cu lazy loading, în Next.js am `app/(features)/[feature]/page.tsx`. Principiile sunt identice — feature isolation, lazy loading, shared components."

**Q: SSR vs SSG vs CSR — când alegi?**

> **Server Components (SSR):** date personalizate per user, auth, date fresh. **SSG:** pagini statice — blog, marketing. **ISR:** date semi-statice — e-commerce. **CSR:** interactivitate post-load.
>
> **Angular parallel:** "În Angular Universal/SSR, luam aceleași decizii. Paginile cu SEO important → SSR. Paginile de dashboard → CSR. Conceptul e identic, implementarea diferă."

**Q: Cum depanezi un component React lent?**

> React DevTools Profiler → identific ce re-renderează inutil. Cauze comune: funcții recreate (useCallback), obiecte recreate (useMemo), child fără React.memo. Computații costisitoare: useMemo. Liste lungi: react-window.
>
> **Angular parallel:** "Procesul e identic cu Angular DevTools. OnPush în Angular = React.memo. `computed()` în Angular = `useMemo`. Virtualizarea cu CDK VirtualScrollViewport = react-window."

**Q: Context vs Zustand vs React Query?**

> Context: theme, locale, auth. Zustand: UI state complex partajat. React Query: date de pe server cu caching. Nu Redux decât cu echipă experimentată.
>
> **Angular parallel:** "E același pattern: Angular DI (servicii) = Context/Zustand. Angular HttpClient + shareReplay = React Query de bază. TanStack Query are acum adapter Angular dacă vrei API identic."

---

*[← 08 - Q&A](./08-Intrebari-si-Raspunsuri.md) | [10 - AI în Producție →](./10-AI-in-Productie.md)*
