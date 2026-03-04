# 09 - Next.js & Frontend Avansat

> Next.js SSR/SSG/CSR, state management (Zustand, React Query), debugging React,
> testare componente async. Acoperă toate întrebările frontend din lista interviului.

---

## 1. Next.js — Structura pentru MVP rapid și mentenabil

### App Router (Next.js 13+) — structura recomandată

```
src/
├── app/                        # App Router — routing bazat pe folder structure
│   ├── layout.tsx              # Root layout (html, body, providers)
│   ├── page.tsx                # Home page (/)
│   ├── loading.tsx             # Loading UI (Suspense automat)
│   ├── error.tsx               # Error boundary
│   ├── not-found.tsx           # 404
│   │
│   ├── (marketing)/            # Route group — nu afectează URL-ul
│   │   └── about/page.tsx      # /about
│   │
│   ├── (app)/                  # Route group — layout separat pentru app
│   │   ├── layout.tsx          # Layout cu sidebar, nav
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
├── lib/                        # Utilities, helpers, clients (db, api)
│   ├── db.ts                   # Prisma client singleton
│   └── auth.ts                 # Auth helpers
│
├── hooks/                      # Custom hooks
├── stores/                     # Zustand stores
└── types/                      # TypeScript types
```

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

### Când alegi SSG

```tsx
// Blog posts, marketing pages, documentație
// Date care nu se schimbă sau se schimbă rar

// Generează pagini la build time pentru toți slug-urile cunoscute
export async function generateStaticParams() {
  const posts = await db.post.findMany({ select: { slug: true } });
  return posts.map(post => ({ slug: post.slug }));
}

async function BlogPost({ params }: { params: { slug: string } }) {
  const post = await db.post.findUnique({ where: { slug: params.slug } });
  return <PostContent post={post} />;
}
```

### Când alegi ISR (Incremental Static Regeneration)

```tsx
// E-commerce: prețuri, stocuri — se schimbă dar nu în timp real
// News site: articole noi la câteva ore

async function ProductCatalog() {
  const products = await fetchProducts();
  return <ProductGrid products={products} />;
}

export const revalidate = 3600; // revalidează la fiecare oră
// SAU: revalidate = 0 (mereu fresh) | false (niciodată — SSG pur)
```

### Când alegi CSR

```tsx
// Date care depind de interacțiunea utilizatorului DUPĂ load
// Căutare, filtrare, paginare client-side
// Widget-uri interactive (charts, maps, editors)

'use client';

function SearchResults() {
  const [query, setQuery] = useState('');
  const { data } = useQuery({    // React Query — server state
    queryKey: ['search', query],
    queryFn: () => searchProducts(query),
    enabled: query.length > 2,
  });

  return (/* ... */);
}
```

---

## 3. State Management — Context vs Zustand vs Redux vs React Query

### Când folosești ce

```
UI State (local):           useState / useReducer în component
Shared UI State (simplu):   Context API (theme, locale, modal open/close)
Shared App State (complex): Zustand
Server/Async State:         React Query (TanStack Query) sau SWR
Global Complex + DevTools:  Redux Toolkit (rar necesar în 2026)
```

### Zustand — simplu, fără boilerplate

```typescript
// stores/cart.store.ts
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

interface CartItem {
  id: string;
  name: string;
  price: number;
  quantity: number;
}

interface CartStore {
  items: CartItem[];
  total: number;
  addItem: (item: Omit<CartItem, 'quantity'>) => void;
  removeItem: (id: string) => void;
  clearCart: () => void;
}

export const useCartStore = create<CartStore>()(
  persist(  // persists în localStorage automat
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
    { name: 'cart-storage' }
  )
);

// Folosire în component (fără Provider!)
function CartIcon() {
  const { items, total } = useCartStore();
  return <div>Cart ({items.length}) — ${total}</div>;
}

// Selector pentru performance — re-render doar când `items.length` se schimbă
function CartBadge() {
  const count = useCartStore(state => state.items.length); // selector granular
  return <span>{count}</span>;
}
```

### React Query (TanStack Query) — server state

