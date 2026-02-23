# Reactivitate în Vue 3 vs Angular (Interview Prep - Senior Frontend Architect)

> Sistemul reactiv Vue 3 bazat pe Proxy, comparație detaliată cu Angular Signals și RxJS.
> ref(), reactive(), computed(), watch(), watchEffect(), toRef(), toRefs().
> Perspective de arhitect: trade-offs, gotchas, optimizări de performanță.

---

## Cuprins

1. [Cum funcționează reactivitatea în Vue 3 (Proxy-based)](#1-cum-funcționează-reactivitatea-în-vue-3-sub-capotă)
2. [ref() în detaliu](#2-ref-în-detaliu)
3. [reactive() în detaliu](#3-reactive-în-detaliu)
4. [computed() - valori derivate](#4-computed---valori-derivate)
5. [watch() - watchers expliciți](#5-watch---watchers-expliciți)
6. [watchEffect() - side effects automate](#6-watcheffect---side-effects-automate)
7. [toRef() și toRefs()](#7-toref-și-torefs)
8. [shallowRef și shallowReactive](#8-shallowref-și-shallowreactive)
9. [Comparație completă: Vue Reactivity vs Angular Signals vs RxJS](#9-comparație-completă-vue-reactivity-vs-angular-signals-vs-rxjs)
10. [Diagrame: cum Vue tracks dependencies](#10-diagrame-cum-vue-tracks-dependencies)
11. [Întrebări de interviu](#11-întrebări-de-interviu)

---

## 1. Cum funcționează reactivitatea în Vue 3 (sub capotă)

### 1.1 Evoluția: Vue 2 → Vue 3

**Vue 2** folosea `Object.defineProperty()` pentru a intercepta get/set pe fiecare proprietate:

```typescript
// Vue 2 - simplificat
Object.defineProperty(obj, 'count', {
  get() {
    // track dependency
    return value
  },
  set(newVal) {
    value = newVal
    // trigger updates
  }
})
```

**Limitări Vue 2:**
- Nu putea detecta adăugarea/ștergerea proprietăților noi (`Vue.set()` era necesar)
- Nu putea detecta modificări pe array prin index (`arr[0] = 'x'` nu funcționa)
- Trebuia să "walk" recursiv la init pe fiecare proprietate
- Performanță slabă pe obiecte mari (inițializare costisitoare)

**Vue 3** folosește **ES6 Proxy** - interceptează ORICE operație pe obiect:

```typescript
// Vue 3 - Proxy interceptează totul
const proxy = new Proxy(target, {
  get(target, key, receiver) { /* ... */ },
  set(target, key, value, receiver) { /* ... */ },
  deleteProperty(target, key) { /* ... */ },
  has(target, key) { /* ... */ },
  ownKeys(target) { /* ... */ }
})
```

**Avantaje Proxy:**
- Detectează proprietăți noi automat
- Detectează ștergeri de proprietăți
- Interceptează array mutations (push, splice, etc.)
- Lazy: proprietățile nested devin reactive doar când sunt accesate
- Performanță mai bună la inițializare

---

### 1.2 Track & Trigger Model

Conceptul fundamental al reactivității Vue 3:

```
Component Render (effect)
    |
    v
Accesează ref.value  ──> track() ──> adaugă effect la dependency set
    |
    v
Modifică ref.value   ──> trigger() ──> re-rulează toate effects dependente
    |
    v
Re-render component (doar cel afectat, NU toată aplicația)
```

**Flux detaliat:**

```
┌─────────────────────────────────────────────────────────┐
│                    REACTIVE SYSTEM                        │
│                                                           │
│  1. Effect (render/computed/watch) pornește               │
│     │                                                     │
│  2. activeEffect = currentEffect                          │
│     │                                                     │
│  3. Effect accesează state.count                          │
│     │                                                     │
│  4. Proxy GET handler → track(target, 'count')            │
│     │                                                     │
│  5. targetMap[target]['count'].add(activeEffect)           │
│     │                                                     │
│  6. Mai târziu: state.count = 5                            │
│     │                                                     │
│  7. Proxy SET handler → trigger(target, 'count')          │
│     │                                                     │
│  8. targetMap[target]['count'].forEach(effect => effect()) │
│     │                                                     │
│  9. Doar effects care depind de 'count' se re-rulează     │
└─────────────────────────────────────────────────────────┘
```

---

### 1.3 Implementare detaliată a Proxy-ului

```typescript
// Simplificat - cum funcționează reactive() intern în Vue 3

// WeakMap: target → Map<key, Set<effect>>
const targetMap = new WeakMap<object, Map<string | symbol, Set<Function>>>()

// Effect-ul activ curent (render function, computed, watcher)
let activeEffect: Function | null = null

function track(target: object, key: string | symbol): void {
  if (!activeEffect) return  // Nimeni nu "ascultă", nu facem nimic

  let depsMap = targetMap.get(target)
  if (!depsMap) {
    targetMap.set(target, (depsMap = new Map()))
  }

  let deps = depsMap.get(key)
  if (!deps) {
    depsMap.set(key, (deps = new Set()))
  }

  deps.add(activeEffect)  // Înregistrează cine citește această proprietate
}

function trigger(target: object, key: string | symbol): void {
  const depsMap = targetMap.get(target)
  if (!depsMap) return

  const deps = depsMap.get(key)
  if (!deps) return

  // Notifică toți watchers/effects care depind de această cheie
  deps.forEach(effect => {
    // Scheduler: effects sunt batched, nu rulează sincron
    queueJob(effect)
  })
}

function reactive<T extends object>(target: T): T {
  return new Proxy(target, {
    get(target, key, receiver) {
      track(target, key)           // Înregistrează cine citește
      const result = Reflect.get(target, key, receiver)

      // Lazy deep reactivity: convertim nested objects doar la acces
      if (typeof result === 'object' && result !== null) {
        return reactive(result)
      }
      return result
    },

    set(target, key, value, receiver) {
      const oldValue = Reflect.get(target, key, receiver)
      const result = Reflect.set(target, key, value, receiver)

      // Trigger doar dacă valoarea s-a schimbat (evită re-renders inutile)
      if (oldValue !== value) {
        trigger(target, key)       // Notifică toți watchers
      }
      return result
    },

    deleteProperty(target, key) {
      const result = Reflect.deleteProperty(target, key)
      trigger(target, key)         // Notifică și la ștergere
      return result
    }
  })
}
```

---

### 1.4 Batching și Scheduler

Vue **NU** face update-uri sincron. Toate modificările sunt **batched**:

```typescript
const count = ref(0)

// Aceste 3 modificări NU generează 3 re-renders
count.value++
count.value++
count.value++
// → Un singur re-render cu count = 3

// Dacă ai nevoie de DOM-ul actualizat:
import { nextTick } from 'vue'

count.value = 10
await nextTick()
// DOM-ul e acum actualizat
console.log(document.querySelector('#count').textContent) // "10"
```

**Flush timing options:**
- `'pre'` - default: rulează înainte de DOM update
- `'post'` - rulează după DOM update
- `'sync'` - rulează sincron (rareori necesar, impact pe performanță)

---

### 1.5 Paralelă cu Angular

| Concept | Vue 3 | Angular |
|---------|-------|---------|
| **Mecanism de bază** | ES6 Proxy | Zone.js (traditional) / Signals (modern) |
| **Dependency tracking** | Automat via Proxy GET | Manual cu Signals / Zone.js change detection |
| **Batching** | Automat (microtask queue) | Zone.js batching / Signal notification batching |
| **Granularitate** | Proprietate individuală | Component tree (Zone.js) / Signal (Signals) |
| **Re-render scope** | Doar component afectat | Component + children (OnPush ajută) |

**Angular Signals (v17+)** au un model similar cu Vue:

```typescript
// Angular Signals
const count = signal(0)        // ≈ ref(0)
const doubled = computed(() => count() * 2)  // ≈ computed()
effect(() => console.log(count()))           // ≈ watchEffect()

// Vue 3
const count = ref(0)
const doubled = computed(() => count.value * 2)
watchEffect(() => console.log(count.value))
```

**Diferența fundamentală:**
- Vue: reactivitatea e **implicită** - accesezi `.value` și Proxy face tracking automat
- Angular Signals: reactivitatea e **explicită** - apelezi `count()` (getter function)
- Angular tradițional: Zone.js face monkey-patch pe async APIs (setTimeout, Promise, etc.) și triggează change detection pe tot component tree-ul

---

## 2. ref() în detaliu

### 2.1 Baza: Ce este ref()?

`ref()` creează un **obiect reactive wrapper** cu proprietatea `.value`:

```typescript
import { ref, type Ref } from 'vue'

// Creează un ref cu o valoare inițială
const count: Ref<number> = ref(0)

// TypeScript inferă tipul automat
const name = ref('Emanuel')           // Ref<string>
const isActive = ref(true)            // Ref<boolean>
const items = ref<string[]>([])       // Ref<string[]> - explicit generic
const user = ref<User | null>(null)   // Ref<User | null> - nullable
```

**De ce `.value`?**
- JavaScript nu poate intercepta citirea/scrierea pe primitive
- `ref()` wrap-uiește primitiva într-un obiect `{ value: T }`
- Proxy-ul e pe acest obiect, interceptând `.value` get/set

```typescript
// Intern, ref arată cam așa:
class RefImpl<T> {
  private _value: T
  public dep: Set<Effect> = new Set()  // dependency set

  constructor(value: T) {
    this._value = isObject(value) ? reactive(value) : value
  }

  get value(): T {
    trackRefValue(this)  // track dependency
    return this._value
  }

  set value(newVal: T) {
    if (hasChanged(newVal, this._value)) {
      this._value = isObject(newVal) ? reactive(newVal) : newVal
      triggerRefValue(this)  // trigger updates
    }
  }
}
```

---

### 2.2 Utilizare cu primitive

```typescript
import { ref } from 'vue'

// Tipuri primitive
const count = ref(0)
const message = ref('Hello')
const isVisible = ref(false)
const price = ref(29.99)

// Citire
console.log(count.value)     // 0
console.log(message.value)   // 'Hello'

// Scriere
count.value++                // 1
count.value = 100            // 100
message.value = 'Salut!'     // 'Salut!'
isVisible.value = !isVisible.value  // toggle

// Comparație
if (count.value > 50) {
  message.value = 'Peste limită'
}
```

---

### 2.3 Utilizare cu obiecte

Când pasezi un obiect în `ref()`, obiectul devine **deep reactive** intern (folosind `reactive()`):

```typescript
interface User {
  id: number
  name: string
  address: {
    city: string
    country: string
  }
  skills: string[]
}

const user = ref<User>({
  id: 1,
  name: 'Emanuel',
  address: {
    city: 'București',
    country: 'România'
  },
  skills: ['Angular', 'Vue', 'TypeScript']
})

// Acces la proprietăți nested (deep reactive)
console.log(user.value.name)              // 'Emanuel'
console.log(user.value.address.city)      // 'București'

// Modificare deep - TRIGGEAZĂ re-render
user.value.name = 'John'
user.value.address.city = 'Cluj'
user.value.skills.push('React')

// REASSIGNMENT - funcționează cu ref() (nu cu reactive()!)
user.value = {
  id: 2,
  name: 'Completely New User',
  address: { city: 'Timișoara', country: 'România' },
  skills: ['Python']
}
```

---

### 2.4 Auto-unwrapping rules

**Regulă 1: Template auto-unwrap**

```vue
<script setup lang="ts">
import { ref } from 'vue'

const count = ref(0)
const user = ref({ name: 'Emanuel' })
</script>

<template>
  <!-- Auto-unwrap - NU scrii .value -->
  <p>Count: {{ count }}</p>
  <p>User: {{ user.name }}</p>

  <!-- Event handlers - tot fără .value -->
  <button @click="count++">Increment</button>

  <!-- v-bind - fără .value -->
  <input :value="count" />

  <!-- ATENȚIE: expresii complexe pot necesita .value -->
  <!-- {{ count.value + 1 }}  ← GREȘIT în template -->
  <!-- {{ count + 1 }}        ← CORECT în template -->
</template>
```

**Regulă 2: Unwrap în reactive()**

```typescript
const count = ref(0)
const state = reactive({
  count  // ref e auto-unwrapped în reactive
})

// Acces FĂRĂ .value
state.count++         // NU state.count.value++
console.log(state.count)  // 1, nu Ref<number>

// Dar: ref-ul original se actualizează bidirecțional
console.log(count.value)  // 1
```

**Regulă 3: Arrays și Maps - NU se unwrap**

```typescript
const items = reactive([ref(1), ref(2), ref(3)])

// NU se unwrap în array
console.log(items[0].value)  // 1, TREBUIE .value
items[0].value = 10          // TREBUIE .value

const map = reactive(new Map([['key', ref('value')]]))
console.log(map.get('key')!.value)  // TREBUIE .value
```

---

### 2.5 ref() vs Angular Signals

```typescript
// ------------ Vue 3 ref() ------------
import { ref, computed, watchEffect } from 'vue'

const count = ref(0)
const doubled = computed(() => count.value * 2)

// Read
console.log(count.value)      // 0
console.log(doubled.value)    // 0

// Write
count.value = 5

// Effect
watchEffect(() => {
  console.log(`Count is: ${count.value}`)
})

// ------------ Angular Signals ------------
import { signal, computed, effect } from '@angular/core'

const count = signal(0)
const doubled = computed(() => count() * 2)

// Read
console.log(count())           // 0
console.log(doubled())         // 0

// Write
count.set(5)
// sau: count.update(v => v + 1)

// Effect
effect(() => {
  console.log(`Count is: ${count()}`)
})
```

**Diferențe cheie:**
| Aspect | Vue ref() | Angular signal() |
|--------|-----------|-----------------|
| **Citire** | `.value` | `()` (function call) |
| **Scriere** | `.value = x` | `.set(x)` / `.update(fn)` |
| **Template** | auto-unwrap | `()` necesar (sau pipes viitoare) |
| **Mutație** | `ref.value++` | `signal.update(v => v + 1)` |
| **Tip intern** | `Ref<T>` | `WritableSignal<T>` |
| **Readonly** | `readonly(ref)` | `signal.asReadonly()` |

---

### 2.6 Pattern-uri avansate cu ref()

```typescript
// Template ref - referință la element DOM
const inputEl = ref<HTMLInputElement | null>(null)

onMounted(() => {
  inputEl.value?.focus()
})

// Component ref
const childComponent = ref<InstanceType<typeof MyComponent> | null>(null)

onMounted(() => {
  childComponent.value?.someMethod()
})
```

```vue
<template>
  <input ref="inputEl" />
  <MyComponent ref="childComponent" />
</template>
```

```typescript
// Custom ref - customRef()
import { customRef } from 'vue'

function useDebouncedRef<T>(value: T, delay = 300) {
  let timeout: ReturnType<typeof setTimeout>

  return customRef<T>((track, trigger) => ({
    get() {
      track()
      return value
    },
    set(newValue: T) {
      clearTimeout(timeout)
      timeout = setTimeout(() => {
        value = newValue
        trigger()
      }, delay)
    }
  }))
}

// Utilizare
const searchQuery = useDebouncedRef('', 500)
// searchQuery.value = 'test' → trigger-ul vine după 500ms
```

---

## 3. reactive() în detaliu

### 3.1 Ce este reactive()?

`reactive()` creează un **Proxy profund reactiv** direct pe un obiect:

```typescript
import { reactive } from 'vue'

// Obiecte simple
const state = reactive({
  count: 0,
  message: 'Hello',
  user: {
    name: 'Emanuel',
    age: 30
  }
})

// Acces direct - FĂRĂ .value
state.count++
state.message = 'Salut'
state.user.name = 'John'  // deep reactive

// Arrays
const list = reactive(['a', 'b', 'c'])
list.push('d')        // reactive
list[0] = 'x'         // reactive (spre deosebire de Vue 2!)
list.splice(1, 1)     // reactive

// Maps și Sets
const map = reactive(new Map<string, number>())
map.set('key', 42)    // reactive

const set = reactive(new Set<string>())
set.add('item')       // reactive
```

---

### 3.2 Deep reactivity

`reactive()` convertește **recursiv** toate proprietățile nested:

```typescript
const state = reactive({
  level1: {
    level2: {
      level3: {
        value: 'deeply nested'
      }
    }
  }
})

// Toate nivelurile sunt reactive
state.level1.level2.level3.value = 'modified'  // Triggerează re-render

// Verifică dacă un obiect e reactive
import { isReactive } from 'vue'

console.log(isReactive(state))                     // true
console.log(isReactive(state.level1))              // true
console.log(isReactive(state.level1.level2))       // true
console.log(isReactive(state.level1.level2.level3)) // true
```

---

### 3.3 GOTCHAS - Pierderea reactivității

**Gotcha 1: Reassignment**

```typescript
let state = reactive({ count: 0 })

// GREȘIT - pierde proxy-ul
// state = reactive({ count: 1 })  // Variabila pointează la un proxy NOU
// Componentele care refereau proxy-ul vechi NU se actualizează

// CORECT - modifică proprietățile
state.count = 1

// Dacă ai NEVOIE de reassignment, folosește ref() cu obiect:
const state2 = ref({ count: 0 })
state2.value = { count: 1 }  // OK - ref poate fi reassigned
```

**Gotcha 2: Destructurare**

```typescript
const state = reactive({
  count: 0,
  name: 'Emanuel'
})

// GREȘIT - pierde reactivitatea
const { count, name } = state
// count e acum doar un number (0), nu mai e reactiv
console.log(count)  // 0
state.count = 5
console.log(count)  // tot 0! Nu s-a actualizat

// CORECT - folosește toRefs()
import { toRefs } from 'vue'
const { count: countRef, name: nameRef } = toRefs(state)
// countRef e Ref<number>, linked la state.count
state.count = 5
console.log(countRef.value)  // 5 - actualizat!
```

**Gotcha 3: Pasare ca argument la funcție**

```typescript
const state = reactive({ count: 0 })

// GREȘIT - pasează valoarea, nu referința
function logCount(count: number) {
  // count e doar un number aici
  console.log(count)
}
logCount(state.count)  // Pierde reactivitatea

// CORECT - pasează obiectul
function logCount2(state: { count: number }) {
  console.log(state.count)  // Reactiv
}

// SAU folosește toRef
import { toRef } from 'vue'
function logCount3(count: Ref<number>) {
  watch(count, (val) => console.log(val))
}
logCount3(toRef(state, 'count'))
```

**Gotcha 4: Spread operator**

```typescript
const state = reactive({ a: 1, b: 2, c: 3 })

// GREȘIT - spread creează un obiect plain
const copy = { ...state }
// copy NU e reactiv

// CORECT - dacă vrei o copie reactivă
const copy2 = reactive({ ...state })
// copy2 e reactiv dar INDEPENDENT de state

// Dacă vrei linked refs:
const refs = toRefs(state)  // { a: Ref, b: Ref, c: Ref }
```

---

### 3.4 reactive() vs ref() - Când folosești fiecare?

| Criteriu | ref() | reactive() |
|----------|-------|------------|
| **Primitive** | Da | Nu |
| **Obiecte** | Da (cu .value) | Da (fără .value) |
| **Reassignment** | Da | Nu (pierde proxy) |
| **Destructurare** | N/A | Pierde reactivitatea |
| **Tip return** | `Ref<T>` | `T` (proxied) |
| **Template** | Auto-unwrap | Direct |
| **Composables return** | Preferat (mai explicit) | Necesită toRefs() |
| **Performance** | Overhead minim (.value) | Overhead minim (Proxy) |

**Recomandarea oficială Vue:**
- Folosește `ref()` ca default - e mai predictibil
- Folosește `reactive()` pentru state complex local al unui component
- În composables, returnează întotdeauna `ref()` sau `toRefs(reactive())

---

### 3.5 Paralelă cu Angular

Angular **NU** are un echivalent direct pentru `reactive()`:

```typescript
// Vue 3 - reactive object
const state = reactive({
  count: 0,
  user: { name: 'Emanuel' }
})
state.count++
state.user.name = 'John'

// Angular - trebuie signals individuale sau un signal cu obiect
const count = signal(0)
const user = signal({ name: 'Emanuel' })

count.update(v => v + 1)
user.update(u => ({ ...u, name: 'John' }))  // Trebuie spread!

// SAU: Angular cu signal pe obiect (deep mutation NU triggerează)
const state2 = signal({ count: 0, user: { name: 'Emanuel' } })
// state2().count++  ← NU triggerează update-ul!
// Trebuie: state2.update(s => ({ ...s, count: s.count + 1 }))
```

**Key insight:** Vue `reactive()` e mult mai ergonomic pentru state complex deoarece mutațiile directe funcționează. Angular Signals necesită immutable updates (spread) sau signals individuale.

---

## 4. computed() - Valori derivate

### 4.1 Basics

`computed()` creează o **valoare derivată cached** care se recalculează automat când dependențele se schimbă:

```typescript
import { ref, computed } from 'vue'

const price = ref(100)
const quantity = ref(3)
const taxRate = ref(0.19)  // TVA 19%

// Computed simplu (readonly)
const subtotal = computed(() => price.value * quantity.value)
const tax = computed(() => subtotal.value * taxRate.value)
const total = computed(() => subtotal.value + tax.value)

console.log(subtotal.value)  // 300
console.log(tax.value)       // 57
console.log(total.value)     // 357

// Modificarea unei dependențe recalculează automat
price.value = 200
console.log(total.value)     // 714 (recalculat automat)
```

---

### 4.2 Caching - de ce contează

```typescript
const list = ref([1, 2, 3, 4, 5, 6, 7, 8, 9, 10])

// COMPUTED - cached, se calculează o singură dată
const expensiveSorted = computed(() => {
  console.log('Sorting...')  // Se logează doar la schimbarea listei
  return [...list.value].sort((a, b) => a - b)
})

// Acces multiplu - NU recalculează
console.log(expensiveSorted.value)  // Sorting... [1,2,3,...]
console.log(expensiveSorted.value)  // [1,2,3,...] (fără "Sorting...")
console.log(expensiveSorted.value)  // [1,2,3,...] (fără "Sorting...")

// FUNCTION în template - se apelează la FIECARE re-render
function getExpensiveSorted() {
  console.log('Sorting...')  // Se logează la FIECARE re-render
  return [...list.value].sort((a, b) => a - b)
}
```

```vue
<template>
  <!-- BINE: computed, cached -->
  <div v-for="item in expensiveSorted" :key="item">{{ item }}</div>

  <!-- RĂU: function call, recalculat la fiecare re-render -->
  <div v-for="item in getExpensiveSorted()" :key="item">{{ item }}</div>
</template>
```

---

### 4.3 Writable computed

```typescript
import { ref, computed } from 'vue'

const firstName = ref('Emanuel')
const lastName = ref('Popescu')

// Writable computed cu get/set
const fullName = computed({
  get: () => `${firstName.value} ${lastName.value}`,
  set: (val: string) => {
    const parts = val.split(' ')
    firstName.value = parts[0] || ''
    lastName.value = parts.slice(1).join(' ') || ''
  }
})

console.log(fullName.value)  // 'Emanuel Popescu'

fullName.value = 'John Doe'
console.log(firstName.value) // 'John'
console.log(lastName.value)  // 'Doe'
```

```vue
<template>
  <!-- v-model pe writable computed -->
  <input v-model="fullName" />
  <p>First: {{ firstName }}, Last: {{ lastName }}</p>
</template>
```

---

### 4.4 Computed cu reactive state

```typescript
const state = reactive({
  items: [
    { id: 1, name: 'Laptop', price: 5000, category: 'tech' },
    { id: 2, name: 'Book', price: 50, category: 'education' },
    { id: 3, name: 'Phone', price: 3000, category: 'tech' }
  ],
  filter: 'all' as string,
  searchQuery: ''
})

// Computed chain
const filteredItems = computed(() => {
  let result = state.items

  if (state.filter !== 'all') {
    result = result.filter(item => item.category === state.filter)
  }

  if (state.searchQuery) {
    const query = state.searchQuery.toLowerCase()
    result = result.filter(item =>
      item.name.toLowerCase().includes(query)
    )
  }

  return result
})

const totalPrice = computed(() =>
  filteredItems.value.reduce((sum, item) => sum + item.price, 0)
)

const itemCount = computed(() => filteredItems.value.length)

// Dependency chain: items/filter/searchQuery → filteredItems → totalPrice/itemCount
```

---

### 4.5 Best practices pentru computed

```typescript
// ✅ BUN: Pure computation, fără side effects
const fullName = computed(() => `${first.value} ${last.value}`)

// ❌ RĂU: Side effects în computed
const fullName = computed(() => {
  console.log('Computing...')           // Side effect!
  localStorage.setItem('name', ...)     // Side effect!
  someOtherRef.value = 'changed'        // Mutație în computed!
  return `${first.value} ${last.value}`
})

// ✅ BUN: Computed în loc de watch + ref
const doubled = computed(() => count.value * 2)

// ❌ RĂU: watch + ref când computed e suficient
const doubled = ref(0)
watch(count, (val) => { doubled.value = val * 2 })  // Overcomplicated

// ✅ BUN: Computed cu debugging (Vue 3.4+)
const doubled = computed(() => count.value * 2, {
  onTrack(e) {
    // Se apelează când o dependență e tracked
    debugger
  },
  onTrigger(e) {
    // Se apelează când computed e invalidat
    debugger
  }
})
```

---

### 4.6 Paralelă cu Angular

```typescript
// ------------ Vue 3 computed() ------------
const count = ref(0)
const doubled = computed(() => count.value * 2)
const tripled = computed(() => count.value * 3)
const sum = computed(() => doubled.value + tripled.value)

// ------------ Angular computed() ------------
const count = signal(0)
const doubled = computed(() => count() * 2)
const tripled = computed(() => count() * 3)
const sum = computed(() => doubled() + tripled())

// ------------ RxJS (Angular traditional) ------------
const count$ = new BehaviorSubject(0)
const doubled$ = count$.pipe(map(v => v * 2))
const tripled$ = count$.pipe(map(v => v * 3))
const sum$ = combineLatest([doubled$, tripled$]).pipe(
  map(([d, t]) => d + t)
)
```

**Key insight:** Vue `computed()` și Angular `computed()` sunt **aproape identice** ca API și comportament. Caching, lazy evaluation, dependency tracking - totul e la fel. Singura diferență e `.value` vs `()`.

---

## 5. watch() - Watchers expliciți

### 5.1 Basics

`watch()` observă una sau mai multe surse reactive și rulează un callback când valorile se schimbă:

```typescript
import { ref, reactive, watch } from 'vue'

const count = ref(0)

// Watch simplu pe un ref
watch(count, (newValue, oldValue) => {
  console.log(`Count: ${oldValue} → ${newValue}`)
})

count.value = 5  // Log: "Count: 0 → 5"
```

---

### 5.2 Surse watchable

```typescript
const count = ref(0)
const name = ref('Emanuel')
const state = reactive({ deep: { nested: { value: 42 } } })

// 1. Watch ref
watch(count, (newVal, oldVal) => {
  console.log(`Count changed: ${oldVal} → ${newVal}`)
})

// 2. Watch getter function
watch(
  () => state.deep.nested.value,
  (newVal, oldVal) => {
    console.log(`Nested value: ${oldVal} → ${newVal}`)
  }
)

// 3. Watch reactive object (deep by default)
watch(state, (newState) => {
  // ATENȚIE: newState === oldState (același obiect proxy)
  console.log('State changed:', newState.deep.nested.value)
})

// 4. Watch array de surse
watch(
  [count, name, () => state.deep.nested.value],
  ([newCount, newName, newNested], [oldCount, oldName, oldNested]) => {
    console.log('Multiple sources changed')
    console.log(`Count: ${oldCount} → ${newCount}`)
    console.log(`Name: ${oldName} → ${newName}`)
    console.log(`Nested: ${oldNested} → ${newNested}`)
  }
)

// GREȘIT - nu funcționează direct pe o proprietate de reactive
// watch(state.deep.nested.value, ...) ← NU! E o valoare plain
// FIX: wrap în getter
// watch(() => state.deep.nested.value, ...)  ← CORECT
```

---

### 5.3 Options

```typescript
const searchQuery = ref('')

watch(searchQuery, (newVal, oldVal) => {
  fetchResults(newVal)
}, {
  // Rulează callback-ul imediat cu valoarea curentă
  immediate: true,

  // Deep watch (necesar pentru ref cu obiect, implicit pentru reactive)
  deep: true,

  // Când se rulează callback-ul relativ la DOM update
  flush: 'post',  // 'pre' (default) | 'post' | 'sync'

  // Limită adâncime deep watch (Vue 3.5+)
  // deep: 2,  // Watch doar primele 2 niveluri

  // Once - rulează o singură dată (Vue 3.4+)
  once: true
})
```

**flush options detaliat:**

```typescript
const count = ref(0)

// flush: 'pre' (DEFAULT)
// Callback rulează ÎNAINTE de DOM update
watch(count, () => {
  // DOM-ul NU e încă actualizat aici
  console.log(document.getElementById('count')?.textContent)  // valoarea VECHE
}, { flush: 'pre' })

// flush: 'post'
// Callback rulează DUPĂ DOM update
watch(count, () => {
  // DOM-ul E actualizat aici
  console.log(document.getElementById('count')?.textContent)  // valoarea NOUĂ
}, { flush: 'post' })
// Alias: watchPostEffect()

// flush: 'sync'
// Callback rulează SINCRON la fiecare modificare (fără batching)
watch(count, () => {
  // Atenție la performanță!
}, { flush: 'sync' })
// Alias: watchSyncEffect()
```

---

### 5.4 Cleanup function

Essential pentru a evita race conditions cu operații asincrone:

```typescript
const searchQuery = ref('')

watch(searchQuery, async (newQuery, oldQuery, onCleanup) => {
  // AbortController pentru a anula request-uri vechi
  const controller = new AbortController()

  // Cleanup se apelează ÎNAINTE de re-rularea callback-ului
  onCleanup(() => {
    controller.abort()
  })

  try {
    const response = await fetch(`/api/search?q=${newQuery}`, {
      signal: controller.signal
    })
    const data = await response.json()
    results.value = data
  } catch (e) {
    if (e instanceof DOMException && e.name === 'AbortError') {
      // Request anulat - normal, ignorăm
      return
    }
    error.value = e as Error
  }
})
```

**Vue 3.5+ syntax cu onWatcherCleanup:**

```typescript
import { watch, onWatcherCleanup } from 'vue'

watch(searchQuery, async (newQuery) => {
  const controller = new AbortController()

  // Noua API - nu mai vine ca parametru
  onWatcherCleanup(() => {
    controller.abort()
  })

  const response = await fetch(`/api/search?q=${newQuery}`, {
    signal: controller.signal
  })
  // ...
})
```

---

### 5.5 Stopping watchers

```typescript
// watch() returnează o funcție stop
const stopWatching = watch(count, (val) => {
  console.log('Count:', val)

  // Self-stop: oprește watcher-ul din interiorul callback-ului
  if (val >= 10) {
    stopWatching()
  }
})

// Oprire manuală
stopWatching()

// Watchers din setup() sunt opriți automat la unmount
// Watchers creați async NU sunt opriți automat!

// GREȘIT - watcher creat async nu se oprește automat
setTimeout(() => {
  // Acest watcher NU va fi oprit la component unmount!
  watch(count, () => { /* ... */ })
}, 1000)

// CORECT - oprește manual sau folosește scope
onMounted(() => {
  const stop = watch(count, () => { /* ... */ })
  onUnmounted(() => stop())
})
```

---

### 5.6 Deep watching - detalii și gotchas

```typescript
const user = ref({
  name: 'Emanuel',
  preferences: {
    theme: 'dark',
    language: 'ro',
    notifications: {
      email: true,
      push: false
    }
  }
})

// Fără deep: NU detectează nested changes
watch(user, (newVal) => {
  console.log('User changed')  // Nu se apelează la nested changes
})

// Cu deep: detectează ORICE nested change
watch(user, (newVal, oldVal) => {
  console.log('User changed (deep)')
  // ATENȚIE: newVal === oldVal pentru nested mutations
  // Ambele pointează la același obiect
}, { deep: true })

// Alternativă: watch pe getter specific
watch(
  () => user.value.preferences.theme,
  (newTheme, oldTheme) => {
    console.log(`Theme: ${oldTheme} → ${newTheme}`)
    // Aici oldVal ȘI newVal sunt corecte (primitive)
  }
)

// Deep watch pe reactive (deep e implicit)
const state = reactive({ nested: { value: 1 } })
watch(state, () => {
  // Deep watching implicit
  console.log('State changed')
})
```

---

### 5.7 Pattern-uri practice

```typescript
// Pattern 1: Debounced search
const searchQuery = ref('')
const results = ref<SearchResult[]>([])
const isSearching = ref(false)

watch(searchQuery, async (query, _, onCleanup) => {
  if (!query.trim()) {
    results.value = []
    return
  }

  isSearching.value = true
  const controller = new AbortController()
  onCleanup(() => controller.abort())

  // Debounce manual
  await new Promise(resolve => setTimeout(resolve, 300))

  try {
    const res = await fetch(`/api/search?q=${query}`, {
      signal: controller.signal
    })
    results.value = await res.json()
  } catch (e) {
    if (!(e instanceof DOMException)) throw e
  } finally {
    isSearching.value = false
  }
})

// Pattern 2: Route params watching
import { useRoute } from 'vue-router'

const route = useRoute()

watch(
  () => route.params.id,
  async (newId) => {
    if (newId) {
      await fetchUser(newId as string)
    }
  },
  { immediate: true }
)

// Pattern 3: Persist to localStorage
const settings = ref({
  theme: 'dark',
  fontSize: 16
})

watch(settings, (val) => {
  localStorage.setItem('settings', JSON.stringify(val))
}, { deep: true })

// Pattern 4: Conditional watching
const isEnabled = ref(true)
const data = ref(0)

watch(
  () => isEnabled.value ? data.value : undefined,
  (val) => {
    if (val !== undefined) {
      console.log('Data changed while enabled:', val)
    }
  }
)
```

---

### 5.8 Paralelă cu Angular

```typescript
// ------------ Vue 3 watch() ------------
const userId = ref(1)

// Watch cu old/new values
watch(userId, (newId, oldId) => {
  console.log(`User changed: ${oldId} → ${newId}`)
  fetchUser(newId)
})

// Watch cu immediate
watch(userId, (id) => {
  fetchUser(id)
}, { immediate: true })

// Watch multiple
watch([firstName, lastName], ([f, l]) => {
  fullName.value = `${f} ${l}`
})

// ------------ Angular effect() (cel mai apropiat) ------------
// Angular effect() NU are old/new values
effect(() => {
  const id = userId()
  console.log('User changed to:', id)
  fetchUser(id)
})
// effect() e mai aproape de watchEffect() decât de watch()

// ------------ RxJS (Angular traditional) ------------
const userId$ = new BehaviorSubject(1)

// Cu old/new values
userId$.pipe(
  pairwise(),
  takeUntilDestroyed()
).subscribe(([oldId, newId]) => {
  console.log(`User changed: ${oldId} → ${newId}`)
  fetchUser(newId)
})

// Cu immediate
userId$.pipe(
  takeUntilDestroyed()
).subscribe(id => {
  fetchUser(id)
})

// Multiple sources
combineLatest([firstName$, lastName$]).pipe(
  takeUntilDestroyed()
).subscribe(([f, l]) => {
  fullName = `${f} ${l}`
})
```

**Diferențe notabile:**
- Vue `watch()` oferă `oldValue` nativ - Angular Signals nu
- Vue `watch()` e lazy default - Angular `effect()` e eager
- Vue `watch()` poate urmări surse specifice - Angular `effect()` auto-tracks
- RxJS `pairwise()` e echivalentul pentru old/new values

---

## 6. watchEffect() - Side Effects Automate

### 6.1 Basics

`watchEffect()` rulează imediat și auto-detectează dependențele:

```typescript
import { ref, watchEffect } from 'vue'

const url = ref('/api/users')
const data = ref(null)
const error = ref<Error | null>(null)
const loading = ref(false)

watchEffect(async () => {
  loading.value = true
  error.value = null

  try {
    const response = await fetch(url.value)  // Vue tracks url.value
    data.value = await response.json()
  } catch (e) {
    error.value = e as Error
  } finally {
    loading.value = false
  }
})

// Când url.value se schimbă, watchEffect se re-rulează automat
url.value = '/api/posts'  // Triggerează re-fetch
```

---

### 6.2 Dependency tracking automat

```typescript
const showActive = ref(true)
const activeCount = ref(5)
const inactiveCount = ref(3)

watchEffect(() => {
  // Dependențele sunt tracked DINAMIC
  if (showActive.value) {
    console.log('Active:', activeCount.value)
    // Dependencies: showActive, activeCount
  } else {
    console.log('Inactive:', inactiveCount.value)
    // Dependencies: showActive, inactiveCount
  }
})

// Dacă showActive e true: tracked = [showActive, activeCount]
// Dacă showActive devine false: tracked = [showActive, inactiveCount]
// activeCount NU mai e tracked dacă showActive e false
```

**IMPORTANT:** Dependențele sunt determinate la fiecare execuție. Dacă un `if` branch se schimbă, dependențele se schimbă.

---

### 6.3 Cleanup

```typescript
// Metoda 1: onCleanup parameter (Vue 3.0+)
watchEffect((onCleanup) => {
  const controller = new AbortController()

  onCleanup(() => {
    controller.abort()
    console.log('Cleanup: previous effect cancelled')
  })

  fetch(url.value, { signal: controller.signal })
    .then(res => res.json())
    .then(json => { data.value = json })
})

// Metoda 2: onWatcherCleanup (Vue 3.5+)
import { watchEffect, onWatcherCleanup } from 'vue'

watchEffect(() => {
  const controller = new AbortController()

  onWatcherCleanup(() => {
    controller.abort()
  })

  fetch(url.value, { signal: controller.signal })
    .then(res => res.json())
    .then(json => { data.value = json })
})

// Metoda 3: DOM event listeners
watchEffect((onCleanup) => {
  const handler = (e: Event) => {
    console.log('Scroll position:', window.scrollY)
  }

  document.addEventListener('scroll', handler)

  onCleanup(() => {
    document.removeEventListener('scroll', handler)
  })
})

// Metoda 4: Timer cleanup
watchEffect((onCleanup) => {
  const interval = setInterval(() => {
    counter.value++
  }, delay.value)  // delay e tracked

  onCleanup(() => clearInterval(interval))
})
```

---

### 6.4 watchEffect vs watch - Detalii

```typescript
const userId = ref(1)
const userData = ref(null)

// watchEffect: auto-track, eager, fără old value
watchEffect(async () => {
  // Rulează IMEDIAT
  // Auto-tracks userId.value
  const res = await fetch(`/api/users/${userId.value}`)
  userData.value = await res.json()
})

// watch: explicit source, lazy, cu old value
watch(userId, async (newId, oldId) => {
  // NU rulează imediat (dacă nu dai immediate: true)
  // Track-ează DOAR userId explicit
  console.log(`Changed from ${oldId} to ${newId}`)
  const res = await fetch(`/api/users/${newId}`)
  userData.value = await res.json()
})
```

### Tabel comparativ complet

| Aspect | `watch()` | `watchEffect()` |
|--------|-----------|-----------------|
| **Dependencies** | Explicit (surse declarate) | Automat (tracked la runtime) |
| **Old value** | Da (`oldVal` parameter) | Nu |
| **Execution** | Lazy (default) | Eager (imediat) |
| **Immediate option** | Da (`immediate: true`) | Întotdeauna imediat |
| **Deep option** | Da | N/A (auto-tracks ce accesezi) |
| **Source types** | ref, reactive, getter, array | N/A (orice ref/reactive accesat) |
| **Conditional deps** | Static (surse fixe) | Dinamic (se schimbă la fiecare run) |
| **Use case principal** | Reacție la surse specifice | Side effects generali |
| **Debugging** | Mai ușor (știi exact ce se schimbă) | Mai greu (deps dinamice) |
| **Analogie RxJS** | `combineLatest().subscribe()` | `autorun()` (MobX-style) |
| **Analogie Angular** | Nu are echivalent direct | `effect()` |

**Regula generală:**
- Folosește `watch()` când ai nevoie de `oldValue` sau vrei control explicit
- Folosește `watchEffect()` pentru side effects simple cu dependențe evidente
- Preferă `watch()` în cod complex (mai ușor de debuggat)

---

### 6.5 watchPostEffect() și watchSyncEffect()

```typescript
import { watchPostEffect, watchSyncEffect } from 'vue'

// Echivalent cu watchEffect cu flush: 'post'
// Rulează DUPĂ DOM update
watchPostEffect(() => {
  // DOM-ul e actualizat aici
  const el = document.querySelector('#my-element')
  if (el) {
    console.log('Element height:', el.clientHeight)
  }
})

// Echivalent cu watchEffect cu flush: 'sync'
// Rulează SINCRON (fără batching) - atenție la performanță!
watchSyncEffect(() => {
  console.log('Count is:', count.value)
  // Se apelează IMEDIAT la fiecare schimbare, fără batching
})
```

---

### 6.6 Paralelă cu Angular

```typescript
// ------------ Vue 3 watchEffect() ------------
const count = ref(0)
const name = ref('Emanuel')

watchEffect(() => {
  console.log(`Count: ${count.value}, Name: ${name.value}`)
  // Auto-tracks: count și name
  // Rulează imediat
  // Re-rulează când count SAU name se schimbă
})

// ------------ Angular effect() ------------
const count = signal(0)
const name = signal('Emanuel')

effect(() => {
  console.log(`Count: ${count()}, Name: ${name()}`)
  // Auto-tracks: count și name
  // Rulează imediat
  // Re-rulează când count SAU name se schimbă
})

// APROAPE IDENTICE!

// Diferența: cleanup
// Vue:
watchEffect((onCleanup) => {
  const sub = someObservable.subscribe()
  onCleanup(() => sub.unsubscribe())
})

// Angular:
effect((onCleanup) => {
  const sub = someObservable.subscribe()
  onCleanup(() => sub.unsubscribe())
})
// Angular 16+: DestroyRef.onDestroy() e mai comun
```

---

## 7. toRef() și toRefs()

### 7.1 toRef() - Singulară proprietate

Creează un `Ref` care e **synchronized** cu proprietatea unui reactive object:

```typescript
import { reactive, toRef, watch } from 'vue'

const state = reactive({
  count: 0,
  name: 'Emanuel',
  settings: {
    theme: 'dark'
  }
})

// toRef creează un ref linked la state.count
const countRef = toRef(state, 'count')

// Modificarea ref-ului modifică state-ul
countRef.value++
console.log(state.count)  // 1

// Modificarea state-ului modifică ref-ul
state.count = 10
console.log(countRef.value)  // 10

// Funcționează cu watch
watch(countRef, (newVal) => {
  console.log('Count changed:', newVal)
})
```

**toRef cu valori default (Vue 3.3+):**

```typescript
interface Props {
  count?: number
  name?: string
}

// toRef poate avea o valoare default
const countRef = toRef(() => props.count ?? 0)
// sau
const countRef = toRef(props, 'count')  // Ref<number | undefined>
```

**toRef cu getter function (Vue 3.3+):**

```typescript
// toRef poate lua un getter function
const doubled = toRef(() => state.count * 2)
// Acesta e readonly și computed-like
console.log(doubled.value)  // 20
```

---

### 7.2 toRefs() - Toate proprietățile

Convertește TOATE proprietățile unui reactive object în Refs individuale:

```typescript
import { reactive, toRefs } from 'vue'

const state = reactive({
  count: 0,
  name: 'Emanuel',
  isActive: true
})

// toRefs creează { count: Ref<number>, name: Ref<string>, isActive: Ref<boolean> }
const refs = toRefs(state)

// Fiecare ref e linked bidirecțional
refs.count.value++
console.log(state.count)  // 1

state.name = 'John'
console.log(refs.name.value)  // 'John'

// DESTRUCTURARE SIGURĂ
const { count, name, isActive } = toRefs(state)
// count, name, isActive sunt Ref-uri reactive!

count.value = 100
console.log(state.count)  // 100
```

---

### 7.3 Pattern principal: Composables

**Acesta e cel mai important use case:**

```typescript
// composable/useCounter.ts
import { reactive, toRefs } from 'vue'

export function useCounter(initialValue = 0) {
  const state = reactive({
    count: initialValue,
    isEven: true,
    history: [] as number[]
  })

  function increment() {
    state.count++
    state.isEven = state.count % 2 === 0
    state.history.push(state.count)
  }

  function decrement() {
    state.count--
    state.isEven = state.count % 2 === 0
    state.history.push(state.count)
  }

  function reset() {
    state.count = initialValue
    state.isEven = initialValue % 2 === 0
    state.history = []
  }

  // RETURN toRefs pentru a permite destructurarea
  return {
    ...toRefs(state),
    increment,
    decrement,
    reset
  }
}

// Component consumer
const { count, isEven, history, increment, decrement, reset } = useCounter(0)
// count, isEven, history sunt Ref-uri reactive
// increment, decrement, reset sunt funcții plain
```

**Alternativă cu ref() direct:**

```typescript
// Unii preferă ref-uri individuale în composables
export function useCounter(initialValue = 0) {
  const count = ref(initialValue)
  const isEven = computed(() => count.value % 2 === 0)
  const history = ref<number[]>([])

  function increment() {
    count.value++
    history.value.push(count.value)
  }

  return { count, isEven, history, increment }
}
```

---

### 7.4 toRefs cu Props

```typescript
// În <script setup>
const props = defineProps<{
  title: string
  count: number
  items: string[]
}>()

// Props sunt reactive dar NU pot fi destructurate direct
// GREȘIT:
// const { title, count } = props  // Pierde reactivitatea

// CORECT:
const { title, count, items } = toRefs(props)
// title, count, items sunt acum Ref<T> reactive

// Sau cu toRef pentru o singură proprietate
const titleRef = toRef(props, 'title')

// Pasare la composable
function useTitle(title: Ref<string>) {
  watch(title, (newTitle) => {
    document.title = newTitle
  }, { immediate: true })
}

useTitle(toRef(props, 'title'))
// SAU
useTitle(title)  // dacă ai folosit toRefs mai sus
```

---

### 7.5 Diferența între toRef, toRefs, și toValue

```typescript
import { ref, reactive, toRef, toRefs, toValue, type MaybeRefOrGetter } from 'vue'

const state = reactive({ count: 0 })

// toRef: proprietate → Ref (linked bidirecțional)
const countRef = toRef(state, 'count')
// typeof countRef = Ref<number>

// toRefs: obiect → { [key]: Ref } (toate linked)
const allRefs = toRefs(state)
// typeof allRefs = { count: Ref<number> }

// toValue (Vue 3.3+): Ref/Getter → valoare plain
const myRef = ref(42)
const myGetter = () => 42

toValue(myRef)       // 42
toValue(myGetter)    // 42
toValue(42)          // 42

// Util în composables care acceptă ref SAU getter SAU valoare
function useDoubled(input: MaybeRefOrGetter<number>) {
  return computed(() => toValue(input) * 2)
}

useDoubled(ref(5))      // computed → 10
useDoubled(() => 5)     // computed → 10
useDoubled(5)           // computed → 10
```

---

### 7.6 Paralelă cu Angular

Angular Signals **nu au un echivalent direct** pentru `toRef`/`toRefs`:

```typescript
// Vue - destructurare sigură cu toRefs
const state = reactive({ count: 0, name: 'test' })
const { count, name } = toRefs(state)

// Angular - nu există necesitate (signals sunt deja "ref-like")
const count = signal(0)
const name = signal('test')
// Nu ai nevoie de toRefs - fiecare signal e deja independent

// Angular - dacă ai un obiect signal
const state = signal({ count: 0, name: 'test' })
// NU poți destructura în signals individuale
// Trebuie signals separate de la început
// SAU computed pentru fiecare proprietate:
const count = computed(() => state().count)
const name = computed(() => state().name)
// Dar acestea sunt READONLY
```

**Key insight:** `toRefs` există în Vue pentru că `reactive()` wrappează tot obiectul. Angular Signals sunt deja individuale, deci nu necesită această conversie. Dar dacă ai un signal cu obiect, Angular nu oferă o modalitate ușoară de a crea writable signals derivate din proprietăți.

---

## 8. shallowRef și shallowReactive

### 8.1 shallowRef

`shallowRef` creează un ref unde doar `.value` assignment e reactive, **nu** proprietățile nested:

```typescript
import { shallowRef, triggerRef } from 'vue'

interface DataRow {
  id: number
  name: string
  metadata: {
    created: Date
    tags: string[]
  }
}

const tableData = shallowRef<DataRow[]>([
  {
    id: 1,
    name: 'Item 1',
    metadata: { created: new Date(), tags: ['a', 'b'] }
  },
  {
    id: 2,
    name: 'Item 2',
    metadata: { created: new Date(), tags: ['c'] }
  }
])

// NU triggerează re-render (mutație nested)
tableData.value[0].name = 'Updated Item 1'     // NU reactive
tableData.value[0].metadata.tags.push('new')    // NU reactive
tableData.value.push({ id: 3, name: 'Item 3', metadata: { created: new Date(), tags: [] } })  // NU reactive

// TRIGGEREAZĂ re-render (reassignment la .value)
tableData.value = [...tableData.value]           // OK - reassignment
tableData.value = tableData.value.map(row => ({
  ...row,
  name: row.name.toUpperCase()
}))                                              // OK - reassignment

// Force trigger (când AI modificat nested dar vrei update)
tableData.value[0].name = 'Forced Update'
triggerRef(tableData)  // Forțează re-render manual
```

---

### 8.2 shallowReactive

`shallowReactive` face reactive doar proprietățile de **top-level**:

```typescript
import { shallowReactive, isReactive } from 'vue'

const state = shallowReactive({
  count: 0,
  user: {
    name: 'Emanuel',
    address: {
      city: 'București'
    }
  },
  items: [1, 2, 3]
})

// Top-level: REACTIVE
state.count++               // Triggerează re-render
state.user = { name: 'New', address: { city: 'Cluj' } }  // Triggerează re-render

// Nested: NU reactive
state.user.name = 'John'         // NU triggerează re-render
state.user.address.city = 'Iași'  // NU triggerează re-render
state.items.push(4)               // NU triggerează re-render

// Verificare
console.log(isReactive(state))       // true
console.log(isReactive(state.user))  // false (nested object NU e reactive)
```

---

### 8.3 Când să folosești shallow variants?

```typescript
// Use case 1: Liste mari (data tables, virtual scrolling)
const hugeList = shallowRef<DataRow[]>([])

async function loadData() {
  const response = await fetch('/api/large-dataset')
  // Un singur assignment → un singur re-render
  hugeList.value = await response.json()
}

function updateRow(id: number, updates: Partial<DataRow>) {
  // Immutable update → triggerea re-render
  hugeList.value = hugeList.value.map(row =>
    row.id === id ? { ...row, ...updates } : row
  )
}

// Use case 2: Third-party object instances
import { Chart } from 'chart.js'

const chartInstance = shallowRef<Chart | null>(null)
// NU vrem Vue să facă deep reactive pe internals-urile Chart.js

onMounted(() => {
  chartInstance.value = new Chart(canvas, config)
})

// Use case 3: Large immutable data (ex: GeoJSON, parsed CSVs)
const geoData = shallowRef<GeoJSON.FeatureCollection | null>(null)
// GeoJSON poate avea mii de features nested
// Deep reactivity ar fi foarte costisitoare

// Use case 4: State management store
const store = shallowReactive({
  // Top-level changes sunt rare (acțiuni de store)
  user: null as User | null,
  products: [] as Product[],
  cart: [] as CartItem[]
})

// Update cu reassignment
function setUser(user: User) {
  store.user = user  // Reactive (top-level)
}

function setProducts(products: Product[]) {
  store.products = products  // Reactive (top-level)
}
```

---

### 8.4 Performance comparison

```
Deep ref/reactive:
┌─────────────────────────────────────────────────────┐
│ ref({ items: [1000 objects with nested properties] }) │
│                                                       │
│ → Proxy pe obiectul principal                         │
│ → Proxy pe fiecare nested object (lazy, dar eventual) │
│ → Track pe fiecare proprietate accesată               │
│ → Memory overhead: HIGH pentru obiecte mari           │
│ → Init time: LOW (lazy) dar eventual HIGH             │
└─────────────────────────────────────────────────────┘

Shallow ref/reactive:
┌─────────────────────────────────────────────────────┐
│ shallowRef([1000 objects with nested properties])     │
│                                                       │
│ → UN SINGUR proxy/ref wrapper pe .value               │
│ → Nested objects rămân plain JS objects               │
│ → Track doar pe .value assignment                     │
│ → Memory overhead: MINIMAL                            │
│ → Init time: MINIMAL                                  │
└─────────────────────────────────────────────────────┘
```

**Guideline:**
- < 100 items cu < 5 niveluri de nesting → `ref()` / `reactive()` (default)
- > 100 items sau obiecte foarte deep → consideră `shallowRef()`
- Third-party instances → întotdeauna `shallowRef()`
- Store state cu updates controlate → `shallowReactive()`

---

### 8.5 Utilități conexe

```typescript
import {
  shallowRef,
  triggerRef,
  isRef,
  isReactive,
  isProxy,
  isReadonly,
  toRaw,
  markRaw
} from 'vue'

// triggerRef - forțează update pe shallowRef
const data = shallowRef({ count: 0 })
data.value.count++
triggerRef(data)  // Forțează re-render

// toRaw - obține obiectul original din proxy
const state = reactive({ count: 0 })
const rawState = toRaw(state)
// rawState NU e reactive, e obiectul plain original

// markRaw - marchează un obiect ca "never reactive"
const thirdPartyLib = markRaw({
  doSomething() { /* ... */ },
  internalState: { /* complex */ }
})

const state2 = reactive({
  lib: thirdPartyLib  // NU devine reactive, rămâne plain
})

console.log(isReactive(state2.lib))  // false

// Type guards
console.log(isRef(ref(0)))          // true
console.log(isReactive(reactive({}))) // true
console.log(isProxy(reactive({})))  // true
console.log(isReadonly(readonly(ref(0)))) // true
```

---

### 8.6 Paralelă cu Angular

Angular Signals sunt **implicit shallow** - nu fac deep tracking:

```typescript
// Angular Signal - e deja "shallow"
const data = signal({ count: 0, nested: { value: 1 } })

// Mutație directă NU triggerează update
data().count++          // NU triggerează
data().nested.value++   // NU triggerează

// Trebuie set() sau update()
data.update(d => ({ ...d, count: d.count + 1 }))  // Triggerează

// Vue shallowRef - comportament similar
const data2 = shallowRef({ count: 0, nested: { value: 1 } })
data2.value.count++             // NU triggerează
data2.value = { ...data2.value, count: data2.value.count + 1 }  // Triggerează
```

**Key insight:** Angular Signals se comportă ca `shallowRef()` by default. Vue `ref()` e deep by default - mai ergonomic dar cu potential overhead. Alegerea shallow în Vue e o **optimizare explicită**, pe când în Angular e **comportamentul standard**.

---

## 9. Comparație completă: Vue Reactivity vs Angular Signals vs RxJS

### 9.1 Tabel principal: Vue 3 vs Angular Signals

| Concept | Vue 3 | Angular Signals (v17+) | Note |
|---------|-------|----------------------|------|
| **Primitive reactivă** | `ref(0)` | `signal(0)` | Aproape identic |
| **Citire valoare (JS)** | `count.value` | `count()` | Vue: property, Angular: function call |
| **Citire valoare (template)** | `{{ count }}` | `{{ count() }}` | Vue auto-unwrap |
| **Setare valoare** | `count.value = 5` | `count.set(5)` | Vue: assignment, Angular: method |
| **Update funcțional** | `count.value++` | `count.update(v => v + 1)` | Vue permite mutație directă |
| **Obiect reactiv deep** | `reactive({})` | Nu există echivalent | Vue: deep, Angular: shallow |
| **Computed readonly** | `computed(() => ...)` | `computed(() => ...)` | Identic! |
| **Writable computed** | `computed({ get, set })` | `linkedSignal()` (parțial) | Vue e mai flexibil |
| **Effect automat** | `watchEffect(() => ...)` | `effect(() => ...)` | Aproape identic |
| **Watch explicit** | `watch(src, cb)` | Nu există direct | Vue-specific |
| **Old/New values** | `watch(src, (new, old) => ...)` | Nu nativ | Trebuie manual în Angular |
| **Cleanup** | `onWatcherCleanup()` | `effect(onCleanup => ...)` | Similar |
| **Readonly** | `readonly(ref)` | `signal.asReadonly()` | Echivalent |
| **Shallow** | `shallowRef()` | Default behavior | Angular e shallow by default |
| **Batch updates** | Automat (microtask) | Automat (notification phase) | Similar |
| **Template integration** | Auto-unwrap | `()` necesar | Vue mai ergonomic |
| **Destructurare** | `toRefs()` | N/A | Vue-specific |
| **Custom ref** | `customRef()` | Nu există | Vue-specific |

---

### 9.2 Tabel: Vue 3 vs RxJS (Angular tradițional)

| Concept | Vue 3 | RxJS (Angular) | Complexitate |
|---------|-------|----------------|-------------|
| **Valoare reactivă** | `ref(0)` | `BehaviorSubject(0)` | Vue: simplu, RxJS: verbose |
| **Citire** | `.value` | `.getValue()` / `\| async` | Vue: proprietate, RxJS: method/pipe |
| **Scriere** | `.value = x` | `.next(x)` | Similar |
| **Derivare** | `computed()` | `combineLatest().pipe(map())` | Vue: mult mai simplu |
| **Side effect** | `watchEffect()` | `.subscribe()` | Similar |
| **Watch changes** | `watch(src, cb)` | `.pipe(pairwise()).subscribe()` | Vue: nativ, RxJS: operator |
| **Multiple sources** | `watch([a, b], cb)` | `combineLatest([a$, b$]).pipe(...)` | Vue: mai simplu |
| **Cleanup** | Automat / `onWatcherCleanup` | `unsubscribe()` / `takeUntilDestroyed()` | Vue: automat, RxJS: manual |
| **Async operation** | `watch + async/await` | `switchMap()` | RxJS: mai puternic |
| **Error handling** | `try/catch` | `catchError()` operator | RxJS: mai composable |
| **Debounce** | `customRef()` / manual | `debounceTime()` | RxJS: nativ |
| **Throttle** | Manual | `throttleTime()` | RxJS: nativ |
| **Cancel prev** | `onCleanup + AbortController` | `switchMap()` | RxJS: mai elegant |
| **Retry** | Manual | `retry()` / `retryWhen()` | RxJS: nativ |
| **Memory mgmt** | Automat (component lifecycle) | Manual (unsubscribe) | Vue: mai sigur |

---

### 9.3 Echivalențe directe: Code comparison

```typescript
// ============================================================
// SCENARIO 1: Counter cu derived values
// ============================================================

// --- Vue 3 ---
const count = ref(0)
const doubled = computed(() => count.value * 2)
const message = computed(() =>
  count.value > 10 ? 'Peste limită' : 'OK'
)

watchEffect(() => {
  console.log(`Count: ${count.value}, Doubled: ${doubled.value}`)
})

count.value++

// --- Angular Signals ---
const count = signal(0)
const doubled = computed(() => count() * 2)
const message = computed(() =>
  count() > 10 ? 'Peste limită' : 'OK'
)

effect(() => {
  console.log(`Count: ${count()}, Doubled: ${doubled()}`)
})

count.update(v => v + 1)

// --- RxJS (Angular) ---
const count$ = new BehaviorSubject(0)
const doubled$ = count$.pipe(map(v => v * 2))
const message$ = count$.pipe(
  map(v => v > 10 ? 'Peste limită' : 'OK')
)

combineLatest([count$, doubled$]).pipe(
  takeUntilDestroyed()
).subscribe(([count, doubled]) => {
  console.log(`Count: ${count}, Doubled: ${doubled}`)
})

count$.next(count$.getValue() + 1)


// ============================================================
// SCENARIO 2: Search cu debounce și cancel
// ============================================================

// --- Vue 3 ---
const query = ref('')
const results = ref([])

watch(query, async (newQuery, _, onCleanup) => {
  const controller = new AbortController()
  onCleanup(() => controller.abort())

  await new Promise(r => setTimeout(r, 300))  // debounce

  const res = await fetch(`/api/search?q=${newQuery}`, {
    signal: controller.signal
  })
  results.value = await res.json()
})

// --- Angular Signals + RxJS hybrid ---
const query = signal('')
// Signals nu au debounce nativ, trebuie RxJS
const query$ = toObservable(query)  // @angular/core/rxjs-interop

const results = signal<any[]>([])
query$.pipe(
  debounceTime(300),
  switchMap(q => fetch(`/api/search?q=${q}`).then(r => r.json())),
  takeUntilDestroyed()
).subscribe(data => results.set(data))

// --- RxJS pur (Angular tradițional) ---
const query$ = new Subject<string>()

const results$ = query$.pipe(
  debounceTime(300),
  distinctUntilChanged(),
  switchMap(q =>
    from(fetch(`/api/search?q=${q}`)).pipe(
      switchMap(res => res.json()),
      catchError(() => of([]))
    )
  )
)


// ============================================================
// SCENARIO 3: Form state management
// ============================================================

// --- Vue 3 ---
const form = reactive({
  name: '',
  email: '',
  age: 0
})

const isValid = computed(() =>
  form.name.length > 0 &&
  form.email.includes('@') &&
  form.age >= 18
)

const isDirty = ref(false)
const originalForm = { ...form }

watch(form, () => {
  isDirty.value = JSON.stringify(form) !== JSON.stringify(originalForm)
}, { deep: true })

// --- Angular Signals ---
const name = signal('')
const email = signal('')
const age = signal(0)

const isValid = computed(() =>
  name().length > 0 &&
  email().includes('@') &&
  age() >= 18
)

// isDirty e mai complex cu signals
const originalValues = { name: '', email: '', age: 0 }
const isDirty = computed(() =>
  name() !== originalValues.name ||
  email() !== originalValues.email ||
  age() !== originalValues.age
)

// --- Angular Reactive Forms (tradițional) ---
const form = new FormGroup({
  name: new FormControl('', Validators.required),
  email: new FormControl('', [Validators.required, Validators.email]),
  age: new FormControl(0, Validators.min(18))
})

const isValid$ = form.statusChanges.pipe(
  map(() => form.valid)
)
const isDirty$ = form.valueChanges.pipe(
  map(() => form.dirty)
)
```

---

### 9.4 Avantaje și dezavantaje

**Vue 3 Reactivity:**
| Avantaje | Dezavantaje |
|----------|-------------|
| Simplu, intuitiv | `.value` poate fi uitat |
| Deep reactive by default | Deep reactivity poate fi costisitor |
| Auto dependency tracking | Debugging mai greu (deps implicite) |
| Funcționează cu mutații directe | Gotchas cu destructurare reactive |
| Template auto-unwrap | |
| Composables pattern curat | |

**Angular Signals:**
| Avantaje | Dezavantaje |
|----------|-------------|
| API minimalist | Nu are deep reactivity |
| Explicit `()` call (no .value confusion) | Trebuie immutable updates cu spread |
| Integrat cu change detection | Nu are watch cu old/new values |
| TypeScript-first | Mai nou, ecosystem în dezvoltare |

**RxJS (Angular tradițional):**
| Avantaje | Dezavantaje |
|----------|-------------|
| Extrem de puternic pentru async | Curba de învățare abruptă |
| Operatori pentru orice scenariu | Verbose pentru cazuri simple |
| Composability excelentă | Memory leaks (forgotten subscriptions) |
| Built-in debounce, retry, etc. | Debugging dificil |
| Stream-based thinking | Overkill pentru state simplu |

---

### 9.5 Migration mental model: Angular → Vue

```
Angular Zone.js          →  Vue ref/reactive (automatic)
Angular OnPush           →  Vue (default - fine-grained)
Angular signal()         →  Vue ref()
Angular computed()       →  Vue computed()
Angular effect()         →  Vue watchEffect()
Angular linkedSignal()   →  Vue computed({ get, set })
BehaviorSubject          →  Vue ref()
combineLatest + map      →  Vue computed()
switchMap                →  Vue watch + async + cleanup
subscribe                →  Vue watchEffect / watch
takeUntilDestroyed       →  Vue (automatic, nu e necesar)
AsyncPipe                →  Vue (nu e necesar, auto-unwrap)
@Input() required        →  Vue defineProps
@Output()                →  Vue defineEmits
```

---

## 10. Diagrame: Cum Vue Tracks Dependencies

### 10.1 Dependency Graph

```
┌─────────────────────────────────────────────────────────┐
│                                                           │
│   ┌──────────┐     track()      ┌───────────────────┐    │
│   │          │ ──────────────>  │                   │    │
│   │  Effect  │                  │   Dependency Set   │    │
│   │ (render, │ <──────────────  │  (Set<Effect>)    │    │
│   │ computed,│     trigger()    │                   │    │
│   │  watch)  │                  │  targetMap:       │    │
│   │          │                  │   WeakMap<        │    │
│   └──────────┘                  │    target,        │    │
│                                  │    Map<key,       │    │
│                                  │     Set<effect>>  │    │
│                                  │   >               │    │
│                                  └───────────────────┘    │
│                                                           │
└─────────────────────────────────────────────────────────┘
```

### 10.2 ref() Internal Structure

```
ref(0):
┌──────────────────────────┐
│    RefImpl               │
│  ┌────────────────────┐  │
│  │ _value: 0          │  │
│  │ dep: Set<Effect>   │  │  ← tracked effects
│  │ __v_isRef: true    │  │  ← marker
│  ├────────────────────┤  │
│  │ get value():       │  │
│  │   track(this)      │  │  ← înregistrează cine citește
│  │   return _value    │  │
│  ├────────────────────┤  │
│  │ set value(newVal): │  │
│  │   if (changed)     │  │
│  │     _value = newVal│  │
│  │     trigger(this)  │  │  ← notifică toți subscribers
│  └────────────────────┘  │
└──────────────────────────┘
```

### 10.3 reactive() Internal Structure

```
reactive({ count: 0, user: { name: 'E' } }):

┌────────────────────────────────────────────────┐
│  Proxy                                          │
│  ┌──────────────────────────────────────────┐   │
│  │  target: { count: 0, user: {...} }       │   │
│  │                                          │   │
│  │  GET handler(target, key):               │   │
│  │    track(target, key)                    │   │
│  │    result = Reflect.get(...)             │   │
│  │    if isObject(result) → reactive(result)│   │  ← lazy deep
│  │    return result                         │   │
│  │                                          │   │
│  │  SET handler(target, key, value):        │   │
│  │    if (changed)                          │   │
│  │      Reflect.set(...)                    │   │
│  │      trigger(target, key)               │   │
│  │    return true                           │   │
│  └──────────────────────────────────────────┘   │
│                                                  │
│  targetMap entry:                                │
│  {                                               │
│    'count' → Set<Effect> [renderEffect, ...]    │
│    'user'  → Set<Effect> [...]                  │
│  }                                               │
└────────────────────────────────────────────────┘
```

### 10.4 Computed Caching Mechanism

```
computed(() => a.value + b.value):

┌──────────────────────────────────────────────┐
│  ComputedRefImpl                              │
│  ┌────────────────────────────────────────┐   │
│  │ _dirty: true     (needs recalculation) │   │
│  │ _value: undefined (cached result)      │   │
│  │ effect: ReactiveEffect                 │   │
│  │ dep: Set<Effect> (subscribers)         │   │
│  ├────────────────────────────────────────┤   │
│  │ get value():                           │   │
│  │   if (_dirty) {                        │   │
│  │     _value = effect.run()  ← calculează│   │
│  │     _dirty = false         ← cachează  │   │
│  │   }                                    │   │
│  │   track(this)                          │   │
│  │   return _value                        │   │
│  └────────────────────────────────────────┘   │
│                                                │
│  Flux:                                         │
│  1. Prima accesare → dirty=true → calculează   │
│  2. A doua accesare → dirty=false → cache      │
│  3. a.value se schimbă → dirty=true            │
│  4. Următoarea accesare → recalculează         │
└──────────────────────────────────────────────┘
```

### 10.5 Full Reactive Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                      Vue 3 Reactive Flow                         │
│                                                                   │
│  ┌────────┐   ┌─────────┐   ┌──────────┐   ┌───────────────┐   │
│  │ State  │   │ Computed │   │ Watcher  │   │  Component    │   │
│  │        │   │          │   │          │   │  Render       │   │
│  │ref(0)  │──>│doubled   │──>│watch()   │   │  Function     │   │
│  │        │   │          │   │          │   │               │   │
│  └────┬───┘   └────┬─────┘   └────┬─────┘   └──────┬────────┘   │
│       │            │              │                 │            │
│       │     track  │       track  │          track  │            │
│       │◄───────────┘◄─────────────┘◄────────────────┘            │
│       │                                                          │
│       │  state.value = 5                                         │
│       │                                                          │
│       ▼                                                          │
│  ┌─────────┐                                                     │
│  │ trigger │                                                     │
│  └────┬────┘                                                     │
│       │                                                          │
│       ├──> Invalidează computed (dirty = true)                   │
│       ├──> Scheduler queue: watcher callback                     │
│       ├──> Scheduler queue: component re-render                  │
│       │                                                          │
│       ▼                                                          │
│  ┌──────────────────┐                                            │
│  │ Microtask Queue  │  (nextTick)                                │
│  │                  │                                            │
│  │ 1. Pre watchers  │                                            │
│  │ 2. Component     │                                            │
│  │    re-render     │                                            │
│  │ 3. Post watchers │                                            │
│  │ 4. DOM update    │                                            │
│  └──────────────────┘                                            │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### 10.6 Watch vs WatchEffect Flow

```
watch(source, callback):
┌──────────────────────────────────────────┐
│                                            │
│  source (explicit)                         │
│     │                                      │
│     ▼                                      │
│  source changed?                           │
│     │                                      │
│     ├─ No → nothing happens               │
│     │                                      │
│     ├─ Yes                                 │
│     │    │                                 │
│     │    ▼                                 │
│     │  callback(newVal, oldVal)            │
│     │                                      │
│  Default: LAZY (nu rulează la creare)      │
│  immediate: true → rulează o dată la start │
│                                            │
└──────────────────────────────────────────┘

watchEffect(callback):
┌──────────────────────────────────────────┐
│                                            │
│  callback() ← rulează IMEDIAT             │
│     │                                      │
│     ├─ accesează ref1.value → track ref1   │
│     ├─ accesează ref2.value → track ref2   │
│     ├─ (condițional) ref3.value → track    │
│     │                                      │
│     ▼                                      │
│  ANY tracked dep changed?                  │
│     │                                      │
│     ├─ Yes → re-run callback()             │
│     │         (cleanup prev first)         │
│     │         (re-track dependencies)      │
│     │                                      │
│  Dependencies: DYNAMIC (per execution)     │
│  No old/new values                         │
│                                            │
└──────────────────────────────────────────┘
```

---

## 11. Întrebări de interviu

### Î1: Cum funcționează reactivitatea în Vue 3 sub capotă?

**Răspuns:**
Vue 3 folosește **ES6 Proxy** pentru a intercepta operațiile GET și SET pe obiecte reactive. Când un effect (render function, computed, watcher) accesează o proprietate reactivă, Vue apelează intern `track()` care înregistrează acel effect ca dependență a proprietății respective într-un `WeakMap<target, Map<key, Set<Effect>>>`. Când proprietatea se modifică, `trigger()` notifică toate effects înregistrate, care sunt adăugate într-o coadă microtask și executate batch. Față de Vue 2 (Object.defineProperty), Proxy oferă interceptare completă: proprietăți noi, ștergeri, acces pe array prin index. Deep reactivity este lazy - obiectele nested devin reactive doar la prima accesare, nu la inițializare. Scheduler-ul Vue batching-uiește multiple modificări într-un singur render cycle, similar cu cum Angular batching-uiește Signal notifications.

---

### Î2: ref() vs reactive() - când folosești fiecare?

**Răspuns:**
`ref()` e recomandat ca **default choice** de documentația oficială Vue. Funcționează cu primitive ȘI obiecte, permite reassignment (`.value = newObject`), e explicit prin `.value` (știi mereu că lucrezi cu ceva reactiv), și e perfect pentru return din composables. `reactive()` e util pentru state local complex al unui component unde vrei access direct fără `.value` și unde nu ai nevoie de reassignment. Gotcha-ul principal cu `reactive()` este pierderea reactivității la destructurare - trebuie `toRefs()`. Ca arhitect, aș standardiza pe `ref()` în echipă pentru consistență, cu excepția cazurilor unde `reactive()` oferă ergonomie semnificativ mai bună (forms complex, state local). Angular developers vor găsi `ref()` mai familiar deoarece e similar cu `signal()` - un wrapper cu getter/setter explicit.

---

### Î3: De ce Vue 3 a trecut de la Object.defineProperty la Proxy?

**Răspuns:**
Trei motive principale: **completitudine**, **performanță**, și **developer experience**. Object.defineProperty nu poate intercepta adăugarea de proprietăți noi (necesita `Vue.set()`), ștergerea de proprietăți, modificarea array-urilor prin index, sau `.length` changes. Proxy interceptează toate aceste operații nativ. Din punct de vedere al performanței, Object.defineProperty trebuia aplicat recursiv la inițializare pe TOATE proprietățile, inclusiv cele nested - Proxy permite lazy deep reactivity (nested objects devin reactive doar la prima accesare). DX-ul s-a îmbunătățit dramatic: nu mai avem nevoie de `Vue.set()`, `Vue.delete()`, sau workaround-uri pentru arrays. Trade-off-ul e că Proxy nu funcționează în IE11, motiv pentru care Vue 3 a abandonat suportul IE11.

---

### Î4: Ce este dependency tracking și cum funcționează?

**Răspuns:**
Dependency tracking e mecanismul prin care Vue știe automat ce effects (render functions, computeds, watchers) depind de ce date reactive. Funcționează prin trei concepte: **activeEffect** (effect-ul curent în execuție), **track()** (apelat la GET pe Proxy), și **trigger()** (apelat la SET). Când un effect rulează, Vue setează `activeEffect` la acel effect. Fiecare GET pe o proprietate reactivă apelează `track()` care adaugă `activeEffect` în Set-ul de dependențe al acelei proprietăți. Structura de stocare e `WeakMap<target, Map<key, Set<Effect>>>` - WeakMap pentru garbage collection automat, Map per proprietate, Set per effect. La modificare, `trigger()` iterează Set-ul și programează re-execuția effects. Acest model e **automat** (spre deosebire de React unde trebuie să listezi dependențele manual în useEffect) și **granular** (fiecare proprietate are propriul set de effects).

---

### Î5: watch() vs watchEffect() - diferențe și use case-uri?

**Răspuns:**
**watch()** e pentru reacții la surse specifice: declari explicit CE urmărești, primești old/new values, și e lazy by default. Folosești watch când ai nevoie de oldValue (logging, undo), când vrei control explicit asupra dependențelor (debugging mai ușor), sau când vrei lazy execution. **watchEffect()** e pentru side effects generali cu auto-dependency tracking: rulează imediat, track-ează automat orice ref/reactive accesat, dar nu oferă old values. Folosești watchEffect pentru data fetching simplu, sincronizare cu external systems, sau effects cu dependențe evidente. Regula mea de arhitect: preferă `watch()` în cod complex (explicit dependencies = debugging mai ușor), folosește `watchEffect()` în cod simplu cu dependențe evidente. watchEffect e echivalentul Angular `effect()`, în timp ce watch nu are un echivalent direct în Angular Signals.

---

### Î6: Cum optimizezi performanța cu shallowRef/shallowReactive?

**Răspuns:**
`shallowRef` și `shallowReactive` elimină overhead-ul deep Proxy wrapping. Le folosesc în trei scenarii: **date mari** (tabele cu mii de rânduri, GeoJSON), **instanțe third-party** (Chart.js, editor instances - nu vrem Vue să intercepteze internal state), și **store state cu updates controlate** (Pinia stores cu acțiuni explicite). Cu shallowRef, doar reassignment-ul la `.value` triggerează update-uri, deci trebuie immutable updates (`[...array]`, `{...obj, prop: newVal}`). `triggerRef()` e un escape hatch pentru a forța un update după mutație directă. Trade-off-ul: câștigi performanță dar pierzi ergonomia mutațiilor directe. Ca regulă: dacă ai < 100 items cu nesting moderat, `ref()` e suficient; pentru dataset-uri mari sau obiecte complexe third-party, `shallowRef()` previne performance issues. Interesant: Angular Signals se comportă ca shallowRef by default.

---

### Î7: Cum gestionezi async operations în watchers?

**Răspuns:**
Provocarea principală cu async în watchers sunt **race conditions** - dacă sursa se schimbă rapid, request-uri vechi pot termina după cele noi. Soluția în Vue e **cleanup function**: `watch(src, async (val, old, onCleanup) => { ... })`. Creez un `AbortController`, înregistrez cleanup-ul cu `onCleanup(() => controller.abort())`, și pasez signal-ul la fetch. Cleanup-ul se apelează automat înainte de re-rularea callback-ului. Pattern-ul complet include: AbortController pentru network cancellation, loading state management, error handling cu verificare AbortError, și opțional debounce manual cu setTimeout în cleanup. Vue 3.5+ introduce `onWatcherCleanup()` ca API separată. Comparativ cu Angular/RxJS: `switchMap` face automat cancel pe previous inner observable, ceea ce e mai elegant. Vue compensează prin simplicitatea async/await vs observable chains.

---

### Î8: Ce probleme poate cauza deep reactivity?

**Răspuns:**
Deep reactivity poate cauza: **performance issues** (obiecte mari cu mii de proprietăți nested generează mulți Proxy wrappers), **circular references** (Vue le gestionează dar pot cauza probleme subtile), **third-party instances corupte** (Proxy-ul poate interfere cu internal state al librăriilor - Chart.js, Monaco Editor, etc.), și **unexpected reactivity** (stocarea unui obiect extern într-un reactive state îl face reactive, ceea ce poate cauza side effects neașteptate). Soluții: `shallowRef()` pentru date mari, `markRaw()` pentru obiecte care nu trebuie niciodată reactive, `toRaw()` pentru a obține obiectul plain din proxy când interacționezi cu API-uri external. Best practice de arhitect: stabilește convenții clare în echipă despre când se folosește deep vs shallow, și documentează explicit când un composable returnează shallow refs.

---

### Î9: Cum testezi cod care folosește reactivitate Vue?

**Răspuns:**
Testarea reactivității Vue implică câteva patterns: pentru **composables**, le testez izolat fără component - creez ref-uri, apelez composable-ul, verific computed values și efectele watch-erilor. `nextTick()` e esențial pentru a aștepta batch updates. Pentru **watchers async**, testez cu fake timers (`vi.useFakeTimers()`) și mock fetch. `flushPromises()` din `@vue/test-utils` rezolvă toate promisiunile pending. Pentru **componente**, `@vue/test-utils` cu `mount()` oferă access la reactive state prin `wrapper.vm`. Pattern de test: setup → act (modifică state) → await nextTick → assert. Important: `watchEffect` rulează sincron la creare deci nu necesită nextTick pentru prima execuție, dar watch cu `immediate: false` necesită modificarea sursei + nextTick. Recomand testarea composables separat de componente - sunt pure functions cu reactive state, ușor de testat unit.

```typescript
// Exemplu test composable
import { nextTick } from 'vue'
import { useCounter } from './useCounter'

describe('useCounter', () => {
  it('should increment and track history', async () => {
    const { count, history, increment } = useCounter(0)

    expect(count.value).toBe(0)
    expect(history.value).toEqual([])

    increment()
    expect(count.value).toBe(1)
    expect(history.value).toEqual([1])

    increment()
    increment()
    expect(count.value).toBe(3)
    expect(history.value).toEqual([1, 2, 3])
  })
})
```

---

### Î10: Compară sistemul reactiv Vue cu Angular Signals.

**Răspuns:**
Vue și Angular Signals au convergit spre un model similar: **fine-grained reactivity cu dependency tracking automat**. Ambele au primitive reactive (ref/signal), derivări cached (computed), și side effects (watchEffect/effect). Diferențele sunt în detalii: Vue oferă **deep reactivity** implicit (Angular signals sunt shallow), **auto-unwrap în template** (Angular necesită `()`), **watch cu old/new values** (Angular nu), și **reactive objects** (Angular nu are echivalent pentru `reactive()`). Angular compensează cu **integrare strânsă cu change detection** (signals bypasează Zone.js), **tip system mai strict** (WritableSignal vs Signal readonly), și **RxJS interop** nativ. Ca arhitect care migrează din Angular, tranziția la Vue reactivity e naturală dacă ai lucrat cu Signals. Cel mai mare adjustment e lipsa operatorilor RxJS - Vue preferă async/await cu cleanup functions, ceea ce e mai simplu dar mai puțin puternic pentru scenarii async complexe (retry, debounce, throttle, merge). Recomand: pentru 90% din cazuri, Vue reactivity e suficientă și mai simplă; pentru streaming/WebSocket/complex async, consideră importul RxJS și în Vue.

---

### Î11: Cum funcționează computed caching și de ce e important?

**Răspuns:**
Computed-urile Vue folosesc un flag intern `_dirty` pentru caching. Când un computed e creat, `_dirty = true`. La prima accesare `.value`, Vue execută getter-ul, stochează rezultatul în `_value`, setează `_dirty = false`. La accesări ulterioare, dacă `_dirty` e false, returnează direct `_value` fără recalculare. Când o dependență a computed-ului se schimbă, Vue setează `_dirty = true` (invalidare), dar NU recalculează imediat - recalcularea e **lazy**, doar la următoarea accesare. Acest model e crucial pentru performanță: dacă ai un computed costisitor (sort pe 10k items), accesat în template de 5 ori, calculul se face o singură dată. Comparativ, o funcție în template s-ar executa de 5 ori la fiecare re-render. Best practice: orice derivare de date care apare în template trebuie să fie computed, nu method. Side effects în computed getter-ul sunt un anti-pattern deoarece ordinea de evaluare nu e garantată.

---

### Î12: Explică pattern-ul composable și rolul reactivității.

**Răspuns:**
Composables sunt **funcții care encapsulează și reutilizează logică stateful** folosind Vue Composition API. Sunt echivalentul Angular services + RxJS patterns, dar mai lightweight. Un composable tipic: creează state intern cu `ref()`/`reactive()`, definește computed derivări, setează watchers pentru side effects, expune state + methods prin return. Reactivitatea e fundamentul: permite composable-ului să fie **self-contained** cu state propriu care se propagă automat la consumeri. Convenții de arhitect: prefixul `use` (useAuth, useFetch), returnează `ref`-uri (nu reactive objects) pentru destructurare sigură, folosește `toRefs()` dacă stateul intern e `reactive()`, documentează ce retururi sunt readonly vs writable. Pattern-ul permite composition: `useAuth()` + `usePermissions()` + `useUserProfile()` pot fi combinate într-un component fără overhead de DI sau module imports complexe ca în Angular.

```typescript
// Composable complet: useFetch
export function useFetch<T>(url: MaybeRefOrGetter<string>) {
  const data = ref<T | null>(null) as Ref<T | null>
  const error = ref<Error | null>(null)
  const isLoading = ref(false)

  async function execute() {
    isLoading.value = true
    error.value = null

    try {
      const response = await fetch(toValue(url))
      if (!response.ok) throw new Error(`HTTP ${response.status}`)
      data.value = await response.json()
    } catch (e) {
      error.value = e as Error
    } finally {
      isLoading.value = false
    }
  }

  // Auto-fetch când URL-ul se schimbă
  watchEffect(() => {
    const currentUrl = toValue(url)
    if (currentUrl) execute()
  })

  return {
    data: readonly(data),
    error: readonly(error),
    isLoading: readonly(isLoading),
    refresh: execute
  }
}

// Utilizare
const { data, error, isLoading, refresh } = useFetch<User[]>('/api/users')
```

---

### Î13: Cum previi memory leaks cu watchers și effects?

**Răspuns:**
Vue gestionează automat cleanup-ul pentru watchers creați **sincron în `setup()`** - se opresc la component unmount. Problemele apar în trei scenarii: **watchers creați async** (setTimeout, after await - nu sunt asociați cu component lifecycle), **event listeners adăugați manual** în watchEffect fără cleanup, și **referințe circulare** între composables. Soluții: pentru watchers async, stochează return value-ul și oprește manual în `onUnmounted()`; folosește întotdeauna `onCleanup` parameter pentru a curăța resources (timers, subscriptions, listeners); în composables complexe, folosește `effectScope()` pentru a grupa și opri toate effects odată. `effectScope()` e echivalentul unui `Subscription` group din RxJS. Best practice de arhitect: fiecare composable trebuie să fie self-cleaning - resursele create intern trebuie curățate intern. Review watchers async și effects cu external subscriptions cu atenție specială.

```typescript
import { effectScope, onScopeDispose } from 'vue'

// effectScope grupează mai multe effects
const scope = effectScope()

scope.run(() => {
  const count = ref(0)

  watchEffect(() => console.log(count.value))
  watch(count, () => { /* ... */ })

  const doubled = computed(() => count.value * 2)
})

// Oprește TOATE effects din scope
scope.stop()

// În composables:
export function useComplexFeature() {
  const scope = effectScope()

  const result = scope.run(() => {
    // Toate effects create aici sunt grouped
    const data = ref(null)
    watchEffect(() => { /* ... */ })
    watch(someSource, () => { /* ... */ })
    return { data }
  })

  onScopeDispose(() => {
    // Cleanup automat când parent scope se distruge
    scope.stop()
  })

  return result!
}
```

---

### Î14: Care sunt diferențele între reactivitatea Vue și React hooks?

**Răspuns:**
Diferențele sunt fundamentale în **model mental** și **implementare**. Vue Composition API e bazat pe **reactivity system** - stateul e reactive by nature, și dependencies sunt tracked automat. React Hooks sunt bazate pe **re-render cycle** - state changes triggerează re-render complet al funcției, și effects trebuie dependențe manuale. Concret: Vue `computed()` cache-uiește automat și nu recalculează dacă deps nu s-au schimbat; React `useMemo()` necesită dependency array manual și poate recalcula inutil la re-renders. Vue `watch()/watchEffect()` track-ează deps automat; React `useEffect()` necesită dependency array (și e o sursă frecventă de bugs). Vue setup() rulează o singură dată; React function component rulează la fiecare render. Vue refs sunt identitate stabilă (nu se recrează); React state necesită closures fresh. Ca arhitect, Vue Composition API e mai predictibil - nu ai stale closures, nu ai dependency array bugs, nu ai hook ordering rules. Trade-off: React-ul e mai explicit despre când se re-render-uiește.

---

### Î15: Cum ai arhitectura state management-ul într-o aplicație Vue mare?

**Răspuns:**
Abordarea pe layere: **component state** (ref/reactive local - formulare, UI state), **composable state** (logică reutilizabilă cu state propriu - useAuth, useFetch), **shared composable state** (singletons cu closures - stare partajată fără store formal), și **Pinia store** (state global, persistent, devtools support). Regula: stateul trebuie să trăiască la cel mai jos nivel posibil. Folosesc Pinia doar pentru state cu adevărat global (user session, theme, notifications) și composables pentru feature state partajat. Pinia folosește intern același sistem reactiv (ref, computed), deci totul e consistent. Comparativ cu Angular: Pinia ≈ NgRx/NGRX dar mult mai simplu (fără actions, reducers, effects separate), composables ≈ Services dar fără DI boilerplate. Recomandarea de arhitect: standardizează pe Pinia pentru global, composables pentru feature-level, ref/reactive pentru local. Evită reactive state în module-level variables fără Pinia - nu ai devtools, nu ai SSR hydration, nu ai hot module replacement support.

```typescript
// Layered state management architecture

// Layer 1: Component local state
const isOpen = ref(false)
const formData = reactive({ name: '', email: '' })

// Layer 2: Composable shared state (singleton pattern)
// composables/useTheme.ts
const theme = ref<'light' | 'dark'>('dark')  // Module-level = singleton

export function useTheme() {
  const isDark = computed(() => theme.value === 'dark')

  function toggle() {
    theme.value = theme.value === 'dark' ? 'light' : 'dark'
  }

  return { theme: readonly(theme), isDark, toggle }
}

// Layer 3: Pinia store (global, devtools, SSR)
// stores/useAuthStore.ts
export const useAuthStore = defineStore('auth', () => {
  const user = ref<User | null>(null)
  const token = ref<string | null>(null)

  const isAuthenticated = computed(() => !!token.value)
  const isAdmin = computed(() => user.value?.role === 'admin')

  async function login(credentials: Credentials) {
    const response = await authApi.login(credentials)
    token.value = response.token
    user.value = response.user
  }

  function logout() {
    token.value = null
    user.value = null
  }

  return { user, isAuthenticated, isAdmin, login, logout }
})
```

---

### Î16: Explică effectScope() și când l-ai folosi.

**Răspuns:**
`effectScope()` creează un **container** pentru reactive effects (watchers, computeds, watchEffects) care pot fi oprite împreună. E echivalentul unui `Subscription` group din RxJS sau al unui `DestroyRef` din Angular. Use case-uri principale: **composables complexe** care creează multiple effects intern (trebuie curățate toate la dispose), **state management** (Pinia folosește intern effectScope pentru fiecare store), **testare** (grupezi effects în scope, oprești la cleanup), și **feature toggles** (activezi/dezactivezi grupuri de effects). `onScopeDispose()` e hook-ul de lifecycle al scope-ului - similar cu `onUnmounted` dar funcționează și în afara componentelor. Nested scopes se opresc automat când parent scope se oprește. Ca arhitect, efectScope e instrumentul care permite composables-urilor să fie cu adevărat autonome și cleanup-able, esențial pentru aplicații mari.

---

### Î17: Care sunt limitările Proxy-based reactivity?

**Răspuns:**
Principalele limitări sunt: **identity comparison** - obiectul original și proxy-ul sunt entități diferite (`target !== reactive(target)`, dar `reactive(target) === reactive(target)` mulțumită caching-ului intern). **ES6 collections** - Map, Set, WeakMap funcționează dar cu overhead suplimentar (Vue override-ește metodele). **Primitive values** - Proxy nu poate wrapa primitive, de aceea avem `ref()` cu `.value`. **Third-party compatibility** - unele librării verifică `instanceof` sau accesează proprietăți interne care pot fi interceptate neașteptat de Proxy. **Performance cu obiecte foarte mari** - deși mai bun ca Vue 2, deep Proxy wrapping pe zeci de mii de proprietăți nested poate încetini. **No IE11** - Proxy nu poate fi polyfilled complet. **Debugging** - Proxy wrappers fac console.log-ul mai greu de citit (trebuie `toRaw()` sau Vue DevTools). Soluții: `shallowRef()` pentru obiecte mari, `markRaw()` pentru third-party instances, `toRaw()` pentru debugging și interop.

---

### Î18: Cum ai migra un component Angular bazat pe RxJS la Vue Composition API?

**Răspuns:**
Procesul de migrare urmează un mapping sistematic. **BehaviorSubject → ref()**: stare cu valoare curentă. **combineLatest + map → computed()**: derivări din multiple surse. **subscribe → watchEffect/watch**: side effects. **switchMap → watch + async + cleanup**: operații async cu cancelation. **pipe operators → composable functions**: encapsulare logică. **takeUntilDestroyed → automatic** (Vue cleanup automat). Pasul critic: identifică ce face fiecare observable chain - 80% din cazuri se simplifică dramatic în Vue. Un `BehaviorSubject.pipe(debounceTime, distinctUntilChanged, switchMap, catchError).subscribe` devine un simplu `watch(src, async (val, old, onCleanup) => { ... })` cu setTimeout și try/catch. Pentru cazurile complexe (WebSocket streams, retry logic, merge multiple async sources), recomand importul RxJS și în Vue - e o librărie standalone, nu Angular-specific. `@vueuse/rxjs` oferă integrare.

```typescript
// Angular RxJS:
private searchQuery$ = new BehaviorSubject('')
private results$ = this.searchQuery$.pipe(
  debounceTime(300),
  distinctUntilChanged(),
  switchMap(q => this.http.get(`/api/search?q=${q}`)),
  catchError(err => of([]))
)

// Vue echivalent:
const searchQuery = ref('')
const results = ref([])
const error = ref(null)

watch(searchQuery, async (query, oldQuery, onCleanup) => {
  if (query === oldQuery) return  // distinctUntilChanged

  const controller = new AbortController()
  onCleanup(() => controller.abort())

  await new Promise(r => setTimeout(r, 300))  // debounceTime

  try {
    const res = await fetch(`/api/search?q=${query}`, {
      signal: controller.signal
    })
    results.value = await res.json()
  } catch (e) {
    if (!(e instanceof DOMException)) {
      results.value = []  // catchError → of([])
    }
  }
})
```

---

### Î19: Explică readonly() și de ce e important pentru architecture.

**Răspuns:**
`readonly()` creează o versiune **deep readonly** a unui reactive object sau ref - orice încercare de mutație generează warning în development și e ignorată. E esențial pentru **encapsulare** în architecture: composables expun state readonly pentru a preveni modificări necontrolate din exterior, stores expun state readonly cu acțiuni explicite pentru mutații. Pattern: stateul intern e `ref()` writable, expus prin `readonly()`, cu funcții dedicate pentru modificări. Acesta e equivalent cu conceptul de `@Output()` emit-only din Angular sau readonly signals. `readonly()` e deep (inclusiv nested properties sunt readonly), și TypeScript enforces-ează la compile time. Diferența față de `as const` sau TypeScript `Readonly<T>`: `readonly()` Vue e runtime enforced (warnings) plus type-safe, nu doar type-level.

```typescript
export function useItems() {
  const items = ref<Item[]>([])
  const selectedId = ref<string | null>(null)

  const selected = computed(() =>
    items.value.find(i => i.id === selectedId.value) ?? null
  )

  function addItem(item: Item) {
    items.value.push(item)
  }

  function selectItem(id: string) {
    selectedId.value = id
  }

  // Expune state READONLY - consumerii nu pot modifica direct
  return {
    items: readonly(items),       // DeepReadonly<Ref<Item[]>>
    selected: readonly(selected), // Readonly computed
    addItem,                      // Singura modalitate de a adăuga
    selectItem                    // Singura modalitate de a selecta
  }
}

// Consumer
const { items, selected, addItem, selectItem } = useItems()

// items.value.push(...)  // TypeScript error + runtime warning
// items.value = [...]    // TypeScript error + runtime warning
addItem({ id: '1', name: 'Test' })  // OK - prin funcția expusă
```

---

### Î20: Cum gestionezi reactivitatea cu TypeScript în Vue 3?

**Răspuns:**
Vue 3 a fost rescris în TypeScript și oferă **type inference excelentă**. `ref<T>()` inferă sau acceptă generic explicit. `reactive()` inferă tipul din argument. `computed()` inferă return type din getter. Gotchas TypeScript: `ref()` returnează `Ref<T>` (nu `T`), deci funcțiile care acceptă ref trebuie typed cu `Ref<T>` sau `MaybeRef<T>`. `reactive()` returnează `UnwrapNestedRefs<T>` (refs nested sunt unwrapped). `toRefs()` returnează `ToRefs<T>` (fiecare proprietate e `Ref`). Pattern avansat: `MaybeRefOrGetter<T>` (Vue 3.3+) pentru composable params care acceptă ref, getter, sau valoare plain. `toValue()` normalizează orice la plain value. `defineProps<T>()` cu generics oferă type-safe props fără runtime overhead. Ca arhitect, recomand strict TypeScript cu `strict: true`, explicite generics pe ref-uri complexe, și `Ref<T | null>` pentru state care poate fi uninitialized.

```typescript
import { ref, computed, type Ref, type MaybeRefOrGetter, toValue } from 'vue'

// Type-safe composable cu generics
function useLocalStorage<T>(
  key: string,
  defaultValue: T
): Ref<T> {
  const stored = localStorage.getItem(key)
  const data = ref<T>(stored ? JSON.parse(stored) : defaultValue) as Ref<T>

  watch(data, (val) => {
    localStorage.setItem(key, JSON.stringify(val))
  }, { deep: true })

  return data
}

// MaybeRefOrGetter pattern
function useDoubled(input: MaybeRefOrGetter<number>) {
  return computed(() => toValue(input) * 2)
}

// Toate variantele funcționează:
const result1 = useDoubled(ref(5))       // Ref<number>
const result2 = useDoubled(() => 5)      // getter
const result3 = useDoubled(5)            // plain value

// defineProps cu TypeScript
const props = defineProps<{
  title: string
  count?: number
  items: Array<{ id: string; name: string }>
  onUpdate?: (value: string) => void
}>()

// defineEmits cu TypeScript
const emit = defineEmits<{
  (e: 'update', value: string): void
  (e: 'delete', id: number): void
}>()

// sau syntax nouă (Vue 3.3+):
const emit = defineEmits<{
  update: [value: string]
  delete: [id: number]
}>()
```

---

## Cheat Sheet Final

```
┌──────────────────────────────────────────────────────────────┐
│                   VUE 3 REACTIVITY CHEAT SHEET                │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  CREARE STATE:                                                 │
│  ref(val)              → Ref<T>       (primitive + objects)    │
│  reactive(obj)         → Proxy<T>     (objects only)           │
│  shallowRef(val)       → ShallowRef   (top-level only)        │
│  shallowReactive(obj)  → ShallowProxy (top-level only)        │
│  readonly(target)      → DeepReadonly  (immutable view)        │
│                                                                │
│  DERIVARE:                                                     │
│  computed(() => ...)           → readonly cached value         │
│  computed({ get, set })        → writable cached value         │
│                                                                │
│  SIDE EFFECTS:                                                 │
│  watch(src, cb, opts?)         → explicit deps, old/new vals  │
│  watchEffect(cb)               → auto deps, eager             │
│  watchPostEffect(cb)           → after DOM update              │
│  watchSyncEffect(cb)           → sync (no batching)            │
│                                                                │
│  CONVERSIE:                                                    │
│  toRef(obj, key)       → Ref linked la proprietate            │
│  toRefs(obj)           → { [key]: Ref } toate proprietățile   │
│  toValue(refOrGetter)  → plain value                          │
│  toRaw(proxy)          → original object                      │
│  markRaw(obj)          → prevent reactive conversion          │
│                                                                │
│  UTILITĂȚI:                                                    │
│  isRef(x)              → boolean                              │
│  isReactive(x)         → boolean                              │
│  isReadonly(x)         → boolean                              │
│  isProxy(x)            → boolean                              │
│  triggerRef(ref)       → force trigger pe shallow ref         │
│  customRef(factory)    → custom get/set logic                 │
│  effectScope()         → group effects                        │
│  nextTick()            → await DOM update                     │
│                                                                │
│  ANGULAR MAPPING:                                              │
│  ref()           ≈  signal()                                  │
│  computed()      ≈  computed()                                │
│  watchEffect()   ≈  effect()                                  │
│  watch()         ≈  (no direct equivalent)                    │
│  reactive()      ≈  (no direct equivalent)                    │
│  readonly()      ≈  signal.asReadonly()                       │
│  shallowRef()    ≈  signal() (default behavior)              │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

> **Notă finală:** Dacă vii din Angular, Vue reactivity va fi mai intuitivă decât crezi.
> ref() e signal(), computed() e computed(), watchEffect() e effect().
> Singura diferență majoră: `.value` în loc de `()` și deep reactivity by default.
> Concentrează-te pe gotchas (destructurare reactive, cleanup async, shallow vs deep)
> și pe pattern-uri (composables, readonly exposure, effectScope).


---

**Următor :** [**03 - Dependency Injection (Provide/Inject)** →](Vue/03-Dependency-Injection-Provide-Inject.md)