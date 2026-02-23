# Arhitectură și Design Patterns în Vue 3 (Interview Prep - Senior Frontend Architect)

> Folder structure pentru proiecte Vue mari, Composables, Smart/Dumb components,
> Design Patterns (Facade, Strategy, Observer), Monorepo cu Nx/Turborepo.
> Paralele detaliate cu Angular Services, Modules, și patterns.

---

## Cuprins

1. [Folder Structure pentru Proiecte Vue Mari](#1-folder-structure-pentru-proiecte-vue-mari)
2. [Composables - Pattern de Reuse](#2-composables---pattern-de-reuse)
3. [Smart vs Dumb Components (Container/Presentational)](#3-smart-vs-dumb-components-containerpresentational)
4. [Design Patterns în Vue](#4-design-patterns-în-vue)
5. [Barrel Exports](#5-barrel-exports)
6. [Monorepo cu Nx/Turborepo + Vue](#6-monorepo-cu-nxturborepo--vue)
7. [Scalarea aplicației - Architecture Decision Records](#7-scalarea-aplicației---architecture-decision-records)
8. [Paralela completă: Angular Architecture vs Vue Architecture](#8-paralela-completă-angular-architecture-vs-vue-architecture)
9. [Întrebări de interviu](#9-întrebări-de-interviu)

---

## 1. Folder Structure pentru Proiecte Vue Mari

### 1.1 Structura Feature-based (RECOMANDATĂ)

```
src/
├── app/
│   ├── App.vue                    # Root component
│   ├── main.ts                    # Entry point
│   └── bootstrap.ts               # App initialization logic
│
├── features/                      # Feature modules (bounded contexts)
│   ├── products/
│   │   ├── components/            # Feature-specific components
│   │   │   ├── ProductList.vue
│   │   │   ├── ProductCard.vue
│   │   │   ├── ProductFilter.vue
│   │   │   └── ProductDetails.vue
│   │   ├── composables/           # Feature-specific composables
│   │   │   ├── useProducts.ts
│   │   │   ├── useProductFilter.ts
│   │   │   └── useProductSearch.ts
│   │   ├── stores/                # Feature-specific Pinia stores
│   │   │   └── useProductStore.ts
│   │   ├── types/                 # Feature types/interfaces
│   │   │   └── product.types.ts
│   │   ├── api/                   # Feature API calls
│   │   │   └── products.api.ts
│   │   ├── constants/             # Feature constants
│   │   │   └── product.constants.ts
│   │   ├── __tests__/             # Feature tests
│   │   │   ├── ProductList.spec.ts
│   │   │   └── useProducts.spec.ts
│   │   ├── routes.ts              # Feature routes
│   │   └── index.ts               # Public API (barrel export)
│   │
│   ├── auth/
│   │   ├── components/
│   │   │   ├── LoginForm.vue
│   │   │   ├── RegisterForm.vue
│   │   │   └── ForgotPassword.vue
│   │   ├── composables/
│   │   │   ├── useAuth.ts
│   │   │   └── usePermissions.ts
│   │   ├── stores/
│   │   │   └── useAuthStore.ts
│   │   ├── guards/
│   │   │   └── authGuard.ts
│   │   ├── types/
│   │   │   └── auth.types.ts
│   │   ├── api/
│   │   │   └── auth.api.ts
│   │   └── index.ts
│   │
│   ├── checkout/
│   │   ├── components/
│   │   │   ├── CartSummary.vue
│   │   │   ├── CheckoutWizard.vue
│   │   │   ├── PaymentForm.vue
│   │   │   └── OrderConfirmation.vue
│   │   ├── composables/
│   │   │   ├── useCart.ts
│   │   │   └── useCheckoutFlow.ts
│   │   ├── stores/
│   │   │   └── useCartStore.ts
│   │   └── index.ts
│   │
│   └── dashboard/
│       ├── components/
│       ├── composables/
│       ├── stores/
│       └── index.ts
│
├── shared/                        # Cross-feature shared code
│   ├── components/                # Shared/generic components
│   │   ├── ui/                    # Design system components
│   │   │   ├── BaseButton.vue
│   │   │   ├── BaseInput.vue
│   │   │   ├── BaseSelect.vue
│   │   │   ├── BaseModal.vue
│   │   │   ├── BaseToast.vue
│   │   │   ├── BaseSpinner.vue
│   │   │   ├── BaseBadge.vue
│   │   │   ├── DataTable.vue
│   │   │   └── Pagination.vue
│   │   ├── layout/
│   │   │   ├── AppHeader.vue
│   │   │   ├── AppSidebar.vue
│   │   │   ├── AppFooter.vue
│   │   │   └── AppLayout.vue
│   │   └── feedback/
│   │       ├── ErrorBoundary.vue
│   │       ├── EmptyState.vue
│   │       └── LoadingSkeleton.vue
│   │
│   ├── composables/               # Shared composables
│   │   ├── useApi.ts
│   │   ├── useDebounce.ts
│   │   ├── useLocalStorage.ts
│   │   ├── useBreakpoints.ts
│   │   ├── usePagination.ts
│   │   ├── useFormValidation.ts
│   │   └── useEventBus.ts
│   │
│   ├── utils/                     # Pure utility functions
│   │   ├── formatters.ts
│   │   ├── validators.ts
│   │   ├── date.utils.ts
│   │   ├── string.utils.ts
│   │   └── array.utils.ts
│   │
│   ├── types/                     # Shared types
│   │   ├── api.types.ts
│   │   ├── common.types.ts
│   │   └── pagination.types.ts
│   │
│   ├── constants/
│   │   ├── app.constants.ts
│   │   ├── routes.constants.ts
│   │   └── http.constants.ts
│   │
│   └── directives/                # Custom Vue directives
│       ├── vClickOutside.ts
│       ├── vTooltip.ts
│       └── vPermission.ts
│
├── plugins/                       # Vue plugins
│   ├── i18n.ts                    # Internationalization
│   ├── pinia.ts                   # State management
│   ├── axios.ts                   # HTTP client
│   └── errorHandler.ts            # Global error handler
│
├── router/                        # Root router configuration
│   ├── index.ts
│   └── guards.ts
│
├── styles/                        # Global styles
│   ├── _variables.scss
│   ├── _mixins.scss
│   ├── _reset.scss
│   └── main.scss
│
└── assets/
    ├── images/
    ├── fonts/
    └── icons/
```

### 1.2 De ce Feature-based?

**Argumente principale:**

1. **Coeziune mare** - tot codul legat de un feature e în același loc
2. **Coupling scăzut** - features comunică prin interfețe publice (barrel exports)
3. **Navigare ușoară** - orice developer găsește rapid codul relevant
4. **Extragere facilă** - un feature poate deveni Micro Frontend (MFE)
5. **Bounded Context (DDD)** - fiecare feature e un context delimitat
6. **Ownership clar** - o echipă poate deține un feature

```
┌─────────────────────────────────────────────────┐
│                   App Shell                      │
│  ┌──────────────────────────────────────────┐   │
│  │              Router (Lazy Load)            │   │
│  └──────┬──────────┬──────────┬─────────────┘   │
│         │          │          │                   │
│  ┌──────▼──┐ ┌─────▼────┐ ┌──▼──────────┐      │
│  │Products │ │  Auth     │ │  Checkout   │      │
│  │Feature  │ │  Feature  │ │  Feature    │      │
│  │         │ │           │ │             │      │
│  │components│ │components│ │ components  │      │
│  │composable│ │composable│ │ composables │      │
│  │stores   │ │stores    │ │ stores      │      │
│  │api      │ │api       │ │ api         │      │
│  └────┬────┘ └────┬─────┘ └──────┬──────┘      │
│       │           │              │               │
│  ┌────▼───────────▼──────────────▼──────────┐   │
│  │           shared/                         │   │
│  │  components/ui  composables  utils  types │   │
│  └──────────────────────────────────────────┘   │
└─────────────────────────────────────────────────┘
```

### 1.3 Reguli de Dependențe

```
REGULĂ STRICTĂ DE IMPORT:

features/products/ ──► shared/          ✅ OK
features/products/ ──► features/auth/   ❌ INTERZIS (coupling direct)
shared/            ──► features/        ❌ INTERZIS (dependency inversă)
features/products/ ──► features/products/index.ts  ✅ OK (intern)
features/checkout/ ──► features/products/index.ts  ⚠️  Doar prin barrel export

DIRECȚIA CORECTĂ:
features → shared → core (unidirecțional)
```

Aceste reguli se pot enforța cu **ESLint** (boundaries plugin) sau **Nx** (module boundaries).

### 1.4 Structura Layer-based (NU recomandată pentru proiecte mari)

```
src/
├── components/          # TOATE componentele
│   ├── ProductList.vue
│   ├── LoginForm.vue
│   ├── CartSummary.vue
│   └── ... (100+ fișiere)
├── composables/         # TOATE composables
│   ├── useProducts.ts
│   ├── useAuth.ts
│   └── ...
├── stores/              # TOATE stores
│   ├── productStore.ts
│   ├── authStore.ts
│   └── ...
├── views/               # TOATE paginile
│   ├── ProductsPage.vue
│   ├── LoginPage.vue
│   └── ...
└── utils/
```

**De ce NU funcționează la scară mare:**

1. **Foldere enorme** - 100+ fișiere în `components/` face navigarea imposibilă
2. **Fără context** - `useData.ts` nu spune nimic despre ce face
3. **Coupling ascuns** - nu e clar ce depinde de ce
4. **Greu de extras** - extragerea unui feature necesită cherry-picking din fiecare folder
5. **Code reviews dificile** - PR-urile ating fișiere din toate folderele

> **Regulă de arhitect:** Layer-based e OK pentru aplicații mici (< 10 componente).
> Feature-based devine necesar de la primele 3-4 features distincte.

### Paralela cu Angular

| Aspect | Angular | Vue |
|--------|---------|-----|
| **Organizare features** | NgModules (sau standalone) | Foldere + barrel exports |
| **Încapsulare** | Module boundary (providedIn) | Convenție + ESLint rules |
| **Lazy loading** | `loadChildren` pe module | `() => import()` pe rute |
| **Shared code** | SharedModule / standalone | `shared/` folder |
| **Public API** | NgModule exports | `index.ts` barrel export |
| **Enforcement** | Nx module boundaries | Nx / ESLint boundaries |

Conceptual, structura feature-based e **identică**. Diferența e că Angular are suport nativ (NgModule), iar Vue se bazează pe **convenții** și **tooling extern** (ESLint, Nx).

---

## 2. Composables - Pattern de Reuse

### 2.1 Ce sunt Composables

**Composables** sunt funcții care încapsulează și reutilizează **logică reactivă** cu Composition API. Sunt echivalentul funcțional al Angular Services, dar fără clase și fără Dependency Injection.

**Convenții:**
- Prefix `use` (ex: `useProducts`, `useAuth`, `useLocalStorage`)
- Returnează un obiect cu **refs**, **computeds** și **funcții**
- Pot folosi lifecycle hooks (`onMounted`, `onUnmounted`)
- Pot fi **stateful** (păstrează state) sau **stateless** (doar logică)

```
┌────────────────────────────────────┐
│         Component A                │
│  const { count, increment }       │
│    = useCounter()                  │
│  // count = ref(0) - instanță A   │
└────────────────────────────────────┘

┌────────────────────────────────────┐
│         Component B                │
│  const { count, increment }       │
│    = useCounter()                  │
│  // count = ref(0) - instanță B   │
│  // DIFERITĂ de instanța A        │
└────────────────────────────────────┘
```

> **Important:** Fiecare apel `useCounter()` creează o **instanță nouă**.
> Aceasta e diferența fundamentală față de Angular Services (care sunt singleton).

### 2.2 Composable simplu

```typescript
// composables/useCounter.ts
import { ref, computed } from 'vue'

export function useCounter(initialValue = 0) {
  const count = ref(initialValue)
  const doubled = computed(() => count.value * 2)
  const isPositive = computed(() => count.value > 0)

  function increment() {
    count.value++
  }

  function decrement() {
    count.value--
  }

  function reset() {
    count.value = initialValue
  }

  function set(value: number) {
    count.value = value
  }

  return {
    // State (readonly expus ca Ref)
    count,
    // Computed
    doubled,
    isPositive,
    // Methods
    increment,
    decrement,
    reset,
    set
  }
}
```

**Utilizare în component:**

```vue
<script setup lang="ts">
import { useCounter } from '@/shared/composables/useCounter'

const { count, doubled, increment, decrement, reset } = useCounter(10)
</script>

<template>
  <div>
    <p>Count: {{ count }}</p>
    <p>Doubled: {{ doubled }}</p>
    <button @click="increment">+</button>
    <button @click="decrement">-</button>
    <button @click="reset">Reset</button>
  </div>
</template>
```

### 2.3 Composable cu Lifecycle Hooks

```typescript
// composables/useWindowSize.ts
import { ref, onMounted, onUnmounted } from 'vue'

export function useWindowSize() {
  const width = ref(window.innerWidth)
  const height = ref(window.innerHeight)
  const isMobile = computed(() => width.value < 768)
  const isTablet = computed(() => width.value >= 768 && width.value < 1024)
  const isDesktop = computed(() => width.value >= 1024)

  function update() {
    width.value = window.innerWidth
    height.value = window.innerHeight
  }

  onMounted(() => {
    window.addEventListener('resize', update)
  })

  onUnmounted(() => {
    window.removeEventListener('resize', update)
  })

  return {
    width,
    height,
    isMobile,
    isTablet,
    isDesktop
  }
}
```

**Cum funcționează lifecycle hooks în composables:**

```
Component mount
    │
    ▼
setup() execută
    │
    ├── useWindowSize() este apelat
    │       │
    │       ├── Creează refs (width, height)
    │       ├── Înregistrează onMounted callback
    │       └── Înregistrează onUnmounted callback
    │
    ▼
Component mounted (lifecycle)
    │
    ├── onMounted din useWindowSize se execută
    │   └── addEventListener('resize', update)
    │
    ▼
... component trăiește ...
    │
    ▼
Component unmounted (lifecycle)
    │
    └── onUnmounted din useWindowSize se execută
        └── removeEventListener('resize', update)
```

> **Atenție:** Composables care folosesc lifecycle hooks trebuie apelate
> **sincron** în `setup()`. Nu le poți apela în callback-uri async.

### 2.4 Composable cu Async + Error Handling

```typescript
// composables/useApi.ts
import { ref, unref, watch, isRef, onMounted, type Ref } from 'vue'

interface UseApiOptions {
  immediate?: boolean    // Fetch imediat? (default: true)
  debounce?: number      // Debounce în ms
  retries?: number       // Număr de retry-uri
  retryDelay?: number    // Delay între retry-uri
}

interface UseApiReturn<T> {
  data: Ref<T | null>
  loading: Ref<boolean>
  error: Ref<Error | null>
  execute: () => Promise<void>
  refresh: () => Promise<void>
}

export function useApi<T>(
  url: string | Ref<string>,
  options: UseApiOptions = {}
): UseApiReturn<T> {
  const {
    immediate = true,
    retries = 0,
    retryDelay = 1000
  } = options

  const data = ref<T | null>(null) as Ref<T | null>
  const loading = ref(false)
  const error = ref<Error | null>(null)
  let abortController: AbortController | null = null

  async function execute(): Promise<void> {
    // Anulează request-ul anterior dacă există
    if (abortController) {
      abortController.abort()
    }
    abortController = new AbortController()

    loading.value = true
    error.value = null

    let lastError: Error | null = null

    for (let attempt = 0; attempt <= retries; attempt++) {
      try {
        const urlValue = unref(url)
        const response = await fetch(urlValue, {
          signal: abortController.signal
        })

        if (!response.ok) {
          throw new Error(`HTTP Error: ${response.status} ${response.statusText}`)
        }

        data.value = await response.json()
        lastError = null
        break // Success - ieșim din loop
      } catch (e) {
        if (e instanceof DOMException && e.name === 'AbortError') {
          return // Request anulat - nu setăm error
        }
        lastError = e instanceof Error ? e : new Error(String(e))

        if (attempt < retries) {
          await new Promise(resolve => setTimeout(resolve, retryDelay))
        }
      }
    }

    if (lastError) {
      error.value = lastError
    }

    loading.value = false
  }

  // Auto-fetch când URL-ul se schimbă (dacă e Ref)
  if (isRef(url)) {
    watch(url, () => execute(), { immediate })
  } else if (immediate) {
    onMounted(() => execute())
  }

  // Cleanup la unmount
  onUnmounted(() => {
    if (abortController) {
      abortController.abort()
    }
  })

  return {
    data,
    loading,
    error,
    execute,
    refresh: execute
  }
}
```

**Utilizare:**

```vue
<script setup lang="ts">
import { useApi } from '@/shared/composables/useApi'
import type { Product } from '@/features/products/types/product.types'

// Fetch automat la mount
const { data: products, loading, error, refresh } = useApi<Product[]>(
  '/api/products',
  { retries: 2 }
)

// Fetch reactiv (se re-fetch când categoryId se schimbă)
const categoryId = ref('electronics')
const { data: categoryProducts } = useApi<Product[]>(
  computed(() => `/api/products?category=${categoryId.value}`)
)
</script>

<template>
  <div>
    <LoadingSkeleton v-if="loading" />
    <ErrorMessage v-else-if="error" :error="error" @retry="refresh" />
    <ProductList v-else-if="products" :products="products" />
  </div>
</template>
```

### 2.5 Composable Patterns

#### Pattern 1: Stateful vs Stateless

```typescript
// STATELESS - fiecare apel creează state nou
export function useCounter(initial = 0) {
  const count = ref(initial)     // Nou ref la fiecare apel
  const increment = () => count.value++
  return { count, increment }
}

// STATEFUL (Singleton) - state partajat între componente
// State-ul e la nivel de modul (în afara funcției)
const globalCount = ref(0)

export function useGlobalCounter() {
  const increment = () => globalCount.value++
  const decrement = () => globalCount.value--
  return {
    count: readonly(globalCount),  // readonly pentru protecție
    increment,
    decrement
  }
}
```

```
STATELESS:                         STATEFUL (Singleton):
┌─────────┐  ┌─────────┐         ┌─────────┐  ┌─────────┐
│Comp A   │  │Comp B   │         │Comp A   │  │Comp B   │
│count: 5 │  │count: 3 │         │count ───┼──┼─► 5     │
│(propriu)│  │(propriu)│         │         │  │         │
└─────────┘  └─────────┘         └─────────┘  └─────────┘
  Diferite instanțe                Aceeași referință
```

#### Pattern 2: Reactive Arguments (acceptă refs)

```typescript
// Composable care acceptă atât valori simple cât și refs
import { type MaybeRefOrGetter, toValue, watch } from 'vue'

export function useTitle(title: MaybeRefOrGetter<string>) {
  // toValue() rezolvă ref, getter, sau valoare simplă
  function updateTitle() {
    document.title = toValue(title)
  }

  // Watch funcționează cu ref/getter, ignoră valori statice
  watch(() => toValue(title), updateTitle, { immediate: true })

  return { updateTitle }
}

// Utilizări posibile:
useTitle('Static Title')                      // Valoare simplă
useTitle(titleRef)                             // Ref
useTitle(() => `Page ${page.value} - App`)     // Getter
```

#### Pattern 3: Composable cu Cleanup

```typescript
// composables/useEventListener.ts
export function useEventListener<K extends keyof WindowEventMap>(
  target: Window | HTMLElement | Ref<HTMLElement | null>,
  event: K,
  handler: (e: WindowEventMap[K]) => void,
  options?: AddEventListenerOptions
) {
  let cleanup: (() => void) | null = null

  function attach(el: Window | HTMLElement) {
    el.addEventListener(event, handler as EventListener, options)
    cleanup = () => el.removeEventListener(event, handler as EventListener, options)
  }

  if (isRef(target)) {
    watch(target, (newEl, oldEl) => {
      if (cleanup) cleanup()
      if (newEl) attach(newEl)
    }, { immediate: true })
  } else {
    onMounted(() => attach(target))
  }

  onUnmounted(() => {
    if (cleanup) cleanup()
  })
}
```

#### Pattern 4: Composables care folosesc alte Composables (Composition)

```typescript
// composables/useProductSearch.ts
export function useProductSearch() {
  // Compoziție - folosim alte composables
  const { debounced: debouncedQuery, value: searchQuery } = useDebounce('', 300)
  const { data: results, loading, error } = useApi<Product[]>(
    computed(() =>
      debouncedQuery.value
        ? `/api/products/search?q=${encodeURIComponent(debouncedQuery.value)}`
        : '/api/products'
    )
  )
  const { width } = useWindowSize()
  const resultsPerPage = computed(() => (width.value < 768 ? 10 : 20))

  const paginatedResults = computed(() =>
    results.value?.slice(0, resultsPerPage.value) ?? []
  )

  return {
    searchQuery,
    results: paginatedResults,
    loading,
    error,
    totalResults: computed(() => results.value?.length ?? 0)
  }
}
```

#### Pattern 5: Convenții de Return

```typescript
// ✅ CORECT - returnează obiect (destructurare pe nume)
export function useCounter() {
  const count = ref(0)
  return { count, increment: () => count.value++ }
}
// Utilizare: const { count, increment } = useCounter()

// ❌ EVITĂ - array return (ca React hooks)
export function useCounter() {
  const count = ref(0)
  return [count, () => count.value++] as const
}
// Utilizare: const [count, increment] = useCounter()
// Problemă: nu se vede ce returnezi, ordinea contează
```

> **Regulă Vue:** Returnează mereu **obiect** din composables.
> Array return e pattern React (useState). Vue preferă named destructuring.

### 2.6 Composable cu Pinia Store Integration

```typescript
// features/products/composables/useProductOperations.ts
import { storeToRefs } from 'pinia'
import { useProductStore } from '../stores/useProductStore'
import { useCartStore } from '@/features/checkout/stores/useCartStore'
import { useApi } from '@/shared/composables/useApi'
import { useNotifications } from '@/shared/composables/useNotifications'

export function useProductOperations() {
  const productStore = useProductStore()
  const cartStore = useCartStore()
  const { notify } = useNotifications()

  // storeToRefs păstrează reactivitatea la destructurare
  const { products, selectedCategory, sortOrder } = storeToRefs(productStore)

  const filteredProducts = computed(() => {
    let result = products.value

    if (selectedCategory.value) {
      result = result.filter(p => p.category === selectedCategory.value)
    }

    if (sortOrder.value === 'price-asc') {
      result = [...result].sort((a, b) => a.price - b.price)
    } else if (sortOrder.value === 'price-desc') {
      result = [...result].sort((a, b) => b.price - a.price)
    }

    return result
  })

  async function addToCart(product: Product) {
    try {
      cartStore.addItem(product)
      notify({ type: 'success', message: `${product.name} adăugat în coș` })
    } catch (e) {
      notify({ type: 'error', message: 'Eroare la adăugare în coș' })
    }
  }

  async function loadProducts() {
    await productStore.fetchProducts()
  }

  return {
    products: filteredProducts,
    selectedCategory,
    sortOrder,
    addToCart,
    loadProducts,
    setCategory: productStore.setCategory,
    setSortOrder: productStore.setSortOrder
  }
}
```

### 2.7 VueUse - Biblioteca de Composables

**VueUse** este cea mai populară bibliotecă de composables Vue, cu 200+ funcții gata de folosit.

**Categorii principale:**

| Categorie | Exemple | Angular Echivalent |
|-----------|---------|-------------------|
| **Browser** | `useLocalStorage`, `useClipboard`, `useMediaQuery` | Nu există direct |
| **Sensors** | `useMouse`, `useScroll`, `useIntersectionObserver` | Angular CDK (parțial) |
| **Animation** | `useTransition`, `useInterval`, `useTimeout` | RxJS timers |
| **State** | `useRefHistory`, `useToggle`, `useCycleList` | Manual cu BehaviorSubject |
| **Network** | `useFetch`, `useWebSocket`, `useEventSource` | HttpClient / RxJS |
| **Utilities** | `useDebounceFn`, `useThrottleFn`, `tryOnMounted` | RxJS operators |

**Exemple populare:**

```typescript
import {
  useLocalStorage,
  useDark,
  useToggle,
  useIntersectionObserver,
  useFetch,
  useClipboard,
  useEventListener,
  useMediaQuery,
  useRefHistory
} from '@vueuse/core'

// Persistare în localStorage cu reactivitate automată
const theme = useLocalStorage('app-theme', 'light')

// Dark mode toggle
const isDark = useDark()
const toggleDark = useToggle(isDark)

// Intersection Observer (lazy loading)
const target = ref<HTMLElement | null>(null)
const isVisible = ref(false)
useIntersectionObserver(target, ([{ isIntersecting }]) => {
  isVisible.value = isIntersecting
})

// Clipboard
const { copy, copied } = useClipboard()
await copy('Text copiat!')

// Media queries
const isLargeScreen = useMediaQuery('(min-width: 1024px)')

// Undo/Redo pentru state
const counter = ref(0)
const { undo, redo, canUndo, canRedo } = useRefHistory(counter)
```

### 2.8 Paralela cu Angular

| Aspect | Vue Composable | Angular Service |
|--------|---------------|-----------------|
| **Definiție** | Funcție (`useXxx()`) | Clasă cu `@Injectable()` |
| **DI (Dependency Injection)** | Nu - import direct | Da - constructor injection |
| **Singleton** | Nu by default (fiecare call = instanță nouă) | Da cu `providedIn: 'root'` |
| **Singleton posibil** | Da - state la nivel de modul | Implicit |
| **Lifecycle hooks** | `onMounted`, `onUnmounted` etc. | Fără (doar `OnDestroy` pe componente) |
| **Reactivitate** | `ref`/`computed` built-in | Manual (`BehaviorSubject`/`Signal`) |
| **Testing** | Import + apel funcție | `TestBed` + inject |
| **State sharing** | Module-level ref (singleton manual) | Singleton service automat |
| **Tipuri de reuse** | Logică + state reactiv | Logică + DI tree |
| **Tree-shaking** | Excelent (funcții) | Bun (providedIn: 'root') |
| **Scoping** | Manual (module-level vs function-level) | `providedIn` / component providers |
| **Ierarhie** | Fără (flat) | DI tree ierarhic |

**Exemplu comparativ:**

```typescript
// === VUE COMPOSABLE ===
// composables/useUserService.ts
import { ref, computed } from 'vue'

const currentUser = ref<User | null>(null)  // Module-level = singleton

export function useUserService() {
  const isLoggedIn = computed(() => currentUser.value !== null)
  const displayName = computed(() => currentUser.value?.name ?? 'Guest')

  async function login(credentials: Credentials): Promise<void> {
    const response = await fetch('/api/auth/login', {
      method: 'POST',
      body: JSON.stringify(credentials)
    })
    currentUser.value = await response.json()
  }

  function logout(): void {
    currentUser.value = null
  }

  return { currentUser: readonly(currentUser), isLoggedIn, displayName, login, logout }
}
```

```typescript
// === ANGULAR SERVICE ===
// services/user.service.ts
@Injectable({ providedIn: 'root' })
export class UserService {
  private currentUser = signal<User | null>(null)

  isLoggedIn = computed(() => this.currentUser() !== null)
  displayName = computed(() => this.currentUser()?.name ?? 'Guest')

  constructor(private http: HttpClient) {}

  login(credentials: Credentials): Observable<User> {
    return this.http.post<User>('/api/auth/login', credentials).pipe(
      tap(user => this.currentUser.set(user))
    )
  }

  logout(): void {
    this.currentUser.set(null)
  }
}
```

**Observații cheie pentru interviu:**
- Vue composables sunt **mai simple** (fără clasă, fără decorator, fără DI)
- Angular services au **DI tree ierarhic** (scoped providers) - Vue nu
- Vue compensează lipsa DI cu **provide/inject** (dar e mai rar folosit)
- Testarea composables e **mai directă** (import + apel), Angular necesită TestBed

---

## 3. Smart vs Dumb Components (Container/Presentational)

### 3.1 Conceptul

```
┌──────────────────────────────────────────────────────┐
│  SMART (Container) Component                          │
│                                                       │
│  - Știe despre store/composables/API                  │
│  - Gestionează state                                  │
│  - Orchestrează logica de business                    │
│  - Puțin sau deloc UI propriu                         │
│  - Conectează Dumb components între ele               │
│                                                       │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────┐ │
│  │ DUMB comp    │  │ DUMB comp    │  │ DUMB comp  │ │
│  │              │  │              │  │            │ │
│  │ - Props in   │  │ - Props in   │  │ - Props in │ │
│  │ - Events out │  │ - Events out │  │ - Events   │ │
│  │ - Pure UI    │  │ - Pure UI    │  │ - Pure UI  │ │
│  │ - Reusable   │  │ - Reusable   │  │ - Reusable │ │
│  └──────────────┘  └──────────────┘  └────────────┘ │
└──────────────────────────────────────────────────────┘
```

### 3.2 Container Components (Smart)

```vue
<!-- features/products/components/ProductListContainer.vue -->
<script setup lang="ts">
import { useProductOperations } from '../composables/useProductOperations'
import { useProductFilter } from '../composables/useProductFilter'
import ProductGrid from './ProductGrid.vue'
import ProductFilterPanel from './ProductFilterPanel.vue'
import LoadingSkeleton from '@/shared/components/feedback/LoadingSkeleton.vue'
import ErrorMessage from '@/shared/components/feedback/ErrorMessage.vue'

// Smart: folosește composables, stores, API
const {
  products,
  loading,
  error,
  addToCart,
  loadProducts,
  selectedCategory,
  setCategory
} = useProductOperations()

const {
  filters,
  activeFilterCount,
  applyFilter,
  clearFilters
} = useProductFilter()

// Smart: gestionează flow-ul
function handleProductClick(productId: string) {
  router.push({ name: 'product-details', params: { id: productId } })
}

function handleAddToCart(product: Product) {
  addToCart(product)
}

// Smart: coordonează fetch-ul inițial
onMounted(() => {
  loadProducts()
})
</script>

<template>
  <!-- Smart: puțin UI propriu, doar layout și condiții -->
  <div class="product-list-page">
    <ProductFilterPanel
      :filters="filters"
      :active-count="activeFilterCount"
      @filter-change="applyFilter"
      @clear="clearFilters"
    />

    <LoadingSkeleton v-if="loading" :count="6" variant="card" />

    <ErrorMessage
      v-else-if="error"
      :error="error"
      @retry="loadProducts"
    />

    <ProductGrid
      v-else
      :products="products"
      @product-click="handleProductClick"
      @add-to-cart="handleAddToCart"
    />
  </div>
</template>
```

### 3.3 Presentational Components (Dumb)

```vue
<!-- features/products/components/ProductGrid.vue -->
<script setup lang="ts">
import type { Product } from '../types/product.types'
import ProductCard from './ProductCard.vue'

// Dumb: primește DOAR props
interface Props {
  products: Product[]
  columns?: 2 | 3 | 4
  showPrices?: boolean
}

const props = withDefaults(defineProps<Props>(), {
  columns: 3,
  showPrices: true
})

// Dumb: emite DOAR events
const emit = defineEmits<{
  productClick: [productId: string]
  addToCart: [product: Product]
}>()
</script>

<template>
  <div
    class="product-grid"
    :class="`grid-cols-${columns}`"
  >
    <ProductCard
      v-for="product in products"
      :key="product.id"
      :product="product"
      :show-price="showPrices"
      @click="emit('productClick', product.id)"
      @add-to-cart="emit('addToCart', product)"
    />
  </div>
</template>

<style scoped>
.product-grid {
  display: grid;
  gap: 1.5rem;
}
.grid-cols-2 { grid-template-columns: repeat(2, 1fr); }
.grid-cols-3 { grid-template-columns: repeat(3, 1fr); }
.grid-cols-4 { grid-template-columns: repeat(4, 1fr); }
</style>
```

```vue
<!-- features/products/components/ProductCard.vue -->
<script setup lang="ts">
import type { Product } from '../types/product.types'
import BaseButton from '@/shared/components/ui/BaseButton.vue'
import BaseBadge from '@/shared/components/ui/BaseBadge.vue'

// Dumb: doar props și events
interface Props {
  product: Product
  showPrice?: boolean
  compact?: boolean
}

const props = withDefaults(defineProps<Props>(), {
  showPrice: true,
  compact: false
})

const emit = defineEmits<{
  click: []
  addToCart: [product: Product]
}>()

// Dumb: computed local (derivă doar din props, nu din store/API)
const formattedPrice = computed(() =>
  new Intl.NumberFormat('ro-RO', {
    style: 'currency',
    currency: 'RON'
  }).format(props.product.price)
)

const isOnSale = computed(() =>
  props.product.originalPrice > props.product.price
)
</script>

<template>
  <article
    class="product-card"
    :class="{ compact }"
    @click="emit('click')"
  >
    <div class="product-card__image">
      <img :src="product.imageUrl" :alt="product.name" loading="lazy" />
      <BaseBadge v-if="isOnSale" variant="danger">Reducere</BaseBadge>
    </div>

    <div class="product-card__body">
      <h3 class="product-card__title">{{ product.name }}</h3>
      <p v-if="!compact" class="product-card__description">
        {{ product.description }}
      </p>

      <div v-if="showPrice" class="product-card__price">
        <span class="current-price">{{ formattedPrice }}</span>
      </div>
    </div>

    <div class="product-card__actions">
      <BaseButton
        variant="primary"
        size="sm"
        @click.stop="emit('addToCart', product)"
      >
        Adaugă în coș
      </BaseButton>
    </div>
  </article>
</template>

<style scoped>
.product-card {
  border: 1px solid var(--border-color);
  border-radius: 8px;
  overflow: hidden;
  transition: box-shadow 0.2s ease;
}
.product-card:hover {
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
}
</style>
```

### 3.4 Ghid: Cum Decizi Smart vs Dumb?

| Întrebare | Smart | Dumb |
|-----------|-------|------|
| Folosește store/composables? | Da | Nu |
| Face API calls? | Da | Nu |
| Are side effects? | Da | Nu |
| Refolosibil în alt context? | De obicei nu | Da |
| Props sunt principala sursă de date? | Nu | Da |
| Emite events pentru acțiuni? | Rar | Mereu |
| Are business logic? | Da | Nu (doar UI logic) |
| Greu de testat izolat? | Mai greu | Ușor |

**Regulă practică:**
- **70-80% Dumb components** (reusable, testabile)
- **20-30% Smart components** (orchestrare, state management)
- Fiecare **pagină/view** e de obicei un Smart component
- **Design system components** sunt mereu Dumb

### 3.5 Paralela cu Angular

```
ANGULAR:                              VUE:

Smart Component                       Smart Component (Container)
├── Injectează Services               ├── Apelează Composables
├── Subscribe la Observables          ├── Folosește refs/computed
├── Dispatch actions                  ├── Apelează store actions
└── Template cu Dumb components       └── Template cu Dumb components

Dumb Component                        Dumb Component
├── @Input() props                    ├── defineProps
├── @Output() events                  ├── defineEmits
├── ChangeDetection.OnPush            ├── (automatic - Vue e reactiv)
└── Pure template                     └── Pure template
```

> **Diferența cheie:** Angular Dumb components beneficiază de `OnPush` change detection
> pentru performanță. Vue nu are nevoie - sistemul reactiv urmărește doar ce se folosește.

---

## 4. Design Patterns în Vue

### 4.1 Facade Pattern

**Scopul:** Simplifică interacțiunea cu subsisteme complexe printr-o interfață unificată.

```
  Component
     │
     ▼
┌─────────────────┐
│   Facade         │  ◄── Interfață simplificată
│ useProductFacade │
└───┬─────┬────┬──┘
    │     │    │
    ▼     ▼    ▼
  Store  API  Analytics    ◄── Subsisteme complexe
```

```typescript
// features/products/composables/useProductFacade.ts
import { computed, type Ref } from 'vue'
import { storeToRefs } from 'pinia'
import { useProductStore } from '../stores/useProductStore'
import { useCartStore } from '@/features/checkout/stores/useCartStore'
import { useApi } from '@/shared/composables/useApi'
import { useAnalytics } from '@/shared/composables/useAnalytics'
import { useNotifications } from '@/shared/composables/useNotifications'
import type { Product, ProductFilter } from '../types/product.types'

export function useProductFacade() {
  // Subsistem 1: Product Store
  const productStore = useProductStore()
  const { selectedCategory, sortOrder } = storeToRefs(productStore)

  // Subsistem 2: Cart Store
  const cartStore = useCartStore()

  // Subsistem 3: API
  const { data: products, loading, error, refresh } = useApi<Product[]>(
    '/api/products'
  )

  // Subsistem 4: Analytics
  const { track } = useAnalytics()

  // Subsistem 5: Notifications
  const { notify } = useNotifications()

  // Facade expune interfață simplificată
  const featuredProducts = computed(() =>
    products.value?.filter(p => p.featured) ?? []
  )

  const filteredProducts = computed(() => {
    let result = products.value ?? []
    if (selectedCategory.value) {
      result = result.filter(p => p.category === selectedCategory.value)
    }
    return result
  })

  const cartItemCount = computed(() => cartStore.totalItems)

  async function addToCart(product: Product): Promise<void> {
    try {
      cartStore.addItem(product)
      track('add_to_cart', { productId: product.id, price: product.price })
      notify({ type: 'success', message: `${product.name} adăugat în coș` })
    } catch (e) {
      notify({ type: 'error', message: 'Eroare la adăugarea în coș' })
      track('add_to_cart_error', { productId: product.id })
    }
  }

  function setFilter(filter: Partial<ProductFilter>): void {
    if (filter.category !== undefined) {
      productStore.setCategory(filter.category)
    }
    if (filter.sortOrder !== undefined) {
      productStore.setSortOrder(filter.sortOrder)
    }
    track('filter_applied', filter)
  }

  // Interfață clară și simplificată
  return {
    // State
    products: filteredProducts,
    featuredProducts,
    loading,
    error,
    cartItemCount,
    selectedCategory,
    sortOrder,
    // Actions
    addToCart,
    setFilter,
    refresh
  }
}
```

**Utilizare în component (simplu, curat):**

```vue
<script setup lang="ts">
import { useProductFacade } from '../composables/useProductFacade'

// O singură linie - componentul nu știe despre store, API, analytics
const {
  products,
  featuredProducts,
  loading,
  addToCart,
  setFilter
} = useProductFacade()
</script>
```

### 4.2 Strategy Pattern

**Scopul:** Definește o familie de algoritmi interschimbabili.

```
┌──────────────────────────┐
│    useSortStrategy        │
│                          │
│  strategies = {          │
│    'price-asc': fn,      │
│    'price-desc': fn,     │
│    'name': fn,           │
│    'newest': fn,         │
│    'popular': fn         │
│  }                       │
│                          │
│  currentStrategy ►───────┼──► sorted result
└──────────────────────────┘
```

```typescript
// composables/useSortStrategy.ts
import { ref, computed, type Ref } from 'vue'

// Definim interfața Strategy
interface SortStrategy<T> {
  key: string
  label: string
  compareFn: (a: T, b: T) => number
}

export function useSortStrategy<T>(
  items: Ref<T[]>,
  strategies: SortStrategy<T>[]
) {
  const currentStrategyKey = ref(strategies[0]?.key ?? '')

  const currentStrategy = computed(() =>
    strategies.find(s => s.key === currentStrategyKey.value) ?? strategies[0]
  )

  const sorted = computed(() => {
    if (!items.value?.length || !currentStrategy.value) return items.value
    return [...items.value].sort(currentStrategy.value.compareFn)
  })

  function setStrategy(key: string): void {
    const exists = strategies.some(s => s.key === key)
    if (exists) {
      currentStrategyKey.value = key
    }
  }

  return {
    sorted,
    currentStrategyKey,
    currentStrategy,
    availableStrategies: strategies,
    setStrategy
  }
}

// Utilizare concretă:
const productSortStrategies: SortStrategy<Product>[] = [
  {
    key: 'price-asc',
    label: 'Preț crescător',
    compareFn: (a, b) => a.price - b.price
  },
  {
    key: 'price-desc',
    label: 'Preț descrescător',
    compareFn: (a, b) => b.price - a.price
  },
  {
    key: 'name-asc',
    label: 'Nume A-Z',
    compareFn: (a, b) => a.name.localeCompare(b.name)
  },
  {
    key: 'newest',
    label: 'Cele mai noi',
    compareFn: (a, b) =>
      new Date(b.createdAt).getTime() - new Date(a.createdAt).getTime()
  },
  {
    key: 'popular',
    label: 'Popularitate',
    compareFn: (a, b) => b.reviewCount - a.reviewCount
  }
]

// În component:
const products = ref<Product[]>([])
const { sorted, setStrategy, availableStrategies } = useSortStrategy(
  products,
  productSortStrategies
)
```

**Strategy Pattern pentru Validare:**

```typescript
// composables/useValidationStrategy.ts
interface ValidationRule<T> {
  name: string
  validate: (value: T) => boolean
  message: string
}

export function useValidation<T>(
  value: Ref<T>,
  rules: ValidationRule<T>[]
) {
  const errors = computed(() =>
    rules
      .filter(rule => !rule.validate(value.value))
      .map(rule => ({ name: rule.name, message: rule.message }))
  )

  const isValid = computed(() => errors.value.length === 0)
  const firstError = computed(() => errors.value[0]?.message ?? null)

  return { errors, isValid, firstError }
}

// Utilizare:
const email = ref('')
const { isValid, firstError } = useValidation(email, [
  {
    name: 'required',
    validate: (v) => v.trim().length > 0,
    message: 'Email-ul este obligatoriu'
  },
  {
    name: 'format',
    validate: (v) => /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(v),
    message: 'Format email invalid'
  }
])
```

### 4.3 Observer Pattern (Event Bus)

**Scopul:** Comunicare loosely-coupled între componente fără relație directă.

> **Notă:** Vue 3 nu mai include Event Bus built-in (Vue 2 avea `$emit` pe instanță).
> Se folosesc composables sau librării externe (`mitt`).

```typescript
// shared/composables/useEventBus.ts
import { onUnmounted } from 'vue'

type EventHandler<T = any> = (payload: T) => void

interface EventBus {
  on<T>(event: string, handler: EventHandler<T>): void
  off<T>(event: string, handler: EventHandler<T>): void
  emit<T>(event: string, payload?: T): void
}

// Singleton la nivel de modul
const listeners = new Map<string, Set<EventHandler>>()

function createEventBus(): EventBus {
  return {
    on<T>(event: string, handler: EventHandler<T>) {
      if (!listeners.has(event)) {
        listeners.set(event, new Set())
      }
      listeners.get(event)!.add(handler)
    },

    off<T>(event: string, handler: EventHandler<T>) {
      listeners.get(event)?.delete(handler)
    },

    emit<T>(event: string, payload?: T) {
      listeners.get(event)?.forEach(handler => handler(payload))
    }
  }
}

const bus = createEventBus()

// Composable cu auto-cleanup
export function useEventBus() {
  const registeredHandlers: Array<{ event: string; handler: EventHandler }> = []

  function on<T>(event: string, handler: EventHandler<T>): void {
    bus.on(event, handler)
    registeredHandlers.push({ event, handler })
  }

  function emit<T>(event: string, payload?: T): void {
    bus.emit(event, payload)
  }

  // Auto-cleanup la unmount
  onUnmounted(() => {
    registeredHandlers.forEach(({ event, handler }) => {
      bus.off(event, handler)
    })
  })

  return { on, emit }
}
```

**Tipizare type-safe cu Event Map:**

```typescript
// types/events.types.ts
interface AppEvents {
  'notification:show': { type: 'success' | 'error'; message: string }
  'cart:updated': { totalItems: number; totalPrice: number }
  'user:logged-in': { userId: string; name: string }
  'user:logged-out': void
  'theme:changed': 'light' | 'dark'
}

// composables/useTypedEventBus.ts
export function useTypedEventBus() {
  const { on: rawOn, emit: rawEmit } = useEventBus()

  function on<K extends keyof AppEvents>(
    event: K,
    handler: (payload: AppEvents[K]) => void
  ): void {
    rawOn(event, handler)
  }

  function emit<K extends keyof AppEvents>(
    event: K,
    payload: AppEvents[K]
  ): void {
    rawEmit(event, payload)
  }

  return { on, emit }
}

// Utilizare:
const { on, emit } = useTypedEventBus()

// Type-safe: TypeScript știe ce payload are fiecare event
on('cart:updated', ({ totalItems, totalPrice }) => {
  console.log(`Cart: ${totalItems} items, ${totalPrice} RON`)
})

emit('notification:show', {
  type: 'success',
  message: 'Produs adăugat!'
})
```

### 4.4 Adapter Pattern (API Adapters)

**Scopul:** Transformă interfața unui sistem extern într-o interfață compatibilă cu aplicația.

```
   Backend API Response          Frontend Model
  ┌─────────────────┐          ┌──────────────────┐
  │ {                │          │ {                 │
  │   product_id: 1, │  ──►    │   id: '1',        │
  │   product_name:  │ Adapter │   name: 'Laptop', │
  │     'Laptop',   │          │   price: 2999,     │
  │   unit_price:   │          │   imageUrl: '...', │
  │     2999.00,    │          │   inStock: true     │
  │   img_url: '...│          │ }                 │
  │   in_stock: 1   │          └──────────────────┘
  │ }                │
  └─────────────────┘
```

```typescript
// features/products/api/product.adapter.ts

// Tipul raw din API (snake_case, structură diferită)
interface ProductApiResponse {
  product_id: number
  product_name: string
  product_description: string | null
  unit_price: number
  original_price: number | null
  img_url: string
  category_info: {
    cat_id: number
    cat_name: string
  }
  in_stock: 0 | 1
  avg_rating: number | null
  review_count: number
  created_at: string
}

// Tipul din frontend (camelCase, structură curată)
interface Product {
  id: string
  name: string
  description: string
  price: number
  originalPrice: number
  imageUrl: string
  category: string
  categoryId: string
  inStock: boolean
  rating: number
  reviewCount: number
  createdAt: Date
  isOnSale: boolean
}

// Adapter: API → Frontend
export function adaptProduct(raw: ProductApiResponse): Product {
  return {
    id: String(raw.product_id),
    name: raw.product_name,
    description: raw.product_description ?? '',
    price: raw.unit_price,
    originalPrice: raw.original_price ?? raw.unit_price,
    imageUrl: raw.img_url,
    category: raw.category_info.cat_name,
    categoryId: String(raw.category_info.cat_id),
    inStock: raw.in_stock === 1,
    rating: raw.avg_rating ?? 0,
    reviewCount: raw.review_count,
    createdAt: new Date(raw.created_at),
    isOnSale: raw.original_price !== null && raw.original_price > raw.unit_price
  }
}

// Adapter: Frontend → API (pentru create/update)
export function adaptProductToApi(product: Partial<Product>): Partial<ProductApiResponse> {
  const result: Record<string, unknown> = {}

  if (product.name !== undefined) result.product_name = product.name
  if (product.description !== undefined) result.product_description = product.description
  if (product.price !== undefined) result.unit_price = product.price
  if (product.imageUrl !== undefined) result.img_url = product.imageUrl

  return result as Partial<ProductApiResponse>
}

export function adaptProducts(raw: ProductApiResponse[]): Product[] {
  return raw.map(adaptProduct)
}
```

**Utilizare în API layer:**

```typescript
// features/products/api/products.api.ts
import { adaptProducts, adaptProduct, adaptProductToApi } from './product.adapter'
import type { Product } from '../types/product.types'
import { httpClient } from '@/plugins/axios'

export const productsApi = {
  async getAll(): Promise<Product[]> {
    const { data } = await httpClient.get('/api/products')
    return adaptProducts(data)  // Adaptare automată
  },

  async getById(id: string): Promise<Product> {
    const { data } = await httpClient.get(`/api/products/${id}`)
    return adaptProduct(data)
  },

  async create(product: Partial<Product>): Promise<Product> {
    const apiData = adaptProductToApi(product)
    const { data } = await httpClient.post('/api/products', apiData)
    return adaptProduct(data)
  }
}
```

### 4.5 Factory Pattern (Component Factories)

**Scopul:** Creează componente sau composables dinamic, bazat pe configurație.

```typescript
// shared/composables/createCrudComposable.ts
// Factory pentru generarea de CRUD composables

interface CrudOptions<T> {
  baseUrl: string
  adapter?: (raw: any) => T
  entityName: string
}

export function createCrudComposable<T extends { id: string }>(
  options: CrudOptions<T>
) {
  const { baseUrl, adapter = (x: any) => x as T, entityName } = options

  return function useCrud() {
    const items = ref<T[]>([]) as Ref<T[]>
    const selectedItem = ref<T | null>(null) as Ref<T | null>
    const loading = ref(false)
    const error = ref<Error | null>(null)

    async function fetchAll(): Promise<void> {
      loading.value = true
      error.value = null
      try {
        const response = await fetch(baseUrl)
        const raw = await response.json()
        items.value = Array.isArray(raw) ? raw.map(adapter) : []
      } catch (e) {
        error.value = new Error(`Eroare la încărcarea ${entityName}`)
      } finally {
        loading.value = false
      }
    }

    async function fetchById(id: string): Promise<T | null> {
      loading.value = true
      try {
        const response = await fetch(`${baseUrl}/${id}`)
        const raw = await response.json()
        const item = adapter(raw)
        selectedItem.value = item
        return item
      } catch (e) {
        error.value = new Error(`Eroare la încărcarea ${entityName} #${id}`)
        return null
      } finally {
        loading.value = false
      }
    }

    async function create(data: Partial<T>): Promise<T | null> {
      loading.value = true
      try {
        const response = await fetch(baseUrl, {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify(data)
        })
        const raw = await response.json()
        const item = adapter(raw)
        items.value.push(item)
        return item
      } catch (e) {
        error.value = new Error(`Eroare la crearea ${entityName}`)
        return null
      } finally {
        loading.value = false
      }
    }

    async function update(id: string, data: Partial<T>): Promise<T | null> {
      loading.value = true
      try {
        const response = await fetch(`${baseUrl}/${id}`, {
          method: 'PUT',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify(data)
        })
        const raw = await response.json()
        const item = adapter(raw)
        const index = items.value.findIndex(i => i.id === id)
        if (index !== -1) items.value[index] = item
        return item
      } catch (e) {
        error.value = new Error(`Eroare la actualizarea ${entityName} #${id}`)
        return null
      } finally {
        loading.value = false
      }
    }

    async function remove(id: string): Promise<boolean> {
      loading.value = true
      try {
        await fetch(`${baseUrl}/${id}`, { method: 'DELETE' })
        items.value = items.value.filter(i => i.id !== id)
        return true
      } catch (e) {
        error.value = new Error(`Eroare la ștergerea ${entityName} #${id}`)
        return false
      } finally {
        loading.value = false
      }
    }

    return {
      items: readonly(items),
      selectedItem: readonly(selectedItem),
      loading: readonly(loading),
      error: readonly(error),
      fetchAll,
      fetchById,
      create,
      update,
      remove
    }
  }
}
```

**Utilizare - generăm composables pentru fiecare entitate:**

```typescript
// features/products/composables/useProducts.ts
import { createCrudComposable } from '@/shared/composables/createCrudComposable'
import { adaptProduct } from '../api/product.adapter'
import type { Product } from '../types/product.types'

export const useProducts = createCrudComposable<Product>({
  baseUrl: '/api/products',
  adapter: adaptProduct,
  entityName: 'produse'
})

// features/auth/composables/useUsers.ts
import { createCrudComposable } from '@/shared/composables/createCrudComposable'
import type { User } from '../types/auth.types'

export const useUsers = createCrudComposable<User>({
  baseUrl: '/api/users',
  entityName: 'utilizatori'
})

// Utilizare în component:
const { items: products, loading, fetchAll, create } = useProducts()
onMounted(() => fetchAll())
```

### 4.6 Provide/Inject Pattern (Dependency Injection în Vue)

**Scopul:** Permite transmiterea dependențelor prin arborele de componente fără props drilling.

```
┌────────────────────────────────┐
│  App.vue                        │
│  provide('theme', themeConfig)  │
│                                 │
│  ┌──────────────────────┐      │
│  │  LayoutComponent      │      │
│  │  (nu știe de theme)   │      │
│  │                       │      │
│  │  ┌────────────────┐  │      │
│  │  │  DeepChild      │  │      │
│  │  │  inject('theme') │  │      │
│  │  │  ✅ Are access  │  │      │
│  │  └────────────────┘  │      │
│  └──────────────────────┘      │
└────────────────────────────────┘
```

```typescript
// shared/composables/useThemeProvider.ts
import { provide, inject, ref, readonly, type InjectionKey, type Ref } from 'vue'

// Chei tipizate pentru type safety
interface ThemeContext {
  theme: Readonly<Ref<'light' | 'dark'>>
  primaryColor: Readonly<Ref<string>>
  toggleTheme: () => void
  setColor: (color: string) => void
}

export const THEME_KEY: InjectionKey<ThemeContext> = Symbol('theme')

// Provider composable (folosit în componenta părinte)
export function useThemeProvider() {
  const theme = ref<'light' | 'dark'>('light')
  const primaryColor = ref('#3b82f6')

  function toggleTheme() {
    theme.value = theme.value === 'light' ? 'dark' : 'light'
  }

  function setColor(color: string) {
    primaryColor.value = color
  }

  const context: ThemeContext = {
    theme: readonly(theme),
    primaryColor: readonly(primaryColor),
    toggleTheme,
    setColor
  }

  provide(THEME_KEY, context)

  return context
}

// Consumer composable (folosit în componente copil)
export function useTheme(): ThemeContext {
  const context = inject(THEME_KEY)

  if (!context) {
    throw new Error(
      'useTheme() trebuie folosit într-un component copil al ThemeProvider'
    )
  }

  return context
}
```

```vue
<!-- App.vue (Provider) -->
<script setup lang="ts">
import { useThemeProvider } from '@/shared/composables/useThemeProvider'

// Providuiește context-ul pentru toți copiii
useThemeProvider()
</script>

<!-- components/DeepChild.vue (Consumer) -->
<script setup lang="ts">
import { useTheme } from '@/shared/composables/useThemeProvider'

// Injectează din orice nivel al arborelui
const { theme, primaryColor, toggleTheme } = useTheme()
</script>

<template>
  <div :class="`theme-${theme}`">
    <button @click="toggleTheme">
      Tema curentă: {{ theme }}
    </button>
  </div>
</template>
```

### 4.7 Paralela cu Angular - Design Patterns

| Pattern | Vue | Angular |
|---------|-----|---------|
| **Facade** | Composable care agregă stores + API | Service care agregă alte services |
| **Strategy** | Composable cu funcții interschimbabile | Service cu strategy interface |
| **Observer** | Event Bus composable / mitt | RxJS Subject / EventEmitter |
| **Adapter** | Funcții pure de transformare | Service / Pipe |
| **Factory** | Funcție care returnează composable | Factory provider / useFactory |
| **DI** | `provide`/`inject` | Constructor injection |
| **Singleton** | Module-level state | `providedIn: 'root'` |
| **Repository** | API layer + adapters | Service cu HttpClient |

---

## 5. Barrel Exports

### 5.1 Ce sunt Barrel Exports

Un **barrel** este un fișier `index.ts` care re-exportă selectiv membrii unui modul/folder, creând o **interfață publică** (public API).

```typescript
// features/products/index.ts (BARREL FILE)

// Componente publice
export { default as ProductList } from './components/ProductList.vue'
export { default as ProductCard } from './components/ProductCard.vue'
export { default as ProductGrid } from './components/ProductGrid.vue'

// Composables publice
export { useProducts } from './composables/useProducts'
export { useProductFilter } from './composables/useProductFilter'

// Store public
export { useProductStore } from './stores/useProductStore'

// Tipuri publice
export type { Product, ProductFilter, ProductCategory } from './types/product.types'

// API public (dacă e necesar)
export { productsApi } from './api/products.api'

// NU exportăm:
// - Componente interne (ProductFilterChip.vue)
// - Helpers interne (formatProductPrice.ts)
// - Constante interne (PRODUCT_SORT_OPTIONS)
// - Adaptere (product.adapter.ts)
```

### 5.2 Beneficii

```
FĂRĂ barrel:                        CU barrel:
import { ProductCard }              import {
  from '../products/                  ProductCard,
    components/ProductCard.vue'       useProducts,
import { useProducts }                Product
  from '../products/                } from '@/features/products'
    composables/useProducts'
import type { Product }             // Un singur import path!
  from '../products/                // Interfață publică clară
    types/product.types'            // Schimbări interne nu afectează
                                    //   consumatorii
```

**Avantaje:**
1. **Încapsulare** - controlezi ce e public și ce e privat
2. **Refactoring safe** - poți restructura intern fără a afecta consumatorii
3. **Import paths scurte** - un singur path per feature
4. **Discoverability** - `index.ts` arată tot API-ul public

### 5.3 Configurare Path Aliases

```typescript
// vite.config.ts
import { defineConfig } from 'vite'
import { resolve } from 'path'

export default defineConfig({
  resolve: {
    alias: {
      '@': resolve(__dirname, 'src'),
      '@features': resolve(__dirname, 'src/features'),
      '@shared': resolve(__dirname, 'src/shared')
    }
  }
})

// tsconfig.json
{
  "compilerOptions": {
    "paths": {
      "@/*": ["./src/*"],
      "@features/*": ["./src/features/*"],
      "@shared/*": ["./src/shared/*"]
    }
  }
}
```

### 5.4 Gotchas - Circular Dependencies

```
PROBLEMĂ:
features/products/index.ts  ──exportă──►  useProducts
       ▲                                        │
       │                                        │
       └──── importă ◄──── features/checkout/   │
                                │                │
                                └── importă ◄────┘
                                     useProducts

SOLUȚIE: Importă direct fișierul, nu barrel-ul
import { useProducts } from '@/features/products/composables/useProducts'
// NU: import { useProducts } from '@/features/products'
```

**Reguli pentru evitarea circular deps:**
1. Features nu importă din barrel-urile altor features (ci direct din fișier)
2. Shared nu importă niciodată din features
3. Folosește ESLint `import/no-cycle` rule
4. Alternativ: shared types/interfaces la granița între features

### 5.5 Auto-import cu unplugin-auto-import

```typescript
// vite.config.ts
import AutoImport from 'unplugin-auto-import/vite'

export default defineConfig({
  plugins: [
    AutoImport({
      imports: ['vue', 'vue-router', 'pinia'],
      dirs: [
        'src/shared/composables',
        'src/shared/utils'
      ],
      dts: 'src/auto-imports.d.ts'
    })
  ]
})

// Acum nu mai trebuie import manual:
// ref, computed, watch, onMounted - automat disponibile
// useLocalStorage, useDebounce - automat din shared/composables
```

### Paralela cu Angular

| Aspect | Vue | Angular |
|--------|-----|---------|
| **Public API** | `index.ts` barrel export | NgModule `exports` array / standalone export |
| **Încapsulare** | Convenție (ce nu e exportat e privat) | Module boundary (declarations vs exports) |
| **Path aliases** | `tsconfig.json` + `vite.config.ts` | `tsconfig.json` |
| **Auto-imports** | `unplugin-auto-import` | Angular Language Service (doar IDE) |
| **Circular deps** | ESLint rules | Angular compiler warning |

---

## 6. Monorepo cu Nx/Turborepo + Vue

### 6.1 De ce Monorepo

```
MONOREPO:                           POLYREPO:
┌─────────────────────────┐        ┌──────┐ ┌──────┐ ┌──────┐
│  Un singur repository    │        │Repo 1│ │Repo 2│ │Repo 3│
│                          │        │App A │ │App B │ │Lib   │
│  apps/app-a              │        └──────┘ └──────┘ └──────┘
│  apps/app-b              │         Separate deploys
│  packages/shared-lib     │         Separate versioning
│                          │         Separate CI/CD
│  Shared deps             │         Duplicated code
│  Atomic commits          │
│  Single CI/CD            │
└─────────────────────────┘
```

**Când Monorepo:**
- Multiple aplicații care share code (MFE, admin + public)
- Design system partajat
- Echipe care lucrează pe features cross-app
- Consistent tooling și linting

**Când Polyrepo:**
- Aplicații complet independente
- Echipe complet independente
- Cicluri de release diferite
- Stack-uri diferite

### 6.2 Nx Workspace cu Vue

```bash
# Creare workspace Nx cu Vue
npx create-nx-workspace@latest my-org --preset=vue

# Structura generată:
my-org/
├── apps/
│   ├── host/                    # Host MFE application
│   │   ├── src/
│   │   │   ├── app/
│   │   │   │   └── App.vue
│   │   │   ├── main.ts
│   │   │   └── styles.css
│   │   ├── project.json         # Nx project config
│   │   ├── vite.config.ts
│   │   └── tsconfig.json
│   │
│   ├── mfe-products/            # Remote MFE - Products
│   │   ├── src/
│   │   │   ├── app/
│   │   │   ├── exposes/         # Module Federation exposes
│   │   │   │   └── Products.vue
│   │   │   └── main.ts
│   │   ├── module-federation.config.ts
│   │   └── project.json
│   │
│   └── mfe-checkout/            # Remote MFE - Checkout
│       ├── src/
│       └── project.json
│
├── packages/                    # Shared libraries
│   ├── ui/                      # Shared component library
│   │   ├── src/
│   │   │   ├── components/
│   │   │   │   ├── BaseButton.vue
│   │   │   │   ├── BaseInput.vue
│   │   │   │   ├── BaseModal.vue
│   │   │   │   └── index.ts
│   │   │   ├── composables/
│   │   │   │   └── index.ts
│   │   │   └── index.ts         # Main barrel export
│   │   ├── package.json
│   │   └── project.json
│   │
│   ├── utils/                   # Shared utilities
│   │   ├── src/
│   │   │   ├── formatters.ts
│   │   │   ├── validators.ts
│   │   │   └── index.ts
│   │   └── package.json
│   │
│   ├── types/                   # Shared TypeScript types
│   │   ├── src/
│   │   │   ├── product.types.ts
│   │   │   ├── user.types.ts
│   │   │   ├── api.types.ts
│   │   │   └── index.ts
│   │   └── package.json
│   │
│   └── api-client/              # Shared API client
│       ├── src/
│       │   ├── client.ts
│       │   ├── interceptors.ts
│       │   └── index.ts
│       └── package.json
│
├── tools/                       # Custom workspace tools
│   └── generators/
│
├── nx.json                      # Nx configuration
├── tsconfig.base.json           # Base TypeScript config
└── package.json
```

### 6.3 Configurare Nx Module Boundaries

```json
// nx.json
{
  "targetDefaults": {
    "build": {
      "dependsOn": ["^build"]
    }
  },
  "plugins": [
    {
      "plugin": "@nx/eslint/plugin",
      "options": {
        "targetName": "lint"
      }
    }
  ]
}
```

```json
// project.json (pentru fiecare proiect)
{
  "name": "mfe-products",
  "tags": ["scope:products", "type:app"]
}

// packages/ui/project.json
{
  "name": "ui",
  "tags": ["scope:shared", "type:lib"]
}
```

```javascript
// .eslintrc.json (la root)
{
  "rules": {
    "@nx/enforce-module-boundaries": [
      "error",
      {
        "allow": [],
        "depConstraints": [
          {
            "sourceTag": "type:app",
            "onlyDependOnLibsWithTags": ["type:lib"]
          },
          {
            "sourceTag": "scope:products",
            "onlyDependOnLibsWithTags": ["scope:products", "scope:shared"]
          },
          {
            "sourceTag": "scope:checkout",
            "onlyDependOnLibsWithTags": ["scope:checkout", "scope:shared"]
          },
          {
            "sourceTag": "scope:shared",
            "onlyDependOnLibsWithTags": ["scope:shared"]
          }
        ]
      }
    ]
  }
}
```

```
REGULI VIZUALE:

  mfe-products ──► packages/ui         ✅
  mfe-products ──► packages/utils      ✅
  mfe-products ──► packages/types      ✅
  mfe-products ──► mfe-checkout        ❌ (app → app interzis)
  packages/ui  ──► mfe-products        ❌ (lib → app interzis)
  packages/ui  ──► packages/types      ✅ (shared → shared OK)
```

### 6.4 Turborepo Setup cu Vue

```json
// turbo.json
{
  "$schema": "https://turbo.build/schema.json",
  "globalDependencies": ["**/.env.*local"],
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**"]
    },
    "lint": {
      "dependsOn": ["^build"]
    },
    "test": {
      "dependsOn": ["^build"]
    },
    "dev": {
      "cache": false,
      "persistent": true
    },
    "type-check": {
      "dependsOn": ["^build"]
    }
  }
}
```

```json
// package.json (root)
{
  "name": "my-monorepo",
  "private": true,
  "workspaces": [
    "apps/*",
    "packages/*"
  ],
  "scripts": {
    "dev": "turbo run dev",
    "build": "turbo run build",
    "lint": "turbo run lint",
    "test": "turbo run test",
    "type-check": "turbo run type-check"
  },
  "devDependencies": {
    "turbo": "^2.0.0"
  }
}
```

```json
// packages/ui/package.json
{
  "name": "@my-org/ui",
  "version": "0.0.0",
  "private": true,
  "main": "./src/index.ts",
  "types": "./src/index.ts",
  "scripts": {
    "build": "vue-tsc --declaration --emitDeclarationOnly",
    "lint": "eslint src/"
  },
  "peerDependencies": {
    "vue": "^3.4.0"
  }
}
```

**Utilizare în aplicații:**

```typescript
// apps/host/src/main.ts
import { BaseButton, BaseInput } from '@my-org/ui'
import { formatCurrency, formatDate } from '@my-org/utils'
import type { Product, User } from '@my-org/types'
```

### 6.5 Nx vs Turborepo

| Aspect | Nx | Turborepo |
|--------|----|-----------|
| **Complexitate** | Mai complex, mai multe features | Mai simplu, focusat pe build |
| **Generatoare** | Da (schematics) | Nu |
| **Module boundaries** | Built-in enforcement | Manual (ESLint) |
| **Cache** | Local + Nx Cloud | Local + Vercel Remote Cache |
| **Affected** | `nx affected` (smart) | `turbo run --filter` |
| **Vue support** | Plugin oficial `@nx/vue` | Agnostic (funcționează cu orice) |
| **Learning curve** | Mai mare | Mai mică |
| **Ideal pentru** | Enterprise, echipe mari | Proiecte medii, simplicitate |

### 6.6 Paralela cu Angular

```
Nx suportă AMBELE framework-uri în ACELAȘI workspace!

monorepo/
├── apps/
│   ├── angular-admin/     # Angular app
│   ├── vue-public/        # Vue app
│   └── vue-mobile/        # Vue app
├── packages/
│   ├── shared-types/      # TypeScript types (framework-agnostic)
│   ├── shared-utils/      # Utility functions (framework-agnostic)
│   ├── ui-angular/        # Angular components
│   └── ui-vue/            # Vue components
└── nx.json
```

> **Punct de interviu:** Într-un monorepo Nx, poți avea Angular și Vue
> side-by-side, partajând types și utils, dar cu UI libraries separate.

---

## 7. Scalarea aplicației - Architecture Decision Records

### 7.1 Ce sunt ADR-urile

**Architecture Decision Records (ADR)** sunt documente scurte care capturează decizii arhitecturale importante, contextul lor și consecințele.

```
docs/
└── adr/
    ├── 001-use-vue3-composition-api.md
    ├── 002-state-management-pinia.md
    ├── 003-feature-based-folder-structure.md
    ├── 004-monorepo-with-nx.md
    ├── 005-api-adapter-pattern.md
    └── template.md
```

**Format ADR:**

```markdown
# ADR-003: Feature-based Folder Structure

## Status
Accepted

## Context
Aplicația a crescut la 15+ features cu 200+ componente.
Structura layer-based actuală face navigarea dificilă
și extragerea de MFE-uri imposibilă.

## Decision
Adoptăm structura feature-based cu:
- Un folder per feature/bounded context
- Barrel exports pentru public API
- Shared folder doar pentru cod cu adevărat partajat
- ESLint boundaries enforcement

## Consequences
### Pozitive
- Navigare ușoară
- Extragere MFE posibilă
- Ownership clar per feature
### Negative
- Migrare necesară din structura veche
- Training echipă pe noile convenții
```

### 7.2 Code Review Guidelines pentru Arhitectură

**Checklist de code review architectural:**

```
ARCHITECTURE REVIEW CHECKLIST:

□ Componentul nou e în feature-ul corect?
□ Composable-ul urmează naming convention (useXxx)?
□ Smart/Dumb separation e respectată?
□ Nu există imports directe cross-feature (fără barrel)?
□ Shared code e generic (nu e specific unui feature)?
□ Tipurile sunt definite (fără 'any' non-justificat)?
□ API calls trec prin adapter (nu raw response în UI)?
□ Barrel export-ul e actualizat pentru members publici noi?
□ Testele unitare acoperă composable-ul nou?
□ Nu există circular dependencies?
```

### 7.3 Dependency Rules - Enforcement

```typescript
// .eslintrc.js - eslint-plugin-boundaries
module.exports = {
  plugins: ['boundaries'],
  settings: {
    'boundaries/elements': [
      { type: 'app', pattern: 'src/app/*' },
      { type: 'features', pattern: 'src/features/*' },
      { type: 'shared', pattern: 'src/shared/*' },
      { type: 'plugins', pattern: 'src/plugins/*' }
    ]
  },
  rules: {
    'boundaries/element-types': [
      'error',
      {
        default: 'disallow',
        rules: [
          // Features pot importa din shared
          {
            from: 'features',
            allow: ['shared']
          },
          // Features NU pot importa din alte features
          {
            from: 'features',
            disallow: ['features'],
            // Excepție: prin barrel export
            except: [{ target: 'features', importKind: 'type' }]
          },
          // Shared NU importă din features
          {
            from: 'shared',
            disallow: ['features', 'app']
          },
          // App importă din orice
          {
            from: 'app',
            allow: ['features', 'shared', 'plugins']
          }
        ]
      }
    ]
  }
}
```

### 7.4 Module Boundaries Visualization

```
    ┌─────────────────────────────────────────────────────────────┐
    │                        APP LAYER                             │
    │    App.vue  │  main.ts  │  router/  │  plugins/              │
    │    Poate importa din: features, shared                       │
    └────────────────────────┬────────────────────────────────────┘
                             │
         ┌───────────────────┼───────────────────┐
         │                   │                   │
         ▼                   ▼                   ▼
    ┌─────────┐       ┌──────────┐       ┌──────────────┐
    │Products │       │  Auth    │       │  Checkout    │
    │Feature  │       │  Feature │       │  Feature     │
    │         │       │          │       │              │
    │ ╔═══╗   │       │ ╔═══╗   │       │ ╔═══╗       │
    │ ║pub║   │       │ ║pub║   │       │ ║pub║       │
    │ ╚═══╝   │       │ ╚═══╝   │       │ ╚═══╝       │
    └────┬────┘       └────┬────┘       └──────┬───────┘
         │                 │                    │
         │    ❌ Nu direct între features ❌     │
         │                 │                    │
         └─────────────────┼────────────────────┘
                           │
                           ▼
    ┌─────────────────────────────────────────────────────────────┐
    │                      SHARED LAYER                            │
    │                                                              │
    │  components/ui   composables   utils   types   constants     │
    │                                                              │
    │  NU importă din features sau app                             │
    └─────────────────────────────────────────────────────────────┘

    ╔═══╗ = Barrel export (index.ts) = Public API
    ╚═══╝
```

### 7.5 Scalarea Progresivă

```
FAZA 1: Monolith mic (1-3 devs)
├── Structură simplă (poți folosi layer-based)
├── Un singur Pinia store
├── Composables în src/composables/
└── Deploy: Single SPA

        │
        ▼ (crește echipa / complexitatea)

FAZA 2: Monolith structurat (3-8 devs)
├── Feature-based structure
├── Pinia stores per feature
├── Barrel exports
├── ESLint boundary rules
└── Deploy: Single SPA cu code splitting

        │
        ▼ (multiple echipe, domenii separate)

FAZA 3: Monorepo (8-20 devs)
├── Nx/Turborepo workspace
├── Shared packages
├── Multiple apps (admin, public, mobile)
├── Module boundary enforcement
└── Deploy: Încă single apps, dar shared code

        │
        ▼ (echipe independente, deploy independent)

FAZA 4: Micro Frontends (20+ devs)
├── Module Federation
├── Independent deployments
├── Shared runtime (Vue, router)
├── Contract testing
└── Deploy: Multiple, independent
```

---

## 8. Paralela completă: Angular Architecture vs Vue Architecture

### 8.1 Tabel Comparativ Complet

| Concept | Angular | Vue 3 |
|---------|---------|-------|
| **Modules** | `NgModule` (sau standalone) | Foldere + barrel exports |
| **Components** | Class-based cu decoratori | `<script setup>` SFC |
| **Services** | `@Injectable` class | Composable function (`useXxx`) |
| **DI** | Ierarhic, built-in | `provide`/`inject` (simplificat) |
| **State management** | NgRx / Signals | Pinia (built-in feel) |
| **Routing** | `@angular/router` | `vue-router` |
| **Lazy loading** | `loadChildren` / `loadComponent` | `() => import()` pe rute |
| **Guards** | `CanActivate`, `CanDeactivate` etc. | `beforeEnter`, navigation guards |
| **Interceptors** | `HttpInterceptor` | Axios interceptors / composables |
| **Resolvers** | `Resolve` guard | `beforeEnter` + composable / loader |
| **Directives** | `@Directive` class | `app.directive()` / `<script>` directive |
| **Pipes** | `@Pipe` class | `computed` / funcții în template |
| **Change Detection** | Zone.js / `OnPush` / Signals | Proxy-based reactivity (automat) |
| **Forms** | Reactive Forms / Template Forms | `v-model` + composables / VeeValidate |
| **HTTP** | `HttpClient` (RxJS-based) | `fetch` / `axios` + composables |
| **Testing** | TestBed + Jasmine/Jest | Vitest + Vue Test Utils |
| **CLI** | `ng` CLI | `create-vue` / Vite |
| **Build** | Webpack / esbuild (Angular 17+) | Vite (esbuild + Rollup) |
| **SSR** | Angular Universal | Nuxt |
| **Static** | Scully / Angular Prerender | Nuxt generate / VitePress |

### 8.2 Guards - Comparație Detaliată

```typescript
// === ANGULAR GUARD ===
// auth.guard.ts
@Injectable({ providedIn: 'root' })
export class AuthGuard implements CanActivate {
  constructor(
    private authService: AuthService,
    private router: Router
  ) {}

  canActivate(
    route: ActivatedRouteSnapshot,
    state: RouterStateSnapshot
  ): boolean | UrlTree {
    if (this.authService.isAuthenticated()) {
      return true
    }
    return this.router.createUrlTree(['/login'], {
      queryParams: { returnUrl: state.url }
    })
  }
}

// În routes:
{
  path: 'dashboard',
  component: DashboardComponent,
  canActivate: [AuthGuard]
}
```

```typescript
// === VUE GUARD ===
// guards/authGuard.ts
import { useAuthStore } from '@/features/auth/stores/useAuthStore'
import type { NavigationGuardWithThis } from 'vue-router'

export const authGuard: NavigationGuardWithThis<undefined> = (to, from, next) => {
  const authStore = useAuthStore()

  if (authStore.isAuthenticated) {
    next()
  } else {
    next({
      name: 'login',
      query: { returnUrl: to.fullPath }
    })
  }
}

// În routes:
{
  path: '/dashboard',
  component: () => import('@/features/dashboard/DashboardPage.vue'),
  beforeEnter: authGuard
}

// SAU global guard:
router.beforeEach((to, from) => {
  const authStore = useAuthStore()

  if (to.meta.requiresAuth && !authStore.isAuthenticated) {
    return { name: 'login', query: { returnUrl: to.fullPath } }
  }
})
```

### 8.3 Interceptors - Comparație Detaliată

```typescript
// === ANGULAR INTERCEPTOR ===
@Injectable()
export class AuthInterceptor implements HttpInterceptor {
  constructor(private authService: AuthService) {}

  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    const token = this.authService.getToken()

    if (token) {
      req = req.clone({
        setHeaders: { Authorization: `Bearer ${token}` }
      })
    }

    return next.handle(req).pipe(
      catchError((error: HttpErrorResponse) => {
        if (error.status === 401) {
          this.authService.logout()
        }
        return throwError(() => error)
      })
    )
  }
}
```

```typescript
// === VUE - AXIOS INTERCEPTOR ===
// plugins/axios.ts
import axios from 'axios'
import { useAuthStore } from '@/features/auth/stores/useAuthStore'
import router from '@/router'

export const httpClient = axios.create({
  baseURL: import.meta.env.VITE_API_URL,
  timeout: 10000,
  headers: { 'Content-Type': 'application/json' }
})

// Request interceptor
httpClient.interceptors.request.use(
  (config) => {
    const authStore = useAuthStore()
    const token = authStore.token

    if (token) {
      config.headers.Authorization = `Bearer ${token}`
    }

    return config
  },
  (error) => Promise.reject(error)
)

// Response interceptor
httpClient.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      const authStore = useAuthStore()
      authStore.logout()
      router.push({ name: 'login' })
    }

    if (error.response?.status === 403) {
      router.push({ name: 'forbidden' })
    }

    return Promise.reject(error)
  }
)
```

### 8.4 Pipes vs Computed/Functions

```typescript
// === ANGULAR PIPE ===
@Pipe({ name: 'formatCurrency', standalone: true })
export class FormatCurrencyPipe implements PipeTransform {
  transform(value: number, currency = 'RON'): string {
    return new Intl.NumberFormat('ro-RO', {
      style: 'currency',
      currency
    }).format(value)
  }
}

