# Testare in Vue 3 (Interview Prep - Senior Frontend Architect)

> Vitest, Vue Test Utils, testare componente, composables, Pinia stores,
> provide/inject mocking. Paralele cu Jest/TestBed/ComponentFixture din Angular.
> Pregatit pentru candidati cu experienta solida in Angular testing (Jest, Jasmine, TestBed).

---

## Cuprins

1. [Vitest - Setup si Configurare](#1-vitest---setup-si-configurare)
2. [Vue Test Utils - mount vs shallowMount](#2-vue-test-utils---mount-vs-shallowmount)
3. [Testare Componente](#3-testare-componente)
4. [Testare Props, Emits, Slots](#4-testare-props-emits-slots)
5. [Testare Composables](#5-testare-composables)
6. [Testare Pinia Stores](#6-testare-pinia-stores)
7. [Testare cu provide/inject Mock](#7-testare-cu-provideinject-mock)
8. [Testare Async (API calls, timers)](#8-testare-async-api-calls-timers)
9. [Testare Vue Router](#9-testare-vue-router)
10. [Component Testing vs E2E](#10-component-testing-vs-e2e)
11. [Paralela completa: Angular Testing vs Vue Testing](#11-paralela-completa-angular-testing-vs-vue-testing)
12. [Intrebari de interviu](#12-intrebari-de-interviu)

---

## 1. Vitest - Setup si Configurare

### 1.1 Ce este Vitest

**Vitest** este test runner-ul nativ pentru ecosistemul Vite. Avantaje cheie:

- **API compatibil cu Jest** - `describe`, `it`, `expect`, `vi.fn()`, `vi.mock()`
- **Mult mai rapid** - ruleaza pe Vite dev server, fara transformari separate
- **TypeScript nativ** - fara `ts-jest` sau configurari suplimentare
- **HMR** - testele se re-ruleaza instant cand modifici codul
- **Built-in coverage** - suporta c8 si istanbul fara pachete aditionale
- **ESM nativ** - suporta ES modules fara configurari speciale

### 1.2 Instalare

```bash
npm install -D vitest @vue/test-utils @vitejs/plugin-vue happy-dom
npm install -D @vitest/coverage-v8    # coverage
npm install -D @vitest/ui             # UI vizualizare
npm install -D @pinia/testing         # testing Pinia
```

### 1.3 Configurare vitest.config.ts

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config'
import vue from '@vitejs/plugin-vue'

export default defineConfig({
  plugins: [vue()],
  test: {
    globals: true,
    environment: 'happy-dom',   // mai rapid decat 'jsdom'
    include: ['src/**/*.{test,spec}.{ts,js}'],
    coverage: {
      provider: 'v8',
      reporter: ['text', 'json', 'html'],
      include: ['src/**/*.{ts,vue}'],
      exclude: ['src/**/*.test.ts', 'src/**/*.d.ts']
    },
    setupFiles: ['./src/test/setup.ts']
  }
})
```

### 1.4 Fisierul de setup global

```typescript
// src/test/setup.ts
import { config } from '@vue/test-utils'
import { createTestingPinia } from '@pinia/testing'
import { vi } from 'vitest'

config.global.plugins = [createTestingPinia({ createSpy: vi.fn })]

config.global.stubs = {
  RouterLink: { template: '<a :href="to"><slot /></a>', props: ['to'] },
  Transition: { template: '<div><slot /></div>' }
}

// Mock APIs care nu exista in happy-dom
global.IntersectionObserver = vi.fn().mockImplementation(() => ({
  observe: vi.fn(), unobserve: vi.fn(), disconnect: vi.fn()
}))
```

### Paralela cu Angular

| Aspect | Angular (Jest) | Vue (Vitest) |
|--------|---------------|--------------|
| Config | jest.config.ts + ts-jest | vitest.config.ts (refoloseste vite.config) |
| Spy function | jest.fn() | vi.fn() |
| Mock module | jest.mock() | vi.mock() |
| Fake timers | jest.useFakeTimers() | vi.useFakeTimers() |
| Coverage | istanbul via jest | v8 sau istanbul |

**Concluzie**: Daca stii Jest, stii 90% din Vitest. API-ul e aproape identic, dar Vitest e semnificativ mai rapid.

---

## 2. Vue Test Utils - mount vs shallowMount

### 2.1 mount() - Randare completa

```typescript
import { mount, shallowMount } from '@vue/test-utils'
import MyComponent from './MyComponent.vue'

// mount - randeaza componenta SI toti copiii
const wrapper = mount(MyComponent, {
  props: { title: 'Test' },
  global: {
    plugins: [createTestingPinia(), router],
    stubs: { HeavyChart: true },       // stub selectiv
    provide: { theme: 'dark' },         // mock provide/inject
    directives: { tooltip: {} }         // mock directive
  },
  slots: {
    default: '<p>Content</p>',
    header: '<h1>Header</h1>'
  },
  attachTo: document.body  // necesar pentru focus, scroll
})

// shallowMount - randeaza DOAR componenta, copiii sunt stub-uri
const shallow = shallowMount(MyComponent, { props: { title: 'Test' } })
```

### 2.2 Cand mount vs shallowMount

| Criteriu | mount() | shallowMount() |
|----------|---------|-----------------|
| Copii | Randati complet | Stub-uri automate |
| Viteza | Mai lent | Mai rapid |
| Use case | **Recomandat ca default** | Componente cu copii foarte grei |

### 2.3 Wrapper API - Metodele esentiale

```typescript
// ===== CAUTARE ELEMENTE =====
wrapper.find('button')                      // CSS selector
wrapper.find('[data-testid="submit"]')      // data-testid
wrapper.findAll('li')                       // multiple
wrapper.findComponent(ChildComp)            // componenta copil
wrapper.find('.optional').exists()           // verifica existenta

// ===== INTERACTIUNI =====
await wrapper.find('button').trigger('click')
await wrapper.find('input').setValue('hello')
await wrapper.find('select').setValue('option2')
await wrapper.find('input[type="checkbox"]').setValue(true)
await wrapper.find('input').trigger('keyup.enter')

// ===== ASSERTIONS =====
expect(wrapper.text()).toContain('Hello')
expect(wrapper.html()).toContain('<div>')
expect(wrapper.classes()).toContain('active')
expect(wrapper.attributes('disabled')).toBeDefined()

// ===== EMITTED EVENTS =====
expect(wrapper.emitted()).toHaveProperty('click')
expect(wrapper.emitted('update')).toHaveLength(1)
expect(wrapper.emitted('update')![0]).toEqual(['new value'])

// ===== PROPS & VM =====
expect(wrapper.props('title')).toBe('Test')
wrapper.vm.someMethod()  // acces direct (evita daca poti)
```

### Paralela cu Angular

| Vue Test Utils | Angular TestBed |
|---------------|-----------------|
| `mount(Comp)` | `TestBed.createComponent(Comp)` |
| `wrapper` | `fixture: ComponentFixture` |
| `wrapper.find('button')` | `debugElement.query(By.css('button'))` |
| `wrapper.trigger('click')` | `triggerEventHandler('click')` |
| `wrapper.text()` | `nativeElement.textContent` |
| `wrapper.emitted('event')` | `spyOn(comp.event, 'emit')` |
| `await nextTick()` | `fixture.detectChanges()` |
| `shallowMount()` | `NO_ERRORS_SCHEMA` + stubs |
| `global.provide` | `TestBed providers` |

**Diferenta fundamentala**: In Angular, `TestBed.configureTestingModule()` necesita configurarea explicita a tuturor dependentelor. In Vue, `mount()` cu `global` options e tot ce ai nevoie.

---

## 3. Testare Componente

### 3.1 Component simplu - Counter

```vue
<!-- Counter.vue -->
<script setup lang="ts">
import { ref, computed } from 'vue'

const props = withDefaults(defineProps<{
  initial?: number
  max?: number
}>(), { initial: 0, max: 100 })

const emit = defineEmits<{
  change: [value: number]
  'limit-reached': [limit: 'max']
}>()

const count = ref(props.initial)
const isAtMax = computed(() => count.value >= props.max)

function increment() {
  if (count.value < props.max) {
    count.value++
    emit('change', count.value)
  } else {
    emit('limit-reached', 'max')
  }
}
</script>

<template>
  <div class="counter">
    <span data-testid="count">{{ count }}</span>
    <button data-testid="increment" :disabled="isAtMax" @click="increment">+</button>
  </div>
</template>
```

### 3.2 Testele pentru Counter

```typescript
// Counter.test.ts
import { mount } from '@vue/test-utils'
import Counter from './Counter.vue'

describe('Counter', () => {
  it('randeaza valoarea initiala default (0)', () => {
    const wrapper = mount(Counter)
    expect(wrapper.find('[data-testid="count"]').text()).toBe('0')
  })

  it('randeaza valoarea initiala din props', () => {
    const wrapper = mount(Counter, { props: { initial: 42 } })
    expect(wrapper.find('[data-testid="count"]').text()).toBe('42')
  })

  it('incrementeaza count la click pe +', async () => {
    const wrapper = mount(Counter)
    await wrapper.find('[data-testid="increment"]').trigger('click')
    expect(wrapper.find('[data-testid="count"]').text()).toBe('1')
  })

  it('emite change cu noua valoare', async () => {
    const wrapper = mount(Counter, { props: { initial: 10 } })
    await wrapper.find('[data-testid="increment"]').trigger('click')

    expect(wrapper.emitted('change')).toHaveLength(1)
    expect(wrapper.emitted('change')![0]).toEqual([11])
  })

  it('emite limit-reached cand ajunge la max', async () => {
    const wrapper = mount(Counter, { props: { initial: 10, max: 10 } })
    await wrapper.find('[data-testid="increment"]').trigger('click')

    expect(wrapper.emitted('limit-reached')![0]).toEqual(['max'])
  })

  it('dezactiveaza butonul + cand e la max', () => {
    const wrapper = mount(Counter, { props: { initial: 10, max: 10 } })
    expect(wrapper.find('[data-testid="increment"]').attributes('disabled')).toBeDefined()
  })
})
```

### 3.3 Component cu dependente externe

```typescript
// UserProfile.test.ts
import { mount, flushPromises } from '@vue/test-utils'
import { createTestingPinia } from '@pinia/testing'
import UserProfile from './UserProfile.vue'

describe('UserProfile', () => {
  function createWrapper(stateOverrides = {}) {
    return mount(UserProfile, {
      global: {
        plugins: [
          createTestingPinia({
            initialState: {
              user: {
                currentUser: { id: 1, name: 'Test User' },
                isAuthenticated: true,
                ...stateOverrides
              }
            }
          })
        ]
      }
    })
  }

  it('afiseaza numele utilizatorului din store', () => {
    const wrapper = createWrapper()
    expect(wrapper.text()).toContain('Test User')
  })

  it('afiseaza mesaj cand nu e autentificat', () => {
    const wrapper = createWrapper({ currentUser: null, isAuthenticated: false })
    expect(wrapper.text()).toContain('Nu esti autentificat')
  })
})
```

---

## 4. Testare Props, Emits, Slots

### 4.1 Testare Props

```typescript
describe('Props', () => {
  it('foloseste valori default', () => {
    const wrapper = mount(Alert, { props: { message: 'Test' } })
    expect(wrapper.props('type')).toBe('info')        // default
    expect(wrapper.props('dismissible')).toBe(false)   // default
  })

  it('reactioneaza la schimbari de props', async () => {
    const wrapper = mount(Alert, { props: { message: 'Initial' } })
    expect(wrapper.text()).toContain('Initial')

    await wrapper.setProps({ message: 'Updated' })
    expect(wrapper.text()).toContain('Updated')
  })

  it('aplica clasa corecta bazata pe type', () => {
    const types = ['success', 'error', 'warning', 'info'] as const
    types.forEach(type => {
      const wrapper = mount(Alert, { props: { message: 'Test', type } })
      expect(wrapper.classes()).toContain(`alert-${type}`)
    })
  })
})
```

### 4.2 Testare Emits

```typescript
describe('Emits', () => {
  it('emite update:modelValue pentru v-model', async () => {
    const wrapper = mount(SearchInput, { props: { modelValue: '' } })
    await wrapper.find('input').setValue('vue testing')

    expect(wrapper.emitted('update:modelValue')![0]).toEqual(['vue testing'])
  })

  it('verifica payload-ul emiterii', async () => {
    const wrapper = mount(UserForm)
    await wrapper.find('[data-testid="name"]').setValue('Ion')
    await wrapper.find('[data-testid="email"]').setValue('ion@test.com')
    await wrapper.find('form').trigger('submit.prevent')

    expect(wrapper.emitted('submit')![0][0]).toEqual({
      name: 'Ion', email: 'ion@test.com'
    })
  })

  it('emite events in ordine corecta', async () => {
    const wrapper = mount(Wizard)
    await wrapper.find('[data-testid="next"]').trigger('click')
    await wrapper.find('[data-testid="next"]').trigger('click')

    const steps = wrapper.emitted('step-change')
    expect(steps![0]).toEqual([1])
    expect(steps![1]).toEqual([2])
  })
})
```

### 4.3 Testare Slots

```vue
<!-- Card.vue -->
<template>
  <div class="card">
    <div class="card-header" v-if="$slots.header">
      <slot name="header" />
    </div>
    <div class="card-body"><slot /></div>
    <div class="card-footer" v-if="$slots.footer">
      <slot name="footer" :year="new Date().getFullYear()" />
    </div>
  </div>
</template>
```

```typescript
describe('Card Slots', () => {
  it('randeaza default slot', () => {
    const wrapper = mount(Card, { slots: { default: '<p>Content</p>' } })
    expect(wrapper.find('.card-body').html()).toContain('<p>Content</p>')
  })

  it('randeaza named slot', () => {
    const wrapper = mount(Card, { slots: { header: '<h1>Title</h1>' } })
    expect(wrapper.find('.card-header').html()).toContain('<h1>Title</h1>')
  })

  it('paseaza scope data la scoped slot', () => {
    const wrapper = mount(Card, {
      slots: {
        footer: `<template #footer="{ year }">
          <span data-testid="year">{{ year }}</span>
        </template>`
      }
    })
    expect(wrapper.find('[data-testid="year"]').text())
      .toBe(new Date().getFullYear().toString())
  })

  it('nu randeaza footer daca slot-ul nu e furnizat', () => {
    const wrapper = mount(Card, { slots: { default: 'Content' } })
    expect(wrapper.find('.card-footer').exists()).toBe(false)
  })
})
```

### Paralela cu Angular

| Concept | Angular | Vue |
|---------|---------|-----|
| Default slot | `<ng-content>` | `<slot />` |
| Named slot | `<ng-content select=".header">` | `<slot name="header" />` |
| Scoped slot | Nu exista direct | `<slot :data="value" />` |
| Testare slot | `nativeElement.querySelector('.projected')` | `wrapper.find('.slot-content')` |

---

## 5. Testare Composables

### 5.1 Composable simplu (fara lifecycle)

```typescript
// composables/useCounter.ts
import { ref, computed } from 'vue'

export function useCounter(initial = 0) {
  const count = ref(initial)
  const doubled = computed(() => count.value * 2)
  const increment = () => count.value++
  const decrement = () => count.value--
  const reset = () => { count.value = initial }
  return { count, doubled, increment, decrement, reset }
}
```

```typescript
// Se testeaza direct ca o functie - nu ai nevoie de mount()
describe('useCounter', () => {
  it('porneste cu valoarea initiala', () => {
    const { count } = useCounter(10)
    expect(count.value).toBe(10)
  })

  it('incrementeaza si decrementeaza', () => {
    const { count, increment, decrement } = useCounter(5)
    increment(); increment()
    expect(count.value).toBe(7)
    decrement()
    expect(count.value).toBe(6)
  })

  it('calculeaza doubled corect', () => {
    const { doubled, increment } = useCounter(5)
    expect(doubled.value).toBe(10)
    increment()
    expect(doubled.value).toBe(12)
  })

  it('reseteaza la valoarea initiala', () => {
    const { count, increment, reset } = useCounter(10)
    increment(); increment()
    reset()
    expect(count.value).toBe(10)
  })
})
```

### 5.2 Composable cu lifecycle hooks - withSetup helper

```typescript
// test/helpers.ts
import { mount } from '@vue/test-utils'
import { defineComponent } from 'vue'

export function withSetup<T>(composable: () => T, options?: { global?: any }): [T, any] {
  let result!: T
  const Wrapper = defineComponent({
    setup() { result = composable(); return {} },
    template: '<div />'
  })
  const wrapper = mount(Wrapper, { global: options?.global })
  return [result, wrapper]
}
```

```typescript
// composables/useApi.ts
import { ref, onMounted, onUnmounted } from 'vue'

export function useApi<T>(url: string) {
  const data = ref<T | null>(null)
  const loading = ref(false)
  const error = ref<string | null>(null)
  let controller: AbortController | null = null

  async function fetchData() {
    loading.value = true
    error.value = null
    controller = new AbortController()
    try {
      const res = await fetch(url, { signal: controller.signal })
      if (!res.ok) throw new Error(`HTTP ${res.status}`)
      data.value = await res.json()
    } catch (e: any) {
      if (e.name !== 'AbortError') error.value = e.message
    } finally {
      loading.value = false
    }
  }

  onMounted(() => fetchData())
  onUnmounted(() => controller?.abort())

  return { data, loading, error, refresh: fetchData }
}
```

```typescript
// composables/useApi.test.ts
import { flushPromises } from '@vue/test-utils'
import { withSetup } from '@/test/helpers'
import { useApi } from './useApi'

describe('useApi', () => {
  beforeEach(() => vi.restoreAllMocks())

  it('incarca date la mount', async () => {
    vi.spyOn(global, 'fetch').mockResolvedValue({
      ok: true, json: () => Promise.resolve([{ id: 1 }])
    } as Response)

    const [result] = withSetup(() => useApi('/api/items'))
    expect(result.loading.value).toBe(true)
    await flushPromises()

    expect(result.data.value).toEqual([{ id: 1 }])
    expect(result.loading.value).toBe(false)
  })

  it('seteaza error la fetch esuat', async () => {
    vi.spyOn(global, 'fetch').mockResolvedValue({ ok: false, status: 404 } as Response)

    const [result] = withSetup(() => useApi('/api/missing'))
    await flushPromises()

    expect(result.error.value).toBe('HTTP 404')
  })

  it('anuleaza request la unmount', async () => {
    const abortSpy = vi.spyOn(AbortController.prototype, 'abort')
    vi.spyOn(global, 'fetch').mockImplementation(() => new Promise(() => {}))

    const [_, wrapper] = withSetup(() => useApi('/api/slow'))
    wrapper.unmount()
    expect(abortSpy).toHaveBeenCalled()
  })
})
```

### Paralela cu Angular

| Vue Composable | Angular Service |
|---------------|-----------------|
| Composable simplu | Service fara DI - testat direct |
| Composable cu lifecycle | Service cu OnDestroy - necesita TestBed |
| `withSetup()` helper | `TestBed.inject(Service)` |
| Composable cu inject | Service cu @Inject - mock prin provide |

---

## 6. Testare Pinia Stores

### 6.1 Testare store izolat

```typescript
// stores/useCartStore.ts
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'

interface CartItem { id: number; name: string; price: number; quantity: number }

export const useCartStore = defineStore('cart', () => {
  const items = ref<CartItem[]>([])
  const totalItems = computed(() => items.value.reduce((s, i) => s + i.quantity, 0))
  const totalPrice = computed(() => items.value.reduce((s, i) => s + i.price * i.quantity, 0))

  function addItem(product: Omit<CartItem, 'quantity'>) {
    const existing = items.value.find(i => i.id === product.id)
    if (existing) existing.quantity++
    else items.value.push({ ...product, quantity: 1 })
  }

  function removeItem(id: number) {
    const idx = items.value.findIndex(i => i.id === id)
    if (idx > -1) items.value.splice(idx, 1)
  }

  function clearCart() { items.value = [] }

  return { items, totalItems, totalPrice, addItem, removeItem, clearCart }
})
```

```typescript
// stores/useCartStore.test.ts
import { setActivePinia, createPinia } from 'pinia'
import { useCartStore } from './useCartStore'

describe('CartStore', () => {
  beforeEach(() => setActivePinia(createPinia()))

  it('porneste cu cos gol', () => {
    const store = useCartStore()
    expect(store.items).toEqual([])
    expect(store.totalItems).toBe(0)
  })

  it('adauga un produs nou', () => {
    const store = useCartStore()
    store.addItem({ id: 1, name: 'Laptop', price: 5000 })
    expect(store.items).toHaveLength(1)
    expect(store.items[0].quantity).toBe(1)
  })

  it('incrementeaza cantitatea la produs duplicat', () => {
    const store = useCartStore()
    store.addItem({ id: 1, name: 'Laptop', price: 5000 })
    store.addItem({ id: 1, name: 'Laptop', price: 5000 })
    expect(store.items).toHaveLength(1)
    expect(store.items[0].quantity).toBe(2)
  })

  it('calculeaza totalPrice corect', () => {
    const store = useCartStore()
    store.addItem({ id: 1, name: 'Laptop', price: 5000 })
    store.addItem({ id: 2, name: 'Mouse', price: 100 })
    store.addItem({ id: 1, name: 'Laptop', price: 5000 })
    expect(store.totalPrice).toBe(10100) // (5000*2) + (100*1)
  })

  it('sterge un produs', () => {
    const store = useCartStore()
    store.addItem({ id: 1, name: 'Laptop', price: 5000 })
    store.removeItem(1)
    expect(store.items).toEqual([])
  })
})
```

### 6.2 Testare store in componente (createTestingPinia)

```typescript
import { mount } from '@vue/test-utils'
import { createTestingPinia } from '@pinia/testing'
import ProductCard from './ProductCard.vue'
import { useCartStore } from '@/stores/useCartStore'

describe('ProductCard cu Pinia', () => {
  it('apeleaza addItem la click (actiuni stub-uite)', async () => {
    const wrapper = mount(ProductCard, {
      props: { product: { id: 1, name: 'Laptop', price: 5000 } },
      global: {
        plugins: [createTestingPinia({ createSpy: vi.fn })]
        // stubActions: true (default) - actiunile sunt vi.fn() automat
      }
    })

    const cartStore = useCartStore()
    await wrapper.find('[data-testid="add-to-cart"]').trigger('click')

    expect(cartStore.addItem).toHaveBeenCalledWith({
      id: 1, name: 'Laptop', price: 5000
    })
  })

  it('afiseaza badge cu initialState', () => {
    const wrapper = mount(ProductCard, {
      props: { product: { id: 1, name: 'Laptop', price: 5000 } },
      global: {
        plugins: [createTestingPinia({
          initialState: {
            cart: { items: [{ id: 1, name: 'Laptop', price: 5000, quantity: 2 }] }
          }
        })]
      }
    })
    expect(wrapper.find('[data-testid="cart-badge"]').text()).toBe('2')
  })

  it('ruleaza actiunile real cand stubActions e false', async () => {
    const wrapper = mount(ProductCard, {
      props: { product: { id: 1, name: 'Laptop', price: 5000 } },
      global: {
        plugins: [createTestingPinia({ stubActions: false })]
      }
    })
    await wrapper.find('[data-testid="add-to-cart"]').trigger('click')
    expect(useCartStore().items).toHaveLength(1)
  })
})
```

### Paralela cu Angular

| Concept | Angular (NgRx) | Vue (Pinia) |
|---------|----------------|-------------|
| Store testing | TestBed + provideMockStore | setActivePinia(createPinia()) |
| Mock in componente | provideMockStore({ initialState }) | createTestingPinia({ initialState }) |
| Stub actions | provideMockActions | stubActions: true (default) |
| Complexitate | **Ridicata** - reducers, effects, selectors | **Scazuta** - o singura functie |

---

## 7. Testare cu provide/inject Mock

### 7.1 Mock-uire provide simpla

```typescript
import { mount } from '@vue/test-utils'
import { ref } from 'vue'
import ThemeToggle from './ThemeToggle.vue'

describe('ThemeToggle', () => {
  it('afiseaza tema curenta', () => {
    const wrapper = mount(ThemeToggle, {
      global: { provide: { theme: ref('dark'), toggleTheme: vi.fn() } }
    })
    expect(wrapper.text()).toContain('dark')
  })

  it('apeleaza toggleTheme la click', async () => {
    const toggleMock = vi.fn()
    const wrapper = mount(ThemeToggle, {
      global: { provide: { theme: ref('light'), toggleTheme: toggleMock } }
    })
    await wrapper.find('[data-testid="toggle"]').trigger('click')
    expect(toggleMock).toHaveBeenCalled()
  })
})
```

### 7.2 Mock-uire cu Symbol keys (InjectionKey)

```typescript
// symbols.ts
import { type InjectionKey, type Ref } from 'vue'

export interface AuthContext {
  user: Ref<{ name: string; role: string } | null>
  isAuthenticated: Ref<boolean>
  logout: () => void
}
export const AuthKey: InjectionKey<AuthContext> = Symbol('auth')
```

```typescript
import { AuthKey } from '@/symbols'

describe('AdminPanel', () => {
  function createWrapper(overrides = {}) {
    const defaultAuth = {
      user: ref({ name: 'Admin', role: 'admin' }),
      isAuthenticated: ref(true),
      logout: vi.fn()
    }
    return mount(AdminPanel, {
      global: {
        provide: { [AuthKey as symbol]: { ...defaultAuth, ...overrides } }
      }
    })
  }

  it('afiseaza panoul admin', () => {
    expect(createWrapper().find('[data-testid="admin-panel"]').exists()).toBe(true)
  })

  it('afiseaza access denied pentru non-admin', () => {
    const wrapper = createWrapper({ user: ref({ name: 'User', role: 'user' }) })
    expect(wrapper.find('[data-testid="access-denied"]').exists()).toBe(true)
  })

  it('apeleaza logout', async () => {
    const logoutMock = vi.fn()
    const wrapper = createWrapper({ logout: logoutMock })
    await wrapper.find('[data-testid="logout-btn"]').trigger('click')
    expect(logoutMock).toHaveBeenCalled()
  })
})
```

### Paralela cu Angular

```typescript
// Angular - mock DI
TestBed.configureTestingModule({
  providers: [
    { provide: AuthService, useValue: mockAuthService },
    { provide: THEME_TOKEN, useValue: 'dark' }
  ]
})

// Vue - echivalent, mult mai simplu
mount(Component, {
  global: { provide: { [AuthKey as symbol]: mockAuth, theme: 'dark' } }
})
```

---

## 8. Testare Async (API calls, timers)

### 8.1 flushPromises

```typescript
import { mount, flushPromises } from '@vue/test-utils'

describe('ProductList - async data', () => {
  it('afiseaza loading apoi datele', async () => {
    vi.spyOn(global, 'fetch').mockResolvedValue({
      ok: true,
      json: () => Promise.resolve([{ id: 1, name: 'Laptop' }])
    } as Response)

    const wrapper = mount(ProductList)
    expect(wrapper.find('[data-testid="loading"]').exists()).toBe(true)

    await flushPromises()

    expect(wrapper.find('[data-testid="loading"]').exists()).toBe(false)
    expect(wrapper.findAll('[data-testid="product-item"]')).toHaveLength(1)
  })

  it('afiseaza eroare la fetch esuat', async () => {
    vi.spyOn(global, 'fetch').mockRejectedValue(new Error('Network error'))
    const wrapper = mount(ProductList)
    await flushPromises()

    expect(wrapper.find('[data-testid="error"]').text()).toContain('Network error')
  })
})
```

### 8.2 Fake Timers - debounce, intervals

```typescript
describe('Debounced Search', () => {
  beforeEach(() => vi.useFakeTimers())
  afterEach(() => vi.useRealTimers())

  it('face debounce la input (300ms)', async () => {
    const fetchSpy = vi.spyOn(global, 'fetch').mockResolvedValue({
      ok: true, json: () => Promise.resolve([])
    } as Response)

    const wrapper = mount(SearchComponent)
    await wrapper.find('input').setValue('v')
    await wrapper.find('input').setValue('vu')
    await wrapper.find('input').setValue('vue')

    expect(fetchSpy).not.toHaveBeenCalled()  // debounce

    vi.advanceTimersByTime(300)
    await flushPromises()

    expect(fetchSpy).toHaveBeenCalledTimes(1) // UN singur call
  })

  it('auto-refresh la fiecare 30 secunde', async () => {
    vi.spyOn(global, 'fetch').mockResolvedValue({
      ok: true, json: () => Promise.resolve([])
    } as Response)

    mount(DashboardWithRefresh)
    await flushPromises()
    expect(fetch).toHaveBeenCalledTimes(1) // initial

    vi.advanceTimersByTime(30000)
    await flushPromises()
    expect(fetch).toHaveBeenCalledTimes(2) // refresh
  })
})
```

### 8.3 Mock Axios

```typescript
vi.mock('axios')
const mockedAxios = vi.mocked(axios)

it('trimite POST request', async () => {
  mockedAxios.post.mockResolvedValue({ data: { id: 1, success: true } })

  const wrapper = mount(CreateUserForm)
  await wrapper.find('[data-testid="name"]').setValue('Ion')
  await wrapper.find('form').trigger('submit.prevent')
  await flushPromises()

  expect(mockedAxios.post).toHaveBeenCalledWith('/api/users', { name: 'Ion' })
})
```

### 8.4 MSW (Mock Service Worker)

```typescript
// test/mocks/handlers.ts
import { http, HttpResponse } from 'msw'

export const handlers = [
  http.get('/api/products', () => {
    return HttpResponse.json([{ id: 1, name: 'Laptop' }])
  }),
  http.post('/api/users', async ({ request }) => {
    const body = await request.json()
    return HttpResponse.json({ id: 1, ...body }, { status: 201 })
  })
]

// test/setup.ts
import { setupServer } from 'msw/node'
import { handlers } from './mocks/handlers'
export const server = setupServer(...handlers)

beforeAll(() => server.listen())
afterEach(() => server.resetHandlers())
afterAll(() => server.close())
```

### Paralela cu Angular

| Concept | Angular | Vue (Vitest) |
|---------|---------|--------------|
| Flush async | fakeAsync + tick() | vi.useFakeTimers + advanceTimersByTime |
| Wait promises | tick() in fakeAsync | await flushPromises() |
| Detect changes | fixture.detectChanges() | await nextTick() (automat dupa trigger) |
| HTTP mock | HttpClientTestingModule | vi.spyOn(fetch) / MSW |
| Verify HTTP | httpMock.expectOne(url).flush(data) | expect(fetch).toHaveBeenCalledWith(url) |

---

## 9. Testare Vue Router

### 9.1 Setup router pentru teste

```typescript
// test/helpers/router.ts
import { createRouter, createMemoryHistory } from 'vue-router'

export function createTestRouter(routes?: any[]) {
  return createRouter({
    history: createMemoryHistory(),
    routes: routes ?? [
      { path: '/', name: 'home', component: { template: '<div>Home</div>' } },
      { path: '/about', name: 'about', component: { template: '<div>About</div>' } },
      { path: '/products/:id', name: 'product', component: { template: '<div>Product</div>' } },
      { path: '/admin', name: 'admin', component: { template: '<div>Admin</div>' }, meta: { requiresAuth: true } },
      { path: '/login', name: 'login', component: { template: '<div>Login</div>' } }
    ]
  })
}
```

### 9.2 Testare componente cu router

```typescript
import { mount, flushPromises } from '@vue/test-utils'
import { createTestRouter } from '@/test/helpers/router'

describe('NavigationBar', () => {
  it('aplica clasa activa pe ruta curenta', async () => {
    const router = createTestRouter()
    await router.push('/about')
    await router.isReady()

    const wrapper = mount(NavigationBar, { global: { plugins: [router] } })
    expect(wrapper.find('[data-testid="nav-about"]').classes()).toContain('router-link-active')
  })

  it('navigheaza la click pe link', async () => {
    const router = createTestRouter()
    await router.push('/')
    await router.isReady()

    const wrapper = mount(NavigationBar, { global: { plugins: [router] } })
    await wrapper.find('[data-testid="nav-about"]').trigger('click')
    await flushPromises()

    expect(router.currentRoute.value.path).toBe('/about')
  })
})
```

### 9.3 Testare Navigation Guards

```typescript
describe('Auth Guard', () => {
  function createGuardedRouter(isAuthenticated: boolean) {
    const router = createTestRouter()
    router.beforeEach((to) => {
      if (to.meta.requiresAuth && !isAuthenticated) {
        return { name: 'login', query: { redirect: to.fullPath } }
      }
    })
    return router
  }

  it('redirectioneaza la login pentru rute protejate', async () => {
    const router = createGuardedRouter(false)
    await router.push('/admin')
    await router.isReady()

    expect(router.currentRoute.value.name).toBe('login')
    expect(router.currentRoute.value.query.redirect).toBe('/admin')
  })

  it('permite accesul cand e autentificat', async () => {
    const router = createGuardedRouter(true)
    await router.push('/admin')
    await router.isReady()

    expect(router.currentRoute.value.name).toBe('admin')
  })
})
```

### 9.4 Mock-uire useRoute fara router real

```typescript
vi.mock('vue-router', () => ({
  useRoute: vi.fn(() => ({
    path: '/products/42',
    params: { id: '42' },
    query: { sort: 'price' },
    name: 'product'
  })),
  useRouter: vi.fn(() => ({
    push: vi.fn(), replace: vi.fn(), back: vi.fn()
  }))
}))
```

### Paralela cu Angular

| Concept | Angular | Vue |
|---------|---------|-----|
| Router testing | RouterTestingModule | createRouter + createMemoryHistory |
| Navigate | router.navigate(['/path']) | router.push('/path') |
| Guards test | TestBed cu guard | router.beforeEach + router.push |
| Wait navigation | fixture.whenStable() | await router.isReady() |

---

## 10. Component Testing vs E2E

### 10.1 Piramida testelor in Vue

```
         /\
        /E2E\        Cypress/Playwright - flows complete
       /------\
      /Component\    VTU + Vitest - componente izolate
     /------------\
    /  Unit Tests   \  Vitest - composables, utils, stores
   /________________\
```

### 10.2 Ce testezi la fiecare nivel

**Unit Tests** (Vitest, fara DOM):
- Composables, utility functions, Pinia stores, validators

**Component Tests** (VTU + Vitest):
- Renderizare, interactiuni utilizator, emits, slot rendering, conditional rendering

**E2E Tests** (Cypress / Playwright):
- Flow-uri complete (login -> actiune -> verificare), cross-browser, responsive

### 10.3 Comparatie

| Aspect | VTU + Vitest | Cypress Component | Playwright E2E |
|--------|-------------|-------------------|----------------|
| Viteza | Foarte rapid (~ms) | Mediu (~s) | Lent (~s) |
| DOM real | Nu (happy-dom) | Da (browser) | Da (browser) |
| Visual testing | Nu | Da | Da (screenshots) |
| Cross-browser | Nu | Da | Da |

### 10.4 Strategia recomandata

```
Component Tests (VTU + Vitest)  : 70% din effort
Unit Tests (Vitest)             : 20% din effort
E2E Tests (Playwright)          : 10% din effort
```

**Recomandare architect**: Investeste cel mai mult in component tests. Sunt rapide, stabile si acopera interactiunile utilizator. E2E doar pentru flow-urile critice de business.

---

## 11. Paralela completa: Angular Testing vs Vue Testing

### 11.1 Tabel comparativ detaliat

| Aspect | Angular | Vue 3 |
|--------|---------|-------|
| **Test runner** | Jest / Karma | **Vitest** |
| **Component testing** | TestBed + ComponentFixture | **Vue Test Utils + mount** |
| **Shallow render** | NO_ERRORS_SCHEMA / stubs | **shallowMount()** |
| **Store testing** | TestBed + provideMockStore | **setActivePinia + createPinia** |
| **Mock store** | provideMockStore({ initialState }) | **createTestingPinia({ initialState })** |
| **Mocking DI** | TestBed providers | **global.provide** |
| **DOM queries** | debugElement.query(By.css()) | **wrapper.find()** |
| **Trigger events** | triggerEventHandler('click') | **trigger('click')** |
| **Detect changes** | fixture.detectChanges() | **await nextTick()** |
| **Async testing** | fakeAsync + tick | **vi.useFakeTimers + flushPromises** |
| **HTTP mock** | HttpClientTestingModule | **vi.spyOn(fetch) / MSW** |
| **Router testing** | RouterTestingModule | **createRouter + createMemoryHistory** |
| **E2E** | Cypress / Playwright | **Cypress / Playwright** |

### 11.2 Setup comparison

```typescript
// ========================================
// ANGULAR - Setup complex (~30 linii)
// ========================================
beforeEach(async () => {
  const userServiceSpy = jasmine.createSpyObj('UserService', ['getUsers'])
  await TestBed.configureTestingModule({
    declarations: [UserListComponent, UserCardComponent],
    imports: [HttpClientTestingModule, RouterTestingModule, MatTableModule],
    providers: [
      { provide: UserService, useValue: userServiceSpy },
      { provide: NotificationService, useValue: { show: jasmine.createSpy() } }
    ],
    schemas: [NO_ERRORS_SCHEMA]
  }).compileComponents()
  fixture = TestBed.createComponent(UserListComponent)
  component = fixture.componentInstance
  userService = TestBed.inject(UserService)
  userService.getUsers.and.returnValue(of([...]))
  fixture.detectChanges()
})

// ========================================
// VUE - Setup simplu (~10 linii)
// ========================================
function createWrapper() {
  vi.spyOn(global, 'fetch').mockResolvedValue({
    ok: true, json: () => Promise.resolve([...])
  } as Response)
  return mount(UserList, {
    global: { plugins: [createTestingPinia(), createTestRouter()] }
  })
}
```

### 11.3 Diferente conceptuale cheie

1. **Nu exista TestBed** - `mount()` cu `global` e tot ce ai nevoie
2. **Nu exista detectChanges() explicit** - `trigger()`/`setValue()` asteapta automat nextTick
3. **Nu exista module hierarchy** - componentele sunt self-contained
4. **DI mocking e trivial** - `global.provide` cu un obiect simplu
5. **Reactivity vs Observables** - testezi `.value` in loc de subscribe/marbles

### 11.4 Migration mental map

```
Angular                        →  Vue Equivalent
────────────────────────────────────────────────
TestBed.configureTestingModule →  mount(Comp, { global: {} })
ComponentFixture               →  VueWrapper
fixture.detectChanges()        →  await nextTick() / automat
debugElement.query(By.css())   →  wrapper.find('selector')
triggerEventHandler('click')   →  trigger('click')
nativeElement.textContent      →  wrapper.text()
@Output EventEmitter spy       →  wrapper.emitted('event')
jasmine.createSpyObj           →  vi.fn() / vi.spyOn()
provideMockStore               →  createTestingPinia
HttpClientTestingModule        →  vi.spyOn(fetch) / MSW
fakeAsync + tick               →  vi.useFakeTimers + advanceTimersByTime
RouterTestingModule            →  createRouter + createMemoryHistory
```

---

## 12. Intrebari de interviu

### Intrebarea 1: De ce Vitest in loc de Jest pentru proiecte Vue 3?

Vitest reutilizeaza configuratia Vite (plugin-uri, alias-uri, transformari), deci nu trebuie sa le configurezi separat ca la Jest cu `ts-jest` si `moduleNameMapper`. Performanta e semnificativ mai buna - Vitest foloseste Vite dev server, ceea ce e mult mai rapid decat webpack/babel chain. TypeScript-ul e suportat nativ. API-ul e compatibil 95% cu Jest, tranzitia e aproape transparenta. La proiecte mari, am vazut timpi redusi cu 40-60%. Singurul motiv pentru Jest ar fi un proiect legacy fara Vite.

### Intrebarea 2: mount() vs shallowMount() - cand ce?

`mount()` e recomandat ca default - randeaza complet inclusiv copiii, teste mai realiste care detecteaza probleme de integrare. `shallowMount()` e util cand copiii au dependente grele (API calls in onMounted, grafice canvas) sau cand componenta are zeci de copii si testul devine prea lent. In practica recomand 80% mount, 20% shallowMount. Compromis bun: `mount()` cu `global.stubs` selectiv doar pentru componentele problematice.

### Intrebarea 3: Cum testezi composables cu lifecycle hooks?

Composable-urile simple se testeaza direct ca functii. Pentru cele cu `onMounted`, `onUnmounted`, `watch` sau `inject`, ai nevoie de un context Vue. Solutia e un helper `withSetup()` care monteaza composable-ul intr-un component wrapper minimal - creeaza un `defineComponent` cu `setup()` care apeleaza composable-ul. Wrapper-ul e necesar si pentru cleanup (unmount). Alternativ, poti crea un component de test dedicat care foloseste composable-ul si testezi prin UI.

### Intrebarea 4: Cum testezi componente cu Pinia stores?

`@pinia/testing` ofera `createTestingPinia()`. Il pasezi ca plugin in `global.plugins`. Cu `stubActions: true` (default), actiunile sunt `vi.fn()` automat - verifici CA actiunea a fost apelata. Cu `stubActions: false`, actiunile ruleaza real - util pentru integrare. `initialState` seteaza starea oricarui store. Pattern: factory function `createWrapper()` cu overrides, `createTestingPinia` cu state-ul dorit, assertions pe store obtinut cu `useStore()` dupa mount.

### Intrebarea 5: Cum mock-uiesti provide/inject in teste?

Direct prin `global.provide` la `mount()`. String keys: `provide: { theme: 'dark' }`. Symbol keys (InjectionKey): `provide: { [MyKey as symbol]: mockValue }`. Valorile trebuie reactive daca componenta se asteapta la reactivitate - wrap-uite in `ref()`. Pattern recomandat: factory function cu defaults si overrides. Diferenta fata de Angular: nu ai nevoie de useValue/useClass/useFactory - pasezi valoarea direct.

### Intrebarea 6: Cum testezi operatii asincrone?

Trei utilitare esentiale: `flushPromises()` rezolva toate Promise-urile pending, `await nextTick()` forteaza update DOM, `vi.useFakeTimers()` controleaza setTimeout/setInterval. Pattern: montezi, `await flushPromises()` pentru date initiale, interactionezi, `await flushPromises()` din nou. `trigger()` si `setValue()` includ automat nextTick. MSW e alternativa superioara pentru mock HTTP - intercepteaza la nivel de network, teste mai realiste.

### Intrebarea 7: Strategie de testare pentru aplicatie Vue enterprise?

Piramida: 70% component tests (VTU), 20% unit tests (Vitest), 10% E2E (Playwright). Component tests acopera interactiunile utilizator si sunt rapide in CI. Unit tests pentru composables, stores si business logic pura. E2E doar pentru flow-uri critice: autentificare, checkout. Infrastructura: Vitest cu happy-dom, MSW, createTestingPinia, factory functions. Coverage: 80% pe business logic, 60% pe UI. CI: unit+component pe fiecare PR (sub 2 min), E2E pe merge in main.

### Intrebarea 8: Cum testezi MFE-uri in Vue?

Fiecare MFE se testeaza independent ca aplicatie completa. Pentru integrare, testezi interfetele: custom events, props de la shell, shared dependencies. Mock-uiesti shell-ul cu provide care simuleaza contextul: `provide: { shellConfig: mockConfig }`. Cross-MFE communication se testeaza cu E2E. Contract testing (Pact) e util pentru verificarea contractelor intre MFE-uri. Shared state (Pinia) se testeaza izolat, apoi integrarea. Recomandare: cat mai putina comunicare intre MFE-uri.

### Intrebarea 9: Compara testing-ul Angular vs Vue.

Vue testing e semnificativ mai simplu. Nu ai TestBed cu module configuration, nu ai declarations/imports/providers ceremony. `mount()` e tot ce ai nevoie. Angular are avantajul unui framework mai structurat - `HttpClientTestingModule` verifica automat request-uri neasteptate, TestBed forteaza configurare explicita. Dar setup-ul e 3-5x mai verbos. Composable-urile se testeaza ca functii simple vs serviciile care necesita intotdeauna TestBed. Pinia testing e dramatic mai simplu decat NgRx. Dezavantajul Vue: echipa trebuie sa-si defineasca propriile conventii.

### Intrebarea 10: Component testing vs E2E - unde tragi linia?

Daca poti testa cu VTU, fa-o acolo - e mai rapid si stabil. E2E cand: testezi interactiunea intre pagini, integrarea cu servicii externe reale, flow-uri care depind de starea browser-ului (cookies, auth sessions), sau responsive behavior. Flow-urile critice (autentificare, plata) merita E2E chiar daca le poti simula. Metrica: daca testul VTU necesita mai mult de 3 mount-uri sau mock-uieste 5+ dependente, probabil e un E2E deghizat. CI: component tests pe fiecare commit (30s), E2E pe PR merge (5-10 min).

### Intrebarea 11 (bonus): Cum abordezi testabilitatea la design?

Testabilitatea e un concern architectural. Principii: **composable extraction** - logica complexa in composable-uri testabile independent. **Prop-driven rendering** - componente deterministe bazate pe props. **data-testid** - selectori stabili, fara dependenta de clase CSS. **Single responsibility** - daca ai describe-uri multiple, componenta trebuie sparta. **Dependency injection** - provide/inject pentru dependente externe, permite mock-uire. **Event-driven** - emit in loc de mutare directa state parinte; emit-urile sunt usor de verificat cu `wrapper.emitted()`.

---

> **Nota finala**: Testarea in Vue 3 e una dintre cele mai placute experiente de testing in frontend.
> Daca vii din Angular, vei aprecia simplicitatea: fara TestBed, fara module configuration,
> fara detectChanges(). `mount()`, `trigger()`, `expect()` - atat de simplu.
> Focus-ul trece de la "cum configurez testul" la "ce vreau sa testez".


---

**Următor :** [**10 - DevOps, CI/CD, Azure** →](Vue/10-DevOps-CICD-Azure.md)