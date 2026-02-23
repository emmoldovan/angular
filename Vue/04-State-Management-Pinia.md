# State Management cu Pinia (Interview Prep - Senior Frontend Architect)

> Pinia - state management oficial pentru Vue 3. Setup stores (Composition API style),
> option stores, storeToRefs, store composition, plugins.
> Comparație cu NgRx, Angular Services cu BehaviorSubject.
> Pregătit pentru candidat cu experiență solidă Angular + NgRx.

---

## Cuprins

1. [De ce Pinia (nu Vuex)](#1-de-ce-pinia-nu-vuex)
2. [Setup Stores (Composition API style) - PREFERAT](#2-setup-stores-composition-api-style---preferat)
3. [Option Stores](#3-option-stores)
4. [storeToRefs() - destructurare reactivă](#4-storetorefs---destructurare-reactivă)
5. [Store Composition (stores dependente)](#5-store-composition-stores-dependente)
6. [Getters (computed properties în store)](#6-getters-computed-properties-în-store)
7. [Actions (metode în store)](#7-actions-metode-în-store)
8. [Plugins (persistence, logging, etc.)](#8-plugins-persistence-logging-etc)
9. [Store Design Patterns](#9-store-design-patterns)
10. [Paralela completă: Pinia vs NgRx vs Angular Services](#10-paralela-completă-pinia-vs-ngrx-vs-angular-services)
11. [Pinia în Micro-frontenduri](#11-pinia-în-micro-frontenduri)
12. [Întrebări de interviu](#12-întrebări-de-interviu)

---

## 1. De ce Pinia (nu Vuex)

### Context istoric

Vuex a fost soluția oficială de state management pentru Vue 2 și Vue 3.
**Eduardo San Martin Morote** (core team Vue, creatorul Vue Router) a creat Pinia
ca experiment pentru Vuex 5, dar a devenit atât de popular încât a fost adoptat
ca **soluția oficială recomandată** de echipa Vue.

Pinia = **Vuex 5** (în practică).

### Problemele Vuex

```typescript
// Vuex 4 - mult boilerplate
const store = createStore({
  state: () => ({
    count: 0
  }),

  // Mutations - SYNC ONLY - eliminat în Pinia
  mutations: {
    INCREMENT(state) {
      state.count++
    },
    SET_COUNT(state, payload: number) {
      state.count = payload
    }
  },

  // Actions - pot fi async, dar trebuie să apeleze mutations
  actions: {
    async fetchCount({ commit }) {
      const count = await api.getCount()
      commit('SET_COUNT', count)  // String-based, fără type safety
    },
    incrementAsync({ commit }) {
      setTimeout(() => {
        commit('INCREMENT')  // String magic - erori la runtime
      }, 1000)
    }
  },

  getters: {
    doubleCount: (state) => state.count * 2
  },

  // Modules - nested, complex
  modules: {
    user: userModule,
    cart: cartModule
  }
})
```

```typescript
// Pinia - simplu, direct, type-safe
export const useCounterStore = defineStore('counter', () => {
  const count = ref(0)
  const doubleCount = computed(() => count.value * 2)

  function increment() {
    count.value++
  }

  async function fetchCount() {
    count.value = await api.getCount()
  }

  return { count, doubleCount, increment, fetchCount }
})
```

### Motivele principale pentru Pinia

1. **Fără mutations** - erau redundante, adăugau boilerplate fără beneficiu real
2. **Flat stores** în loc de nested modules - fiecare store e independent
3. **TypeScript nativ** - inferență completă, nu trebuie declarații suplimentare
4. **Composition API nativ** - setup stores = composables cu state global
5. **Extremely lightweight** - ~1.5KB gzipped
6. **DevTools integration** - time travel, edit state, timeline
7. **SSR support** - out of the box
8. **Hot Module Replacement** - state se păstrează la HMR
9. **Plugin system** - extensibil (persistence, sync, logging)

### Tabel comparativ: Vuex 4 vs Pinia

| Aspect | Vuex 4 | Pinia |
|---|---|---|
| **Mutations** | Da (obligatorii, sync) | Nu (eliminat complet) |
| **Actions** | Da (async, commit mutations) | Da (sync + async, directe) |
| **Modules** | Nested modules, namespaced | Flat stores independente |
| **TypeScript** | Parțial (multe type assertions) | Nativ, inferență completă |
| **Composition API** | Prin plugin/workaround | Nativ (setup stores) |
| **Bundle size** | ~5KB gzipped | ~1.5KB gzipped |
| **DevTools** | Da | Da (mai bune) |
| **SSR** | Complex | Simplu, built-in |
| **$reset()** | Manual | Automat (option stores) |
| **Plugin system** | Da | Da (mai simplu) |
| **Hot Module Replacement** | Parțial | Complet |
| **Status** | Legacy / Maintenance | **Oficial, recomandat** |

### Paralela cu Angular

| Concept | Vuex | Pinia | NgRx | Angular Service |
|---|---|---|---|---|
| **State mutation** | mutations (sync) | Direct ref update | Reducers (pure fn) | BehaviorSubject.next() |
| **Side effects** | actions | actions | Effects (RxJS) | Metode async |
| **Computed** | getters | computed() | Selectors | pipe(map()) |
| **Boilerplate** | Mediu | Minimal | Foarte mare | Minimal |

**Key insight**: Pinia e similar ca nivel de complexitate cu **Angular Services + BehaviorSubject**,
nu cu NgRx. Dacă vii din Angular, gândește-te la Pinia stores ca la Injectable services
cu state reactiv.

---

## 2. Setup Stores (Composition API style) - PREFERAT

### Conceptul fundamental

**Setup stores** folosesc o funcție (la fel ca `setup()` din componente sau composables).
Definești state cu `ref()`, getters cu `computed()`, actions ca funcții normale.

Acesta este **pattern-ul recomandat** pentru proiecte noi.

### Structura de bază

```typescript
// stores/useUserStore.ts
import { defineStore } from 'pinia'
import { ref, computed, watch } from 'vue'
import type { User, LoginCredentials, AuthResponse } from '@/types'
import { authApi } from '@/api/auth'

export const useUserStore = defineStore('user', () => {
  // ═══════════════════════════════════════════
  // STATE (echivalent state() în option store)
  // ═══════════════════════════════════════════
  const user = ref<User | null>(null)
  const token = ref<string | null>(null)
  const refreshToken = ref<string | null>(null)
  const loading = ref(false)
  const error = ref<string | null>(null)
  const lastLoginAt = ref<Date | null>(null)

  // ═══════════════════════════════════════════
  // GETTERS (echivalent getters în option store)
  // ═══════════════════════════════════════════
  const isAuthenticated = computed(() => !!user.value && !!token.value)

  const displayName = computed(() => {
    if (!user.value) return 'Guest'
    return `${user.value.firstName} ${user.value.lastName}`
  })

  const isAdmin = computed(() => user.value?.role === 'admin')

  const isPremium = computed(() =>
    user.value?.subscription === 'premium' || isAdmin.value
  )

  const initials = computed(() => {
    if (!user.value) return '?'
    return `${user.value.firstName[0]}${user.value.lastName[0]}`.toUpperCase()
  })

  // ═══════════════════════════════════════════
  // ACTIONS (echivalent actions în option store)
  // ═══════════════════════════════════════════
  async function login(credentials: LoginCredentials): Promise<void> {
    loading.value = true
    error.value = null

    try {
      const response: AuthResponse = await authApi.login(credentials)
      user.value = response.user
      token.value = response.token
      refreshToken.value = response.refreshToken
      lastLoginAt.value = new Date()
    } catch (e) {
      error.value = e instanceof Error ? e.message : 'Login failed'
      throw e  // Re-throw pentru a permite componente să reacționeze
    } finally {
      loading.value = false
    }
  }

  async function logout(): Promise<void> {
    try {
      if (token.value) {
        await authApi.logout(token.value)
      }
    } catch {
      // Logout local chiar dacă API-ul eșuează
      console.warn('Logout API call failed, clearing local state')
    } finally {
      $reset()
    }
  }

  async function refreshSession(): Promise<void> {
    if (!refreshToken.value) {
      throw new Error('No refresh token available')
    }

    try {
      const response = await authApi.refresh(refreshToken.value)
      token.value = response.token
      refreshToken.value = response.refreshToken
    } catch (e) {
      // Refresh failed - force logout
      $reset()
      throw e
    }
  }

  async function updateProfile(data: Partial<User>): Promise<void> {
    if (!user.value) throw new Error('Not authenticated')

    loading.value = true
    error.value = null

    try {
      const updated = await authApi.updateProfile(user.value.id, data)
      user.value = { ...user.value, ...updated }
    } catch (e) {
      error.value = e instanceof Error ? e.message : 'Update failed'
      throw e
    } finally {
      loading.value = false
    }
  }

  // $reset - în setup stores trebuie definit manual
  // (option stores îl au automat)
  function $reset(): void {
    user.value = null
    token.value = null
    refreshToken.value = null
    loading.value = false
    error.value = null
    lastLoginAt.value = null
  }

  // ═══════════════════════════════════════════
  // WATCHERS (bonus - nu există în option stores)
  // ═══════════════════════════════════════════
  watch(token, (newToken) => {
    if (newToken) {
      // Setează header-ul Authorization pentru API calls
      authApi.setAuthHeader(newToken)
    } else {
      authApi.clearAuthHeader()
    }
  })

  // ═══════════════════════════════════════════
  // RETURN - IMPORTANT: tot ce returnezi e public
  // Ce nu returnezi rămâne privat store-ului
  // ═══════════════════════════════════════════
  return {
    // State
    user,
    token,
    loading,
    error,
    lastLoginAt,
    // refreshToken NU e returnat - rămâne privat!

    // Getters
    isAuthenticated,
    displayName,
    isAdmin,
    isPremium,
    initials,

    // Actions
    login,
    logout,
    refreshSession,
    updateProfile,
    $reset
  }
})
```

### Utilizarea în componente

```vue
<!-- components/UserProfile.vue -->
<script setup lang="ts">
import { useUserStore } from '@/stores/useUserStore'
import { storeToRefs } from 'pinia'
import { ref as vueRef } from 'vue'

const userStore = useUserStore()

// ✅ CORECT: storeToRefs pentru state și getters (păstrează reactivitatea)
const {
  user,
  isAuthenticated,
  displayName,
  isAdmin,
  loading,
  error
} = storeToRefs(userStore)

// ✅ CORECT: destructurare directă pentru actions (funcții, nu valori reactive)
const { login, logout, updateProfile } = userStore

// Local state
const editMode = vueRef(false)
const formData = vueRef({ firstName: '', lastName: '' })

function startEdit() {
  if (user.value) {
    formData.value = {
      firstName: user.value.firstName,
      lastName: user.value.lastName
    }
    editMode.value = true
  }
}

async function saveProfile() {
  try {
    await updateProfile(formData.value)
    editMode.value = false
  } catch {
    // error e deja setat în store
  }
}

async function handleLogin(credentials: LoginCredentials) {
  try {
    await login(credentials)
    // Redirect sau altă logică
  } catch {
    // error e deja setat în store
  }
}
</script>

<template>
  <div class="user-profile">
    <!-- Loading State -->
    <div v-if="loading" class="loading-spinner">
      Se încarcă...
    </div>

    <!-- Error State -->
    <div v-if="error" class="error-banner" role="alert">
      {{ error }}
    </div>

    <!-- Authenticated State -->
    <template v-if="isAuthenticated">
      <div class="profile-header">
        <h2>{{ displayName }}</h2>
        <span v-if="isAdmin" class="badge badge-admin">Admin</span>
      </div>

      <!-- View Mode -->
      <div v-if="!editMode" class="profile-view">
        <p><strong>Email:</strong> {{ user?.email }}</p>
        <p><strong>Rol:</strong> {{ user?.role }}</p>
        <button @click="startEdit" :disabled="loading">
          Editează profil
        </button>
        <button @click="logout" :disabled="loading">
          Logout
        </button>
      </div>

      <!-- Edit Mode -->
      <form v-else @submit.prevent="saveProfile" class="profile-edit">
        <label>
          Prenume:
          <input v-model="formData.firstName" required />
        </label>
        <label>
          Nume:
          <input v-model="formData.lastName" required />
        </label>
        <button type="submit" :disabled="loading">Salvează</button>
        <button type="button" @click="editMode = false">Anulează</button>
      </form>
    </template>

    <!-- Unauthenticated State -->
    <LoginForm v-else @submit="handleLogin" />
  </div>
</template>
```

### Private state (avantaj setup stores)

```typescript
export const useAuthStore = defineStore('auth', () => {
  // PUBLIC - returnat
  const user = ref<User | null>(null)
  const isAuthenticated = computed(() => !!user.value)

  // PRIVAT - NU returnat, nu e accesibil din afara store-ului
  const _refreshTimer = ref<ReturnType<typeof setInterval> | null>(null)
  const _retryCount = ref(0)
  const _maxRetries = 3

  function _startAutoRefresh() {
    _stopAutoRefresh()
    _refreshTimer.value = setInterval(() => {
      _refreshToken()
    }, 15 * 60 * 1000) // 15 minute
  }

  function _stopAutoRefresh() {
    if (_refreshTimer.value) {
      clearInterval(_refreshTimer.value)
      _refreshTimer.value = null
    }
  }

  async function _refreshToken() {
    // Logică internă de refresh
    _retryCount.value++
    if (_retryCount.value >= _maxRetries) {
      _stopAutoRefresh()
      await logout()
    }
  }

  async function login(credentials: LoginCredentials) {
    // ... login logic
    _retryCount.value = 0
    _startAutoRefresh()
  }

  async function logout() {
    _stopAutoRefresh()
    user.value = null
  }

  // Doar ce returnezi e public
  return {
    user,
    isAuthenticated,
    login,
    logout
    // _refreshTimer, _retryCount, etc. sunt PRIVATE
  }
})
```

### Paralela cu Angular

```typescript
// Angular Service - echivalentul setup store-ului
@Injectable({ providedIn: 'root' })
export class UserService {
  // State = BehaviorSubject (privat) + Observable (public)
  private _user = new BehaviorSubject<User | null>(null)
  private _loading = new BehaviorSubject<boolean>(false)
  private _error = new BehaviorSubject<string | null>(null)

  // Getters = Observable cu pipe()
  readonly user$ = this._user.asObservable()
  readonly loading$ = this._loading.asObservable()
  readonly error$ = this._error.asObservable()

  readonly isAuthenticated$ = this._user.pipe(
    map(user => !!user)
  )

  readonly displayName$ = this._user.pipe(
    map(user => user ? `${user.firstName} ${user.lastName}` : 'Guest')
  )

  // Action = metodă
  async login(credentials: LoginCredentials): Promise<void> {
    this._loading.next(true)
    this._error.next(null)
    try {
      const response = await firstValueFrom(this.http.post<AuthResponse>('/api/login', credentials))
      this._user.next(response.user)
    } catch (e) {
      this._error.next(e.message)
      throw e
    } finally {
      this._loading.next(false)
    }
  }
}
```

**Diferențe cheie**:

| Aspect | Pinia Setup Store | Angular Service |
|---|---|---|
| **State** | `ref()` - reactiv automat | `BehaviorSubject` - manual subscribe |
| **Private state** | Nu returna din setup | `private` keyword |
| **Computed** | `computed()` - auto-cached | `pipe(map())` - manual |
| **Template binding** | Direct (cu storeToRefs) | `async` pipe sau subscribe |
| **Cleanup** | Automat (Vue reactivity) | Manual unsubscribe / takeUntilDestroyed |
| **DevTools** | Built-in Pinia DevTools | Nicio soluție standard |
| **$reset** | Manual în setup stores | Manual |

---

## 3. Option Stores

### Structura

Option stores folosesc un obiect cu `state`, `getters`, și `actions` - similar cu
Options API din componente sau cu Vuex store modules.

```typescript
// stores/useProductStore.ts
import { defineStore } from 'pinia'
import type { Product, ProductFilter, SortOption } from '@/types'
import { productApi } from '@/api/products'

interface ProductState {
  products: Product[]
  selectedProduct: Product | null
  loading: boolean
  error: string | null
  filter: ProductFilter
  sortBy: SortOption
  currentPage: number
  totalPages: number
  pageSize: number
}

export const useProductStore = defineStore('product', {
  // STATE - trebuie să fie funcție (ca data() din Options API)
  state: (): ProductState => ({
    products: [],
    selectedProduct: null,
    loading: false,
    error: null,
    filter: {
      category: null,
      minPrice: 0,
      maxPrice: Infinity,
      searchQuery: ''
    },
    sortBy: 'name-asc',
    currentPage: 1,
    totalPages: 0,
    pageSize: 20
  }),

  // GETTERS - computed properties pe baza state-ului
  getters: {
    // Getter simplu
    productCount: (state) => state.products.length,

    // Getter cu logică complexă
    filteredProducts: (state): Product[] => {
      let result = [...state.products]

      if (state.filter.category) {
        result = result.filter(p => p.category === state.filter.category)
      }
      if (state.filter.searchQuery) {
        const query = state.filter.searchQuery.toLowerCase()
        result = result.filter(p =>
          p.name.toLowerCase().includes(query) ||
          p.description.toLowerCase().includes(query)
        )
      }
      if (state.filter.minPrice > 0) {
        result = result.filter(p => p.price >= state.filter.minPrice)
      }
      if (state.filter.maxPrice < Infinity) {
        result = result.filter(p => p.price <= state.filter.maxPrice)
      }

      return result
    },

    // Getter care accesează alt getter - folosește `this`
    filteredProductCount(): number {
      return this.filteredProducts.length
    },

    // Getter cu parametru (returnează funcție)
    getProductById: (state) => {
      return (id: string): Product | undefined => {
        return state.products.find(p => p.id === id)
      }
    },

    // Getter care depinde de alt store
    productsWithDiscount(): Product[] {
      const userStore = useUserStore()
      const discount = userStore.isPremium ? 0.1 : 0
      return this.filteredProducts.map(p => ({
        ...p,
        finalPrice: p.price * (1 - discount)
      }))
    },

    // Pagination getters
    paginatedProducts(): Product[] {
      const start = (this.currentPage - 1) * this.pageSize
      return this.filteredProducts.slice(start, start + this.pageSize)
    },

    hasNextPage: (state) => state.currentPage < state.totalPages,
    hasPrevPage: (state) => state.currentPage > 1
  },

  // ACTIONS - metode (sync + async)
  actions: {
    // Async action - fetch data
    async fetchProducts(): Promise<void> {
      this.loading = true
      this.error = null

      try {
        const response = await productApi.getAll({
          page: this.currentPage,
          pageSize: this.pageSize,
          sort: this.sortBy
        })
        this.products = response.data
        this.totalPages = response.totalPages
      } catch (e) {
        this.error = e instanceof Error ? e.message : 'Failed to fetch products'
      } finally {
        this.loading = false
      }
    },

    // Sync action
    setFilter(filter: Partial<ProductFilter>): void {
      this.filter = { ...this.filter, ...filter }
      this.currentPage = 1  // Reset la pagina 1 la schimbarea filtrului
    },

    // Action care apelează alt action
    async selectProduct(id: string): Promise<void> {
      const existing = this.getProductById(id)
      if (existing) {
        this.selectedProduct = existing
        return
      }

      // Dacă nu e în lista curentă, fetch individual
      this.loading = true
      try {
        this.selectedProduct = await productApi.getById(id)
      } catch (e) {
        this.error = e instanceof Error ? e.message : 'Product not found'
      } finally {
        this.loading = false
      }
    },

    // Pagination actions
    async goToPage(page: number): Promise<void> {
      if (page < 1 || page > this.totalPages) return
      this.currentPage = page
      await this.fetchProducts()
    },

    async nextPage(): Promise<void> {
      if (this.hasNextPage) {
        await this.goToPage(this.currentPage + 1)
      }
    },

    async prevPage(): Promise<void> {
      if (this.hasPrevPage) {
        await this.goToPage(this.currentPage - 1)
      }
    },

    setSortBy(sort: SortOption): void {
      this.sortBy = sort
      this.currentPage = 1
    }
  }
})
```

### $reset() în Option Stores - funcționează automat

```typescript
const productStore = useProductStore()

// Resetează state-ul la valorile inițiale din state()
productStore.$reset()
// products → [], selectedProduct → null, loading → false, etc.
```

**ATENȚIE**: `$reset()` **NU funcționează automat** în setup stores!
Trebuie definit manual (vezi secțiunea 2).

### Când să folosești Option Store vs Setup Store

| Criteriu | Setup Store | Option Store |
|---|---|---|
| **Composition API fans** | ✅ Identic cu composables | ❌ Stil diferit |
| **Private state** | ✅ Nu returnezi | ❌ Totul e public |
| **Watchers în store** | ✅ watch() direct | ❌ Nu suportă |
| **$reset() automat** | ❌ Manual | ✅ Automat |
| **Composable reuse** | ✅ Import composables | ⚠️ Posibil dar awkward |
| **Familiar Vuex devs** | ❌ Stil nou | ✅ Similar Vuex |
| **Complex logic** | ✅ Mai flexibil | ⚠️ Limitat la structură |
| **Recomandat oficial** | ✅ **Da** | ✅ Da (dar setup preferat) |

**Recomandare**: Folosește **setup stores** pentru proiecte noi. Option stores sunt OK
pentru echipe care vin de la Vuex și vor o tranziție ușoară.

### Paralela cu Angular

- **Option Store** ~ **NgRx** (structură fixă: state/getters/actions ↔ state/selectors/reducers+effects)
- **Setup Store** ~ **Angular Service** (libertate totală în structurare)

---

## 4. storeToRefs() - destructurare reactivă

### Problema

Când destructurezi un store Pinia, **state-ul și getters pierd reactivitatea**:

```typescript
const store = useUserStore()

// ❌ GREȘIT - pierde reactivitatea!
const { user, isAuthenticated, displayName } = store

// user nu se mai actualizează când store-ul se schimbă!
console.log(user)  // { name: 'John' }
store.login(otherCredentials)
console.log(user)  // Tot { name: 'John' } - STALE!
```

### Soluția: storeToRefs()

```typescript
import { storeToRefs } from 'pinia'

const store = useUserStore()

// ✅ CORECT - storeToRefs convertește state și getters în Ref-uri
const { user, isAuthenticated, displayName } = storeToRefs(store)

// Acum sunt reactive!
console.log(user.value)  // { name: 'John' }
store.login(otherCredentials)
console.log(user.value)  // { name: 'Jane' } - REACTIV!
```

### Regula de aur

```typescript
const store = useUserStore()

// STATE + GETTERS → storeToRefs()
const { user, token, loading, error, isAuthenticated, displayName } = storeToRefs(store)

// ACTIONS → destructurare directă (funcțiile NU sunt reactive)
const { login, logout, updateProfile } = store
```

### Ce face storeToRefs() tehnic

```typescript
// Simplificat - ce face storeToRefs() intern
function storeToRefs(store) {
  const refs = {}

  for (const key in store) {
    const value = store[key]
    // Doar ref-uri și computed-uri (nu funcții/actions)
    if (isRef(value) || isComputed(value)) {
      refs[key] = toRef(store, key)
    }
  }

  return refs
}
```

### storeToRefs() vs toRefs()

```typescript
import { toRefs } from 'vue'
import { storeToRefs } from 'pinia'

const store = useUserStore()

// toRefs() din Vue - funcționează DAR include și metode interne Pinia
const allRefs = toRefs(store)  // Include $id, $state, $patch, etc.

// storeToRefs() din Pinia - filtrează doar state și getters
const cleanRefs = storeToRefs(store)  // Doar state + getters, curat
```

**Întotdeauna folosește `storeToRefs()` din Pinia, nu `toRefs()` din Vue.**

### Pattern complet în componentă

```vue
<script setup lang="ts">
import { useNotificationStore } from '@/stores/useNotificationStore'
import { storeToRefs } from 'pinia'

const notifStore = useNotificationStore()

// Reactive refs
const {
  notifications,    // state
  unreadCount,      // getter
  hasUnread,        // getter
  loading           // state
} = storeToRefs(notifStore)

// Non-reactive functions
const {
  fetchNotifications,
  markAsRead,
  markAllAsRead,
  dismissNotification
} = notifStore

// Fetch la mount
fetchNotifications()
</script>

<template>
  <div class="notifications">
    <header>
      <h3>Notificări ({{ unreadCount }})</h3>
      <button
        v-if="hasUnread"
        @click="markAllAsRead"
        :disabled="loading"
      >
        Marchează toate ca citite
      </button>
    </header>

    <ul>
      <li
        v-for="notif in notifications"
        :key="notif.id"
        :class="{ unread: !notif.read }"
      >
        <span>{{ notif.message }}</span>
        <button @click="markAsRead(notif.id)">✓</button>
        <button @click="dismissNotification(notif.id)">×</button>
      </li>
    </ul>
  </div>
</template>
```

### Paralela cu Angular

| Concept | Vue (Pinia) | Angular |
|---|---|---|
| **Store access** | `useStore()` | `inject(Service)` |
| **Reactive state** | `storeToRefs()` | `service.state$` (Observable) |
| **Lost reactivity** | Destructurare fără storeToRefs | `subscribe()` fără `async` pipe |
| **Template binding** | Direct (ref.value auto-unwrap) | `async` pipe sau signal() |
| **Cleanup** | Automat | `takeUntilDestroyed()` sau manual unsubscribe |

În Angular cu NgRx:
```typescript
// NgRx - selectori returnează Observable, mereu reactiv
this.user$ = this.store.select(selectUser)
this.isAuthenticated$ = this.store.select(selectIsAuthenticated)
// Template: {{ user$ | async }}
```

În Angular nu ai problema „pierderii reactivității" la destructurare, deoarece
Observable-urile sunt mereu lazy și reactive. Dar ai problema **memory leaks** dacă nu
faci unsubscribe - ceea ce Pinia + storeToRefs() rezolvă automat.

---

## 5. Store Composition (stores dependente)

### Conceptul

Store composition = un store care **folosește alt store**. Pinia permite acest lucru
simplu - importezi și apelezi `useOtherStore()` direct în setup function.

### Exemplu: Cart Store care depinde de User Store și Product Store

```typescript
// stores/useCartStore.ts
import { defineStore } from 'pinia'
import { ref, computed, watch } from 'vue'
import { useUserStore } from './useUserStore'
import { useProductStore } from './useProductStore'
import type { CartItem, CheckoutResult } from '@/types'
import { cartApi } from '@/api/cart'

export const useCartStore = defineStore('cart', () => {
  // ═══════════════════════════════════════════
  // DEPENDENT STORES
  // ═══════════════════════════════════════════
  const userStore = useUserStore()
  const productStore = useProductStore()

  // ═══════════════════════════════════════════
  // STATE
  // ═══════════════════════════════════════════
  const items = ref<CartItem[]>([])
  const loading = ref(false)
  const error = ref<string | null>(null)
  const couponCode = ref<string | null>(null)
  const couponDiscount = ref(0)

  // ═══════════════════════════════════════════
  // GETTERS (folosesc state local + state din alte stores)
  // ═══════════════════════════════════════════
  const itemCount = computed(() =>
    items.value.reduce((sum, item) => sum + item.quantity, 0)
  )

  const subtotal = computed(() =>
    items.value.reduce((sum, item) => sum + item.price * item.quantity, 0)
  )

  // Discount bazat pe user role (din userStore)
  const memberDiscount = computed(() => {
    if (userStore.isAdmin) return 0.20   // 20% admin
    if (userStore.isPremium) return 0.10 // 10% premium
    return 0
  })

  const totalDiscount = computed(() =>
    Math.max(memberDiscount.value, couponDiscount.value)
  )

  const total = computed(() =>
    subtotal.value * (1 - totalDiscount.value)
  )

  const isEmpty = computed(() => items.value.length === 0)

  // ═══════════════════════════════════════════
  // ACTIONS
  // ═══════════════════════════════════════════
  function addItem(productId: string, quantity: number = 1): void {
    // Verifică dacă produsul există (din productStore)
    const product = productStore.getProductById(productId)
    if (!product) {
      error.value = 'Product not found'
      return
    }

    const existingItem = items.value.find(item => item.productId === productId)

    if (existingItem) {
      existingItem.quantity += quantity
    } else {
      items.value.push({
        productId,
        name: product.name,
        price: product.price,
        quantity,
        image: product.thumbnail
      })
    }
  }

  function removeItem(productId: string): void {
    items.value = items.value.filter(item => item.productId !== productId)
  }

  function updateQuantity(productId: string, quantity: number): void {
    if (quantity <= 0) {
      removeItem(productId)
      return
    }
    const item = items.value.find(i => i.productId === productId)
    if (item) {
      item.quantity = quantity
    }
  }

  async function applyCoupon(code: string): Promise<boolean> {
    try {
      const result = await cartApi.validateCoupon(code)
      if (result.valid) {
        couponCode.value = code
        couponDiscount.value = result.discount
        return true
      }
      error.value = 'Cupon invalid'
      return false
    } catch {
      error.value = 'Eroare la validarea cuponului'
      return false
    }
  }

  async function checkout(): Promise<CheckoutResult> {
    // Verificare autentificare (din userStore)
    if (!userStore.isAuthenticated) {
      throw new Error('Trebuie să fii autentificat pentru a finaliza comanda')
    }

    if (isEmpty.value) {
      throw new Error('Coșul este gol')
    }

    loading.value = true
    error.value = null

    try {
      const result = await cartApi.checkout({
        items: items.value,
        couponCode: couponCode.value,
        userId: userStore.user!.id
      })

      // Golește coșul după checkout reușit
      $reset()

      return result
    } catch (e) {
      error.value = e instanceof Error ? e.message : 'Checkout failed'
      throw e
    } finally {
      loading.value = false
    }
  }

  function $reset(): void {
    items.value = []
    loading.value = false
    error.value = null
    couponCode.value = null
    couponDiscount.value = 0
  }

  // ═══════════════════════════════════════════
  // WATCHERS
  // ═══════════════════════════════════════════
  // Golește coșul când user-ul face logout
  watch(
    () => userStore.isAuthenticated,
    (isAuth) => {
      if (!isAuth) {
        $reset()
      }
    }
  )

  return {
    items,
    loading,
    error,
    couponCode,
    itemCount,
    subtotal,
    memberDiscount,
    totalDiscount,
    total,
    isEmpty,
    addItem,
    removeItem,
    updateQuantity,
    applyCoupon,
    checkout,
    $reset
  }
})
```

### Atenție la dependențe circulare

```typescript
// ❌ GREȘIT - dependență circulară la nivel de modul
// storeA.ts
import { useStoreB } from './storeB'  // storeB importă storeA → circular!

export const useStoreA = defineStore('a', () => {
  const storeB = useStoreB()  // Eroare la init!
})

// ✅ CORECT - apelează useStore() ÎNĂUNTRUL acțiunilor, nu la top-level
export const useStoreA = defineStore('a', () => {
  const data = ref('')

  function doSomething() {
    // Apelează useStoreB() doar când funcția e executată,
    // nu la definirea store-ului
    const storeB = useStoreB()
    storeB.someAction()
  }

  return { data, doSomething }
})
```

### Pattern: Orchestrator Store

```typescript
// stores/useCheckoutStore.ts - orchestrează mai multe stores
export const useCheckoutStore = defineStore('checkout', () => {
  const userStore = useUserStore()
  const cartStore = useCartStore()
  const orderStore = useOrderStore()
  const notifStore = useNotificationStore()

  const step = ref<'cart' | 'shipping' | 'payment' | 'confirmation'>('cart')
  const processing = ref(false)

  async function processCheckout(paymentDetails: PaymentDetails) {
    processing.value = true

    try {
      // 1. Validare user
      if (!userStore.isAuthenticated) {
        throw new Error('Not authenticated')
      }

      // 2. Checkout din cart
      const checkoutResult = await cartStore.checkout()

      // 3. Crează order
      const order = await orderStore.createOrder({
        ...checkoutResult,
        payment: paymentDetails,
        userId: userStore.user!.id
      })

      // 4. Notificare
      notifStore.addNotification({
        type: 'success',
        message: `Comanda #${order.id} a fost plasată cu succes!`
      })

      step.value = 'confirmation'
      return order
    } catch (e) {
      notifStore.addNotification({
        type: 'error',
        message: 'Eroare la procesarea comenzii'
      })
      throw e
    } finally {
      processing.value = false
    }
  }

  return { step, processing, processCheckout }
})
```

### Paralela cu Angular

| Concept | Pinia | Angular |
|---|---|---|
| **Store composition** | `useOtherStore()` în setup | `inject(OtherService)` în constructor |
| **Circular deps** | Lazy call în actions | Angular DI rezolvă (dar evitați) |
| **Orchestrator** | Store care importă alte stores | Facade Service |
| **NgRx equivalent** | N/A | Effects care dispatch alte actions |

```typescript
// Angular - Facade Service (echivalent orchestrator store)
@Injectable({ providedIn: 'root' })
export class CheckoutFacade {
  constructor(
    private userService: UserService,
    private cartService: CartService,
    private orderService: OrderService,
    private notifService: NotificationService
  ) {}

  async processCheckout(payment: PaymentDetails): Promise<Order> {
    // Orchestrează servicii multiple - identic conceptual
  }
}
```

---

## 6. Getters (computed properties în store)

### Getters în Setup Stores

```typescript
export const useAnalyticsStore = defineStore('analytics', () => {
  const events = ref<AnalyticsEvent[]>([])
  const dateRange = ref<{ start: Date; end: Date }>({
    start: new Date(Date.now() - 30 * 24 * 60 * 60 * 1000),
    end: new Date()
  })

  // Getter simplu
  const totalEvents = computed(() => events.value.length)

  // Getter cu filtrare
  const filteredEvents = computed(() =>
    events.value.filter(e =>
      e.timestamp >= dateRange.value.start &&
      e.timestamp <= dateRange.value.end
    )
  )

  // Getter derivat din alt getter
  const eventsByType = computed(() => {
    const grouped: Record<string, AnalyticsEvent[]> = {}
    for (const event of filteredEvents.value) {
      if (!grouped[event.type]) {
        grouped[event.type] = []
      }
      grouped[event.type].push(event)
    }
    return grouped
  })

  // Getter cu calcul complex (memoizat automat de computed)
  const statistics = computed(() => ({
    total: filteredEvents.value.length,
    byType: eventsByType.value,
    avgPerDay: filteredEvents.value.length /
      Math.max(1, Math.ceil(
        (dateRange.value.end.getTime() - dateRange.value.start.getTime()) /
        (24 * 60 * 60 * 1000)
      )),
    topEvent: Object.entries(eventsByType.value)
      .sort(([, a], [, b]) => b.length - a.length)[0]?.[0] ?? 'none'
  }))

  // Getter "factory" - returnează funcție (NU e cached per argument!)
  const getEventsByType = computed(() => {
    return (type: string) => events.value.filter(e => e.type === type)
  })

  return {
    events,
    dateRange,
    totalEvents,
    filteredEvents,
    eventsByType,
    statistics,
    getEventsByType
  }
})
```

### Getters cu argumente - pattern avansat

```typescript
// Getter cu argumente (factory pattern)
// ATENȚIE: funcția returnată NU e cached per argument
const getProductById = computed(() => {
  return (id: string) => products.value.find(p => p.id === id)
})

// Utilizare
const product = store.getProductById('abc-123')
```

```typescript
// Pentru caching per argument, folosește o Map internă
const _productCache = computed(() => {
  const map = new Map<string, Product>()
  for (const product of products.value) {
    map.set(product.id, product)
  }
  return map
})

const getProductById = computed(() => {
  return (id: string) => _productCache.value.get(id)
})
// Acum lookup-ul e O(1) și cache-ul se invalidează doar când products se schimbă
```

### Paralela cu Angular

```typescript
// NgRx Selectors - echivalentul getters
// selectors/product.selectors.ts
export const selectProducts = (state: AppState) => state.products.items

export const selectFilteredProducts = createSelector(
  selectProducts,
  selectActiveFilter,
  (products, filter) => products.filter(p => matchesFilter(p, filter))
)

export const selectProductById = (id: string) => createSelector(
  selectProducts,
  (products) => products.find(p => p.id === id)
)

// Utilizare în componentă
this.products$ = this.store.select(selectFilteredProducts)
```

| Aspect | Pinia Getters | NgRx Selectors |
|---|---|---|
| **Definire** | `computed()` | `createSelector()` |
| **Memoizare** | Automat (Vue computed) | Automat (reselect) |
| **Compoziție** | Computed din computed | Selector din selectors |
| **Cu argumente** | Factory function | Factory selector |
| **Tipare** | Inferență automată | Explicit typing necesar |

---

## 7. Actions (metode în store)

### Sync Actions

```typescript
export const useUIStore = defineStore('ui', () => {
  const sidebarOpen = ref(true)
  const theme = ref<'light' | 'dark'>('light')
  const locale = ref('ro')
  const toasts = ref<Toast[]>([])

  // Sync actions - modifică state direct
  function toggleSidebar() {
    sidebarOpen.value = !sidebarOpen.value
  }

  function setTheme(newTheme: 'light' | 'dark') {
    theme.value = newTheme
    document.documentElement.setAttribute('data-theme', newTheme)
  }

  function addToast(toast: Omit<Toast, 'id'>) {
    const id = crypto.randomUUID()
    toasts.value.push({ ...toast, id })

    // Auto-remove după duration
    if (toast.duration !== Infinity) {
      setTimeout(() => {
        removeToast(id)
      }, toast.duration ?? 5000)
    }
  }

  function removeToast(id: string) {
    toasts.value = toasts.value.filter(t => t.id !== id)
  }

  return { sidebarOpen, theme, locale, toasts, toggleSidebar, setTheme, addToast, removeToast }
})
```

### Async Actions cu error handling

```typescript
export const useArticleStore = defineStore('article', () => {
  const articles = ref<Article[]>([])
  const currentArticle = ref<Article | null>(null)
  const loading = ref(false)
  const error = ref<string | null>(null)
  const saving = ref(false)

  async function fetchArticles(params?: ArticleQueryParams): Promise<void> {
    loading.value = true
    error.value = null

    try {
      articles.value = await articleApi.getAll(params)
    } catch (e) {
      error.value = e instanceof Error ? e.message : 'Failed to fetch articles'
      articles.value = [] // Reset pe eroare
    } finally {
      loading.value = false
    }
  }

  async function fetchArticleById(id: string): Promise<void> {
    loading.value = true
    error.value = null

    try {
      currentArticle.value = await articleApi.getById(id)
    } catch (e) {
      error.value = e instanceof Error ? e.message : 'Article not found'
      currentArticle.value = null
    } finally {
      loading.value = false
    }
  }

  // Optimistic update
  async function toggleLike(articleId: string): Promise<void> {
    const article = articles.value.find(a => a.id === articleId)
    if (!article) return

    // Salvează starea veche
    const previousLiked = article.liked
    const previousLikeCount = article.likeCount

    // Optimistic: update imediat UI-ul
    article.liked = !article.liked
    article.likeCount += article.liked ? 1 : -1

    try {
      // Trimite la server
      await articleApi.toggleLike(articleId)
    } catch {
      // Rollback pe eroare
      article.liked = previousLiked
      article.likeCount = previousLikeCount
      error.value = 'Nu s-a putut actualiza like-ul'
    }
  }

  // Action cu debounce (salvare draft)
  let saveTimeout: ReturnType<typeof setTimeout> | null = null

  function autoSaveDraft(content: string): void {
    if (saveTimeout) clearTimeout(saveTimeout)

    saveTimeout = setTimeout(async () => {
      if (!currentArticle.value) return
      saving.value = true
      try {
        await articleApi.saveDraft(currentArticle.value.id, { content })
      } catch {
        error.value = 'Auto-save failed'
      } finally {
        saving.value = false
      }
    }, 2000) // Debounce 2 secunde
  }

  // Action care apelează alte actions din alte stores
  async function publishArticle(id: string): Promise<void> {
    const notifStore = useNotificationStore()
    const analyticsStore = useAnalyticsStore()

    saving.value = true
    error.value = null

    try {
      const published = await articleApi.publish(id)

      // Update local state
      const index = articles.value.findIndex(a => a.id === id)
      if (index !== -1) {
        articles.value[index] = published
      }
      if (currentArticle.value?.id === id) {
        currentArticle.value = published
      }

      // Cross-store actions
      notifStore.addNotification({
        type: 'success',
        message: `Articolul "${published.title}" a fost publicat!`
      })

      analyticsStore.trackEvent({
        type: 'article_published',
        metadata: { articleId: id }
      })
    } catch (e) {
      error.value = e instanceof Error ? e.message : 'Publish failed'
      notifStore.addNotification({
        type: 'error',
        message: 'Nu s-a putut publica articolul'
      })
      throw e
    } finally {
      saving.value = false
    }
  }

  return {
    articles,
    currentArticle,
    loading,
    error,
    saving,
    fetchArticles,
    fetchArticleById,
    toggleLike,
    autoSaveDraft,
    publishArticle
  }
})
```

### $patch() - batch updates

```typescript
const store = useUserStore()

// $patch cu obiect
store.$patch({
  user: newUser,
  loading: false,
  error: null
})

// $patch cu funcție (mai flexibil - util pentru arrays)
store.$patch((state) => {
  state.items.push(newItem)
  state.total = state.items.length
  state.lastUpdated = new Date()
})
```

### $onAction() - interceptare actions

```typescript
// În componentă sau plugin
const store = useUserStore()

const unsubscribe = store.$onAction(
  ({
    name,       // numele acțiunii
    store,      // instanța store-ului
    args,       // argumentele acțiunii
    after,      // callback după succes
    onError     // callback pe eroare
  }) => {
    const startTime = Date.now()

    console.log(`Action ${name} started with args:`, args)

    after((result) => {
      console.log(
        `Action ${name} completed in ${Date.now() - startTime}ms`,
        result
      )
    })

    onError((error) => {
      console.error(
        `Action ${name} failed after ${Date.now() - startTime}ms`,
        error
      )
    })
  }
)

// Cleanup
onUnmounted(() => {
  unsubscribe()
})
```

### Paralela cu Angular

| Pattern | Pinia | NgRx | Angular Service |
|---|---|---|---|
| **Sync mutation** | Direct assignment | Reducer (pure fn) | BehaviorSubject.next() |
| **Async action** | async function | Effect + Action | async method |
| **Optimistic update** | Manual (save → update → rollback) | Same pattern + Actions | Same pattern |
| **Cross-store** | useOtherStore() în action | Dispatch action din Effect | inject(OtherService) |
| **Interceptor** | $onAction() | Meta-reducers | RxJS tap() |
| **Batch update** | $patch() | Single action, reducer handles all | Multiple .next() calls |

---

## 8. Plugins (persistence, logging, etc.)

### Plugin API - structura

```typescript
import type { PiniaPluginContext } from 'pinia'

// Un plugin Pinia e o funcție care primește context
function myPlugin(context: PiniaPluginContext) {
  // context.pinia    - instanța Pinia
  // context.app      - instanța Vue (Vue 3)
  // context.store    - store-ul curent
  // context.options   - opțiunile defineStore()

  // Poți adăuga proprietăți la store
  context.store.myCustomProp = 'hello'

  // Poți adăuga state reactiv
  context.store.$state.myPluginState = ref('initial')

  // Poți returna un obiect care se merge cu store-ul
  return {
    pluginValue: 'added by plugin'
  }
}
```

### Plugin: Logger complet

```typescript
// plugins/piniaLogger.ts
import type { PiniaPluginContext } from 'pinia'

interface LoggerOptions {
  enabled?: boolean
  logActions?: boolean
  logMutations?: boolean
  logLevel?: 'log' | 'debug' | 'info'
  collapsed?: boolean
  filter?: (storeName: string, actionName: string) => boolean
}

export function createPiniaLogger(options: LoggerOptions = {}) {
  const {
    enabled = import.meta.env.DEV,
    logActions = true,
    logMutations = true,
    logLevel = 'log',
    collapsed = true,
    filter = () => true
  } = options

  return ({ store }: PiniaPluginContext) => {
    if (!enabled) return

    // Log Actions
    if (logActions) {
      store.$onAction(({ name, args, after, onError }) => {
        if (!filter(store.$id, name)) return

        const startTime = performance.now()
        const groupMethod = collapsed ? 'groupCollapsed' : 'group'

        console[groupMethod](
          `%c[Pinia Action] %c${store.$id}.${name}`,
          'color: #888',
          'color: #4CAF50; font-weight: bold'
        )
        console[logLevel]('Arguments:', args)
        console[logLevel]('State before:', JSON.parse(JSON.stringify(store.$state)))

        after((result) => {
          const duration = performance.now() - startTime
          console[logLevel]('Result:', result)
          console[logLevel]('State after:', JSON.parse(JSON.stringify(store.$state)))
          console[logLevel](`Duration: ${duration.toFixed(2)}ms`)
          console.groupEnd()
        })

        onError((error) => {
          const duration = performance.now() - startTime
          console.error('Error:', error)
          console[logLevel](`Duration: ${duration.toFixed(2)}ms`)
          console.groupEnd()
        })
      })
    }

    // Log State Mutations
    if (logMutations) {
      store.$subscribe((mutation, state) => {
        console[logLevel](
          `%c[Pinia Mutation] %c${store.$id}`,
          'color: #888',
          'color: #FF9800; font-weight: bold',
          {
            type: mutation.type,
            storeId: mutation.storeId,
            events: mutation.events,
            state: JSON.parse(JSON.stringify(state))
          }
        )
      })
    }
  }
}

// Utilizare în main.ts
import { createPiniaLogger } from '@/plugins/piniaLogger'

const pinia = createPinia()

pinia.use(createPiniaLogger({
  enabled: import.meta.env.DEV,
  logActions: true,
  logMutations: true,
  collapsed: true,
  filter: (store, action) => {
    // Exclude stores zgomotoase
    return store !== 'ui' || action !== 'updateMousePosition'
  }
}))
```

### Plugin: Persistence cu pinia-plugin-persistedstate

```typescript
// main.ts
import { createPinia } from 'pinia'
import piniaPersistedstate from 'pinia-plugin-persistedstate'

const pinia = createPinia()
pinia.use(piniaPersistedstate)

app.use(pinia)
```

```typescript
// stores/useSettingsStore.ts - SETUP STORE cu persistence
export const useSettingsStore = defineStore('settings', () => {
  const theme = ref<'light' | 'dark'>('light')
  const locale = ref('ro')
  const fontSize = ref(16)
  const sidebarCollapsed = ref(false)
  const recentSearches = ref<string[]>([])

  function setTheme(newTheme: 'light' | 'dark') {
    theme.value = newTheme
  }

  function setLocale(newLocale: string) {
    locale.value = newLocale
  }

  function addRecentSearch(query: string) {
    recentSearches.value = [
      query,
      ...recentSearches.value.filter(s => s !== query)
    ].slice(0, 10)
  }

  return {
    theme, locale, fontSize, sidebarCollapsed, recentSearches,
    setTheme, setLocale, addRecentSearch
  }
}, {
  // Opțiuni persistence - al treilea argument defineStore
  persist: {
    key: 'app-settings',              // Cheia în storage
    storage: localStorage,             // sau sessionStorage
    pick: ['theme', 'locale', 'fontSize'], // Doar aceste proprietăți se persistă
    // omit: ['recentSearches'],       // Alternativ: exclude proprietăți
  }
})
```

```typescript
// Persistence avansată - multiple storage locations
export const useAuthStore = defineStore('auth', () => {
  const token = ref<string | null>(null)
  const refreshToken = ref<string | null>(null)
  const user = ref<User | null>(null)
  const rememberMe = ref(false)

  // ... actions

  return { token, refreshToken, user, rememberMe }
}, {
  persist: [
    {
      // Token-ul în sessionStorage (se pierde la închiderea tab-ului)
      key: 'auth-token',
      storage: sessionStorage,
      pick: ['token']
    },
    {
      // User și rememberMe în localStorage (persistă)
      key: 'auth-user',
      storage: localStorage,
      pick: ['user', 'rememberMe']
    },
    {
      // Refresh token - custom storage (ex: secure cookie wrapper)
      key: 'auth-refresh',
      storage: {
        getItem(key: string) {
          // Citire din cookie httpOnly sau alt mecanism
          return getCookie(key)
        },
        setItem(key: string, value: string) {
          setCookie(key, value, { secure: true, sameSite: 'strict' })
        },
        removeItem(key: string) {
          deleteCookie(key)
        }
      },
      pick: ['refreshToken']
    }
  ]
})
```

### Plugin custom: Undo/Redo

```typescript
// plugins/piniaUndoRedo.ts
import type { PiniaPluginContext } from 'pinia'
import { ref, toRaw } from 'vue'

interface UndoRedoOptions {
  maxHistory?: number
  // Store IDs care suportă undo/redo
  stores?: string[]
}

export function createUndoRedo(options: UndoRedoOptions = {}) {
  const { maxHistory = 50, stores } = options

  return ({ store }: PiniaPluginContext) => {
    // Skip dacă store-ul nu e în lista permisă
    if (stores && !stores.includes(store.$id)) return

    const history = ref<string[]>([])
    const future = ref<string[]>([])

    // Salvează starea inițială
    history.value.push(JSON.stringify(toRaw(store.$state)))

    // Ascultă schimbări de state
    store.$subscribe((_, state) => {
      const serialized = JSON.stringify(toRaw(state))
      const lastState = history.value[history.value.length - 1]

      // Evită duplicate consecutive
      if (serialized !== lastState) {
        history.value.push(serialized)
        future.value = []  // Reset redo pe schimbare nouă

        // Limitează istoria
        if (history.value.length > maxHistory) {
          history.value.shift()
        }
      }
    })

    return {
      canUndo: computed(() => history.value.length > 1),
      canRedo: computed(() => future.value.length > 0),

      undo() {
        if (history.value.length <= 1) return

        const current = history.value.pop()!
        future.value.push(current)

        const previous = history.value[history.value.length - 1]
        store.$patch(JSON.parse(previous))
      },

      redo() {
        if (future.value.length === 0) return

        const next = future.value.pop()!
        history.value.push(next)
        store.$patch(JSON.parse(next))
      },

      clearHistory() {
        history.value = [JSON.stringify(toRaw(store.$state))]
        future.value = []
      }
    }
  }
}

// Utilizare
const pinia = createPinia()
pinia.use(createUndoRedo({
  maxHistory: 30,
  stores: ['editor', 'canvas']  // Doar aceste stores au undo/redo
}))
```

```vue
<!-- Utilizare undo/redo în componentă -->
<script setup lang="ts">
const editorStore = useEditorStore()

// Proprietățile undo/redo sunt adăugate de plugin
</script>

<template>
  <div class="toolbar">
    <button
      @click="editorStore.undo()"
      :disabled="!editorStore.canUndo"
    >
      Undo
    </button>
    <button
      @click="editorStore.redo()"
      :disabled="!editorStore.canRedo"
    >
      Redo
    </button>
  </div>
</template>
```

### Plugin: Loading state global

```typescript
// plugins/piniaLoading.ts
import type { PiniaPluginContext } from 'pinia'

export function piniaLoadingPlugin({ store }: PiniaPluginContext) {
  const activeActions = ref(new Set<string>())

  store.$onAction(({ name, after, onError }) => {
    activeActions.value.add(name)

    const cleanup = () => {
      activeActions.value.delete(name)
    }

    after(cleanup)
    onError(cleanup)
  })

  return {
    activeActions: computed(() => [...activeActions.value]),
    isActionLoading: computed(() => {
      return (actionName: string) => activeActions.value.has(actionName)
    }),
    hasActiveActions: computed(() => activeActions.value.size > 0)
  }
}
```

### Paralela cu Angular

| Plugin Pinia | Echivalent Angular |
|---|---|
| **Persistence** | Manual (localStorage service) sau @ngrx/store cu meta-reducer |
| **Logger** | NgRx Store DevTools / custom meta-reducer |
| **Undo/Redo** | ngrx-undo / custom meta-reducer |
| **Loading tracking** | @ngrx/component-store sau HTTP interceptor |

Pinia plugins sunt mai simple decât NgRx meta-reducers:
- Plugin Pinia = o funcție simplă
- NgRx meta-reducer = higher-order function pe reducer + integrare cu effects

---

## 9. Store Design Patterns

### Pattern 1: Feature-based Stores (un store per domeniu)

```
stores/
├── useAuthStore.ts         # Autentificare
├── useUserStore.ts         # User profile, preferences
├── useProductStore.ts      # Catalog produse
├── useCartStore.ts         # Coș de cumpărături
├── useOrderStore.ts        # Comenzi
├── useNotificationStore.ts # Notificări
├── useSearchStore.ts       # Search state
└── useUIStore.ts           # UI global state
```

**Regulă**: un store per domeniu/funcționalitate. NU un store monolitic.

### Pattern 2: Composable + Store (separare logică reutilizabilă de state global)

```typescript
// composables/useSearch.ts - logică reutilizabilă, FĂRĂ state global
import { ref, computed, watch } from 'vue'
import { useDebounceFn } from '@vueuse/core'

export function useSearch<T>(
  searchFn: (query: string) => Promise<T[]>,
  options: { debounce?: number; minLength?: number } = {}
) {
  const { debounce = 300, minLength = 2 } = options

  const query = ref('')
  const results = ref<T[]>([]) as Ref<T[]>
  const loading = ref(false)
  const error = ref<string | null>(null)

  const hasResults = computed(() => results.value.length > 0)
  const canSearch = computed(() => query.value.length >= minLength)

  const performSearch = useDebounceFn(async () => {
    if (!canSearch.value) {
      results.value = []
      return
    }

    loading.value = true
    error.value = null

    try {
      results.value = await searchFn(query.value)
    } catch (e) {
      error.value = e instanceof Error ? e.message : 'Search failed'
      results.value = []
    } finally {
      loading.value = false
    }
  }, debounce)

  watch(query, () => {
    performSearch()
  })

  function clear() {
    query.value = ''
    results.value = []
    error.value = null
  }

  return {
    query, results, loading, error,
    hasResults, canSearch, clear
  }
}
```

```typescript
// stores/useGlobalSearchStore.ts - state GLOBAL de search
import { defineStore } from 'pinia'
import { useSearch } from '@/composables/useSearch'
import { searchApi } from '@/api/search'

export const useGlobalSearchStore = defineStore('globalSearch', () => {
  // Reutilizează composable-ul în store
  const productSearch = useSearch(
    (q) => searchApi.searchProducts(q),
    { debounce: 500 }
  )

  const articleSearch = useSearch(
    (q) => searchApi.searchArticles(q),
    { debounce: 500 }
  )

  // State specific store-ului
  const recentSearches = ref<string[]>([])
  const activeTab = ref<'products' | 'articles'>('products')

  function addToRecent(query: string) {
    recentSearches.value = [
      query,
      ...recentSearches.value.filter(s => s !== query)
    ].slice(0, 10)
  }

  function searchAll(query: string) {
    productSearch.query.value = query
    articleSearch.query.value = query
    addToRecent(query)
  }

  return {
    productSearch,
    articleSearch,
    recentSearches,
    activeTab,
    searchAll
  }
})
```

### Pattern 3: UI State Store (modal, toast, sidebar)

```typescript
// stores/useUIStore.ts
export const useUIStore = defineStore('ui', () => {
  // Sidebar
  const sidebarOpen = ref(true)
  const sidebarWidth = ref(280)

  // Modal stack
  const modals = ref<ModalConfig[]>([])

  // Toasts
  const toasts = ref<Toast[]>([])

  // Global loading overlay
  const globalLoading = ref(false)
  const globalLoadingMessage = ref('')

  // Breakpoint tracking
  const windowWidth = ref(window.innerWidth)
  const isMobile = computed(() => windowWidth.value < 768)
  const isTablet = computed(() => windowWidth.value >= 768 && windowWidth.value < 1024)
  const isDesktop = computed(() => windowWidth.value >= 1024)

  // Modal actions
  function openModal(config: ModalConfig) {
    modals.value.push({ ...config, id: crypto.randomUUID() })
  }

  function closeModal(id?: string) {
    if (id) {
      modals.value = modals.value.filter(m => m.id !== id)
    } else {
      modals.value.pop()  // Închide ultimul modal
    }
  }

  function closeAllModals() {
    modals.value = []
  }

  const activeModal = computed(() =>
    modals.value.length > 0 ? modals.value[modals.value.length - 1] : null
  )

  const hasModals = computed(() => modals.value.length > 0)

  // Toast actions
  function showToast(toast: Omit<Toast, 'id'>) {
    const id = crypto.randomUUID()
    toasts.value.push({ ...toast, id })

    const duration = toast.duration ?? 5000
    if (duration > 0) {
      setTimeout(() => dismissToast(id), duration)
    }
  }

  function dismissToast(id: string) {
    toasts.value = toasts.value.filter(t => t.id !== id)
  }

  // Convenience methods
  function showSuccess(message: string) {
    showToast({ type: 'success', message })
  }

  function showError(message: string) {
    showToast({ type: 'error', message, duration: 8000 })
  }

  function showWarning(message: string) {
    showToast({ type: 'warning', message })
  }

  // Sidebar actions
  function toggleSidebar() {
    sidebarOpen.value = !sidebarOpen.value
  }

  // Loading overlay
  function startLoading(message = 'Se încarcă...') {
    globalLoading.value = true
    globalLoadingMessage.value = message
  }

  function stopLoading() {
    globalLoading.value = false
    globalLoadingMessage.value = ''
  }

  // Window resize listener
  function _handleResize() {
    windowWidth.value = window.innerWidth
  }

  if (typeof window !== 'undefined') {
    window.addEventListener('resize', _handleResize)
  }

  return {
    sidebarOpen, sidebarWidth,
    modals, activeModal, hasModals,
    toasts,
    globalLoading, globalLoadingMessage,
    windowWidth, isMobile, isTablet, isDesktop,
    openModal, closeModal, closeAllModals,
    showToast, dismissToast, showSuccess, showError, showWarning,
    toggleSidebar,
    startLoading, stopLoading
  }
})
```

### Pattern 4: Normalized State (pentru entități relaționale)

```typescript
// stores/useEntityStore.ts - pattern de normalizare
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'

interface NormalizedState<T extends { id: string }> {
  ids: string[]
  entities: Record<string, T>
}

function useNormalizedState<T extends { id: string }>() {
  const ids = ref<string[]>([])
  const entities = ref<Record<string, T>>({})

  const all = computed(() =>
    ids.value.map(id => entities.value[id]).filter(Boolean)
  )

  const count = computed(() => ids.value.length)

  function getById(id: string): T | undefined {
    return entities.value[id]
  }

  function setMany(items: T[]) {
    for (const item of items) {
      if (!ids.value.includes(item.id)) {
        ids.value.push(item.id)
      }
      entities.value[item.id] = item
    }
  }

  function setOne(item: T) {
    if (!ids.value.includes(item.id)) {
      ids.value.push(item.id)
    }
    entities.value[item.id] = item
  }

  function removeOne(id: string) {
    ids.value = ids.value.filter(i => i !== id)
    delete entities.value[id]
  }

  function updateOne(id: string, changes: Partial<T>) {
    if (entities.value[id]) {
      entities.value[id] = { ...entities.value[id], ...changes }
    }
  }

  function clear() {
    ids.value = []
    entities.value = {}
  }

  return {
    ids, entities, all, count,
    getById, setMany, setOne, removeOne, updateOne, clear
  }
}

// Utilizare concretă
export const useTaskStore = defineStore('task', () => {
  const {
    ids, entities, all, count,
    getById, setMany, setOne, removeOne, updateOne, clear
  } = useNormalizedState<Task>()

  const loading = ref(false)
  const error = ref<string | null>(null)

  // Getters specifice domeniu
  const completedTasks = computed(() =>
    all.value.filter(t => t.status === 'completed')
  )

  const pendingTasks = computed(() =>
    all.value.filter(t => t.status === 'pending')
  )

  const completionRate = computed(() => {
    if (count.value === 0) return 0
    return completedTasks.value.length / count.value
  })

  // Actions
  async function fetchTasks() {
    loading.value = true
    try {
      const tasks = await taskApi.getAll()
      setMany(tasks)
    } catch (e) {
      error.value = e instanceof Error ? e.message : 'Failed'
    } finally {
      loading.value = false
    }
  }

  async function toggleComplete(id: string) {
    const task = getById(id)
    if (!task) return

    const newStatus = task.status === 'completed' ? 'pending' : 'completed'
    updateOne(id, { status: newStatus })

    try {
      await taskApi.updateStatus(id, newStatus)
    } catch {
      // Rollback
      updateOne(id, { status: task.status })
    }
  }

  return {
    ids, entities, all, count,
    loading, error,
    completedTasks, pendingTasks, completionRate,
    getById, fetchTasks, toggleComplete,
    setMany, setOne, removeOne
  }
})
```

### Paralela cu Angular

| Pattern | Pinia | Angular |
|---|---|---|
| **Feature stores** | Un store per feature | Un service per feature sau NgRx feature state |
| **Composable + Store** | composable() + defineStore() | Utility service + Feature service |
| **UI State** | useUIStore | UI Service sau ComponentStore |
| **Normalized state** | Custom helper function | @ngrx/entity (EntityAdapter) |
| **Orchestrator** | Checkout store | Facade service |

NgRx **EntityAdapter** (din `@ngrx/entity`) este echivalentul pattern-ului
de normalizare, dar mult mai matur și cu mai multe features built-in
(sorting, selectori predefiniti). În Pinia trebuie să construiești manual.

---

## 10. Paralela completă: Pinia vs NgRx vs Angular Services

### Tabel comparativ complet

| Aspect | Pinia | NgRx | Angular Service + BehaviorSubject |
|---|---|---|---|
| **Filosofie** | Simplu, pragmatic | Strict, Redux pattern | Flexibil, OOP |
| **Complexitate** | Scăzută | Ridicată | Scăzută-Medie |
| **Boilerplate** | Minimal | Foarte mare | Minimal |
| **Learning curve** | ~30 minute | Zile / Săptămâni | ~1 oră |
| **State** | `ref()` | Reducer (pure fn) | `BehaviorSubject` |
| **Computed** | `computed()` | `createSelector()` | `pipe(map())` |
| **Side effects** | Direct în actions | Effects (RxJS streams) | Direct în methods |
| **Immutability** | Optional (nu enforced) | Enforced (freeze) | Optional |
| **DevTools** | Da (timeline, edit) | Da (puternic, time-travel) | Nu (fără DevTools standard) |
| **Time travel** | Da | Da | Nu |
| **TypeScript** | Nativ, inferență completă | Bun, dar verbose | Nativ |
| **Bundle size** | ~1.5KB | ~15KB+ (cu effects) | 0 (built-in) |
| **SSR** | Built-in | Soluții 3rd party | Manual |
| **Plugins** | Da (persistence, etc.) | Meta-reducers, effects | Interceptors, decorators |
| **Testing** | Simple (no TestBed) | Complex (TestBed + mock store) | Mediu (TestBed) |
| **Scalabilitate** | Foarte bună | Excelentă | Bună |
| **Best for** | 90% din aplicații | Aplicații enterprise foarte complexe | Orice dimensiune |

### Cod echivalent: State management pentru un TODO list

**Pinia (Setup Store)**:

```typescript
// stores/useTodoStore.ts
export const useTodoStore = defineStore('todo', () => {
  const todos = ref<Todo[]>([])
  const filter = ref<'all' | 'active' | 'completed'>('all')
  const loading = ref(false)

  const filteredTodos = computed(() => {
    switch (filter.value) {
      case 'active': return todos.value.filter(t => !t.completed)
      case 'completed': return todos.value.filter(t => t.completed)
      default: return todos.value
    }
  })

  const remainingCount = computed(() =>
    todos.value.filter(t => !t.completed).length
  )

  async function fetchTodos() {
    loading.value = true
    try {
      todos.value = await todoApi.getAll()
    } finally {
      loading.value = false
    }
  }

  async function addTodo(title: string) {
    const todo = await todoApi.create({ title, completed: false })
    todos.value.push(todo)
  }

  async function toggleTodo(id: string) {
    const todo = todos.value.find(t => t.id === id)
    if (!todo) return
    todo.completed = !todo.completed
    try {
      await todoApi.update(id, { completed: todo.completed })
    } catch {
      todo.completed = !todo.completed  // rollback
    }
  }

  function setFilter(newFilter: 'all' | 'active' | 'completed') {
    filter.value = newFilter
  }

  return {
    todos, filter, loading,
    filteredTodos, remainingCount,
    fetchTodos, addTodo, toggleTodo, setFilter
  }
})
```

**NgRx (Actions + Reducer + Selectors + Effects)**:

```typescript
// ═══ ACTIONS (todo.actions.ts) ═══
export const TodoActions = createActionGroup({
  source: 'Todo',
  events: {
    'Load Todos': emptyProps(),
    'Load Todos Success': props<{ todos: Todo[] }>(),
    'Load Todos Failure': props<{ error: string }>(),
    'Add Todo': props<{ title: string }>(),
    'Add Todo Success': props<{ todo: Todo }>(),
    'Toggle Todo': props<{ id: string }>(),
    'Toggle Todo Success': props<{ id: string; completed: boolean }>(),
    'Toggle Todo Failure': props<{ id: string; completed: boolean }>(),
    'Set Filter': props<{ filter: 'all' | 'active' | 'completed' }>()
  }
})

// ═══ REDUCER (todo.reducer.ts) ═══
export interface TodoState {
  todos: Todo[]
  filter: 'all' | 'active' | 'completed'
  loading: boolean
  error: string | null
}

const initialState: TodoState = {
  todos: [],
  filter: 'all',
  loading: false,
  error: null
}

export const todoReducer = createReducer(
  initialState,
  on(TodoActions.loadTodos, (state) => ({ ...state, loading: true })),
  on(TodoActions.loadTodosSuccess, (state, { todos }) => ({
    ...state, todos, loading: false
  })),
  on(TodoActions.loadTodosFailure, (state, { error }) => ({
    ...state, error, loading: false
  })),
  on(TodoActions.addTodoSuccess, (state, { todo }) => ({
    ...state, todos: [...state.todos, todo]
  })),
  on(TodoActions.toggleTodoSuccess, (state, { id, completed }) => ({
    ...state,
    todos: state.todos.map(t => t.id === id ? { ...t, completed } : t)
  })),
  on(TodoActions.toggleTodoFailure, (state, { id, completed }) => ({
    ...state,
    todos: state.todos.map(t => t.id === id ? { ...t, completed } : t)
  })),
  on(TodoActions.setFilter, (state, { filter }) => ({ ...state, filter }))
)

// ═══ SELECTORS (todo.selectors.ts) ═══
export const selectTodoState = (state: AppState) => state.todo
export const selectTodos = createSelector(selectTodoState, s => s.todos)
export const selectFilter = createSelector(selectTodoState, s => s.filter)
export const selectLoading = createSelector(selectTodoState, s => s.loading)

export const selectFilteredTodos = createSelector(
  selectTodos, selectFilter,
  (todos, filter) => {
    switch (filter) {
      case 'active': return todos.filter(t => !t.completed)
      case 'completed': return todos.filter(t => t.completed)
      default: return todos
    }
  }
)

export const selectRemainingCount = createSelector(
  selectTodos,
  (todos) => todos.filter(t => !t.completed).length
)

// ═══ EFFECTS (todo.effects.ts) ═══
@Injectable()
export class TodoEffects {
  loadTodos$ = createEffect(() =>
    this.actions$.pipe(
      ofType(TodoActions.loadTodos),
      switchMap(() =>
        this.todoApi.getAll().pipe(
          map(todos => TodoActions.loadTodosSuccess({ todos })),
          catchError(error => of(TodoActions.loadTodosFailure({ error: error.message })))
        )
      )
    )
  )

  addTodo$ = createEffect(() =>
    this.actions$.pipe(
      ofType(TodoActions.addTodo),
      switchMap(({ title }) =>
        this.todoApi.create({ title, completed: false }).pipe(
          map(todo => TodoActions.addTodoSuccess({ todo }))
        )
      )
    )
  )

  toggleTodo$ = createEffect(() =>
    this.actions$.pipe(
      ofType(TodoActions.toggleTodo),
      switchMap(({ id }) => {
        const todo = // ... get current todo
        return this.todoApi.update(id, { completed: !todo.completed }).pipe(
          map(() => TodoActions.toggleTodoSuccess({ id, completed: !todo.completed })),
          catchError(() => of(TodoActions.toggleTodoFailure({ id, completed: todo.completed })))
        )
      })
    )
  )

  constructor(
    private actions$: Actions,
    private todoApi: TodoApiService
  ) {}
}
```

**Angular Service + BehaviorSubject**:

```typescript
// services/todo.service.ts
@Injectable({ providedIn: 'root' })
export class TodoService {
  private _todos = new BehaviorSubject<Todo[]>([])
  private _filter = new BehaviorSubject<'all' | 'active' | 'completed'>('all')
  private _loading = new BehaviorSubject<boolean>(false)

  readonly todos$ = this._todos.asObservable()
  readonly filter$ = this._filter.asObservable()
  readonly loading$ = this._loading.asObservable()

  readonly filteredTodos$ = combineLatest([this.todos$, this.filter$]).pipe(
    map(([todos, filter]) => {
      switch (filter) {
        case 'active': return todos.filter(t => !t.completed)
        case 'completed': return todos.filter(t => t.completed)
        default: return todos
      }
    })
  )

  readonly remainingCount$ = this.todos$.pipe(
    map(todos => todos.filter(t => !t.completed).length)
  )

  constructor(private http: HttpClient) {}

  async fetchTodos(): Promise<void> {
    this._loading.next(true)
    try {
      const todos = await firstValueFrom(this.http.get<Todo[]>('/api/todos'))
      this._todos.next(todos)
    } finally {
      this._loading.next(false)
    }
  }

  async addTodo(title: string): Promise<void> {
    const todo = await firstValueFrom(
      this.http.post<Todo>('/api/todos', { title, completed: false })
    )
    this._todos.next([...this._todos.value, todo])
  }

  async toggleTodo(id: string): Promise<void> {
    const todos = this._todos.value
    const todo = todos.find(t => t.id === id)
    if (!todo) return

    const updated = { ...todo, completed: !todo.completed }
    this._todos.next(todos.map(t => t.id === id ? updated : t))

    try {
      await firstValueFrom(this.http.patch(`/api/todos/${id}`, { completed: updated.completed }))
    } catch {
      this._todos.next(todos)  // rollback
    }
  }

  setFilter(filter: 'all' | 'active' | 'completed'): void {
    this._filter.next(filter)
  }
}
```

### Concluzia comparației

**Pinia** (48 linii) vs **NgRx** (120+ linii) vs **Angular Service** (55 linii)

- **Pinia** = cel mai puțin boilerplate, cel mai intuitiv
- **NgRx** = cel mai strict, cel mai verbose, cel mai bun pentru echipe mari cu discipline de cod
- **Angular Service** = similar ca simplitate cu Pinia, dar fără DevTools built-in

**Sfat pentru interviuri**: Dacă ești întrebat „De ce nu NgRx-like în Vue?", răspunsul e:
Pinia oferă aceleași beneficii (predictabilitate, DevTools, testabilitate) fără boilerplate-ul
excesiv. NgRx e excelent pentru aplicații enterprise masive cu echipe de 20+ devs,
dar pentru 90% din aplicații Pinia e soluția corectă.

---

## 11. Pinia în Micro-frontenduri

### Provocări

Într-o arhitectură micro-frontend (Module Federation, Single-SPA, etc.), fiecare
MFE este o aplicație Vue independentă cu propria instanță Pinia.

**Problemele principale**:

1. **Multiple Pinia instances** - fiecare MFE are propriul `createPinia()`
2. **Store ID collisions** - două MFE-uri pot avea un store cu id `'user'`
3. **Shared state** - cum partajezi state între MFE-uri?
4. **Singleton vs isolated** - când vrei state partajat vs izolat?

### Strategia 1: Izolare completă (recomandat)

```typescript
// MFE-A/main.ts
const piniaA = createPinia()
appA.use(piniaA)

// MFE-B/main.ts
const piniaB = createPinia()
appB.use(piniaB)

// Fiecare MFE are stores complet izolate
// Comunicare între MFE-uri: Custom Events / Event Bus
```

```typescript
// shared/eventBus.ts - comunicare între MFE-uri
class EventBus {
  private handlers = new Map<string, Set<Function>>()

  on(event: string, handler: Function) {
    if (!this.handlers.has(event)) {
      this.handlers.set(event, new Set())
    }
    this.handlers.get(event)!.add(handler)
    return () => this.handlers.get(event)?.delete(handler)
  }

  emit(event: string, data?: unknown) {
    this.handlers.get(event)?.forEach(handler => handler(data))
  }
}

// Singleton global (pe window)
export const eventBus = (window as any).__EVENT_BUS__ ??= new EventBus()
```

```typescript
// MFE-A: emite event când user-ul se loghează
const userStore = useUserStore()

watch(() => userStore.isAuthenticated, (isAuth) => {
  eventBus.emit('auth:changed', {
    isAuthenticated: isAuth,
    user: isAuth ? userStore.user : null
  })
})
```

```typescript
// MFE-B: ascultă event-ul
const localUserState = ref<User | null>(null)

eventBus.on('auth:changed', (data: { isAuthenticated: boolean; user: User | null }) => {
  localUserState.value = data.user
})
```

### Strategia 2: Shared Pinia (Module Federation)

```typescript
// webpack.config.js (Module Federation)
module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      shared: {
        vue: { singleton: true },
        pinia: { singleton: true },  // O singură instanță Pinia
        // Shared stores
        '@shared/stores': { singleton: true }
      }
    })
  ]
}
```

```typescript
// shared/stores/useSharedAuthStore.ts
// Prefix stores partajate pentru a evita coliziuni
export const useSharedAuthStore = defineStore('shared:auth', () => {
  // ...state partajat între toate MFE-urile
})
```

### Strategia 3: Store ID namespacing

```typescript
// Funcție helper pentru prefix
function createNamespacedStore<T>(
  mfeName: string,
  storeId: string,
  setupFn: () => T
) {
  return defineStore(`${mfeName}:${storeId}`, setupFn)
}

// MFE-A
export const useUserStore = createNamespacedStore('mfe-a', 'user', () => {
  // ...
})
// Store ID = 'mfe-a:user'

// MFE-B
export const useUserStore = createNamespacedStore('mfe-b', 'user', () => {
  // ...
})
// Store ID = 'mfe-b:user'
```

### Best practices pentru MFE + Pinia

1. **Izolare by default** - fiecare MFE cu propriul Pinia
2. **Custom Events** pentru comunicare inter-MFE
3. **Shared stores** doar pentru state cu adevărat global (auth, theme)
4. **Namespace store IDs** cu prefix MFE
5. **Nu partaja** Pinia instance decât dacă Module Federation-ul o cere
6. **Contracte clare** între MFE-uri (interfețe TypeScript pentru events)

### Paralela cu Angular

| Aspect | Vue MFE + Pinia | Angular MFE |
|---|---|---|
| **Izolare** | Pinia instance separată | Injector separat per MFE |
| **Shared state** | Shared Pinia / Event Bus | Shared Injectable / Custom Events |
| **NgRx MFE** | N/A | Shared store sau federated state |
| **Communication** | Custom Events, eventBus | Custom Events, RxJS Subject partajat |
| **Namespace** | Store ID prefix | Feature state key prefix |

---

## 12. Întrebări de interviu

### Întrebarea 1: De ce Pinia și nu Vuex?

**Răspuns**: Pinia este succesorul oficial al Vuex, creat de același Eduardo San Martin Morote.
Eliminarea mutations reduce boilerplate-ul semnificativ - în Vuex aveai commit('MUTATION')
cu string magic, fără type safety, doar pentru a respecta principiul separării sync/async.
Pinia are flat stores în loc de nested modules, ceea ce simplifică mental model-ul.
TypeScript support-ul e nativ - nu mai ai nevoie de augmentări de tip sau declarații
suplimentare. Bundle size-ul e de 3x mai mic (~1.5KB vs ~5KB). De asemenea, setup stores
permit reutilizarea pattern-urilor din Composition API, inclusiv composables, ceea ce
face codul mai consistent între componente și stores.

### Întrebarea 2: Setup store vs Option store - când folosești fiecare?

**Răspuns**: Setup stores (Composition API style) sunt recomandate pentru proiecte noi.
Oferă private state (ce nu returnezi din setup nu e accesibil), watchers direct în store,
și reutilizarea composables. Option stores sunt utile când migrezi de la Vuex (structura
e similară) sau când ai nevoie de `$reset()` automat. Trade-off-ul principal: setup stores
nu au `$reset()` automat (trebuie implementat manual), dar câștigi flexibilitate totală.
Ca architect, aș standardiza pe setup stores în proiect și aș crea un helper pentru
`$reset()` dacă e necesar frecvent.

### Întrebarea 3: De ce ai nevoie de storeToRefs()?

**Răspuns**: Când destructurezi un store Pinia, pierzi reactivitatea state-ului și
getters-ilor. `storeToRefs()` convertește proprietățile reactive (ref-uri și computed)
în Ref-uri care mențin conexiunea la store. Actions (funcțiile) nu au nevoie de
storeToRefs() deoarece sunt funcții simple, nu valori reactive.
Diferența față de `toRefs()` din Vue este că `storeToRefs()` filtrează proprietățile
interne Pinia ($id, $state, $patch, etc.), returnând doar state și getters.
E similar conceptual cu NgRx selectors: `store.select()` returnează Observable-uri
care mențin reactivitatea, exact cum `storeToRefs()` menține refs reactive.

### Întrebarea 4: Cum gestionezi store composition (stores dependente)?

**Răspuns**: În setup stores, apelezi `useOtherStore()` direct. Important: pentru
dependențe circulare, apelează `useOtherStore()` înăuntrul acțiunilor, nu la
top-level-ul setup function. Asta asigură lazy initialization.
Pattern-ul Orchestrator Store e util când ai un flow complex care implică
mai multe stores (ex: checkout = auth + cart + orders + notifications).
Trade-off: prea multă compunere între stores poate duce la un graf de dependențe
greu de urmărit. O regulă bună: maximum 2-3 nivele de dependență. Dacă ai
mai mult, refactorizezi în composables sau extragi logica într-un layer separat.

### Întrebarea 5: Cum persisți state-ul?

**Răspuns**: Plugin-ul `pinia-plugin-persistedstate` este soluția standard. Suportă
multiple storage backends (localStorage, sessionStorage, custom), serializare selectivă
(pick/omit pentru a alege ce se persistă), și multiple storage locations per store.
Ca architect, câteva considerații: nu persista niciodată token-uri de autentificare
în localStorage (vulnerabil la XSS) - folosește httpOnly cookies sau sessionStorage.
Persistă doar date care au sens la refresh (preferințe user, draft-uri, filtre).
Pentru SSR, trebuie custom storage care verifică `typeof window !== 'undefined'`.
Alternativa e să rehidratezi state-ul din API la fiecare load, ceea ce e mai safe
dar mai lent.

### Întrebarea 6: Cum testezi Pinia stores?

**Răspuns**: Testarea Pinia stores e remarcabil de simplă comparativ cu NgRx.
Creezi o instanță Pinia fresh cu `createPinia()`, setezi `setActivePinia()`, și
instanțiezi store-ul. Nu ai nevoie de TestBed sau mock complex.

```typescript
import { setActivePinia, createPinia } from 'pinia'
import { useUserStore } from '@/stores/useUserStore'

describe('useUserStore', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
  })

  it('should login successfully', async () => {
    vi.spyOn(authApi, 'login').mockResolvedValue({
      user: { id: '1', name: 'Test' },
      token: 'abc'
    })

    const store = useUserStore()
    await store.login({ email: 'test@test.com', password: '123' })

    expect(store.isAuthenticated).toBe(true)
    expect(store.user?.name).toBe('Test')
    expect(store.loading).toBe(false)
  })

  it('should handle login error', async () => {
    vi.spyOn(authApi, 'login').mockRejectedValue(new Error('Invalid'))

    const store = useUserStore()
    await expect(store.login({ email: 'x', password: 'y' })).rejects.toThrow()

    expect(store.isAuthenticated).toBe(false)
    expect(store.error).toBe('Invalid')
  })
})
```

Trade-off vs NgRx: testele NgRx sunt mai granulare (testezi reducers, selectors, effects
independent), ceea ce e avantajos pentru logică foarte complexă. Testele Pinia sunt
integration tests pe store - mai simple dar mai puțin izolate.

### Întrebarea 7: Cum ai structura stores într-o aplicație mare?

**Răspuns**: Feature-based organization cu categorii clare:

```
stores/
├── auth/
│   └── useAuthStore.ts
├── user/
│   └── useUserStore.ts
├── products/
│   ├── useProductStore.ts
│   └── useProductFilterStore.ts
├── cart/
│   └── useCartStore.ts
├── orders/
│   └── useOrderStore.ts
├── ui/
│   ├── useUIStore.ts
│   └── useModalStore.ts
└── shared/
    └── useAppConfigStore.ts