```typescript
// Server state = date care trăiesc pe server, tu le caching local
// React Query gestionează: loading, error, caching, refetch, optimistic updates

// 1. Setup provider în layout
// app/layout.tsx
'use client';
const queryClient = new QueryClient();

export default function RootLayout({ children }) {
  return (
    <html>
      <body>
        <QueryClientProvider client={queryClient}>
          {children}
        </QueryClientProvider>
      </body>
    </html>
  );
}

// 2. Fetch cu useQuery
function UserProfile({ userId }: { userId: string }) {
  const { data: user, isLoading, error } = useQuery({
    queryKey: ['user', userId],      // cache key — schimbă userId → refetch automat
    queryFn: () => fetchUser(userId),
    staleTime: 5 * 60 * 1000,        // consider fresh 5 minute
    retry: 3,                        // retry de 3 ori pe eroare
  });

  if (isLoading) return <Skeleton />;
  if (error) return <ErrorMessage error={error} />;
  return <UserCard user={user} />;
}

// 3. Mutații cu useMutation + invalidare cache
function UpdateProfileForm({ userId }: { userId: string }) {
  const queryClient = useQueryClient();

  const mutation = useMutation({
    mutationFn: (data: UpdateUserDto) => updateUser(userId, data),
    onSuccess: () => {
      // Invalidează cache-ul — refetch automat
      queryClient.invalidateQueries({ queryKey: ['user', userId] });
    },
    onError: (error) => {
      toast.error(error.message);
    },
  });

  return (
    <form onSubmit={(e) => {
      e.preventDefault();
      mutation.mutate({ name: e.target.name.value });
    }}>
      <input name="name" />
      <button disabled={mutation.isPending}>
        {mutation.isPending ? 'Saving...' : 'Save'}
      </button>
    </form>
  );
}
```

### Context API — când e suficient

```typescript
// Context e bun pentru: theme, locale, auth user, modal state
// NU pentru: date care se schimbă des (cauzeaza re-render pe tot arborele)

const ThemeContext = createContext<'light' | 'dark'>('light');

function ThemeProvider({ children }: { children: ReactNode }) {
  const [theme, setTheme] = useState<'light' | 'dark'>('light');

  return (
    <ThemeContext.Provider value={theme}>
      {children}
    </ThemeContext.Provider>
  );
}

// Optimizare: separă value de setter dacă se schimbă rar
```

---

## 4. Debugging și optimizarea unui React component lent

### Pas 1: Identifică problema cu React DevTools Profiler

```
React DevTools → Profiler tab → Record → interacționezi → Stop
→ Flame chart arată ce componente au rerenderat și cât timp au luat
→ Caută componente care re-renderează fără motiv (fără props schimbate)
```

### Pas 2: Cauze comune și fix-uri

```tsx
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
function Parent() {
  const config = { color: 'red', size: 'large' }; // nou la fiecare render
  return <Child config={config} />;
}

// ✅
const CONFIG = { color: 'red', size: 'large' }; // în afara componentei
// SAU
const config = useMemo(() => ({ color: 'red', size: 'large' }), []);

// CAUZA 3: Child nu e memo-izat
// ❌ Child re-renderează chiar dacă props nu s-au schimbat
const Child = ({ name }) => <div>{name}</div>;

// ✅ Skip re-render dacă props sunt identice (shallow comparison)
const Child = React.memo(({ name }) => <div>{name}</div>);

// CAUZA 4: State update în context cauzează re-render tuturor consumatorilor
// Fix: Zustand cu selectori granulari (re-render doar când valoarea selectată se schimbă)
const count = useStore(state => state.count); // nu re-renderează dacă altceva din store se schimbă
```

### Pas 3: Computații costisitoare

```tsx
// Filtrare/sortare pe array mare — fără memo: recalculează la fiecare render
const filteredProducts = useMemo(() =>
  products
    .filter(p => p.category === selectedCategory)
    .sort((a, b) => a.price - b.price),
  [products, selectedCategory] // recalculează DOAR când acestea se schimbă
);

// Virtualizare pentru liste lungi (1000+ iteme)
import { FixedSizeList } from 'react-window';

function BigList({ items }) {
  return (
    <FixedSizeList height={500} itemCount={items.length} itemSize={50} width="100%">
      {({ index, style }) => (
        <div style={style}>{items[index].name}</div>
      )}
    </FixedSizeList>
  );
}
```

---

## 5. Testarea componentelor cu async data fetching