// Template: {{ product.price | formatCurrency }}
// Standalone: {{ product.price | formatCurrency:'EUR' }}
```

```typescript
// === VUE - Funcție / Computed ===
// shared/utils/formatters.ts
export function formatCurrency(value: number, currency = 'RON'): string {
  return new Intl.NumberFormat('ro-RO', {
    style: 'currency',
    currency
  }).format(value)
}

// Template: {{ formatCurrency(product.price) }}
// SAU cu computed:
const formattedPrice = computed(() => formatCurrency(product.value.price))
// Template: {{ formattedPrice }}
```

> **Observație:** Vue nu are Pipes. Se folosesc funcții importate sau computed properties.
> Avantaj: funcțiile sunt tree-shakeable și testabile fără framework.
> Dezavantaj: lipsa pipe-ului din template (mai puțin elegant pentru chain-uri).

### 8.5 Rezumatul Diferențelor Arhitecturale

```
ANGULAR                              VUE
═══════                              ═══

Framework COMPLET                    Framework PROGRESIV
(opinionated)                        (un-opinionated)

   │                                    │
   ▼                                    ▼

Everything built-in:                 Core mic, ecosistem:
- Router                             - vue-router
- Forms                              - pinia
- HTTP                               - vite
- Testing                            - vitest
- CLI                                - VeeValidate
- Animations                         - axios
- i18n (lib)                         - vue-i18n

   │                                    │
   ▼                                    ▼