```

Principii: un store per domeniu (max 200-300 linii), split dacă crește prea mult.
Stores UI separate de stores de date. Composables pentru logică reutilizabilă care
nu necesită state global. Store-urile nu ar trebui să conțină business logic complexă -
aceasta aparține unui layer de services/composables, iar store-ul doar gestionează
state-ul rezultat. Asta menține stores thin și testabile.

### Întrebarea 8: Cum funcționează Pinia în contextul micro-frontendurilor?

**Răspuns**: Fiecare MFE ar trebui să aibă propria instanță Pinia, izolată complet.
Comunicarea între MFE-uri se face prin Custom Events sau un Event Bus partajat pe
window - NU prin partajarea instanței Pinia. Dacă folosești Module Federation și
chiar ai nevoie de stores partajate (ex: auth), configurează `pinia: { singleton: true }`
în shared config și namespace-ează store IDs cu prefix MFE.
Trade-off: izolarea completă = duplicare state (fiecare MFE își menține propriul
user state). Shared Pinia = coupling mai mare, risc de coliziuni, dar consistență.
Recomandarea mea: izolare completă cu event-based sync pentru datele critice
(auth, theme), deoarece reduce coupling-ul și permite deploy independent.

### Întrebarea 9: Cum gestionezi optimistic updates?

**Răspuns**: Pattern: salvează starea curentă, aplică update-ul imediat în UI,
trimite request-ul la server, rollback pe eroare.

```typescript
async function toggleFavorite(productId: string) {
  const product = products.value.find(p => p.id === productId)
  if (!product) return

  // Snapshot pentru rollback
  const previousState = product.isFavorite

  // Optimistic update
  product.isFavorite = !product.isFavorite

  try {
    await api.toggleFavorite(productId)
  } catch {
    // Rollback
    product.isFavorite = previousState
    uiStore.showError('Operația a eșuat. Te rugăm să încerci din nou.')
  }
}
```

Trade-off: optimistic updates dau UX excelent (UI instant), dar adaugă complexitate
(gestionare rollback, stări inconsistente temporare). Folosește-le pentru acțiuni
frecvente și reversibile (like, favorite, toggle). NU pentru acțiuni ireversibile
(delete, purchase) unde confirmarea serverului e obligatorie.

### Întrebarea 10: Compară Pinia cu NgRx - avantaje și dezavantaje

**Răspuns**:

**Pinia avantaje**: boilerplate minimal, learning curve de 30 minute, TypeScript
inferență nativă, bundle size mic, flexibilitate în structurare, testare simplă.

**Pinia dezavantaje**: nu enforced immutability (developerii pot muta state direct
în moduri imprevizibile), mai puțină disciplină impusă de framework, DevTools mai
puțin puternice decât NgRx Store DevTools.

**NgRx avantaje**: immutability enforced, pattern strict (reduce erori în echipe mari),
DevTools foarte puternice, effects management excellent pentru flows complexe
async, selectors memoizate performante.

**NgRx dezavantaje**: boilerplate enorm (actions, reducers, effects, selectors
pentru ORICE change), learning curve abruptă, bundle size mare, over-engineering
pentru aplicații mici/medii.

**Concluzie architect**: pentru 90% din aplicații Vue, Pinia e alegerea corectă.
Echivalentul NgRx în Vue ar fi util doar pentru aplicații cu sute de
entități interconectate și echipe foarte mari (20+ devs) unde disciplina
codului e critică. Dar chiar și în Vue ecosystem, nu există un echivalent NgRx
matur - și nu e nevoie, pentru că Pinia rezolvă problemele suficient de bine
la o fracțiune din complexitate.

### Întrebarea 11: Când ai folosi Pinia vs provide/inject vs composables?

**Răspuns**: Depinde de scopul și durata de viață a state-ului:

- **Pinia store**: state global, partajat între componente care nu sunt în
  relație părinte-copil, persistat între navigări de rute. Ex: auth, cart,
  notificări, theme.

- **provide/inject**: state scoped la un sub-arbore de componente. Ex: form
  context partajat între FormField children, theme override local. Atenție:
  nu are DevTools, debugging dificil la scară mare.

- **Composables** (fără store): logică reutilizabilă cu state LOCAL per instanță.
  Ex: useSearch(), useForm(), usePagination(). Fiecare componentă care apelează
  composable-ul primește propria instanță de state.

**Regula**: Dacă două componente care nu sunt în relație directă trebuie să
partajeze state, folosește Pinia. Dacă e state local reutilizabil, composable.
Dacă e state de sub-arbore, provide/inject.

### Întrebarea 12: Cum gestionezi error handling global în stores?

**Răspuns**: Mai multe strategii complementare:

```typescript
// 1. Plugin global de error handling
function piniaErrorHandler({ store }: PiniaPluginContext) {
  store.$onAction(({ name, onError }) => {
    onError((error) => {
      const uiStore = useUIStore()

      if (error instanceof AuthError) {
        // Force logout
        const authStore = useAuthStore()
        authStore.logout()
        uiStore.showError('Sesiunea a expirat. Te rugăm să te loghezi din nou.')
      } else if (error instanceof NetworkError) {
        uiStore.showError('Eroare de rețea. Verifică conexiunea.')
      } else {
        uiStore.showError(`Eroare în ${store.$id}.${name}: ${error.message}`)
      }

      // Report to monitoring (Sentry, etc.)
      errorReporter.capture(error, {
        store: store.$id,
        action: name
      })
    })
  })
}
```

```typescript
// 2. Composable de error handling reutilizabil
function useAsyncAction() {
  const loading = ref(false)
  const error = ref<string | null>(null)

  async function execute<T>(fn: () => Promise<T>): Promise<T | null> {
    loading.value = true
    error.value = null
    try {
      const result = await fn()
      return result
    } catch (e) {
      error.value = e instanceof Error ? e.message : 'Unknown error'
      return null
    } finally {
      loading.value = false
    }
  }

  return { loading, error, execute }
}