```tsx
// Setup: vitest + React Testing Library + MSW (Mock Service Worker)
// MSW interceptează fetch/axios la nivel de Service Worker — realistic!

// 1. Mock server setup (src/mocks/server.ts)
import { setupServer } from 'msw/node';
import { http, HttpResponse } from 'msw';

export const server = setupServer(
  http.get('/api/users/:id', ({ params }) => {
    return HttpResponse.json({
      id: params.id,
      name: 'Test User',
      email: 'test@example.com',
    });
  }),

  http.get('/api/users', () => {
    return HttpResponse.json([
      { id: '1', name: 'Alice' },
      { id: '2', name: 'Bob' },
    ]);
  })
);

// vitest.setup.ts
beforeAll(() => server.listen());
afterEach(() => server.resetHandlers()); // reset după fiecare test!
afterAll(() => server.close());

// 2. Test pentru component cu fetch
import { render, screen, waitFor } from '@testing-library/react';
import { QueryClientProvider, QueryClient } from '@tanstack/react-query';

function renderWithProviders(ui: React.ReactElement) {
  const queryClient = new QueryClient({
    defaultOptions: { queries: { retry: false } } // nu retry în teste!
  });
  return render(
    <QueryClientProvider client={queryClient}>{ui}</QueryClientProvider>
  );
}

describe('UserProfile', () => {
  it('shows loading state initially', () => {
    renderWithProviders(<UserProfile userId="1" />);
    expect(screen.getByRole('progressbar')).toBeInTheDocument();
  });

  it('shows user data after fetch', async () => {
    renderWithProviders(<UserProfile userId="1" />);

    await waitFor(() => {
      expect(screen.getByText('Test User')).toBeInTheDocument();
    });
    expect(screen.getByText('test@example.com')).toBeInTheDocument();
  });

  it('shows error state when fetch fails', async () => {
    // Override handler pentru acest test specific
    server.use(
      http.get('/api/users/:id', () => {
        return HttpResponse.error();
      })
    );

    renderWithProviders(<UserProfile userId="1" />);

    await waitFor(() => {
      expect(screen.getByText(/failed to load/i)).toBeInTheDocument();
    });
  });
});

// 3. Test pentru user interactions cu async effects
import userEvent from '@testing-library/user-event';

it('submits form and shows success message', async () => {
  const user = userEvent.setup(); // setup înainte de render
  renderWithProviders(<CreateUserForm />);

  await user.type(screen.getByLabelText('Name'), 'Emanuel');
  await user.type(screen.getByLabelText('Email'), 'em@test.com');
  await user.click(screen.getByRole('button', { name: /submit/i }));

  await waitFor(() => {
    expect(screen.getByText(/user created/i)).toBeInTheDocument();
  });
});
```

---

## Întrebări de interviu — Frontend

**Q: Cum structurezi un proiect Next.js pentru MVP rapid?**
> Feature-based folder structure (nu type-based). Server Components by default pentru tot ce nu e interactiv. shadcn/ui pentru componente UI gata. Zustand pentru state simplu. React Query pentru server state. Route Handlers ca BFF. Vercel pentru deployment. Regula: nu construiesc ce nu am nevoie acum.

**Q: SSR vs SSG vs CSR — când alegi?**
> **SSR (Server Component async):** date personalizate per user, auth, date care trebuie fresh. **SSG:** pagini statice cu date la build time — blog, marketing, docs. **ISR:** date care se schimbă dar nu real-time — e-commerce, news. **CSR:** interactivitate post-load — search, filtrare, editors.

**Q: Cum depanezi un component React lent?**
> React DevTools Profiler → identific ce re-renderează inutil. Cauze comune: funcții recreate la fiecare render (fix: useCallback), obiecte recreate (fix: useMemo sau constantă), child fără React.memo. Computații costisitoare: useMemo. Liste lungi: react-window pentru virtualizare.

**Q: Context vs Zustand vs React Query?**
> Context: theme, locale, auth user — date care nu se schimbă des. Zustand: UI state complex partajat între componente (cart, modal, filters). React Query: orice date de pe server — gestionează loading, error, caching, invalidare. Nu folosesc Redux decât dacă echipa are deja experiență cu el.

---

*[← 08 - Q&A](./08-Intrebari-si-Raspunsuri.md) | [10 - AI în Producție →](./10-AI-in-Productie.md)*
