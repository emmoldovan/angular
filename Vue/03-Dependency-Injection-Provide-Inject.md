# Dependency Injection in Vue 3 - provide/inject (Interview Prep - Senior Frontend Architect)

> provide() si inject() in Vue 3, Symbol keys, app-level provides,
> reactive provides. Comparatie detaliata cu Angular DI hierarchy.
> Pregatit pentru candidati cu experienta solida in Angular.

---

## Cuprins

1. [Ce este Dependency Injection si de ce](#1-ce-este-dependency-injection-si-de-ce)
2. [provide() si inject() - Basics](#2-provide-si-inject---basics)
3. [App-level provides](#3-app-level-provides)
4. [Component-level provides](#4-component-level-provides)
5. [Symbol keys pentru collision avoidance](#5-symbol-keys-pentru-collision-avoidance)
6. [Reactive provides](#6-reactive-provides)
7. [Factory pattern cu provide/inject](#7-factory-pattern-cu-provideinject)
8. [Paralela completa: Angular DI vs Vue provide/inject](#8-paralela-completa-angular-di-vs-vue-provideinject)
9. [Cand folosesti provide/inject vs Pinia vs Props](#9-cand-folosesti-provideinject-vs-pinia-vs-props)
10. [Intrebari de interviu](#10-intrebari-de-interviu)

---

## 1. Ce este Dependency Injection si de ce

### 1.1 Definitie

**Dependency Injection (DI)** este un design pattern in care un obiect sau functie primeste dependentele de care are nevoie din exterior, in loc sa le creeze singur. In contextul frontend-ului, DI rezolva problema transmiterii datelor si serviciilor prin mai multe niveluri de componente.

### 1.2 Problema pe care o rezolva - Prop Drilling

```
App
 └── Layout
      └── Sidebar
           └── UserMenu
                └── Avatar   <-- Trebuie sa stie tema curenta
```

**Fara DI (prop drilling):**

```vue
<!-- App.vue -->
<template>
  <Layout :theme="theme" />
</template>

<!-- Layout.vue -->
<template>
  <Sidebar :theme="theme" />   <!-- doar paseaza mai departe -->
</template>

<!-- Sidebar.vue -->
<template>
  <UserMenu :theme="theme" />  <!-- doar paseaza mai departe -->
</template>

<!-- UserMenu.vue -->
<template>
  <Avatar :theme="theme" />    <!-- doar paseaza mai departe -->
</template>

<!-- Avatar.vue - singurul care FOLOSESTE theme -->
<template>
  <img :class="theme" />
</template>
```

**Probleme cu prop drilling:**
- Componentele intermediare (Layout, Sidebar, UserMenu) **nu folosesc** `theme`, doar il paseaza
- Orice schimbare in structura necesita modificari in **toate** componentele din lant
- Semnatura componentelor devine poluata cu props irelevante
- Refactorizarea este costisitoare si fragila

### 1.3 Solutia DI in Vue - provide/inject

```
App (provide: theme)
 └── Layout              <-- nu stie de theme
      └── Sidebar        <-- nu stie de theme
           └── UserMenu  <-- nu stie de theme
                └── Avatar (inject: theme)  <-- consuma direct
```

**Cu provide/inject:**
- Doar **provider-ul** si **consumatorul** stiu despre dependenta
- Componentele intermediare raman **curate**
- Structura poate fi refactorizata **liber**

### 1.4 Vue provide/inject vs Angular DI - Diferente fundamentale

| Aspect | Angular DI | Vue provide/inject |
|--------|-----------|-------------------|
| Complexitate | Foarte puternic, complex | Simplu, minimalist |
| Instantiere | Automata (clasa + constructor) | Manuala (tu creezi obiectul) |
| Rezolvare | Automata prin constructor injection | Explicita prin `inject()` |
| Module-level | Da (NgModule providers) | Nu (Vue nu are modules) |
| Built-in patterns | Factory, Value, Class, Existing | Doar key-value pairs |

> **Nota pentru Angular devs:** In Angular, DI este **coloana vertebrala** a framework-ului.
> In Vue, provide/inject este **un tool** printre altele. Nu inlocuieste Pinia, composables,
> sau props - le **complementeaza**.

---

## 2. provide() si inject() - Basics

### 2.1 Sintaxa fundamentala

**provide()** - furnizeaza o valoare intr-un component (sau app root).
**inject()** - consuma valoarea din cel mai apropiat ancestor care o furnizeaza.

```vue
<!-- ParentComponent.vue - Provider -->
<script setup lang="ts">
import { provide, ref } from 'vue'

const theme = ref<'dark' | 'light'>('dark')
const locale = ref<string>('ro')

// provide(key, value)
provide('theme', theme)      // key = string, value = Ref<string>
provide('locale', locale)    // fiecare provide e un key-value pair
</script>

<template>
  <div>
    <h1>Parent Component</h1>
    <!-- Orice copil, nepot, etc. poate inject 'theme' sau 'locale' -->
    <MiddleComponent />
  </div>
</template>
```

```vue
<!-- MiddleComponent.vue - NU stie despre theme/locale -->
<script setup lang="ts">
// Nu importa nimic legat de theme/locale
</script>

<template>
  <div>
    <h2>Middle Component</h2>
    <DeepChildComponent />
  </div>
</template>
```

```vue
<!-- DeepChildComponent.vue - Consumer (poate fi la orice nivel sub Parent) -->
<script setup lang="ts">
import { inject } from 'vue'
import type { Ref } from 'vue'

// Varianta 1: Fara default - poate fi undefined
const theme = inject<Ref<string>>('theme')
// Type: Ref<string> | undefined

// Varianta 2: Cu default value
const themeWithDefault = inject<Ref<string>>('theme', ref('light'))
// Type: Ref<string>

// Varianta 3: Non-null assertion (cand esti SIGUR ca exista provider)
const themeAsserted = inject('theme')!
// Type: string (pierde tipul corect fara generic)

// Varianta 4: Cu factory function ca default (lazy evaluation)
const themeFactory = inject<Ref<string>>('theme', () => ref('light'), true)
// Al treilea argument `true` indica ca default-ul este o factory function
</script>

<template>
  <div :class="theme">
    <p>Tema curenta: {{ theme }}</p>
  </div>
</template>
```

### 2.2 Cum functioneaza rezolvarea (Resolution)

```
inject('theme') cauta:
  1. Componenta curenta -> NU (nu e provider pentru sine insusi)
  2. Parintele direct -> Are provide('theme')? -> DA -> Returneaza valoarea
                                                -> NU -> Continua in sus
  3. Bunicul -> Are provide('theme')? -> DA -> Returneaza valoarea
                                        -> NU -> Continua in sus
  4. ... tot in sus ...
  5. App root -> Are app.provide('theme')? -> DA -> Returneaza valoarea
                                             -> NU -> Returneaza undefined / default
```

**Key points:**
- inject() cauta **in sus** in arborele de componente (nearest parent first)
- Daca nu gaseste provider, returneaza **undefined** (sau default value daca e specificata)
- **NU** functioneaza "in jos" sau "lateral" - doar **ascendent**
- Daca mai multi ancestori furnizeaza aceeasi cheie, **cel mai apropiat** castiga (shadowing)

### 2.3 Shadowing (Override pe nivel)

```vue
<!-- GrandParent.vue -->
<script setup lang="ts">
import { provide } from 'vue'
provide('color', 'red')
</script>

<!-- Parent.vue -->
<script setup lang="ts">
import { provide } from 'vue'
provide('color', 'blue')  // Suprascrie 'red' pentru tot subtree-ul
</script>

<!-- Child.vue -->
<script setup lang="ts">
import { inject } from 'vue'
const color = inject('color')  // 'blue' (cel mai apropiat provider)
</script>
```

### Paralela cu Angular

In Angular, shadowing functioneaza similar prin **hierarchical injectors**:

```typescript
// Angular - Hierarchical DI
@Component({
  providers: [{ provide: ColorService, useValue: 'red' }]  // GrandParent
})
export class GrandParentComponent {}

@Component({
  providers: [{ provide: ColorService, useValue: 'blue' }]  // Parent
})
export class ParentComponent {}

@Component({})
export class ChildComponent {
  // Primeste 'blue' - cel mai apropiat injector
  constructor(private color: ColorService) {}
}
```

**Diferenta cheie:** In Angular, injector hierarchy include si **Module Injectors** si **Element Injectors**. In Vue, exista doar **component tree** hierarchy.

---

## 3. App-level provides

### 3.1 Configurare in main.ts

**App-level provides** sunt disponibile in **TOATE** componentele aplicatiei.

```typescript
// main.ts
import { createApp } from 'vue'
import { createPinia } from 'pinia'
import App from './App.vue'
import type { AppConfig } from './types'

const app = createApp(App)

// ---- App-level provides ----

// Valori simple
app.provide('apiUrl', 'https://api.example.com')
app.provide('appVersion', '2.1.0')
app.provide('environment', import.meta.env.MODE)

// Obiecte complexe
const appConfig: AppConfig = {
  apiUrl: import.meta.env.VITE_API_URL,
  wsUrl: import.meta.env.VITE_WS_URL,
  maxRetries: 3,
  timeout: 30000,
  features: {
    darkMode: true,
    notifications: true,
    analytics: import.meta.env.PROD
  }
}
app.provide('appConfig', appConfig)

// Plugins
app.use(createPinia())

app.mount('#app')
```

### 3.2 Consum in orice componenta

```vue
<!-- Oriunde in aplicatie -->
<script setup lang="ts">
import { inject } from 'vue'
import type { AppConfig } from '@/types'

const apiUrl = inject<string>('apiUrl')
const config = inject<AppConfig>('appConfig')

// Folosire
async function fetchData() {
  const response = await fetch(`${apiUrl}/users`)
  // ...
}
</script>
```

### 3.3 Use cases tipice pentru app-level provides

| Use Case | Exemplu | De ce app-level |
|----------|---------|-----------------|
| API Configuration | URL-uri, timeout-uri | Toate componentele au nevoie |
| Tema aplicatiei | `theme: 'dark'` | Global, rar se schimba |
| Locale / i18n | `locale: 'ro'` | Afecteaza toata aplicatia |
| Feature flags | `{ darkMode: true }` | Disponibile peste tot |
| Logger instance | `logger: new Logger()` | Cross-cutting concern |
| Auth state simplu | `currentUser` | Acces universal |

### 3.4 Pattern: Plugin cu app.provide()

```typescript
// plugins/analytics.ts
import type { App } from 'vue'
import type { InjectionKey } from 'vue'

export interface AnalyticsService {
  track(event: string, properties?: Record<string, unknown>): void
  identify(userId: string, traits?: Record<string, unknown>): void
  page(name: string): void
}

export const AnalyticsKey: InjectionKey<AnalyticsService> = Symbol('analytics')

export function createAnalyticsPlugin(config: { apiKey: string }) {
  return {
    install(app: App) {
      const analytics: AnalyticsService = {
        track(event, properties) {
          console.log(`[Analytics] Track: ${event}`, properties)
          // Implementare reala cu Mixpanel/Amplitude/etc.
        },
        identify(userId, traits) {
          console.log(`[Analytics] Identify: ${userId}`, traits)
        },
        page(name) {
          console.log(`[Analytics] Page: ${name}`)
        }
      }

      app.provide(AnalyticsKey, analytics)
    }
  }
}

// main.ts
import { createAnalyticsPlugin } from './plugins/analytics'

app.use(createAnalyticsPlugin({ apiKey: 'ak_123' }))

// Orice componenta
import { inject } from 'vue'
import { AnalyticsKey } from '@/plugins/analytics'

const analytics = inject(AnalyticsKey)!
analytics.track('button_click', { button: 'submit' })
```

### Paralela cu Angular

| Vue app.provide() | Angular equivalent |
|-------------------|-------------------|
| `app.provide('apiUrl', url)` | `{ provide: 'API_URL', useValue: url }` in root module |
| `app.provide(AnalyticsKey, service)` | `providedIn: 'root'` pe serviciu |
| Plugin cu `install()` | `@NgModule({ providers: [...] })` |
| Disponibil in toate componentele | Disponibil in toate componentele |

**Diferenta:** In Angular, `providedIn: 'root'` este **tree-shakable** - daca nimeni nu injecteaza serviciul, el nu ajunge in bundle. In Vue, `app.provide()` furnizeaza valoarea **intotdeauna**, dar daca valoarea este un obiect creat manual, tree-shaking depinde de cum e structurat codul.

---

## 4. Component-level provides

### 4.1 Provide la nivel de componenta

Cand folosesti `provide()` intr-un component (nu in `main.ts`), valoarea este disponibila **doar in subtree-ul** acelui component.

```vue
<!-- UserDashboard.vue - Provider scoped -->
<script setup lang="ts">
import { provide, ref, reactive } from 'vue'

// Aceste valori sunt disponibile DOAR in copiii lui UserDashboard
const dashboardState = reactive({
  selectedTab: 'overview',
  isLoading: false,
  filters: {
    dateRange: 'last7days',
    status: 'all'
  }
})

const notifications = ref<Notification[]>([])

provide('dashboardState', dashboardState)
provide('notifications', notifications)
</script>

<template>
  <div class="dashboard">
    <DashboardHeader />
    <DashboardSidebar />
    <DashboardContent />
    <!-- Toate aceste componente (si copiii lor) pot inject dashboardState -->
  </div>
</template>
```

```vue
<!-- DashboardSidebar.vue - Consumer -->
<script setup lang="ts">
import { inject } from 'vue'

const state = inject('dashboardState')!

function selectTab(tab: string) {
  state.selectedTab = tab  // reactive - se propaga automat
}
</script>

<template>
  <nav>
    <button
      v-for="tab in ['overview', 'analytics', 'settings']"
      :key="tab"
      :class="{ active: state.selectedTab === tab }"
      @click="selectTab(tab)"
    >
      {{ tab }}
    </button>
  </nav>
</template>
```

### 4.2 Fiecare instanta = scope nou

**Important:** Daca componenta provider este instantiata de mai multe ori, **fiecare instanta** creaza propriul scope de provide.

```vue
<!-- ProductCard.vue - Provider cu scope per instanta -->
<script setup lang="ts">
import { provide, reactive } from 'vue'

interface Props {
  product: Product
}

const props = defineProps<Props>()

// Fiecare ProductCard furnizeaza propriul product context
const productContext = reactive({
  product: props.product,
  isInCart: false,
  quantity: 0
})

provide('productContext', productContext)
</script>

<template>
  <div class="product-card">
    <ProductImage />
    <ProductDetails />
    <AddToCartButton />
    <!-- Fiecare grup primeste product-ul PROPRIEI instante ProductCard -->
  </div>
</template>
```

```vue
<!-- Utilizare - doua instante cu scope-uri separate -->
<template>
  <div class="product-grid">
    <!-- Instanta 1: productContext = { product: laptop, ... } -->
    <ProductCard :product="laptop" />

    <!-- Instanta 2: productContext = { product: phone, ... } -->
    <ProductCard :product="phone" />
  </div>
</template>
```

### 4.3 Pattern: Compound Components cu provide/inject

```vue
<!-- Tabs.vue - Parent Provider -->
<script setup lang="ts">
import { provide, ref } from 'vue'
import type { InjectionKey, Ref } from 'vue'

export interface TabsContext {
  activeTab: Ref<string>
  setActiveTab: (id: string) => void
  registerTab: (id: string, label: string) => void
}

export const TabsKey: InjectionKey<TabsContext> = Symbol('tabs')

const activeTab = ref<string>('')
const tabs = ref<Array<{ id: string; label: string }>>([])

provide(TabsKey, {
  activeTab,
  setActiveTab: (id: string) => { activeTab.value = id },
  registerTab: (id: string, label: string) => {
    if (!tabs.value.find(t => t.id === id)) {
      tabs.value.push({ id, label })
    }
  }
})
</script>

<template>
  <div class="tabs">
    <slot />
  </div>
</template>
```

```vue
<!-- Tab.vue - Child Consumer -->
<script setup lang="ts">
import { inject, onMounted } from 'vue'
import { TabsKey } from './Tabs.vue'

interface Props {
  id: string
  label: string
}

const props = defineProps<Props>()
const tabsContext = inject(TabsKey)!

onMounted(() => {
  tabsContext.registerTab(props.id, props.label)
})
</script>

<template>
  <div
    v-show="tabsContext.activeTab.value === id"
    class="tab-panel"
  >
    <slot />
  </div>
</template>
```

```vue
<!-- Utilizare -->
<template>
  <Tabs>
    <Tab id="general" label="General">
      <p>Continut general</p>
    </Tab>
    <Tab id="security" label="Securitate">
      <p>Setari de securitate</p>
    </Tab>
    <Tab id="billing" label="Facturare">
      <p>Informatii facturare</p>
    </Tab>
  </Tabs>
</template>
```

### Paralela cu Angular

In Angular, **component-level providers** functioneaza identic conceptual:

```typescript
// Angular - Component-level DI
@Injectable()
export class DashboardStateService {
  selectedTab = signal('overview');
  isLoading = signal(false);
}

@Component({
  selector: 'app-user-dashboard',
  providers: [DashboardStateService],  // <-- Scoped la acest component
  template: `
    <app-dashboard-header />
    <app-dashboard-sidebar />
    <app-dashboard-content />
  `
})
export class UserDashboardComponent {
  constructor(private state: DashboardStateService) {}
}

// Fiecare instanta a UserDashboardComponent primeste propria
// instanta de DashboardStateService
```

**Diferente:**
- Angular **instantiaza automat** serviciul cand e declarat in `providers`
- Vue necesita **crearea manuala** a obiectului in `provide()`
- Angular suporta **viewProviders** (vizibil doar in template, nu in content projection) - Vue nu are echivalent direct

---

## 5. Symbol keys pentru collision avoidance

### 5.1 Problema cu string keys

```typescript
// plugin-a.ts
provide('config', { color: 'red' })

// plugin-b.ts
provide('config', { apiKey: '123' })  // COLIZIUNE! Suprascrie config-ul din plugin-a
```

Cand aplicatia creste si foloseste mai multe librarii/plugin-uri, **string keys** pot cauza coliziuni neintentionate.

### 5.2 Symbol() ca key

```typescript
// Fiecare Symbol() este UNIC - nu poate fi duplicat
const key1 = Symbol('config')
const key2 = Symbol('config')

console.log(key1 === key2)  // false - chiar daca au aceeasi descriere
```

### 5.3 InjectionKey<T> - Type-safe keys

Vue ofera tipul **InjectionKey<T>** care combina unicitatea Symbol cu type safety:

```typescript
// injection-keys.ts
import type { InjectionKey, Ref, ComputedRef } from 'vue'

// ---- Definire tipuri ----

export interface UserContext {
  id: string
  name: string
  email: string
  role: 'admin' | 'editor' | 'viewer'
  permissions: string[]
}

export interface ThemeContext {
  mode: 'dark' | 'light' | 'system'
  primaryColor: string
  fontSize: 'sm' | 'md' | 'lg'
}

export interface NotificationContext {
  notifications: Ref<Notification[]>
  unreadCount: ComputedRef<number>
  markAsRead: (id: string) => void
  markAllAsRead: () => void
  dismiss: (id: string) => void
}

// ---- Definire keys ----

export const UserKey: InjectionKey<Ref<UserContext>> = Symbol('user')
export const ThemeKey: InjectionKey<ThemeContext> = Symbol('theme')
export const NotificationKey: InjectionKey<NotificationContext> = Symbol('notifications')
export const ApiUrlKey: InjectionKey<string> = Symbol('apiUrl')
export const LoggerKey: InjectionKey<Logger> = Symbol('logger')
```

### 5.4 Utilizare cu InjectionKey

```vue
<!-- Provider.vue -->
<script setup lang="ts">
import { provide, ref, reactive } from 'vue'
import { UserKey, ThemeKey } from '@/injection-keys'
import type { UserContext, ThemeContext } from '@/injection-keys'

// TypeScript stie ca valoarea trebuie sa fie Ref<UserContext>
provide(UserKey, ref<UserContext>({
  id: '1',
  name: 'Emanuel',
  email: 'emanuel@example.com',
  role: 'admin',
  permissions: ['read', 'write', 'delete']
}))

// TypeScript stie ca valoarea trebuie sa fie ThemeContext
provide(ThemeKey, reactive<ThemeContext>({
  mode: 'dark',
  primaryColor: '#3b82f6',
  fontSize: 'md'
}))

// EROARE TypeScript - tipul nu se potriveste:
// provide(UserKey, { wrong: 'data' })  // TS Error
</script>
```

```vue
<!-- Consumer.vue -->
<script setup lang="ts">
import { inject } from 'vue'
import { UserKey, ThemeKey } from '@/injection-keys'

// Fully typed! TypeScript stie exact ce tipuri primesti
const user = inject(UserKey)
// Type: Ref<UserContext> | undefined

const theme = inject(ThemeKey)
// Type: ThemeContext | undefined

// Acces type-safe
if (user?.value) {
  console.log(user.value.name)        // OK
  console.log(user.value.role)        // OK
  // console.log(user.value.age)      // TS Error - nu exista 'age'
}

if (theme) {
  console.log(theme.mode)             // OK - 'dark' | 'light' | 'system'
  console.log(theme.primaryColor)     // OK - string
}
</script>
```

### 5.5 Organizare keys in proiecte mari

```
src/
  injection-keys/
    index.ts          # Re-export all keys
    auth.keys.ts      # Auth-related keys
    theme.keys.ts     # Theme-related keys
    api.keys.ts       # API-related keys
    feature.keys.ts   # Feature flag keys
```

```typescript
// injection-keys/auth.keys.ts
import type { InjectionKey, Ref, ComputedRef } from 'vue'

export interface AuthState {
  user: Ref<User | null>
  token: Ref<string | null>
  isAuthenticated: ComputedRef<boolean>
  isAdmin: ComputedRef<boolean>
  login: (credentials: LoginCredentials) => Promise<void>
  logout: () => Promise<void>
  refreshToken: () => Promise<void>
}

export const AuthKey: InjectionKey<AuthState> = Symbol('auth')
```

```typescript
// injection-keys/index.ts
export { AuthKey } from './auth.keys'
export type { AuthState } from './auth.keys'

export { ThemeKey } from './theme.keys'
export type { ThemeContext } from './theme.keys'

export { ApiKey } from './api.keys'
export type { ApiConfig } from './api.keys'
```

### Paralela cu Angular

| Vue InjectionKey<T> | Angular InjectionToken<T> |
|---------------------|--------------------------|
| `Symbol('auth')` | `new InjectionToken<AuthService>('auth')` |
| Type-safe la provide si inject | Type-safe la provide si inject |
| Doar tip (nu are factory) | Suporta `factory` function |
| Exportat ca constanta | Exportat ca constanta |

```typescript
// Angular equivalent
import { InjectionToken } from '@angular/core'

export interface AuthState {
  user: Signal<User | null>
  isAuthenticated: Signal<boolean>
  login: (credentials: LoginCredentials) => Promise<void>
  logout: () => Promise<void>
}

// Angular InjectionToken poate avea factory default
export const AUTH_TOKEN = new InjectionToken<AuthState>('auth', {
  providedIn: 'root',
  factory: () => {
    // Creat automat daca nimeni nu il overridde
    return createDefaultAuthState()
  }
})

// Vue InjectionKey NU are factory - trebuie furnizat explicit
export const AuthKey: InjectionKey<AuthState> = Symbol('auth')
```

---

## 6. Reactive provides

### 6.1 Provide cu ref() - valori reactive

Cand furnizezi un `ref()`, consumatorii primesc **aceeasi referinta reactiva**. Orice schimbare in provider se propaga automat.

```vue
<!-- ThemeProvider.vue -->
<script setup lang="ts">
import { provide, ref } from 'vue'
import { ThemeKey } from '@/injection-keys'

const currentTheme = ref<'dark' | 'light'>('dark')

provide(ThemeKey, currentTheme)

function toggleTheme() {
  currentTheme.value = currentTheme.value === 'dark' ? 'light' : 'dark'
}
</script>

<template>
  <div :class="`theme-${currentTheme}`">
    <button @click="toggleTheme">Toggle Theme</button>
    <slot />  <!-- Copiii primesc reactive ref -->
  </div>
</template>
```

```vue
<!-- DeepChild.vue -->
<script setup lang="ts">
import { inject, watch } from 'vue'
import { ThemeKey } from '@/injection-keys'

const theme = inject(ThemeKey)!

// Reactiv - se declanseaza cand provider-ul schimba tema
watch(theme, (newTheme) => {
  console.log('Tema s-a schimbat:', newTheme)
})
</script>

<template>
  <!-- Se actualizeaza automat cand provider schimba tema -->
  <div :class="`card-${theme}`">
    <p>Tema curenta: {{ theme }}</p>
  </div>
</template>
```

### 6.2 Provide cu reactive()

```vue
<!-- StateProvider.vue -->
<script setup lang="ts">
import { provide, reactive } from 'vue'

interface AppState {
  user: { name: string; role: string } | null
  preferences: {
    theme: string
    language: string
    notifications: boolean
  }
  isLoading: boolean
}

const state = reactive<AppState>({
  user: null,
  preferences: {
    theme: 'dark',
    language: 'ro',
    notifications: true
  },
  isLoading: false
})

provide('appState', state)
</script>
```

### 6.3 Provide cu computed() - readonly reactive

```vue
<!-- Provider.vue -->
<script setup lang="ts">
import { provide, ref, computed } from 'vue'

const items = ref<CartItem[]>([])

// Consumatorii primesc o valoare reactiva READONLY
provide('cartCount', computed(() => items.value.length))
provide('cartTotal', computed(() =>
  items.value.reduce((sum, item) => sum + item.price * item.quantity, 0)
))
</script>
```

```vue
<!-- Consumer.vue -->
<script setup lang="ts">
import { inject } from 'vue'
import type { ComputedRef } from 'vue'

const cartCount = inject<ComputedRef<number>>('cartCount')!
const cartTotal = inject<ComputedRef<number>>('cartTotal')!

// cartCount.value = 10  // NU merge - computed este readonly
console.log(cartCount.value)  // OK - citire
</script>
```

### 6.4 BEST PRACTICE: readonly + update methods

**Regula de aur:** Furnizeaza valori **readonly** si **metode explicite** de actualizare. Aceasta previne mutatii necontrolate din componente copil.

```typescript
// composables/useThemeProvider.ts
import { ref, readonly, computed, provide } from 'vue'
import type { InjectionKey, Ref, ComputedRef } from 'vue'

export interface ThemeState {
  // Readonly values
  mode: Readonly<Ref<'dark' | 'light' | 'system'>>
  primaryColor: Readonly<Ref<string>>
  isDark: ComputedRef<boolean>

  // Update methods
  setMode: (mode: 'dark' | 'light' | 'system') => void
  setPrimaryColor: (color: string) => void
  toggleMode: () => void
}

export const ThemeKey: InjectionKey<ThemeState> = Symbol('theme')

export function provideTheme(initialMode: 'dark' | 'light' | 'system' = 'dark') {
  const mode = ref<'dark' | 'light' | 'system'>(initialMode)
  const primaryColor = ref('#3b82f6')

  const themeState: ThemeState = {
    // Readonly - consumatorii NU pot modifica direct
    mode: readonly(mode),
    primaryColor: readonly(primaryColor),
    isDark: computed(() => {
      if (mode.value === 'system') {
        return window.matchMedia('(prefers-color-scheme: dark)').matches
      }
      return mode.value === 'dark'
    }),

    // Update methods - singura modalitate de a schimba starea
    setMode: (newMode) => { mode.value = newMode },
    setPrimaryColor: (color) => { primaryColor.value = color },
    toggleMode: () => {
      mode.value = mode.value === 'dark' ? 'light' : 'dark'
    }
  }

  provide(ThemeKey, themeState)
  return themeState
}
```

```vue
<!-- App.vue -->
<script setup lang="ts">
import { provideTheme } from '@/composables/useThemeProvider'

const theme = provideTheme('dark')
</script>

<template>
  <div :class="{ dark: theme.isDark.value }">
    <router-view />
  </div>
</template>
```

```vue
<!-- SettingsPanel.vue -->
<script setup lang="ts">
import { inject } from 'vue'
import { ThemeKey } from '@/composables/useThemeProvider'

const theme = inject(ThemeKey)!

// theme.mode.value = 'light'  // TypeScript Error! Readonly
theme.setMode('light')          // OK - prin metoda expusa
theme.toggleMode()              // OK
</script>

<template>
  <div>
    <h3>Tema: {{ theme.mode.value }}</h3>
    <p>Dark mode: {{ theme.isDark.value ? 'Da' : 'Nu' }}</p>

    <button @click="theme.setMode('dark')">Dark</button>
    <button @click="theme.setMode('light')">Light</button>
    <button @click="theme.setMode('system')">System</button>
    <button @click="theme.toggleMode()">Toggle</button>
  </div>
</template>
```

### 6.5 Anti-pattern: Mutatie directa fara readonly

```vue
<!-- RAUUUU - NU face asta -->
<script setup lang="ts">
import { provide, ref } from 'vue'

const user = ref({ name: 'Emanuel', role: 'admin' })
provide('user', user)  // Orice copil poate modifica direct!
</script>

<!-- Child.vue - poate face orice -->
<script setup lang="ts">
const user = inject('user')!
user.value.role = 'viewer'  // Mutatie necontrolata!
user.value = null            // Distruge tot state-ul!
</script>
```

```vue
<!-- BINE - readonly + metode -->
<script setup lang="ts">
import { provide, ref, readonly } from 'vue'

const user = ref({ name: 'Emanuel', role: 'admin' })
provide('user', {
  current: readonly(user),
  updateRole: (role: string) => { user.value.role = role }
})
</script>
```

### Paralela cu Angular

Pattern-ul `readonly` + `update methods` este **identic conceptual** cu Angular services care expun Observables:

```typescript
// Angular - pattern similar
@Injectable({ providedIn: 'root' })
export class ThemeService {
  private modeSubject = new BehaviorSubject<'dark' | 'light'>('dark');

  // Readonly (Observable, nu Subject)
  readonly mode$ = this.modeSubject.asObservable();
  readonly isDark$ = this.mode$.pipe(map(m => m === 'dark'));

  // Update methods
  setMode(mode: 'dark' | 'light') {
    this.modeSubject.next(mode);
  }

  toggleMode() {
    const current = this.modeSubject.getValue();
    this.modeSubject.next(current === 'dark' ? 'light' : 'dark');
  }
}

// Cu Signals (Angular 16+) - si mai similar cu Vue
@Injectable({ providedIn: 'root' })
export class ThemeService {
  private _mode = signal<'dark' | 'light'>('dark');

  readonly mode = this._mode.asReadonly();     // Similar cu readonly(ref)
  readonly isDark = computed(() => this._mode() === 'dark');

  setMode(mode: 'dark' | 'light') {
    this._mode.set(mode);
  }
}
```

---

## 7. Factory pattern cu provide/inject

### 7.1 Composable pattern: createXxxProvider() + useXxx()

Acesta este **pattern-ul principal** pentru DI avansat in Vue. Combina `provide/inject` cu **composables** pentru a obtine ceva similar cu Angular services.

```typescript
// composables/auth/types.ts
export interface User {
  id: string
  name: string
  email: string
  role: 'admin' | 'editor' | 'viewer'
  avatar?: string
}

export interface LoginCredentials {
  email: string
  password: string
}

export interface AuthService {
  // State (readonly)
  user: Readonly<Ref<User | null>>
  token: Readonly<Ref<string | null>>
  isAuthenticated: ComputedRef<boolean>
  isAdmin: ComputedRef<boolean>
  isLoading: Ref<boolean>
  error: Ref<string | null>

  // Actions
  login: (credentials: LoginCredentials) => Promise<void>
  logout: () => Promise<void>
  refreshToken: () => Promise<string>
  updateProfile: (data: Partial<User>) => Promise<void>
}
```

```typescript
// composables/auth/useAuth.ts
import { ref, computed, readonly, provide, inject } from 'vue'
import type { InjectionKey, Ref, ComputedRef } from 'vue'
import type { User, LoginCredentials, AuthService } from './types'

// Key - unic si type-safe
export const AuthKey: InjectionKey<AuthService> = Symbol('auth')

/**
 * Creeaza si furnizeaza AuthService.
 * Apeleaza in componenta root (App.vue) sau in layout.
 */
export function createAuthProvider(apiUrl: string): AuthService {
  // Private state
  const user = ref<User | null>(null)
  const token = ref<string | null>(null)
  const isLoading = ref(false)
  const error = ref<string | null>(null)

  // Private helper
  async function apiCall<T>(endpoint: string, options?: RequestInit): Promise<T> {
    const headers: Record<string, string> = {
      'Content-Type': 'application/json'
    }
    if (token.value) {
      headers['Authorization'] = `Bearer ${token.value}`
    }

    const response = await fetch(`${apiUrl}${endpoint}`, {
      ...options,
      headers: { ...headers, ...options?.headers }
    })

    if (!response.ok) {
      throw new Error(`API Error: ${response.status}`)
    }

    return response.json()
  }

  // Actions
  async function login(credentials: LoginCredentials): Promise<void> {
    isLoading.value = true
    error.value = null

    try {
      const result = await apiCall<{ user: User; token: string }>(
        '/auth/login',
        {
          method: 'POST',
          body: JSON.stringify(credentials)
        }
      )
      user.value = result.user
      token.value = result.token
      localStorage.setItem('auth_token', result.token)
    } catch (e) {
      error.value = e instanceof Error ? e.message : 'Login failed'
      throw e
    } finally {
      isLoading.value = false
    }
  }

  async function logout(): Promise<void> {
    try {
      await apiCall('/auth/logout', { method: 'POST' })
    } finally {
      user.value = null
      token.value = null
      localStorage.removeItem('auth_token')
    }
  }

  async function refreshToken(): Promise<string> {
    const result = await apiCall<{ token: string }>('/auth/refresh', {
      method: 'POST'
    })
    token.value = result.token
    localStorage.setItem('auth_token', result.token)
    return result.token
  }

  async function updateProfile(data: Partial<User>): Promise<void> {
    const updated = await apiCall<User>('/auth/profile', {
      method: 'PATCH',
      body: JSON.stringify(data)
    })
    user.value = updated
  }

  // Public service object
  const authService: AuthService = {
    // Readonly state
    user: readonly(user),
    token: readonly(token),
    isAuthenticated: computed(() => !!user.value),
    isAdmin: computed(() => user.value?.role === 'admin'),
    isLoading,
    error,

    // Actions
    login,
    logout,
    refreshToken,
    updateProfile
  }

  // Provide in component tree
  provide(AuthKey, authService)

  return authService
}

/**
 * Consuma AuthService din cel mai apropiat provider.
 * Arunca eroare daca provider-ul lipseste.
 */
export function useAuth(): AuthService {
  const auth = inject(AuthKey)

  if (!auth) {
    throw new Error(
      'AuthService not found. ' +
      'Did you forget to call createAuthProvider() in a parent component?'
    )
  }

  return auth
}
```

### 7.2 Utilizare

```vue
<!-- App.vue - Setup provider -->
<script setup lang="ts">
import { createAuthProvider } from '@/composables/auth/useAuth'

// Creeaza si furnizeaza AuthService
const auth = createAuthProvider(import.meta.env.VITE_API_URL)

// Initializare - verifica token existent
const savedToken = localStorage.getItem('auth_token')
if (savedToken) {
  auth.refreshToken().catch(() => {
    // Token invalid, redirect la login
  })
}
</script>

<template>
  <RouterView />
</template>
```

```vue
<!-- LoginForm.vue - Consumer -->
<script setup lang="ts">
import { ref } from 'vue'
import { useAuth } from '@/composables/auth/useAuth'
import { useRouter } from 'vue-router'

const auth = useAuth()  // Injecteaza AuthService
const router = useRouter()

const email = ref('')
const password = ref('')

async function handleLogin() {
  try {
    await auth.login({ email: email.value, password: password.value })
    router.push('/dashboard')
  } catch {
    // auth.error contine deja mesajul
  }
}
</script>

<template>
  <form @submit.prevent="handleLogin">
    <div v-if="auth.error.value" class="error">
      {{ auth.error.value }}
    </div>

    <input v-model="email" type="email" placeholder="Email" />
    <input v-model="password" type="password" placeholder="Parola" />

    <button type="submit" :disabled="auth.isLoading.value">
      {{ auth.isLoading.value ? 'Se conecteaza...' : 'Login' }}
    </button>
  </form>
</template>
```

```vue
<!-- UserAvatar.vue - Consumer -->
<script setup lang="ts">
import { useAuth } from '@/composables/auth/useAuth'

const { user, isAuthenticated } = useAuth()
</script>

<template>
  <div v-if="isAuthenticated.value" class="avatar">
    <img :src="user.value?.avatar" :alt="user.value?.name" />
    <span>{{ user.value?.name }}</span>
  </div>
  <div v-else>
    <router-link to="/login">Login</router-link>
  </div>
</template>
```

### 7.3 Pattern avansat: Factory cu dependinte

```typescript
// composables/useApiClient.ts
import type { InjectionKey } from 'vue'

export interface ApiClient {
  get<T>(endpoint: string): Promise<T>
  post<T>(endpoint: string, data: unknown): Promise<T>
  put<T>(endpoint: string, data: unknown): Promise<T>
  delete(endpoint: string): Promise<void>
}

export const ApiClientKey: InjectionKey<ApiClient> = Symbol('apiClient')

export function createApiClientProvider(baseUrl: string): ApiClient {
  const client: ApiClient = {
    async get<T>(endpoint: string): Promise<T> {
      const res = await fetch(`${baseUrl}${endpoint}`)
      return res.json()
    },
    async post<T>(endpoint: string, data: unknown): Promise<T> {
      const res = await fetch(`${baseUrl}${endpoint}`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(data)
      })
      return res.json()
    },
    async put<T>(endpoint: string, data: unknown): Promise<T> {
      const res = await fetch(`${baseUrl}${endpoint}`, {
        method: 'PUT',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(data)
      })
      return res.json()
    },
    async delete(endpoint: string): Promise<void> {
      await fetch(`${baseUrl}${endpoint}`, { method: 'DELETE' })
    }
  }

  provide(ApiClientKey, client)
  return client
}

export function useApiClient(): ApiClient {
  const client = inject(ApiClientKey)
  if (!client) throw new Error('ApiClient not provided')
  return client
}
```

```typescript
// composables/useUserService.ts - depinde de ApiClient
import type { InjectionKey } from 'vue'
import { useApiClient } from './useApiClient'

export interface UserService {
  users: Readonly<Ref<User[]>>
  isLoading: Ref<boolean>
  fetchUsers: () => Promise<void>
  createUser: (data: CreateUserDto) => Promise<User>
  updateUser: (id: string, data: Partial<User>) => Promise<User>
  deleteUser: (id: string) => Promise<void>
}

export const UserServiceKey: InjectionKey<UserService> = Symbol('userService')

export function createUserServiceProvider(): UserService {
  // Injecteaza dependinta!
  const api = useApiClient()  // Depinde de ApiClient

  const users = ref<User[]>([])
  const isLoading = ref(false)

  const service: UserService = {
    users: readonly(users),
    isLoading,
    async fetchUsers() {
      isLoading.value = true
      try {
        users.value = await api.get<User[]>('/users')
      } finally {
        isLoading.value = false
      }
    },
    async createUser(data) {
      const user = await api.post<User>('/users', data)
      users.value.push(user)
      return user
    },
    async updateUser(id, data) {
      const updated = await api.put<User>(`/users/${id}`, data)
      const index = users.value.findIndex(u => u.id === id)
      if (index >= 0) users.value[index] = updated
      return updated
    },
    async deleteUser(id) {
      await api.delete(`/users/${id}`)
      users.value = users.value.filter(u => u.id !== id)
    }
  }

  provide(UserServiceKey, service)
  return service
}

export function useUserService(): UserService {
  const service = inject(UserServiceKey)
  if (!service) throw new Error('UserService not provided')
  return service
}
```

```vue
<!-- App.vue - Wiring -->
<script setup lang="ts">
import { createApiClientProvider } from '@/composables/useApiClient'
import { createUserServiceProvider } from '@/composables/useUserService'
import { createAuthProvider } from '@/composables/auth/useAuth'

// Ordinea conteaza! ApiClient trebuie creat inainte de UserService
createApiClientProvider(import.meta.env.VITE_API_URL)
createAuthProvider(import.meta.env.VITE_API_URL)
createUserServiceProvider()  // Injecteaza ApiClient automat
</script>
```

### Paralela cu Angular

**Factory pattern-ul** Vue de mai sus este echivalentul manual al Angular DI:

```typescript
// Angular - totul e automat
@Injectable({ providedIn: 'root' })
export class ApiClientService {
  constructor(private http: HttpClient) {}  // Injectat automat

  get<T>(endpoint: string) {
    return this.http.get<T>(`${this.baseUrl}${endpoint}`);
  }
}

@Injectable({ providedIn: 'root' })
export class UserService {
  // Angular rezolva ApiClientService automat
  constructor(private api: ApiClientService) {}

  fetchUsers() {
    return this.api.get<User[]>('/users');
  }
}
```

**Diferente esentiale:**

| Aspect | Angular | Vue |
|--------|---------|-----|
| Dependency resolution | **Automata** (constructor) | **Manuala** (inject() in factory) |
| Ordinea crearii | **Automata** (DI rezolva graficul) | **Manuala** (tu decizi ordinea) |
| Circular dependencies | Detectate de framework | Eroare runtime (stack overflow) |
| Lazy loading | Suportat nativ | Trebuie implementat manual |
| Error messages | Descriptive (NullInjectorError) | Custom (throw new Error) |

---

## 8. Paralela completa: Angular DI vs Vue provide/inject

### 8.1 Tabel comparativ detaliat

| Aspect | Angular DI | Vue provide/inject |
|--------|-----------|-------------------|
| **Registrare** | `providers` array, `providedIn` | `provide()` / `app.provide()` |
| **Token** | `InjectionToken<T>` | `InjectionKey<T>` / string |
| **Scope** | root, module, component | app, component subtree |
| **Multi providers** | Da (`multi: true`) | Nu direct (dar poti cu arrays) |
| **Factory** | `useFactory`, `deps` | Manual (composable pattern) |
| **Optional** | `@Optional()` / `inject(token, { optional: true })` | `inject(key, defaultValue)` |
| **Class instantiation** | Automat (class constructor) | Manual (trebuie sa creezi instanta) |
| **Singleton** | `providedIn: 'root'` | `app.provide()` |
| **Hierarchical** | Da (complex hierarchy) | Da (simplu - nearest parent) |
| **Tree-shakable** | Da (cu `providedIn`) | Da (by default) |
| **Testing** | `TestBed.configureTestingModule` | `app.provide()` in test setup |
| **Type safety** | Excelent | Bun (cu InjectionKey) |
| **Circular deps** | Detectate si raportate | Nu sunt detectate |
| **Lazy loading** | Suportat nativ | Manual |
| **Decorator-based** | Da (`@Injectable`, `@Inject`) | Nu (function-based) |
| **Module scope** | Da (NgModule providers) | Nu (Vue nu are modules) |
| **viewProviders** | Da | Nu |
| **Resolution modifiers** | `@Self`, `@SkipSelf`, `@Host` | Nu |
| **Inject flags** | `InjectFlags` enum | Nu |

### 8.2 Echivalente directe

```typescript
// ---- SINGLETON SERVICE ----

// Angular
@Injectable({ providedIn: 'root' })
export class NotificationService {
  private notifications = signal<Notification[]>([]);
  readonly notifications$ = this.notifications.asReadonly();
  // ...
}

// Vue
// injection-keys.ts
export const NotificationKey: InjectionKey<NotificationService> = Symbol('notifications')

// main.ts
app.provide(NotificationKey, createNotificationService())
```

```typescript
// ---- COMPONENT-SCOPED SERVICE ----

// Angular
@Component({
  providers: [FormStateService]  // Nova instanta per component
})
export class MyFormComponent {}

// Vue
// MyForm.vue
const formState = reactive({ /* ... */ })
provide(FormStateKey, formState)  // Scoped la subtree
```

```typescript
// ---- OPTIONAL INJECTION ----

// Angular
constructor(@Optional() private logger?: LoggerService) {
  if (this.logger) {
    this.logger.log('Component created');
  }
}

// Angular 14+ functional
private logger = inject(LoggerService, { optional: true });

// Vue
const logger = inject(LoggerKey, undefined)  // undefined daca lipseste
// sau cu default
const logger = inject(LoggerKey, createDefaultLogger())
```

```typescript
// ---- FACTORY PROVIDER ----

// Angular
{
  provide: API_CLIENT,
  useFactory: (http: HttpClient, config: ConfigService) => {
    return new ApiClient(http, config.apiUrl);
  },
  deps: [HttpClient, ConfigService]
}

// Vue - manual in composable
function createApiClient(): ApiClient {
  const config = inject(ConfigKey)!  // Rezolvare manuala
  // ... creeare client
  provide(ApiClientKey, client)
  return client
}
```

```typescript
// ---- MULTI PROVIDERS ----

// Angular
{ provide: HTTP_INTERCEPTORS, useClass: AuthInterceptor, multi: true },
{ provide: HTTP_INTERCEPTORS, useClass: LoggingInterceptor, multi: true },
{ provide: HTTP_INTERCEPTORS, useClass: ErrorInterceptor, multi: true }

// Vue - simulat cu array
const interceptors: HttpInterceptor[] = [
  new AuthInterceptor(),
  new LoggingInterceptor(),
  new ErrorInterceptor()
]
provide(InterceptorsKey, interceptors)
```

```typescript
// ---- RESOLUTION MODIFIERS ----

// Angular
constructor(
  @Self() private ownService: MyService,       // Doar in acest injector
  @SkipSelf() private parentService: MyService, // Doar in parinti
  @Host() private hostService: MyService        // Doar in host component
) {}

// Vue - NU are echivalent direct
// inject() cauta INTOTDEAUNA de la parent in sus
// Nu poti restrictiona la "doar parintele direct" sau "doar host"
```

### 8.3 Key differences - Ce sa subliniezi la interviu

**1. Angular DI e mult mai puternic si complex**
- Class-based cu automatic instantiation
- Factory patterns built-in (`useFactory`, `useClass`, `useValue`, `useExisting`)
- Resolution modifiers (`@Self()`, `@SkipSelf()`, `@Host()`)
- Multi providers
- Module-level si Environment injectors

**2. Vue provide/inject e simplu dar suficient**
- Function-based (nu class-based)
- Trebuie combinat cu **composables** pentru patterns avansate
- Mai putin "magic" - tu controlezi totul explicit
- Mai usor de inteles si depanat

**3. Angular DI rezolva dependente automat**
- Constructor injection - framework-ul creaza si injecteaza
- Vue necesita `inject()` explicit si creare manuala

**4. Vue NU are concept de "module-level" providers**
- Angular are NgModule providers, standalone component providers, route providers
- Vue are doar app-level si component-level

**5. Testabilitate**
- Angular: `TestBed.configureTestingModule({ providers: [...] })`
- Vue: `app.provide()` in test setup sau mock direct in composable

---

## 9. Cand folosesti provide/inject vs Pinia vs Props

### 9.1 Decision Framework

| Situatie | Solutie | De ce |
|----------|---------|-------|
| Date de la parent la child direct | **Props** | Cel mai simplu, explicit, type-safe |
| Date la 2-3 nivele adancime | **Props** (prop drilling e OK) | Simplicitate, trasabilitate |
| Date partajate in intreg subtree | **provide/inject** | Evita prop drilling |
| State global (auth, theme) simplu | **app.provide()** | Usor de configurat |
| State global complex cu devtools | **Pinia** | Devtools, SSR, plugins |
| Business logic partajata | **Composables** (fara DI) | Functii pure, testabile |
| Cross-cutting concerns | **Pinia + composables** | State + logica separata |
| Compound components (Tabs, Form) | **provide/inject** | Comunicare parent-child implicita |
| Server state (API data) | **TanStack Query** | Cache, refetch, optimistic updates |
| Form state complex | **VeeValidate / FormKit** | Validare, state management specific |

### 9.2 Flowchart de decizie

```
Datele sunt partajate intre componente?
├── NU → State local (ref/reactive in component)
└── DA → Sunt partajate doar intre parent si copii directi?
    ├── DA → Props + Events
    └── NU → Sunt partajate in tot app-ul?
        ├── DA → E state complex cu multe actiuni?
        │   ├── DA → Pinia store
        │   └── NU → app.provide() cu composable
        └── NU → E in cadrul unui subtree specific?
            ├── DA → Component-level provide/inject
            └── NU → Pinia store
```

### 9.3 Exemple concrete

**Props - comunicare directa:**

```vue
<!-- BINE: Props pentru 1-2 nivele -->
<template>
  <UserProfile :user="currentUser" @update="handleUpdate" />
</template>
```

**provide/inject - subtree shared state:**

```vue
<!-- BINE: provide/inject pentru compound components -->
<template>
  <DataTable :data="users">
    <DataColumn field="name" label="Nume" sortable />
    <DataColumn field="email" label="Email" />
    <DataColumn field="role" label="Rol" filterable />
    <!-- DataColumn-urile injecteaza context din DataTable -->
  </DataTable>
</template>
```

**Pinia - global state complex:**

```typescript
// BINE: Pinia pentru state complex cu devtools
export const useCartStore = defineStore('cart', () => {
  const items = ref<CartItem[]>([])
  const total = computed(() =>
    items.value.reduce((sum, i) => sum + i.price * i.quantity, 0)
  )

  function addItem(product: Product) { /* ... */ }
  function removeItem(id: string) { /* ... */ }
  function updateQuantity(id: string, qty: number) { /* ... */ }
  function clearCart() { /* ... */ }

  return { items, total, addItem, removeItem, updateQuantity, clearCart }
})
```

**Composable fara DI - logica reutilizabila:**

```typescript
// BINE: Composable simplu pentru logica
export function useDebounce<T>(value: Ref<T>, delay: number = 300): Ref<T> {
  const debounced = ref(value.value) as Ref<T>
  let timeout: ReturnType<typeof setTimeout>

  watch(value, (newVal) => {
    clearTimeout(timeout)
    timeout = setTimeout(() => {
      debounced.value = newVal
    }, delay)
  })

  return debounced
}
```

### 9.4 Anti-patterns

```typescript
// RAUUU: provide/inject pentru date care ar trebui sa fie props
// Daca e doar parent -> child, foloseste props!
provide('buttonSize', 'large')  // In parent
const size = inject('buttonSize')  // In child direct -> Foloseste prop!

// RAUUU: Pinia pentru state care e doar intr-un subtree
// UserDashboard si copiii sai -> provide/inject, nu Pinia
const useDashboardStore = defineStore('dashboard', () => { /* ... */ })

// RAUUU: provide/inject global fara InjectionKey
app.provide('data', complexObject)  // String key + fara tip

// BINE: provide/inject global cu InjectionKey
app.provide(DataKey, complexObject)  // Type-safe, fara coliziuni
```

### Paralela cu Angular

| Situatie | Angular | Vue |
|----------|---------|-----|
| State global | Service `providedIn: 'root'` | Pinia store sau `app.provide()` |
| State subtree | Service in `providers` | Component `provide()` |
| Props directe | `@Input()` / `input()` | `defineProps()` |
| Business logic | Service (class) | Composable (function) |
| Compound components | Service in parent | `provide/inject` |
| State management | NgRx / Signal Store | Pinia |

---

## 10. Intrebari de interviu

### Intrebarea 1: Explica diferenta fundamentala intre DI in Angular si provide/inject in Vue

**Raspuns:**

Diferenta fundamentala este nivelul de **automatizare si complexitate**. Angular DI este un **sistem complet** de dependency injection: rezolva dependente automat prin constructor injection, gestioneaza ciclul de viata al instantelor, suporta factory patterns, multi-providers si resolution modifiers (`@Self`, `@SkipSelf`, `@Host`). Angular DI are **hierarchical injectors** pe mai multe niveluri: Environment (root, platform), Module si Element.

Vue `provide/inject` este un mecanism **simplu si explicit**: furnizezi o valoare cu `provide(key, value)` si o consumi cu `inject(key)`. Nu exista instantiere automata, nu exista resolution modifiers, nu exista module-level scoping. Pentru patterns avansate, Vue se bazeaza pe **composables** - functii care combina `provide/inject` cu reactive state. Aceasta simplitate este intentionata: Vue prefera compozitia explicita fata de magia implicita.

---

### Intrebarea 2: Cand ai folosi provide/inject vs Pinia?

**Raspuns:**

**provide/inject** se foloseste cand state-ul este **scoped la un subtree** de componente. De exemplu: un form wizard unde wizard-ul furnizeaza state-ul pasilor, un DataTable care furnizeaza configuratia coloanelor, sau un tema provider care ofera tema curenta. Avantajul este ca state-ul este **izolat** la acel subtree si nu polueaza global scope.

**Pinia** se foloseste cand state-ul este **global si complex**: cos de cumparaturi, autentificare cu multe actiuni, date cache-uite care trebuie accesate din pagini diferite. Pinia ofera **devtools** excelente, **SSR support**, **plugin system** si persistenta. Regula generala: daca ai nevoie de devtools pentru debugging sau daca state-ul este partajat intre routes, alege Pinia. Daca state-ul este local unui subtree, provide/inject e suficient si mai curat.

---

### Intrebarea 3: Cum previi coliziunile de keys?

**Raspuns:**

Folosesc **InjectionKey<T>** din Vue, care este un `Symbol` tipizat. Fiecare `Symbol()` este unic by definition, deci nu pot exista coliziuni chiar daca doua librarii diferite folosesc aceeasi descriere. Pattern-ul recomandat este sa centralizezi toate key-urile intr-un fisier `injection-keys.ts` (sau un director cu fisiere pe domeniu). Fiecare key este exportat ca o constanta cu tipul generic complet: `export const AuthKey: InjectionKey<AuthService> = Symbol('auth')`. Aceasta ofera atat unicitate cat si type safety la provide si inject.

---

### Intrebarea 4: Cum faci provide/inject type-safe?

**Raspuns:**

Prin combinarea **InjectionKey<T>** cu interfete TypeScript explicite. Definesc interfata completa a serviciului (ce valori reactive expune, ce metode are), creez un `InjectionKey<T>` tipizat cu acea interfata, apoi TypeScript impune tipul corect atat la `provide()` cat si la `inject()`. Pattern-ul complet include: `InjectionKey<T>` pentru key, interfata pentru contract, `readonly()` pentru valori care nu ar trebui modificate direct, si `ComputedRef` pentru valori derivate. Consumatorul primeste automat tipul corect si IDE-ul ofera autocomplete complet.

---

### Intrebarea 5: Cum testezi componente care folosesc inject?

**Raspuns:**

Exista doua abordari principale. **Prima:** in test setup, creez o aplicatie wrapper care furnizeaza mock-urile necesare folosind `app.provide()`. Cu Vitest si Vue Test Utils:

```typescript
import { mount } from '@vue/test-utils'

const wrapper = mount(MyComponent, {
  global: {
    provide: {
      [AuthKey as symbol]: {
        user: ref({ name: 'Test User' }),
        isAuthenticated: computed(() => true),
        login: vi.fn(),
        logout: vi.fn()
      }
    }
  }
})
```

**A doua abordare:** daca folosesc pattern-ul `createXxxProvider()` + `useXxx()`, pot testa composable-ul izolat cu `withSetup()` helper care ruleaza composable-ul intr-un component temporar. Aceasta permite testarea unitara a logicii fara a monta componente UI.

---

### Intrebarea 6: Care sunt limitarile provide/inject comparativ cu Angular DI?

**Raspuns:**

**Sase limitari principale:** (1) Nu exista **instantiere automata** - trebuie sa creezi manual obiectele si sa le furnizezi. (2) Nu exista **resolution modifiers** (`@Self`, `@SkipSelf`, `@Host`) - inject cauta intotdeauna in sus pana la root. (3) Nu exista **multi-providers** nativi - daca ai nevoie de un array de implementari, trebuie sa il gestionezi manual. (4) Nu exista **module-level scoping** - doar app-level si component-level. (5) Nu exista **detectie automata a dependintelor circulare** - obtii stack overflow in loc de eroare descriptiva. (6) Nu exista **factory pattern built-in** cu dependency resolution automata - trebuie sa chemi `inject()` manual in factory function.

---

### Intrebarea 7: Cum ai implementa un service singleton in Vue?

**Raspuns:**

Doua abordari. **Abordarea 1: app.provide()** - furnizez serviciul la nivel de aplicatie in `main.ts`. Aceasta garanteaza o singura instanta accesibila in toata aplicatia:

```typescript
// main.ts
const authService = createAuthService()
app.provide(AuthKey, authService)
```

**Abordarea 2: module-level singleton** - export direct din modul, fara provide/inject:

```typescript
// services/logger.ts
export const logger = createLogger({ level: 'info' })
// Import direct oriunde: import { logger } from '@/services/logger'
```

Abordarea 1 este preferata cand ai nevoie de **reactivity** si **testabilitate** (poti inlocui in teste). Abordarea 2 este OK pentru utilitare **stateless** (logger, formatter). In Angular, echivalentul este `providedIn: 'root'` care combina ambele avantaje.

---

### Intrebarea 8: Cum gestionezi situatia cand un provider lipseste?

**Raspuns:**

**Trei strategii:** (1) **Default value** - `inject(Key, defaultValue)` returneaza un fallback semnificativ. Util cand componenta poate functiona si fara provider. (2) **Error throw** - in functia `useXxx()`, verific daca `inject()` returneaza undefined si arunc o eroare descriptiva: `throw new Error('Provider not found. Did you forget to call createXxxProvider()?')`. Aceasta da un mesaj clar developerului. (3) **Optional pattern** - `inject(Key, undefined)` si apoi gestionez conditional in component cu `v-if` sau optional chaining.

Pattern-ul recomandat pentru servicii critice (auth, API client) este **error throw**. Pentru configuratii optionale (tema, locale), folosesc **default value**. Aceasta reflecta pattern-ul Angular: `@Optional()` pentru dependinte optionale, eroare automata `NullInjectorError` pentru cele obligatorii.

---

### Intrebarea 9: Poti folosi provide/inject in composables (in afara componentelor)?

**Raspuns:**

**Da, dar cu conditii.** `provide()` si `inject()` functioneaza in orice context care are acces la **component instance** curent. Aceasta include: `setup()` function, composables apelate din `setup()`, si lifecycle hooks. **NU** functioneaza in: functii asincrone dupa un `await` (se pierde contextul), event handlers, setTimeout/setInterval callbacks, sau cod complet in afara componentelor.

Pattern-ul recomandat este sa creezi composable-uri care apeleaza `provide()` sau `inject()` **sincron** in `setup()`. Daca ai nevoie de inject intr-un context asincron, salveaza referinta inainte de `await`:

```typescript
export function useMyComposable() {
  const service = inject(ServiceKey)!  // OK - sincron in setup
  // NU: await something(); const service = inject(ServiceKey)  // Pierde context
  return { service }
}
```

---

### Intrebarea 10: Cum ai migra un Angular service complex la Vue?

**Raspuns:**

**Proces in 5 pasi:**

1. **Analizeaza serviciul Angular:** identifica state-ul (proprietati), actiunile (metode), dependintele (constructor injection) si scope-ul (`providedIn`).

2. **Defineste interfata TypeScript:** creez interfata care descrie contractul serviciului - ce expune read-only, ce metode are. Aceasta inlocuieste clasa Angular.

3. **Creez composable-ul factory:** `createXxxProvider()` care: creaza state reactiv (`ref`, `reactive`), defineste actiunile ca functii, rezolva dependinte cu `inject()`, furnizeaza serviciul cu `provide()`.

4. **Creez consumer composable:** `useXxx()` care injecteaza si valideaza existenta provider-ului.

5. **Wire-up in App.vue:** apelez factory-urile in ordinea corecta (dependintele inainte de dependenti).

Exemplu concret - Angular `NotificationService` cu Subject-uri devine Vue composable cu `ref` + `readonly` + metode. `BehaviorSubject` devine `ref()`, `.pipe(map())` devine `computed()`, `.subscribe()` devine `watch()`.

---

### Intrebarea 11: Cum ai proiecta DI pentru o aplicatie Vue enterprise?

**Raspuns:**

**Arhitectura pe straturi:**

**Layer 1 - Infrastructure (app.provide):** API client, logger, analytics, feature flags, configuratie. Furnizate in `main.ts`, disponibile global. Echivalent Angular: `providedIn: 'root'`.

**Layer 2 - Domain Services (composables cu provide/inject):** Auth service, notification service, permission service. Pattern-ul `createXxxProvider()` + `useXxx()`. Furnizate in `App.vue` sau layout components.

**Layer 3 - Feature State (Pinia stores):** State complex specific feature-urilor: cart, product catalog, user management. Pinia pentru devtools si SSR support.

**Layer 4 - Component Scoped (component provide):** Compound components (DataTable, Form Wizard, Tabs). State local partajat in subtree. Fiecare instanta are propriul scope.

**Conventii:** toate InjectionKey-urile in `src/injection-keys/`, toate composable-urile urmeaza pattern-ul `createXxxProvider()` + `useXxx()`, testele furnizeaza mock-uri prin `global.provide`, documentatie inline cu JSDoc.

---

### Intrebarea 12: Ce pattern ai folosi pentru un sistem de permisiuni cu provide/inject?

**Raspuns:**

As crea un **PermissionService** cu provide/inject care expune permisiunile reactive si directive/composable-uri helper:

```typescript
// composables/usePermissions.ts
export interface PermissionService {
  permissions: Readonly<Ref<string[]>>
  roles: Readonly<Ref<string[]>>
  can: (permission: string) => boolean
  canAny: (permissions: string[]) => boolean
  canAll: (permissions: string[]) => boolean
  hasRole: (role: string) => boolean
}

export const PermissionKey: InjectionKey<PermissionService> = Symbol('permissions')

export function createPermissionProvider() {
  const permissions = ref<string[]>([])
  const roles = ref<string[]>([])

  // Reactive permission checks
  function can(permission: string): boolean {
    return permissions.value.includes(permission)
  }

  function canAny(perms: string[]): boolean {
    return perms.some(p => permissions.value.includes(p))
  }

  function canAll(perms: string[]): boolean {
    return perms.every(p => permissions.value.includes(p))
  }

  function hasRole(role: string): boolean {
    return roles.value.includes(role)
  }

  const service: PermissionService = {
    permissions: readonly(permissions),
    roles: readonly(roles),
    can,
    canAny,
    canAll,
    hasRole
  }

  provide(PermissionKey, service)
  return { ...service, _setPermissions: (p: string[]) => { permissions.value = p } }
}

export function usePermissions(): PermissionService {
  const service = inject(PermissionKey)
  if (!service) throw new Error('PermissionService not provided')
  return service
}
```

Plus o **directiva** `v-can` pentru template:

```typescript
// directives/vCan.ts
export const vCan: Directive<HTMLElement, string> = {
  mounted(el, binding) {
    const permissions = inject(PermissionKey)!
    if (!permissions.can(binding.value)) {
      el.style.display = 'none'
    }
  }
}

// Utilizare: <button v-can="'users.delete'">Sterge</button>
```

In Angular, echivalentul ar fi un service cu `*ngIf="permissionService.can('users.delete')"` sau o structural directive custom `*appCan="'users.delete'"`.

---

### Intrebarea 13 (Bonus): Explain how Vue's provide/inject relates to the Inversion of Control principle

**Raspuns:**

**Inversion of Control (IoC)** inseamna ca o componenta nu isi creaza dependintele, ci le primeste din exterior. In Vue, `provide/inject` implementeaza IoC astfel: componenta copil **nu stie** cum e creat serviciul sau de unde vine - doar il **injecteaza** si il foloseste. Componenta parinte (provider) **decide** ce implementare furnizeaza. Aceasta permite: (1) **Substituire in teste** - furnizezi mock-uri in loc de servicii reale, (2) **Flexibilitate** - acelasi copil poate primi implementari diferite in contexte diferite (shadowing), (3) **Decuplare** - consumatorul depinde de **interfata**, nu de implementare.

Diferenta fata de Angular: in Angular, IoC container-ul (Injector) este **framework-level**, gestioneaza ciclul de viata complet al instantelor. In Vue, IoC este **manual** - tu esti container-ul, tu decizi cand si cum creezi, dar principiul este acelasi. Vue prefera **explicit IoC** (tu faci wiring-ul), Angular prefera **implicit IoC** (framework-ul face wiring-ul).

---

> **Sfat final pentru interviu:** Cand discuti despre DI in Vue, subliniaza ca intelegi **de ce**
> Vue a ales o abordare mai simpla: composability over complexity, explicit over magic.
> Arata ca stii Angular DI in profunzime, dar ca apreciezi si simplitatea Vue.
> Cel mai bun semnal pe care il poti trimite este ca intelegi trade-off-urile,
> nu ca preferi un framework sau altul.


---

**Următor :** [**04 - State Management cu Pinia** →](Vue/04-State-Management-Pinia.md)