O modalitate "corectă"              Multe modalități valide
(Angular Way)                        (alegi ce vrei)

   │                                    │
   ▼                                    ▼

Curba de învățare                   Curba de învățare
ABRUPTĂ dar COMPLETĂ               LINĂ dar NECESITĂ DECIZII
```

---

## 9. Întrebări de interviu

### Întrebarea 1: Cum structurezi un proiect Vue mare cu 20+ features?

**Răspuns:**
Folosesc o structură **feature-based** (bounded contexts din DDD). Fiecare feature are propriul folder cu `components/`, `composables/`, `stores/`, `types/`, `api/` și un `index.ts` ca barrel export (public API). Codul partajat merge în `shared/` cu subfoldere pentru UI components, composables și utils. Regulile de import sunt stricte: features importă din shared dar nu între ele direct. Enforcement cu ESLint boundaries plugin. Această structură permite extragerea oricărui feature într-un Micro Frontend dacă e necesar, iar echipele pot lucra independent pe features separate fără conflicte.

### Întrebarea 2: Ce sunt composables și cum diferă de Angular services?

**Răspuns:**
Composables sunt funcții care încapsulează logică reactivă reutilizabilă, cu convenția `useXxx()`. Diferențele majore față de Angular services: nu folosesc clase ci funcții, nu au DI (import direct), fiecare apel creează o instanță nouă (nu sunt singleton by default), pot folosi lifecycle hooks (`onMounted`, `onUnmounted`), și au reactivitate built-in cu `ref`/`computed`. Pentru singleton behavior, mut state-ul la nivel de modul (în afara funcției). Testarea e mai simplă: import + apel, fără TestBed. Dezavantajul: lipsa DI-ului ierarhic face scoping-ul mai greu (în Angular poți avea service scoped la component/module).

### Întrebarea 3: Cum asiguri separarea responsabilităților în Vue?

**Răspuns:**
Aplic pattern-ul **Smart/Dumb components**: 70-80% din componente sunt Dumb (primesc props, emit events, zero logic de business, reusable), iar 20-30% sunt Smart (folosesc stores, composables, gestionează state, orchestrează flow-ul). Business logic-ul merge în composables sau Pinia stores, nu în componente. API calls trec prin api layer cu adapters pentru transformarea datelor. Validarea merge în composables dedicate. Styling-ul e encapsulat cu `scoped` CSS. Această separare face testarea unitară mult mai ușoară și componentele Dumb devin reutilizabile instant.

### Întrebarea 4: Cum previi circular dependencies?

**Răspuns:**
Câteva strategii: 1) Reguli stricte de import unidirecționale (features → shared, niciodată invers). 2) Barrel exports cu atenție - când un feature importă din alt feature, importă direct fișierul, nu barrel-ul. 3) Shared types la granița între features (dacă Products și Checkout share un tip, tipul merge în shared/types). 4) ESLint `import/no-cycle` rule în CI. 5) Dacă două features trebuie să comunice, folosesc un Event Bus sau un shared store mai degrabă decât import direct. 6) Nx module boundaries dacă sunt în monorepo. Cele mai periculoase sunt circularitățile indirecte (A → B → C → A), de aceea automatizez detecția.

### Întrebarea 5: Smart vs Dumb components - cum decizi?

**Răspuns:**
Regula mea: dacă componenta accesează store, face API calls, sau are side effects → Smart. Dacă funcționează doar cu props și emite events → Dumb. Concret: fiecare **pagină** (route-level) e Smart, fiecare **card/list/form field** e Dumb. Design system components sunt mereu Dumb. Semn că ceva e greșit: un Dumb component care importă un store, sau un Smart component cu 200 linii de template. Testez Dumb components cu props mock-uite (fast, isolated). Smart components le testez integration-style sau le las pentru E2E. Convenție de naming: `ProductListContainer.vue` (Smart) vs `ProductCard.vue` (Dumb).

### Întrebarea 6: Cum scalezi de la monolith la Micro Frontends?

**Răspuns:**
Scalarea e progresivă: **Faza 1** - monolith cu structura feature-based (pregătire). **Faza 2** - monorepo cu Nx, features devin librării separate, shared code în packages. **Faza 3** - Module Federation cu Webpack/Vite, fiecare feature devine un remote deployable independent. Cerința fundamentală pentru MFE: features trebuie să fie **self-contained** - propriile componente, store, API, types. Comunicarea cross-MFE prin event bus sau shared state minimal. Shared runtime (Vue, Pinia, Router) se încarcă o singură dată de host. Cel mai important: nu faci MFE de la început. Structura feature-based e necesară și suficientă până ai echipe care au nevoie de deploy independent.

### Întrebarea 7: Cum implementezi un design system shared?

**Răspuns:**
Creez un package `@org/ui` în monorepo cu: componente Base* (BaseButton, BaseInput, BaseModal etc.), tokens CSS (variabile pentru culori, spacing, typography), composables pentru UI patterns (useDialog, useToast). Fiecare componentă are props tipizate, events documentate, și Storybook stories. Publicarea: în monorepo e package intern (workspace dependency), cross-repo e npm privat. Versionare semantică strictă. Componentele sunt framework-agnostic unde posibil (CSS tokens) și Vue-specific unde necesar (SFC). Testez vizual cu Chromatic/Storybook. Documentez cu VitePress sau Storybook autodocs.

### Întrebarea 8: Cum gestionezi dependențe între features?

**Răspuns:**
Regula de bază: features NU importă direct una din alta. Dacă Products are nevoie de date din Auth, folosesc una din: 1) **Shared store** - un store minimal accesibil ambelor features. 2) **Event bus** - Products emite event, Auth ascultă. 3) **Shared types** - interfețele comune merg în `shared/types/`. 4) **Props/Events** - componenta părinte (Smart) face bridge-ul. 5) **Route params** - transmit doar ID-uri între features. Dacă două features trebuie să comunice intens, reevaluez: poate ar trebui să fie un singur feature, sau poate trebuie un shared sub-domain.

### Întrebarea 9: Ce design patterns folosești frecvent în Vue?

**Răspuns:**
Cele mai folosite: 1) **Facade** - composable care simplifică interacțiunea cu multiple stores + API-uri. Componentul apelează un singur `useProductFacade()` în loc de 5 imports. 2) **Strategy** - composable cu algoritmi interschimbabili (sorting, filtering, validation rules). 3) **Adapter** - funcții de transformare API response → frontend model și invers, izolând componenta de structura backend-ului. 4) **Factory** - funcții generice care creează composables CRUD tipizate per entitate. 5) **Observer** - Event Bus tipizat pentru comunicare cross-feature. 6) **Provide/Inject** - pentru DI-like behavior (theme, config, auth context).

### Întrebarea 10: Cum faci code splitting eficient?

**Răspuns:**
Trei nivele de splitting: 1) **Route-level** - fiecare rută e lazy-loaded cu `() => import('./ProductsPage.vue')`, Vite generează automat chunk-uri separate. 2) **Component-level** - componente grele folosesc `defineAsyncComponent` cu loading/error states. 3) **Library-level** - librării mari (chart library, rich editor) se importă dinamic. Configurez Vite manual chunks pentru vendor code stabil (vue, pinia). Folosesc `vite-plugin-compression` pentru gzip/brotli. Analizez bundle-ul cu `rollup-plugin-visualizer`. Prefetch hint-uri pentru rute probabile. Regulă: niciun chunk inițial peste 200KB gzipped.

### Întrebarea 11: Monorepo vs polyrepo - când ce?

**Răspuns:**
**Monorepo** când: partajez cod între aplicații (design system, types, utils), am echipe care lucrează cross-app, vreau consistență în tooling/linting, atomic commits sunt necesare (schimb API + consumer în același commit). **Polyrepo** când: aplicațiile sunt complet independente, echipele nu partajează cod, lifecycle-urile de release sunt decuplate, stack-urile sunt diferite. În practică, aleg monorepo cu Nx pentru 80% din cazuri enterprise. Turborepo pentru proiecte mai mici. Polyrepo doar dacă e o aplicație cu adevărat standalone. Atenție: monorepo nu e monolith - fiecare app se deploy-ează independent.

### Întrebarea 12: Cum enforci architecture rules?

**Răspuns:**
Automatizez totul: 1) **ESLint boundaries plugin** - validează reguli de import (features nu importă între ele). 2) **Nx module boundaries** (dacă monorepo) - enforce la build time. 3) **Husky pre-commit hooks** - linting înainte de commit. 4) **CI pipeline** - lint + type-check + test obligatorii. 5) **ADR-uri** - decizii documentate și revizuibile. 6) **PR templates** - checklist architectural. 7) **eslint-plugin-import** - previne circular deps. 8) **Custom ESLint rules** dacă e necesar (ex: enforce naming conventions). Fără automatizare, regulile se degradează. Cu automatizare, nimeni nu poate face merge fără respectarea lor.

### Întrebarea 13: Cum gestionezi state management la scară mare?

**Răspuns:**
Cu Pinia, structurez store-urile per feature: fiecare feature are propriul store (`useProductStore`, `useCartStore`). State-ul global minimal merge într-un store dedicat (`useAppStore` - tema, locale, user curent). Combin store-uri în composables Facade când un component are nevoie de date din multiple surse. Reguli: nu accesez store-uri direct în template (trec prin composable), folosesc `storeToRefs` pentru destructurare reactivă, persistez doar ce e necesar cu `pinia-plugin-persistedstate`. Dacă un store depășește 200 de linii, e semn că trebuie split-uit.

### Întrebarea 14: Provide/Inject vs Pinia - când ce?

**Răspuns:**
**Pinia** pentru state global sau per-feature care trebuie persistent, accesibil din orice componentă, și cu devtools support. **Provide/Inject** pentru state scoped la un subtree de componente: configurare formular (form context), theme overrides, feature flags locale. Exemplu: un `FormProvider` care furnizează validare și submission logic copiilor săi - nu are rost să fie global. Provide/Inject e echivalentul Vue al Angular component-level providers. Regulă: dacă state-ul trebuie accesat din componente care nu sunt copii direcți/indirecți, folosesc Pinia.

### Întrebarea 15: Cum abordezi migrarea de la Angular la Vue?

**Răspuns:**
Abordare incrementală cu **Strangler Fig Pattern**: 1) Setup Vue alături de Angular (Module Federation sau iframe). 2) Features noi în Vue, features vechi rămân în Angular. 3) Migrez features pe rând, de la cele mai independente. 4) Shared types/utils sunt framework-agnostic din start. 5) Design system migrat cu adapter (wrappere Vue peste componente Angular temporar). Nu rescriu tot odată - risc prea mare. Păstrez API layer identic (backend nu se schimbă). Echipa învață Vue pe features noi, apoi migrează cu experiență. Timeline realist: 6-12 luni pentru aplicație medie, features mari pot fi 2-3 sprints fiecare.

### Întrebarea 16: Ce anti-patterns arhitecturale ai întâlnit în proiecte Vue?

**Răspuns:**
Cele mai comune: 1) **God Component** - un component cu 1000+ linii care face totul (soluție: split în Smart + Dumb). 2) **Store-in-template** - accesare directă a store-ului din template fără composable intermediar (soluție: Facade composable). 3) **Prop Drilling** - transmiterea props prin 5 nivele de componente (soluție: provide/inject sau store). 4) **Business Logic în Components** - validări, calcule, transformări în `<script setup>` (soluție: mută în composable). 5) **Fără Adapter Pattern** - componente care știu structura API response-ului (soluție: API adapter layer). 6) **Import spaghetti** - features care importă circular (soluție: barrel exports + ESLint enforcement).

---

> **Notă finală:** Arhitectura Vue la nivel Senior Frontend Architect presupune
> aplicarea acelorași principii SOLID și Clean Architecture ca în Angular,
> dar cu instrumente diferite. Composables înlocuiesc Services, barrel exports
> înlocuiesc NgModules, ESLint boundaries înlocuiesc DI scoping.
> Gândirea arhitecturală rămâne identică - doar implementarea diferă.


---

**Următor :** [**07 - Performanta si Optimizare** →](Vue/07-Performanta-si-Optimizare.md)