// Utilizare în store
export const useDataStore = defineStore('data', () => {
  const { loading, error, execute } = useAsyncAction()
  const items = ref<Item[]>([])

  async function fetchItems() {
    const result = await execute(() => api.getItems())
    if (result) {
      items.value = result
    }
  }

  return { items, loading, error, fetchItems }
})
```

Trade-off: error handling global (plugin) vs local (per store). Global e convenabil
dar poate masca erori specifice. Local e explicit dar repetitiv. Recomandare:
plugin global pentru erori comune (network, auth), handling specific per store
pentru business logic errors.

### Întrebarea 13: Cum implementezi real-time updates cu Pinia (WebSockets)?

**Răspuns**: Composable de WebSocket care alimentează store-ul:

```typescript
// composables/useWebSocket.ts
export function useStoreWebSocket(url: string) {
  const ws = ref<WebSocket | null>(null)
  const connected = ref(false)

  function connect() {
    ws.value = new WebSocket(url)

    ws.value.onopen = () => {
      connected.value = true
    }

    ws.value.onmessage = (event) => {
      const message = JSON.parse(event.data)
      handleMessage(message)
    }

    ws.value.onclose = () => {
      connected.value = false
      // Auto-reconnect
      setTimeout(connect, 3000)
    }
  }

  function handleMessage(message: WSMessage) {
    switch (message.type) {
      case 'notification': {
        const notifStore = useNotificationStore()
        notifStore.addNotification(message.payload)
        break
      }
      case 'order:updated': {
        const orderStore = useOrderStore()
        orderStore.updateOrder(message.payload)
        break
      }
      case 'chat:message': {
        const chatStore = useChatStore()
        chatStore.addMessage(message.payload)
        break
      }
    }
  }

  function disconnect() {
    ws.value?.close()
    ws.value = null
    connected.value = false
  }

  return { connected, connect, disconnect }
}
```

Principiul e separarea: WebSocket handler-ul e un composable/service independent,
store-urile rămân pure state containers. Handler-ul apelează actions pe stores.
Asta e echivalentul NgRx Effects care ascultă WebSocket events și dispatch-uiesc
actions.

### Întrebarea 14: Cum gestionezi form state - Pinia vs local state?

**Răspuns**: Form state ar trebui să fie aproape întotdeauna LOCAL (în componentă
sau composable). Pinia e pentru state partajat, iar un form e de obicei scoped
la o singură componentă.

Excepții unde Pinia e justificat:
- **Multi-step wizard** - form state trebuie să persiste între componente/rute
- **Draft auto-save** - state-ul formularului trebuie supraviețuit unui refresh
- **Collaborative editing** - mai mulți utilizatori editează același document

```typescript
// Multi-step wizard - justifică Pinia
export const useWizardStore = defineStore('wizard', () => {
  const step = ref(1)
  const formData = ref<WizardFormData>({
    // Step 1
    personalInfo: { name: '', email: '' },
    // Step 2
    addressInfo: { street: '', city: '' },
    // Step 3
    paymentInfo: { method: 'card' }
  })

  function updateStep(stepNum: number, data: Partial<WizardFormData>) {
    Object.assign(formData.value, data)
    step.value = stepNum
  }

  function nextStep() { step.value++ }
  function prevStep() { step.value-- }

  function $reset() {
    step.value = 1
    formData.value = { /* initial values */ }
  }

  return { step, formData, updateStep, nextStep, prevStep, $reset }
}, {
  persist: {
    key: 'wizard-draft',
    storage: sessionStorage
  }
})
```

### Întrebarea 15: Care sunt anti-pattern-urile comune cu Pinia?

**Răspuns**:

1. **God Store** - un singur store cu tot state-ul aplicației. Soluție: split
   pe domenii/features, maximum 200-300 linii per store.

2. **Store pentru local state** - form inputs, toggle-uri locale puse în Pinia.
   Soluție: local state rămâne local, Pinia doar pentru state partajat.

3. **Business logic în store** - calculări complexe, validări, transformări direct
   în actions. Soluție: extrage în services/composables, store-ul doar stochează.

4. **Destructurare fără storeToRefs()** - pierderea reactivității. Soluție:
   linting rule custom sau code review checklist.

5. **Dependențe circulare între stores** - storeA importă storeB care importă storeA
   la top level. Soluție: lazy imports în actions, nu la definire.

6. **Mutații directe din componente** - `store.items.push(newItem)` din template
   sau script. Soluție: mereu prin actions, chiar și pentru mutații simple.
   Asta menține predictabilitatea și permite interceptare cu $onAction().

7. **Prea multe stores fine-grained** - un store per entitate mică. Soluție:
   grupează pe domeniu, un store pentru întregul feature cu sub-state.

---

## Rezumat rapid pentru interviu

| Concept | Ce trebuie să știi |
|---|---|
| **Pinia vs Vuex** | Pinia = Vuex 5, fără mutations, flat stores, TS nativ |
| **Setup vs Option** | Setup preferat (private state, watchers, composables) |
| **storeToRefs()** | Obligatoriu la destructurare state/getters, NU pentru actions |
| **Store composition** | useOtherStore() în setup, lazy în actions pt circular deps |
| **Getters** | computed() în setup, memoizate automat, factory pt argumente |
| **Actions** | Sync + async, optimistic updates, $patch pentru batch |
| **Plugins** | Persistence, logging, undo/redo - funcții simple pe context |
| **Patterns** | Feature stores, composable+store, normalized state, orchestrator |
| **vs NgRx** | 3x mai puțin cod, similar ca beneficii, fără enforced immutability |
| **vs Angular Services** | Foarte similar, dar cu DevTools și plugin ecosystem |
| **MFE** | Izolare per default, events pentru comunicare, namespace IDs |
| **Testing** | Simplu: createPinia() + setActivePinia() + instanțiezi store |


---

**Următor :** [**05 - Micro-Frontenduri & Module Federation** →](Vue/05-Micro-Frontenduri-Module-Federation.md)