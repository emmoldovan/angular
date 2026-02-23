# Vue 3 Core - Composition API (Interview Prep - Senior Frontend Architect)

> Composition API în Vue 3.5+, de la setup() la `<script setup>`,
> ref/reactive, lifecycle hooks, template syntax, componente.
> Include paralele detaliate cu Angular la fiecare concept.
> Pregătit pentru Emanuel Moldovan - tranziție Angular → Vue.

---

## Cuprins

1. [Composition API Overview](#1-composition-api-overview)
2. [setup() și `<script setup>`](#2-setup-și-script-setup)
3. [ref() vs reactive()](#3-ref-vs-reactive)
4. [computed()](#4-computed)
5. [watch() și watchEffect()](#5-watch-și-watcheffect)
6. [Lifecycle Hooks](#6-lifecycle-hooks)
7. [Template Syntax](#7-template-syntax)
8. [Components - Props, Emits, Slots](#8-components---props-emits-slots)
9. [defineProps/defineEmits cu TypeScript](#9-definepropsdefineemits-cu-typescript---advanced)
10. [Tabel Complet: Angular → Vue Mapping](#10-tabel-complet-angular--vue-mapping)
11. [Întrebări de interviu](#11-întrebări-de-interviu)

---

## 1. Composition API Overview

### Ce este Composition API?

**Composition API** este un set de funcții introduse în Vue 3 care permit organizarea logicii componentelor după **funcționalitate** (feature), nu după **tip** (data, methods, computed, etc.). A apărut ca răspuns la limitările **Options API** în aplicații mari.

### Problemele Options API în proiecte mari

Options API (stilul clasic Vue 2) organizează codul după **tipul opțiunii**:

```vue
<script>
export default {
  // Toate datele într-un loc
  data() {
    return {
      searchQuery: '',
      searchResults: [],
      filterCategory: 'all',
      filterResults: [],
      paginationPage: 1,
      paginationTotal: 0
    }
  },
  // Toate computed într-un loc (amestec de features)
  computed: {
    filteredResults() { /* ... */ },
    paginatedResults() { /* ... */ },
    searchSummary() { /* ... */ }
  },
  // Toate metodele într-un loc
  methods: {
    search() { /* ... */ },
    applyFilter() { /* ... */ },
    goToPage() { /* ... */ }
  },
  // Toate watchers într-un loc
  watch: {
    searchQuery() { /* ... */ },
    filterCategory() { /* ... */ }
  }
}
</script>
```

**Problema:** Logica pentru "search", "filter" și "pagination" este **fragmentată** în 4 secțiuni diferite. Într-o componentă cu 500+ linii, trebuie să sari constant între secțiuni pentru a înțelege o singură funcționalitate.

### Cum rezolvă Composition API

Cu Composition API, organizezi codul **pe funcționalitate**:

```vue
<script setup lang="ts">
// ======= SEARCH FEATURE =======
const searchQuery = ref('')
const searchResults = ref<Item[]>([])
const searchSummary = computed(() => `${searchResults.value.length} rezultate`)

async function search() {
  searchResults.value = await api.search(searchQuery.value)
}

watch(searchQuery, debounce(search, 300))

// ======= FILTER FEATURE =======
const filterCategory = ref('all')
const filteredResults = computed(() =>
  searchResults.value.filter(r =>
    filterCategory.value === 'all' || r.category === filterCategory.value
  )
)

function applyFilter(category: string) {
  filterCategory.value = category
}

// ======= PAGINATION FEATURE =======
const paginationPage = ref(1)
const paginationTotal = computed(() => Math.ceil(filteredResults.value.length / 10))
const paginatedResults = computed(() => {
  const start = (paginationPage.value - 1) * 10
  return filteredResults.value.slice(start, start + 10)
})
</script>
```

### Și mai bine: extragerea în Composables

Fiecare bloc de logică poate fi extras într-un **composable** (funcție reutilizabilă):

```typescript
// composables/useSearch.ts
export function useSearch() {
  const query = ref('')
  const results = ref<Item[]>([])
  const summary = computed(() => `${results.value.length} rezultate`)

  async function search() {
    results.value = await api.search(query.value)
  }

  watch(query, debounce(search, 300))

  return { query, results, summary, search }
}
```

```vue
<script setup lang="ts">
import { useSearch } from '@/composables/useSearch'
import { useFilter } from '@/composables/useFilter'
import { usePagination } from '@/composables/usePagination'

const { query, results, summary } = useSearch()
const { category, filtered, applyFilter } = useFilter(results)
const { page, total, paginated } = usePagination(filtered)
</script>
```

### Options API vs Composition API

| Aspect | Options API | Composition API |
|--------|-------------|-----------------|
| **Organizare cod** | Pe tip (data, methods, computed) | Pe funcționalitate (feature) |
| **Reutilizare logică** | Mixins (problematice) | Composables (funcții pure) |
| **TypeScript** | Suport limitat, tipuri implicite | Suport excelent, inferență completă |
| **Code splitting** | Dificil - totul e în obiectul options | Natural - funcții independente |
| **Învățare** | Mai ușor pentru începători | Necesită înțelegere reactivitate |
| **Boilerplate** | Mediu (this, return) | Minim cu `<script setup>` |
| **Tree-shaking** | Limitat | Excelent - import explicit |
| **Scalabilitate** | Problematică la 500+ linii | Scalează bine cu composables |

### Important: Composition API NU înlocuiește Options API

Ambele stiluri sunt suportate și vor rămâne suportate. **Composition API** este recomandat pentru:
- Proiecte noi
- Aplicații mari și complexe
- Echipe care folosesc TypeScript
- Logică care trebuie reutilizată între componente

**Options API** rămâne valid pentru:
- Componente simple
- Prototipuri rapide
- Echipe care vin din Vue 2

### Paralela cu Angular

Angular nu are această dualitate Options API / Composition API. În Angular, **totul este class-based** cu decoratori:

```typescript
// Angular - mereu class-based
@Component({ /* ... */ })
export class SearchComponent {
  query = signal('');
  results = signal<Item[]>([]);

  search() { /* ... */ }
}
```

Conceptual însă, evoluția Angular e similară:
- **NgModules → Standalone Components** = reducerea boilerplate-ului (similar cu trecerea la `<script setup>`)
- **RxJS → Signals** = simplificarea reactivității (similar cu trecerea la ref/computed)
- **Services cu DI → Standalone functions** = Vue Composables sunt conceptual similar cu funcțiile standalone care folosesc `inject()`

**Diferența cheie:** În Angular, logica reutilizabilă stă în **Services** (clase injectabile). În Vue, stă în **Composables** (funcții pure care returnează stare reactivă).

---

## 2. setup() și `<script setup>`

### setup() Function - Entry Point

Funcția `setup()` este **punctul de intrare** pentru Composition API într-o componentă Vue. Se execută **înainte** ca componenta să fie creată (înainte de `beforeCreate`).

```vue
<script>
import { ref, computed, onMounted } from 'vue'

export default {
  props: {
    initialCount: {
      type: Number,
      default: 0
    }
  },
  emits: ['count-changed'],
  setup(props, context) {
    // props este reactiv (dar NU destructura direct!)
    // context contine: attrs, slots, emit, expose

    const count = ref(props.initialCount)
    const doubled = computed(() => count.value * 2)

    function increment() {
      count.value++
      context.emit('count-changed', count.value)
    }

    onMounted(() => {
      console.log('Componenta montată, count:', count.value)
    })

    // OBLIGATORIU: returnezi tot ce vrei sa expui la template
    return {
      count,
      doubled,
      increment
    }
  }
}
</script>

<template>
  <button @click="increment">
    Count: {{ count }} (doubled: {{ doubled }})
  </button>
</template>
```

**Parametrii setup():**
- `props` - obiect reactiv cu props-urile componentei. **Nu destructura direct** - pierzi reactivitatea
- `context` - obiect non-reactiv cu: `attrs`, `slots`, `emit`, `expose`

### `<script setup>` - Syntax Sugar (PREFERAT)

`<script setup>` este un **compiler macro** care simplifică dramatic setup-ul:

```vue
<script setup lang="ts">
import { ref, computed, onMounted } from 'vue'

// Props și Emits - compiler macros (nu se importă!)
const props = defineProps<{
  initialCount?: number
}>()

const emit = defineEmits<{
  'count-changed': [value: number]
}>()

// Tot ce declari la top-level este automat expus la template
const count = ref(props.initialCount ?? 0)
const doubled = computed(() => count.value * 2)

function increment() {
  count.value++
  emit('count-changed', count.value)
}

onMounted(() => {
  console.log('Componenta montată, count:', count.value)
})
</script>

<template>
  <button @click="increment">
    Count: {{ count }} (doubled: {{ doubled }})
  </button>
</template>
```

### Avantajele `<script setup>` față de setup() clasic

| Aspect | setup() clasic | `<script setup>` |
|--------|---------------|-------------------|
| **Return statement** | Obligatoriu - totul trebuie returnat | Nu e necesar - top-level bindings sunt automat expuse |
| **Props/Emits** | Declarate în options object | `defineProps` / `defineEmits` compiler macros |
| **Boilerplate** | `export default { setup() { ... return {...} } }` | Direct cod la top-level |
| **Performance** | Standard | Mai bun - compilatorul optimizează template bindings |
| **TypeScript** | Funcționează, dar mai verbose | Inferență completă, mai puțin cod |
| **IDE Support** | Bun | Excelent cu Volar |

### Compiler Macros - nu se importă!

Aceste funcții sunt **compiler macros** - sunt procesate la build time, nu la runtime:

```vue
<script setup lang="ts">
// NU importa aceste funcții - sunt globale în <script setup>
// import { defineProps, defineEmits } from 'vue'  // GREȘIT!

const props = defineProps<{ title: string }>()
const emit = defineEmits<{ click: [] }>()

// defineExpose - controlează ce vede părintele prin template ref
defineExpose({
  publicMethod: () => console.log('accesibil din părinte')
})

// defineOptions - setează opțiuni ale componentei
defineOptions({
  inheritAttrs: false,
  name: 'MyCustomComponent'
})

// defineSlots - typed slots (Vue 3.3+)
const slots = defineSlots<{
  default: (props: { message: string }) => any
  header: () => any
}>()

// defineModel - simplified v-model (Vue 3.4+)
const modelValue = defineModel<string>({ required: true })
</script>
```

### Comparație completă: Options API → setup() → `<script setup>`

**Options API (vechi):**

```vue
<script>
export default {
  data() {
    return {
      count: 0
    }
  },
  computed: {
    doubled() {
      return this.count * 2
    }
  },
  methods: {
    increment() {
      this.count++
    }
  },
  mounted() {
    console.log('mounted')
  }
}
</script>

<template>
  <button @click="increment">{{ count }} ({{ doubled }})</button>
</template>
```

**Composition API cu setup():**

```vue
<script>
import { ref, computed, onMounted } from 'vue'

export default {
  setup() {
    const count = ref(0)
    const doubled = computed(() => count.value * 2)

    function increment() {
      count.value++
    }

    onMounted(() => {
      console.log('mounted')
    })

    return { count, doubled, increment }
  }
}
</script>

<template>
  <button @click="increment">{{ count }} ({{ doubled }})</button>
</template>
```

**`<script setup>` - PREFERAT:**

```vue
<script setup lang="ts">
import { ref, computed, onMounted } from 'vue'

const count = ref(0)
const doubled = computed(() => count.value * 2)

function increment() {
  count.value++
}

onMounted(() => {
  console.log('mounted')
})
</script>

<template>
  <button @click="increment">{{ count }} ({{ doubled }})</button>
</template>
```

### Paralela cu Angular

| Vue | Angular | Notă |
|-----|---------|------|
| `setup()` | `constructor` + `ngOnInit` | setup() rulează înainte de creare; constructor + ngOnInit rulează la inițializare |
| `<script setup>` top-level | Corpul clasei componentei | Ambele declară stare și metode direct |
| `defineProps()` | `input()` / `@Input()` | Compiler macro vs decorator |
| `defineEmits()` | `output()` / `@Output()` | Compiler macro vs decorator |
| `defineExpose()` | Public methods pe clasă (accesate via `viewChild()`) | Vue ascunde totul implicit |
| Eliminarea `return {}` | Nu are echivalent | Angular expune automat tot ce e public pe clasă |
| `defineOptions({ name })` | `@Component({ selector })` | Metadata componentă |

**Observație importantă:** `<script setup>` a eliminat boilerplate-ul similar cu cum standalone components au eliminat NgModules în Angular. Ambele framework-uri se îndreaptă spre **mai puțin boilerplate, mai multă convenție**.

---

## 3. ref() vs reactive()

### ref() - Wrapper reactiv universal

`ref()` creează o **referință reactivă** care funcționează cu **orice tip de valoare**: primitive, obiecte, array-uri.

```typescript
import { ref } from 'vue'

// Primitive
const count = ref(0)              // Ref<number>
const name = ref('Emanuel')       // Ref<string>
const isActive = ref(true)        // Ref<boolean>
const nothing = ref(null)         // Ref<null>

// Obiecte și array-uri
const user = ref({
  name: 'Emanuel',
  age: 30,
  skills: ['Angular', 'TypeScript']
})                                 // Ref<{ name: string; age: number; skills: string[] }>

const items = ref<string[]>([])   // Ref<string[]>

// Acces cu .value în <script>
count.value++                      // 1
name.value = 'John'                // OK
user.value.name = 'John'           // OK - nested access
user.value = { name: 'Jane', age: 25, skills: [] }  // OK - reasignare completă

// În <template> - automat unwrapped (fără .value)
// {{ count }} nu {{ count.value }}
```

### reactive() - Proxy pentru obiecte

`reactive()` creează un **proxy reactiv** direct pe obiect. **Nu funcționează cu primitive.**

```typescript
import { reactive } from 'vue'

// Obiecte
const state = reactive({
  count: 0,
  name: 'Emanuel',
  nested: {
    deep: true
  }
})

state.count++                     // Direct, fără .value
state.name = 'John'               // Direct
state.nested.deep = false          // Reactivitate profundă (deep)

// Array-uri
const items = reactive<string[]>([])
items.push('item1')               // OK

// NU funcționează cu primitive!
// const count = reactive(0)      // EROARE TypeScript
// const name = reactive('Emanuel') // EROARE TypeScript
```

### Gotchas cu reactive() - DE REȚINUT

#### 1. Nu poți reasigna întregul obiect

```typescript
const state = reactive({ count: 0, name: 'Emanuel' })

// GREȘIT - pierzi reactivitatea!
// state = reactive({ count: 1, name: 'John' })
// Nu poți face asta. Variabila `state` pointează la proxy-ul original.

// CORECT - modifici proprietățile individual
state.count = 1
state.name = 'John'

// Sau folosești Object.assign
Object.assign(state, { count: 1, name: 'John' })
```

#### 2. Destructurarea pierde reactivitatea

```typescript
const state = reactive({ count: 0, name: 'Emanuel' })

// GREȘIT - variabilele nu mai sunt reactive!
const { count, name } = state
// count este acum un simplu number (0), nu mai e reactiv
// Modificarea state.count NU va actualiza variabila count

// CORECT - folosește toRefs()
import { toRefs } from 'vue'
const { count, name } = toRefs(state)
// count este acum Ref<number> - reactiv!
count.value++  // actualizează și state.count

// Sau toRef() pentru o singură proprietate
import { toRef } from 'vue'
const countRef = toRef(state, 'count')
```

#### 3. Pierdere reactivitate la pasarea ca argument

```typescript
const state = reactive({ count: 0 })

// GREȘIT - pasezi valoarea, nu referința
function logCount(count: number) {
  console.log(count)  // va fi mereu 0
}
logCount(state.count)

// CORECT cu ref()
const count = ref(0)
function logCount(count: Ref<number>) {
  console.log(count.value)  // va fi valoarea curentă
}
logCount(count)
```

### ref() vs reactive() - Tabel comparativ

| Aspect | ref() | reactive() |
|--------|-------|------------|
| **Tipuri suportate** | Primitive + Obiecte + Array-uri | Doar Obiecte și Array-uri |
| **Acces valoare** | `.value` în script, automat în template | Direct (fără .value) |
| **Reasignare** | `myRef.value = newValue` - OK | Nu poți reasigna obiectul |
| **Destructurare** | N/A (e deja o referință) | Pierde reactivitatea (necesită `toRefs()`) |
| **TypeScript** | `Ref<T>` - inferență excelentă | Tipul obiectului direct |
| **Nested reactivity** | Deep reactive pentru obiecte | Deep reactive |
| **Pasare ca argument** | Referința se păstrează | Valoarea se copiază la destructurare |
| **Recomandare oficială** | **DA - folosește ref() pentru tot** | Doar când vrei să eviți .value |
| **Performanță** | Minim overhead (wrapper) | Ușor mai bun (fără wrapper) |
| **Consistency** | Mereu .value - predictibil | Uneori cu, uneori fără .value |

### De ce ref() este recomandat

**Recomandarea oficială Vue:** Folosește `ref()` pentru tot. Motivele:

1. **Consistență** - mereu `.value`, fără surprize
2. **Funcționează cu orice tip** - primitive și obiecte
3. **Reasignare** - poți înlocui complet valoarea
4. **Destructurare safe** - nu pierzi reactivitatea accidental
5. **TypeScript** - `Ref<T>` e clar și explicit
6. **Composables** - returnarea ref-urilor din composables e pattern-ul standard

```typescript
// Pattern recomandat - ref() pentru tot
const count = ref(0)
const user = ref<User | null>(null)
const items = ref<Item[]>([])
const isLoading = ref(false)

// NU acest pattern cu reactive()
const state = reactive({
  count: 0,
  user: null as User | null,
  items: [] as Item[],
  isLoading: false
})
```

### shallowRef() și shallowReactive()

Pentru obiecte mari unde nu ai nevoie de **deep reactivity**:

```typescript
import { shallowRef, triggerRef } from 'vue'

// shallowRef - doar .value assignment trigger-uiește reactivitate
const heavyObject = shallowRef({ nested: { deep: { value: 1 } } })

// NU trigger-uiește update-ul UI
heavyObject.value.nested.deep.value = 2

// Trigger-uiește update-ul UI
heavyObject.value = { nested: { deep: { value: 2 } } }

// Sau forțează trigger manual
heavyObject.value.nested.deep.value = 2
triggerRef(heavyObject)
```

### Paralela cu Angular

| Vue | Angular Signals | Notă |
|-----|----------------|------|
| `ref(0)` | `signal(0)` | Ambele creează referință reactivă |
| `ref.value` | `signal()` (getter call) | Vue: `.value`, Angular: `()` |
| `ref.value = x` | `signal.set(x)` | Vue: assignment, Angular: method call |
| `ref.value++` | `signal.update(v => v + 1)` | Vue: direct mutation, Angular: update callback |
| `reactive({...})` | Nu are echivalent direct | Angular Signals sunt mereu "ref-like" |
| `toRefs()` | Nu e necesar | Angular nu are problema destructurării |
| `shallowRef()` | `signal()` (e shallow by default) | Angular signals sunt shallow implicit |

**Diferența fundamentală:** Angular Signals nu au dualitatea ref/reactive. `signal()` funcționează cu orice tip de valoare și este mereu accesat cu `()`. Vue are două abordări, dar recomandarea oficială (`ref()` pentru tot) face experiența similară cu Angular Signals.

```typescript
// Vue
const count = ref(0)
console.log(count.value)    // citire
count.value = 5             // scriere

// Angular
const count = signal(0)
console.log(count())        // citire
count.set(5)                // scriere
```

---

## 4. computed()

### Valori derivate cu cache automat

`computed()` creează o **valoare reactivă derivată** care se recalculează automat când dependențele se schimbă. Rezultatul este **cached** - se recalculează doar când dependențele se modifică.

```typescript
import { ref, computed } from 'vue'

const firstName = ref('Emanuel')
const lastName = ref('Moldovan')
const items = ref<Product[]>([])

// Computed readonly - cel mai comun
const fullName = computed(() => `${firstName.value} ${lastName.value}`)
// fullName.value === 'Emanuel Moldovan'
// fullName.value = 'Test'  // EROARE - readonly!

// Computed cu dependențe complexe
const expensiveItems = computed(() =>
  items.value
    .filter(item => item.price > 100)
    .sort((a, b) => b.price - a.price)
)

// Computed bazat pe alt computed (chaining)
const totalExpensive = computed(() =>
  expensiveItems.value.reduce((sum, item) => sum + item.price, 0)
)
```

### Computed Writable (get/set)

Util când vrei ca o valoare derivată să poată fi și setată:

```typescript
const firstName = ref('Emanuel')
const lastName = ref('Moldovan')

// Writable computed
const fullName = computed({
  get: () => `${firstName.value} ${lastName.value}`,
  set: (newValue: string) => {
    const [first, ...rest] = newValue.split(' ')
    firstName.value = first
    lastName.value = rest.join(' ')
  }
})

// Citire
console.log(fullName.value)  // 'Emanuel Moldovan'

// Scriere - declanșează setter-ul
fullName.value = 'John Doe'
console.log(firstName.value)  // 'John'
console.log(lastName.value)   // 'Doe'
```

### Computed vs Methods

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

const items = ref<number[]>([1, 2, 3, 4, 5])

// COMPUTED - cached, se recalculează DOAR când items se schimbă
const total = computed(() => {
  console.log('computed evaluat')  // se apelează o singură dată
  return items.value.reduce((a, b) => a + b, 0)
})

// METHOD - se execută la FIECARE renderare
function getTotal(): number {
  console.log('method apelat')  // se apelează la fiecare render
  return items.value.reduce((a, b) => a + b, 0)
}
</script>

<template>
  <!-- Computed - cached -->
  <p>{{ total }}</p>
  <p>{{ total }}</p>
  <!-- "computed evaluat" apare O SINGURĂ DATĂ -->

  <!-- Method - recalculat la fiecare utilizare -->
  <p>{{ getTotal() }}</p>
  <p>{{ getTotal() }}</p>
  <!-- "method apelat" apare DE DOUĂ ORI -->
</template>
```

### Best Practices computed()

```typescript
// 1. Folosește computed() pentru orice derivare de stare
const isFormValid = computed(() =>
  name.value.length > 0 && email.value.includes('@')
)

// 2. NU face side effects în computed
// GREȘIT:
const bad = computed(() => {
  apiService.log('computed evaluat')  // side effect!
  return items.value.length
})

// CORECT - folosește watch() sau watchEffect() pentru side effects
const good = computed(() => items.value.length)
watchEffect(() => {
  apiService.log(`Avem ${good.value} item-uri`)
})

// 3. Evită computed-uri prea costisitoare fără necesitate
// Dacă valoarea se schimbă rar, computed e perfect (cached)
// Dacă valoarea trebuie recalculată la fiecare frame, folosește method

// 4. Poți folosi computed cu generics
function useFiltered<T>(items: Ref<T[]>, predicate: (item: T) => boolean) {
  return computed(() => items.value.filter(predicate))
}
```

### Paralela cu Angular

| Vue | Angular | Notă |
|-----|---------|------|
| `computed(() => ...)` | `computed(() => ...)` | **Identic conceptual!** |
| Cache automat | Cache automat | Ambele recalculează doar când dependențele se schimbă |
| Dependency tracking automat | Dependency tracking automat | Ambele urmăresc automat ce ref/signal citești |
| Writable computed (get/set) | Nu are echivalent direct (dar poți cu signal + effect) | Vue e mai flexibil aici |
| `computed()` returnează `ComputedRef` | `computed()` returnează `Signal` | Acces: `.value` vs `()` |

```typescript
// Vue
const firstName = ref('Emanuel')
const fullName = computed(() => `${firstName.value} Moldovan`)
console.log(fullName.value)  // 'Emanuel Moldovan'

// Angular
const firstName = signal('Emanuel')
const fullName = computed(() => `${firstName()} Moldovan`)
console.log(fullName())  // 'Emanuel Moldovan'
```

**Observație:** Computed în ambele framework-uri folosesc **lazy evaluation** - se calculează prima dată când sunt citite, nu la declarare. Și ambele sunt **cached** - se recalculează doar când dependențele se schimbă.

---

## 5. watch() și watchEffect()

### watch() - Urmărire explicită

`watch()` urmărește **surse specifice** și execută un callback când acestea se schimbă:

```typescript
import { ref, watch } from 'vue'

const count = ref(0)
const name = ref('Emanuel')

// Watch o singură sursă
watch(count, (newValue, oldValue) => {
  console.log(`count: ${oldValue} → ${newValue}`)
})

// Watch cu opțiuni
watch(count, (newValue, oldValue) => {
  console.log(`count: ${oldValue} → ${newValue}`)
}, {
  immediate: true,   // execută imediat la inițializare
  once: true,        // execută o singură dată (Vue 3.4+)
  flush: 'post'      // execută DUPĂ actualizarea DOM-ului
})

// Watch multiple surse
watch([count, name], ([newCount, newName], [oldCount, oldName]) => {
  console.log(`count: ${oldCount} → ${newCount}`)
  console.log(`name: ${oldName} → ${newName}`)
})

// Watch un getter (expresie)
watch(
  () => count.value * 2,
  (doubled) => {
    console.log(`doubled: ${doubled}`)
  }
)
```

### Watch pe obiecte

```typescript
const user = ref({ name: 'Emanuel', age: 30 })
const state = reactive({ count: 0, nested: { deep: true } })

// Watch ref cu obiect - deep by default de la Vue 3.5
watch(user, (newUser, oldUser) => {
  console.log('user changed:', newUser)
})

// Watch proprietate specifică a unui reactive
watch(
  () => state.count,
  (newCount) => {
    console.log('count:', newCount)
  }
)

// Watch deep explicit
watch(
  () => state.nested,
  (newNested) => {
    console.log('nested changed:', newNested)
  },
  { deep: true }
)
```

### watchEffect() - Urmărire automată

`watchEffect()` **detectează automat** dependențele și re-execută callback-ul când acestea se schimbă:

```typescript
import { ref, watchEffect } from 'vue'

const count = ref(0)
const name = ref('Emanuel')

// Detectează automat că depinde de count.value și name.value
watchEffect(() => {
  console.log(`count: ${count.value}, name: ${name.value}`)
})
// Se execută IMEDIAT la declarare (spre deosebire de watch)

// Cu cleanup
watchEffect((onCleanup) => {
  const controller = new AbortController()

  fetch(`/api/user/${name.value}`, { signal: controller.signal })
    .then(res => res.json())
    .then(data => console.log(data))

  onCleanup(() => {
    controller.abort()  // anulează request-ul anterior
  })
})
```

### onWatcherCleanup() (Vue 3.5+)

```typescript
import { ref, watch, onWatcherCleanup } from 'vue'

const searchQuery = ref('')

watch(searchQuery, (newQuery) => {
  const controller = new AbortController()

  fetch(`/api/search?q=${newQuery}`, { signal: controller.signal })
    .then(res => res.json())
    .then(data => { /* procesare */ })

  // Cleanup - se apelează când watcher-ul re-execută sau componenta e distrusă
  onWatcherCleanup(() => {
    controller.abort()
  })
})
```

### watch() vs watchEffect()

| Aspect | watch() | watchEffect() |
|--------|---------|---------------|
| **Surse** | Explicite (le declari) | Automat detectate |
| **Execuție inițială** | Nu (decât cu `immediate: true`) | Da - mereu |
| **Acces old value** | Da - `(newVal, oldVal)` | Nu |
| **Lazy** | Da | Nu - rulează imediat |
| **Când folosești** | Reacție la schimbări specifice | Side effects cu dependențe multiple |
| **Cleanup** | În callback sau `onWatcherCleanup` | `onCleanup` parameter |
| **Control** | Maxim - alegi exact ce urmărești | Automat - poate urmări prea mult |

### Oprirea unui watcher

```typescript
// watch() și watchEffect() returnează o funcție de stop
const stopWatch = watch(count, (newVal) => {
  console.log(newVal)
})

const stopEffect = watchEffect(() => {
  console.log(count.value)
})

// Oprire manuală
stopWatch()
stopEffect()

// Watcher-ele declarate în setup() sunt oprite AUTOMAT
// la unmount-ul componentei
```

### Paralela cu Angular

| Vue | Angular | Notă |
|-----|---------|------|
| `watch(source, callback)` | `effect(() => { source(); callback() })` | Angular effect se comportă ca watchEffect |
| `watchEffect(() => ...)` | `effect(() => ...)` | Ambele au auto-tracking |
| `watch(source, cb, { immediate: true })` | `effect()` (rulează mereu imediat) | Angular effect e mereu "immediate" |
| `onCleanup` / `onWatcherCleanup` | `effect` cu `onCleanup` | Ambele suportă cleanup |
| Oprire manuală | `effect` returnează `EffectRef` cu `.destroy()` | Similar |
| `watch(source, cb)` pe props | `effect()` care citește `input()()` | Ambele detectează schimbări |

```typescript
// Vue - watch explicit
watch(count, (newVal, oldVal) => {
  console.log(`${oldVal} → ${newVal}`)
})

// Angular - nu ai acces direct la oldVal în effect()
effect(() => {
  console.log(`count is now: ${count()}`)
})

// Angular - pentru oldVal ai nevoie de pattern explicit
effect(() => {
  const current = count()
  // trebuie să stochezi manual previous value
})
```

**Diferență cheie:** Vue oferă `watch()` cu acces la `oldValue` out-of-the-box. Angular `effect()` nu are acest concept - trebuie să gestionezi manual valoarea anterioară.

---

## 6. Lifecycle Hooks

### Diagrama lifecycle-ului

```
Creare componentă
    │
    ├── setup() / <script setup> ← AICI scrii cod Composition API
    │
    ├── onBeforeMount()
    │       │
    │       ▼
    ├── onMounted()           ← DOM disponibil
    │       │
    │   ┌───┴───┐
    │   │ UPDATE │ (când se schimbă starea reactivă)
    │   │       │
    │   ├── onBeforeUpdate()
    │   │       │
    │   ├── onUpdated()       ← DOM actualizat
    │   │       │
    │   └───┬───┘
    │       │
    ├── onBeforeUnmount()
    │       │
    ▼       ▼
    onUnmounted()             ← Cleanup
```

### Toate hook-urile disponibile

```vue
<script setup lang="ts">
import {
  onBeforeMount,
  onMounted,
  onBeforeUpdate,
  onUpdated,
  onBeforeUnmount,
  onUnmounted,
  onErrorCaptured,
  onActivated,
  onDeactivated
} from 'vue'

// ═══ CREARE ═══
// setup() / <script setup> top-level - echivalent constructor + ngOnInit
console.log('1. setup - componenta se inițializează')

// ═══ MONTARE ═══
onBeforeMount(() => {
  // DOM-ul NU e disponibil încă
  console.log('2. beforeMount - înainte de render')
})

onMounted(() => {
  // DOM-ul ESTE disponibil
  // Aici faci: fetch data, attach event listeners, inițializare librării DOM
  console.log('3. mounted - DOM gata')
})

// ═══ ACTUALIZARE ═══
onBeforeUpdate(() => {
  // Starea s-a schimbat, DOM-ul nu s-a actualizat încă
  console.log('4. beforeUpdate - înainte de re-render')
})

onUpdated(() => {
  // DOM-ul s-a actualizat
  // ATENȚIE: nu modifica starea aici - poți cauza loop infinit!
  console.log('5. updated - DOM actualizat')
})

// ═══ DISTRUGERE ═══
onBeforeUnmount(() => {
  // Componenta încă funcționează, dar urmează să fie distrusă
  console.log('6. beforeUnmount')
})

onUnmounted(() => {
  // Cleanup: timers, event listeners, subscriptions
  console.log('7. unmounted - cleanup done')
})

// ═══ ERORI ═══
onErrorCaptured((error, instance, info) => {
  // Captează erori din componente copil
  console.error('Eroare captată:', error, info)
  return false  // previne propagarea erorii
})

// ═══ KEEP-ALIVE ═══
onActivated(() => {
  // Componenta a fost activată din cache (KeepAlive)
  console.log('activated')
})

onDeactivated(() => {
  // Componenta a fost dezactivată (pusă în cache de KeepAlive)
  console.log('deactivated')
})
</script>
```

### Exemplu practic - Fetch data + Cleanup

```vue
<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue'

interface User {
  id: number
  name: string
  email: string
}

const users = ref<User[]>([])
const isLoading = ref(true)
const error = ref<string | null>(null)

let intervalId: ReturnType<typeof setInterval> | null = null
let abortController: AbortController | null = null

onMounted(async () => {
  // DOM e disponibil - safe să accesezi elemente
  await fetchUsers()

  // Setup polling
  intervalId = setInterval(fetchUsers, 30000)
})

onUnmounted(() => {
  // CLEANUP - echivalent ngOnDestroy
  if (intervalId) {
    clearInterval(intervalId)
    intervalId = null
  }
  if (abortController) {
    abortController.abort()
    abortController = null
  }
})

async function fetchUsers() {
  // Anulează request-ul anterior dacă există
  abortController?.abort()
  abortController = new AbortController()

  try {
    isLoading.value = true
    error.value = null

    const response = await fetch('/api/users', {
      signal: abortController.signal
    })

    if (!response.ok) throw new Error(`HTTP ${response.status}`)

    users.value = await response.json()
  } catch (e) {
    if (e instanceof Error && e.name !== 'AbortError') {
      error.value = e.message
    }
  } finally {
    isLoading.value = false
  }
}
</script>

<template>
  <div v-if="isLoading" class="spinner">Se încarcă...</div>
  <div v-else-if="error" class="error">Eroare: {{ error }}</div>
  <ul v-else>
    <li v-for="user in users" :key="user.id">
      {{ user.name }} - {{ user.email }}
    </li>
  </ul>
</template>
```

### Exemplu: useTemplateRef() (Vue 3.5+)

```vue
<script setup lang="ts">
import { useTemplateRef, onMounted } from 'vue'

// Vue 3.5+ - useTemplateRef
const canvasRef = useTemplateRef<HTMLCanvasElement>('canvas')

// Alternativ (stil mai vechi, dar funcțional)
// const canvasRef = ref<HTMLCanvasElement | null>(null)

onMounted(() => {
  // DOM e gata, ref-ul e populat
  const ctx = canvasRef.value?.getContext('2d')
  if (ctx) {
    ctx.fillStyle = 'blue'
    ctx.fillRect(0, 0, 100, 100)
  }
})
</script>

<template>
  <canvas ref="canvas" width="400" height="300" />
</template>
```

### Lifecycle: Angular vs Vue - Tabel complet

| Angular | Vue Composition API | Moment | Notă |
|---------|-------------------|--------|------|
| `constructor` | `setup()` / `<script setup>` top-level | La creare | Inițializare stare, DI |
| `ngOnInit` | `onMounted()` (sau top-level `setup`) | După prima inițializare | Vue: DOM gata; Angular: bindings ready |
| `ngOnChanges` | `watch(() => props.X, ...)` | Când props/inputs se schimbă | Vue: watch explicit pe props |
| `ngAfterViewInit` | `onMounted()` | DOM-ul copil e gata | Vue are un singur hook pentru mount |
| `ngAfterContentInit` | `onMounted()` | Content projection gata | Vue nu separă view de content |
| `ngAfterViewChecked` | `onUpdated()` | După fiecare re-render | Vue: doar când DOM-ul chiar se schimbă |
| `ngDoCheck` | `onBeforeUpdate()` (loosely) | Înainte de verificare schimbări | Diferit conceptual |
| `ngOnDestroy` | `onUnmounted()` | La distrugerea componentei | Cleanup identic conceptual |
| Nu are echivalent | `onActivated()` / `onDeactivated()` | KeepAlive cache | Angular nu are KeepAlive built-in |
| `ErrorHandler` | `onErrorCaptured()` | Eroare în copii | Vue: per componentă, Angular: global |

### Paralela cu Angular

**Observații cheie pentru tranziție:**

1. **Mai puține lifecycle hooks** - Vue are mai puține hook-uri decât Angular. Nu ai `ngOnChanges` - folosești `watch()` pe props. Nu ai separare `AfterViewInit` / `AfterContentInit` - `onMounted()` acoperă ambele.

2. **setup() este cel mai important** - Majoritatea codului stă direct în `<script setup>`, nu în hook-uri.

3. **Cleanup pattern identic** - `onUnmounted()` este folosit exact ca `ngOnDestroy()` - cleanup timers, subscriptions, event listeners.

```typescript
// Angular
@Component({ /* ... */ })
export class UserComponent implements OnInit, OnDestroy {
  private destroy$ = new Subject<void>();
  users = signal<User[]>([]);

  constructor(private http: HttpClient) {}

  ngOnInit() {
    this.http.get<User[]>('/api/users')
      .pipe(takeUntil(this.destroy$))
      .subscribe(users => this.users.set(users));
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}

// Vue - echivalent
// <script setup lang="ts">
const users = ref<User[]>([])
let controller: AbortController | null = null

onMounted(async () => {
  controller = new AbortController()
  const res = await fetch('/api/users', { signal: controller.signal })
  users.value = await res.json()
})

onUnmounted(() => {
  controller?.abort()
})
// </script>
```

---

## 7. Template Syntax

### Interpolation

```vue
<template>
  <!-- Text interpolation - identic cu Angular -->
  <p>{{ message }}</p>
  <p>{{ count + 1 }}</p>
  <p>{{ isActive ? 'Da' : 'Nu' }}</p>
  <p>{{ user.name.toUpperCase() }}</p>

  <!-- HTML raw (atenție XSS!) -->
  <div v-html="rawHtml"></div>
  <!-- Angular echivalent: [innerHTML]="rawHtml" -->
</template>
```

### Directive - v-if / v-else-if / v-else

```vue
<template>
  <!-- Vue -->
  <div v-if="status === 'loading'">
    Se încarcă...
  </div>
  <div v-else-if="status === 'error'">
    Eroare: {{ errorMessage }}
  </div>
  <div v-else-if="items.length === 0">
    Nu există date.
  </div>
  <div v-else>
    <ul>
      <li v-for="item in items" :key="item.id">{{ item.name }}</li>
    </ul>
  </div>

  <!-- Grupare fără element wrapper - folosește <template> -->
  <template v-if="isLoggedIn">
    <h1>Bun venit!</h1>
    <p>Ai {{ notifications }} notificări.</p>
  </template>
</template>
```

**Angular echivalent:**

```html
<!-- Angular (noul control flow) -->
@if (status === 'loading') {
  <div>Se încarcă...</div>
} @else if (status === 'error') {
  <div>Eroare: {{ errorMessage }}</div>
} @else if (items.length === 0) {
  <div>Nu există date.</div>
} @else {
  <div>
    <ul>
      @for (item of items; track item.id) {
        <li>{{ item.name }}</li>
      }
    </ul>
  </div>
}
```

### v-for - Iterare

```vue
<template>
  <!-- Array de obiecte -->
  <ul>
    <li v-for="item in items" :key="item.id">
      {{ item.name }} - {{ item.price }}
    </li>
  </ul>

  <!-- Cu index -->
  <ul>
    <li v-for="(item, index) in items" :key="item.id">
      {{ index + 1 }}. {{ item.name }}
    </li>
  </ul>

  <!-- Iterare obiect -->
  <div v-for="(value, key, index) in userObject" :key="key">
    {{ index }}. {{ key }}: {{ value }}
  </div>

  <!-- Range -->
  <span v-for="n in 10" :key="n">{{ n }} </span>
  <!-- Output: 1 2 3 4 5 6 7 8 9 10 -->

  <!-- v-for cu v-if - folosește <template> pentru a evita conflicte -->
  <template v-for="item in items" :key="item.id">
    <li v-if="item.isVisible">{{ item.name }}</li>
  </template>

  <!-- IMPORTANT: v-if are prioritate mai mare decât v-for pe același element -->
  <!-- NU pune v-if și v-for pe același element! -->
</template>
```

**Angular echivalent:**

```html
<!-- Angular -->
@for (item of items; track item.id; let i = $index) {
  <li>{{ i + 1 }}. {{ item.name }}</li>
} @empty {
  <li>Nu există elemente.</li>
}
```

**Diferență importantă:** Vue `v-for` nu are `@empty` block built-in. Trebuie `v-if` separat:

```vue
<template>
  <ul v-if="items.length > 0">
    <li v-for="item in items" :key="item.id">{{ item.name }}</li>
  </ul>
  <p v-else>Nu există elemente.</p>
</template>
```

### v-show vs v-if

```vue
<template>
  <!-- v-if: adaugă/elimină elementul din DOM -->
  <div v-if="isVisible">Sunt adăugat/eliminat din DOM</div>

  <!-- v-show: toggle display:none (elementul rămâne în DOM) -->
  <div v-show="isVisible">Sunt mereu în DOM, doar ascuns cu CSS</div>
</template>
```

| Aspect | v-if | v-show |
|--------|------|--------|
| **Mecanism** | Adaugă/elimină din DOM | Toggle `display: none` |
| **Cost inițial** | Mai mic (nu renderează dacă false) | Mai mare (renderează mereu) |
| **Cost toggle** | Mai mare (distruge și recreează) | Mai mic (doar CSS) |
| **Când folosești** | Condiții care se schimbă rar | Toggle-uri frecvente |
| **Angular echivalent** | `@if` / `*ngIf` | `[hidden]` / `[style.display]` |
| **Suportă v-else** | Da | Nu |
| **Funcționează pe template** | Da | Nu |

### v-bind - Attribute Binding

```vue
<template>
  <!-- Syntax complet -->
  <img v-bind:src="imageUrl" v-bind:alt="imageAlt" />

  <!-- Shorthand cu : (PREFERAT) -->
  <img :src="imageUrl" :alt="imageAlt" />

  <!-- Same-name shorthand (Vue 3.4+) -->
  <img :src :alt />
  <!-- echivalent cu :src="src" :alt="alt" -->

  <!-- Dynamic attribute name -->
  <div :[attributeName]="value">...</div>

  <!-- Bind multiple attributes dintr-un obiect -->
  <div v-bind="attributesObject">...</div>
  <!-- dacă attributesObject = { id: 'app', class: 'main' } -->
  <!-- devine: <div id="app" class="main"> -->

  <!-- Boolean attributes -->
  <button :disabled="isSubmitting">Submit</button>
  <!-- Angular: <button [disabled]="isSubmitting"> -->

  <!-- Props pentru componente copil -->
  <UserCard :user="currentUser" :show-avatar="true" />
  <!-- Angular: <app-user-card [user]="currentUser" [showAvatar]="true"> -->
</template>
```

**Angular echivalent:**
- Vue `:attr="value"` = Angular `[attr]="value"`
- Vue `v-bind="obj"` = Angular nu are echivalent direct (trebuie fiecare atribut separat)

### v-on - Event Binding

```vue
<template>
  <!-- Syntax complet -->
  <button v-on:click="handleClick">Click</button>

  <!-- Shorthand cu @ (PREFERAT) -->
  <button @click="handleClick">Click</button>

  <!-- Inline expression -->
  <button @click="count++">Increment</button>

  <!-- Cu argument event -->
  <button @click="handleClick($event)">Click</button>

  <!-- Event modifiers -->
  <form @submit.prevent="onSubmit">
    <!-- .prevent = preventDefault() -->
  </form>

  <a @click.stop="handleClick">
    <!-- .stop = stopPropagation() -->
  </a>

  <button @click.once="doOnce">
    <!-- .once = se execută o singură dată -->
  </button>

  <div @click.self="onDivClick">
    <!-- .self = doar dacă target === currentTarget -->
    <button @click="onButtonClick">Button</button>
  </div>

  <!-- Combinare modifiers -->
  <form @submit.stop.prevent="onSubmit">...</form>

  <!-- Key modifiers -->
  <input @keyup.enter="submit" />
  <input @keyup.esc="cancel" />
  <input @keyup.tab="nextField" />
  <input @keyup.delete="removeItem" />
  <input @keyup.space="togglePlay" />

  <!-- Combinații de taste -->
  <input @keyup.ctrl.enter="submitAndClose" />
  <input @keyup.alt.s="save" />
  <div @click.ctrl="selectMultiple">...</div>

  <!-- Mouse modifiers -->
  <div @click.left="leftClick" />
  <div @click.right="rightClick" />
  <div @click.middle="middleClick" />

  <!-- Dynamic event name -->
  <button @[eventName]="handler">...</button>
</template>
```

**Angular echivalent:**
- Vue `@click="handler"` = Angular `(click)="handler()"`
- Vue `.prevent` modifier = Angular: trebuie `$event.preventDefault()` manual
- Vue `.stop` modifier = Angular: trebuie `$event.stopPropagation()` manual
- Vue `@keyup.enter` = Angular `(keyup.enter)` (similar!)

### v-model - Two-Way Binding

```vue
<script setup lang="ts">
import { ref } from 'vue'

const name = ref('')
const age = ref(0)
const isActive = ref(false)
const selectedColor = ref('red')
const selectedColors = ref<string[]>([])
const message = ref('')
</script>

<template>
  <!-- Input text -->
  <input v-model="name" placeholder="Nume" />
  <!-- Echivalent cu: <input :value="name" @input="name = $event.target.value" /> -->

  <!-- Input number -->
  <input v-model.number="age" type="number" />
  <!-- .number modifier - convertește la number automat -->

  <!-- Checkbox -->
  <input v-model="isActive" type="checkbox" />

  <!-- Radio -->
  <input v-model="selectedColor" type="radio" value="red" /> Roșu
  <input v-model="selectedColor" type="radio" value="blue" /> Albastru

  <!-- Multiple checkboxes → array -->
  <input v-model="selectedColors" type="checkbox" value="red" /> Roșu
  <input v-model="selectedColors" type="checkbox" value="blue" /> Albastru
  <input v-model="selectedColors" type="checkbox" value="green" /> Verde
  <!-- selectedColors = ['red', 'green'] dacă sunt bifate -->

  <!-- Select -->
  <select v-model="selectedColor">
    <option value="red">Roșu</option>
    <option value="blue">Albastru</option>
  </select>

  <!-- Textarea -->
  <textarea v-model="message" />

  <!-- Modifiers -->
  <input v-model.lazy="name" />      <!-- actualizare la 'change', nu 'input' -->
  <input v-model.trim="name" />      <!-- trim automat whitespace -->
  <input v-model.number="age" />     <!-- convertește la number -->
</template>
```

**Angular echivalent:**
- Vue `v-model="name"` = Angular `[(ngModel)]="name"` (cu FormsModule)
- Vue `v-model` pe componente custom = Angular `[(ngModel)]` cu ControlValueAccessor SAU `model()` signal input

### Class și Style Binding

```vue
<template>
  <!-- ═══ CLASS BINDING ═══ -->

  <!-- Object syntax -->
  <div :class="{ active: isActive, 'text-danger': hasError }">
    <!-- class="active" dacă isActive e true -->
    <!-- class="active text-danger" dacă ambele sunt true -->
  </div>

  <!-- Array syntax -->
  <div :class="[activeClass, errorClass]">
    <!-- activeClass = 'active', errorClass = 'text-danger' -->
    <!-- class="active text-danger" -->
  </div>

  <!-- Combinat cu clase statice -->
  <div class="base-class" :class="{ active: isActive }">
    <!-- class="base-class active" -->
  </div>

  <!-- Array cu condiții -->
  <div :class="[isActive ? 'active' : '', errorClass]">...</div>

  <!-- Array cu object syntax -->
  <div :class="[{ active: isActive }, errorClass]">...</div>

  <!-- Computed class -->
  <div :class="computedClasses">...</div>

  <!-- ═══ STYLE BINDING ═══ -->

  <!-- Object syntax -->
  <div :style="{ color: textColor, fontSize: fontSize + 'px' }">
    <!-- camelCase SAU kebab-case cu quotes -->
  </div>

  <div :style="{ 'font-size': fontSize + 'px' }">...</div>

  <!-- Object ref -->
  <div :style="styleObject">...</div>

  <!-- Array syntax (merge multiple style objects) -->
  <div :style="[baseStyles, overrideStyles]">...</div>

  <!-- Auto-prefixing - Vue adaugă automat vendor prefixes -->
  <div :style="{ transform: 'rotate(45deg)' }">...</div>
</template>

<script setup lang="ts">
import { ref, computed } from 'vue'

const isActive = ref(true)
const hasError = ref(false)
const textColor = ref('#333')
const fontSize = ref(16)

const computedClasses = computed(() => ({
  'is-active': isActive.value,
  'has-error': hasError.value,
  'is-large': fontSize.value > 20
}))

const styleObject = ref({
  color: '#333',
  fontSize: '16px',
  fontWeight: 'bold'
})
</script>
```

**Angular echivalent:**

| Vue | Angular | Exemplu |
|-----|---------|---------|
| `:class="{ active: isActive }"` | `[ngClass]="{ active: isActive }"` | Object syntax - identic! |
| `:class="[cls1, cls2]"` | `[ngClass]="[cls1, cls2]"` | Array syntax - identic! |
| `:class="computedClasses"` | `[ngClass]="computedClasses"` | Computed - identic! |
| `:style="{ color: c }"` | `[ngStyle]="{ color: c }"` | Object syntax - identic! |
| `:style="[s1, s2]"` | `[ngStyle]` (nu suportă array nativ) | Vue e mai flexibil |
| `class="static"` + `:class="dynamic"` | `class="static"` + `[ngClass]="dynamic"` | Merge automat |

### Tabel rezumativ Template Syntax: Vue vs Angular

| Funcționalitate | Vue 3 | Angular 17+ | Notă |
|-----------------|-------|-------------|------|
| **Interpolation** | `{{ expr }}` | `{{ expr }}` | Identic |
| **Attribute binding** | `:attr="val"` | `[attr]="val"` | Sintaxă diferită, concept identic |
| **Event binding** | `@event="handler"` | `(event)="handler()"` | Sintaxă diferită |
| **Two-way binding** | `v-model="val"` | `[(ngModel)]="val"` | Vue: built-in; Angular: necesită FormsModule |
| **Conditional** | `v-if` / `v-else` | `@if` / `@else` | Ambele elimină din DOM |
| **Show/Hide** | `v-show` | `[hidden]` | Vue: directivă dedicată |
| **Loop** | `v-for="x in list"` | `@for (x of list; track x.id)` | Angular: track obligatoriu |
| **Empty state** | `v-if` separat | `@empty { }` | Angular mai elegant aici |
| **Class binding** | `:class="{ a: true }"` | `[ngClass]="{ a: true }"` | Aproape identic |
| **Style binding** | `:style="{ color: c }"` | `[ngStyle]="{ color: c }"` | Aproape identic |
| **Raw HTML** | `v-html="html"` | `[innerHTML]="html"` | Ambele: atenție XSS |
| **Template ref** | `ref="name"` | `#name` | Acces la element DOM |
| **Event modifier** | `@click.prevent` | Manual: `$event.preventDefault()` | Vue mai concis |
| **Key events** | `@keyup.enter` | `(keyup.enter)` | Similar |
| **Slot / Projection** | `<slot>` | `<ng-content>` | Concept identic |
| **Dynamic component** | `<component :is="comp">` | `ngComponentOutlet` | Vue mai simplu |
| **Teleport** | `<Teleport to="#target">` | `cdkPortal` (Angular CDK) | Vue built-in |

### Paralela cu Angular

**Cele mai importante diferențe de sintaxă:**

1. **Binding syntax** - Vue folosește `:` și `@`, Angular folosește `[]` și `()`
2. **Directives** - Vue folosește `v-` prefix (v-if, v-for, v-model), Angular folosește `@` blocks sau `*` prefix
3. **Modifiers** - Vue are `.prevent`, `.stop`, `.once` pe evenimente; Angular nu are echivalent (faci manual)
4. **v-model** - Vue: works out of the box pe orice input. Angular: necesită `FormsModule` import
5. **Track in v-for** - Vue: `:key` atribut separat. Angular: `track` integrat în `@for`

**Ce e similar:** Interpolation `{{ }}`, conceptul de binding, lifecycle, component composition. Dacă știi Angular templates, Vue templates vin natural - doar sintaxa e diferită.

---

## 8. Components - Props, Emits, Slots

### Props - Primirea datelor de la părinte

#### Runtime Declaration

```vue
<script setup lang="ts">
// Runtime declaration - validare la runtime
const props = defineProps({
  // String obligatoriu
  title: {
    type: String,
    required: true
  },
  // Number cu default
  count: {
    type: Number,
    default: 0
  },
  // Boolean (default false implicit dacă nu e pasat)
  isVisible: {
    type: Boolean,
    default: true
  },
  // Array cu factory function pentru default
  items: {
    type: Array as PropType<string[]>,
    default: () => []
  },
  // Object cu validare custom
  user: {
    type: Object as PropType<User>,
    required: true,
    validator: (value: User) => {
      return value.name.length > 0 && value.age > 0
    }
  },
  // Union types
  size: {
    type: String as PropType<'sm' | 'md' | 'lg'>,
    default: 'md'
  }
})
</script>
```

#### Type-based Declaration (PREFERAT cu TypeScript)

```vue
<script setup lang="ts">
// Type-based - PREFERAT cu TypeScript
// Compilatorul inferă validarea din tipuri

interface Props {
  title: string
  count?: number
  isVisible?: boolean
  items?: string[]
  user: User
  size?: 'sm' | 'md' | 'lg'
  onCustomEvent?: (id: number) => void
}

const props = defineProps<Props>()

// Cu default values - withDefaults()
const props = withDefaults(defineProps<Props>(), {
  count: 0,
  isVisible: true,
  items: () => [],           // factory function pentru obiecte/array-uri
  size: 'md'
})

// Acces la props
console.log(props.title)     // direct, fără .value
console.log(props.count)     // 0 (default)
</script>
```

#### Reactive Props Destructure (Vue 3.5+)

```vue
<script setup lang="ts">
// Vue 3.5+ - destructurare reactivă! (înainte pierdeai reactivitatea)
const { title, count = 0, items = [] } = defineProps<{
  title: string
  count?: number
  items?: string[]
}>()

// title, count, items sunt reactive automat
// Poți folosi direct în watch()
watch(() => count, (newCount) => {
  console.log('count changed:', newCount)
})
</script>
```

#### Props în template

```vue
<template>
  <!-- Pasare props -->
  <ChildComponent
    title="Static Title"
    :count="dynamicCount"
    :user="currentUser"
    :items="['a', 'b', 'c']"
    is-visible
  />
  <!-- is-visible fără valoare = true (boolean shorthand) -->

  <!-- Spread all props -->
  <ChildComponent v-bind="propsObject" />
</template>
```

### Angular echivalent Props

```typescript
// Angular - @Input() decorator (clasic)
@Component({ /* ... */ })
export class ChildComponent {
  @Input({ required: true }) title!: string;
  @Input() count = 0;
  @Input() isVisible = true;
}

// Angular - input() signal (modern)
@Component({ /* ... */ })
export class ChildComponent {
  title = input.required<string>();
  count = input(0);             // cu default
  isVisible = input(true);
}
```

| Vue defineProps | Angular Input | Notă |
|----------------|---------------|------|
| `defineProps<{ title: string }>()` | `title = input.required<string>()` | Ambele type-safe |
| `withDefaults(defineProps<P>(), { count: 0 })` | `count = input(0)` | Angular mai concis cu defaults |
| `props.title` (direct) | `this.title()` (signal getter) | Vue: proprietate, Angular: funcție |
| Props sunt readonly | Inputs sunt readonly | Ambele: one-way data flow |
| Reactive destructure (3.5+) | Nu are echivalent | Vue permite destructurare |

### Emits - Comunicare copil → părinte

#### Runtime Declaration

```vue
<script setup lang="ts">
// Runtime declaration
const emit = defineEmits(['update', 'delete', 'search'])

// Emitere
function handleUpdate(id: number) {
  emit('update', id)
}

function handleDelete(id: number) {
  emit('delete', id)
}

function handleSearch(query: string, page: number) {
  emit('search', query, page)
}
</script>
```

#### Type-based Declaration (PREFERAT)

```vue
<script setup lang="ts">
// Type-based - PREFERAT
// Definești exact ce parametri acceptă fiecare event
const emit = defineEmits<{
  update: [id: number]
  delete: [id: number]
  search: [query: string, page: number]
  'status-change': [status: 'active' | 'inactive']
}>()

// Emitere - TypeScript verifică parametrii
emit('update', 42)                    // OK
emit('search', 'vue', 1)             // OK
// emit('update', 'string')           // EROARE TypeScript!
// emit('search', 'vue')              // EROARE - lipsește page!
// emit('unknown-event')              // EROARE - event necunoscut!
</script>
```

#### Ascultare emits în părinte

```vue
<template>
  <!-- Ascultare în părinte -->
  <ChildComponent
    @update="handleUpdate"
    @delete="handleDelete"
    @search="(query, page) => performSearch(query, page)"
    @status-change="onStatusChange"
  />
</template>

<script setup lang="ts">
function handleUpdate(id: number) {
  console.log('Update item:', id)
}

function handleDelete(id: number) {
  console.log('Delete item:', id)
}

function performSearch(query: string, page: number) {
  console.log('Search:', query, 'Page:', page)
}

function onStatusChange(status: 'active' | 'inactive') {
  console.log('Status:', status)
}
</script>
```

### Angular echivalent Emits

```typescript
// Angular - @Output() clasic
@Component({ /* ... */ })
export class ChildComponent {
  @Output() update = new EventEmitter<number>();
  @Output() delete = new EventEmitter<number>();

  handleUpdate(id: number) {
    this.update.emit(id);
  }
}

// Angular - output() modern
@Component({ /* ... */ })
export class ChildComponent {
  update = output<number>();
  delete = output<number>();

  handleUpdate(id: number) {
    this.update.emit(id);
  }
}
```

| Vue defineEmits | Angular Output | Notă |
|----------------|----------------|------|
| `emit('update', id)` | `this.update.emit(id)` | Aproape identic |
| `defineEmits<{ update: [id: number] }>()` | `update = output<number>()` | Ambele type-safe |
| Event name: kebab-case recomandat | Event name: camelCase | Convenție diferită |
| `@update="handler"` (în template) | `(update)="handler($event)"` | Sintaxă binding diferită |
| Multipli parametri nativ | Un singur parametru (obiect wrapper) | Vue mai flexibil |

### Slots - Content Projection

#### Default Slot

```vue
<!-- Card.vue - componentă cu slot -->
<template>
  <div class="card">
    <div class="card-body">
      <slot>
        <!-- Fallback content (afișat dacă părintele nu trimite conținut) -->
        <p>Conținut implicit</p>
      </slot>
    </div>
  </div>
</template>

<!-- Utilizare în părinte -->
<template>
  <Card>
    <p>Acesta este conținutul cardului.</p>
  </Card>

  <!-- Fără conținut - va afișa fallback-ul -->
  <Card />
</template>
```

#### Named Slots

```vue
<!-- PageLayout.vue -->
<template>
  <div class="page">
    <header>
      <slot name="header" />
    </header>

    <main>
      <slot />  <!-- default slot (fără name) -->
    </main>

    <aside>
      <slot name="sidebar" />
    </aside>

    <footer>
      <slot name="footer">
        <p>Footer implicit &copy; 2024</p>
      </slot>
    </footer>
  </div>
</template>

<!-- Utilizare în părinte -->
<template>
  <PageLayout>
    <template #header>
      <h1>Titlul paginii</h1>
      <nav>
        <a href="/">Acasă</a>
        <a href="/about">Despre</a>
      </nav>
    </template>

    <!-- default slot - conținut fără template #name -->
    <article>
      <p>Conținutul principal al paginii.</p>
    </article>

    <template #sidebar>
      <ul>
        <li>Link 1</li>
        <li>Link 2</li>
      </ul>
    </template>

    <!-- footer nu e specificat - va afișa fallback -->
  </PageLayout>
</template>
```

#### Scoped Slots (slot cu date de la copil)

```vue
<!-- DataList.vue - expune date prin slot -->
<script setup lang="ts">
interface Props {
  items: any[]
}
const props = defineProps<Props>()
</script>

<template>
  <ul>
    <li v-for="(item, index) in items" :key="index">
      <!-- Expune item și index către părinte -->
      <slot name="item" :item="item" :index="index" :isLast="index === items.length - 1">
        <!-- Fallback: afișare simplă -->
        {{ item }}
      </slot>
    </li>
  </ul>

  <div v-if="items.length === 0">
    <slot name="empty">
      <p>Nu există elemente.</p>
    </slot>
  </div>
</template>

<!-- Utilizare cu scoped slot -->
<template>
  <DataList :items="users">
    <template #item="{ item: user, index, isLast }">
      <div :class="{ 'border-bottom': !isLast }">
        <strong>{{ index + 1 }}.</strong>
        {{ user.name }} - {{ user.email }}
      </div>
    </template>

    <template #empty>
      <div class="no-data">
        <img src="/empty.svg" alt="No data" />
        <p>Nu există utilizatori.</p>
      </div>
    </template>
  </DataList>
</template>
```

#### Typed Slots (Vue 3.3+)

```vue
<script setup lang="ts">
interface User {
  id: number
  name: string
  email: string
}

defineProps<{
  items: User[]
}>()

// defineSlots pentru type safety
defineSlots<{
  default: (props: { greeting: string }) => any
  item: (props: { item: User; index: number }) => any
  empty: () => any
}>()
</script>
```

### Angular echivalent Slots

| Vue Slot | Angular | Exemplu |
|----------|---------|---------|
| `<slot />` (default) | `<ng-content />` | Content projection simplă |
| `<slot name="header" />` | `<ng-content select="[header]" />` | Named projection |
| `#header` (în părinte) | `header` attribute pe element | Selecție content |
| Scoped slot `#item="{ data }"` | `<ng-template let-data>` + context | Scoped: Vue mai ergonomic |
| Fallback content în slot | Nu are echivalent direct | Vue: conținut default în `<slot>` |
| `defineSlots<{...}>()` | Nu are echivalent typed | Vue 3.3+ avantaj TypeScript |

```html
<!-- Angular - content projection -->
<app-card>
  <div header>
    <h1>Title</h1>
  </div>
  <p>Main content</p>
  <div footer>
    <button>Action</button>
  </div>
</app-card>

<!-- app-card.component.html -->
<div class="card">
  <div class="header">
    <ng-content select="[header]" />
  </div>
  <div class="body">
    <ng-content />
  </div>
  <div class="footer">
    <ng-content select="[footer]" />
  </div>
</div>
```

### defineModel() (Vue 3.4+)

`defineModel()` simplifică **v-model pe componente custom**:

```vue
<!-- ÎNAINTE (Vue 3.3 și anterior) - verbose -->
<script setup lang="ts">
const props = defineProps<{
  modelValue: string
}>()
const emit = defineEmits<{
  'update:modelValue': [value: string]
}>()

// Trebuia să gestionezi manual:
function onInput(event: Event) {
  emit('update:modelValue', (event.target as HTMLInputElement).value)
}
</script>

<template>
  <input :value="modelValue" @input="onInput" />
</template>
```

```vue
<!-- ACUM (Vue 3.4+) - cu defineModel -->
<script setup lang="ts">
// defineModel creează automat prop + emit
const model = defineModel<string>({ required: true })
// model este un Ref<string> care se sincronizează cu v-model al părintelui
</script>

<template>
  <!-- v-model direct pe model ref -->
  <input v-model="model" />
</template>

<!-- Utilizare în părinte -->
<!-- <CustomInput v-model="userName" /> -->
```

#### Multiple v-model bindings

```vue
<!-- UserForm.vue -->
<script setup lang="ts">
const firstName = defineModel<string>('firstName', { required: true })
const lastName = defineModel<string>('lastName', { required: true })
const age = defineModel<number>('age', { default: 0 })
</script>

<template>
  <input v-model="firstName" placeholder="Prenume" />
  <input v-model="lastName" placeholder="Nume" />
  <input v-model.number="age" type="number" placeholder="Vârsta" />
</template>

<!-- Utilizare -->
<!--
<UserForm
  v-model:first-name="user.firstName"
  v-model:last-name="user.lastName"
  v-model:age="user.age"
/>
-->
```

### Angular echivalent defineModel

```typescript
// Angular - model() signal input (Angular 17.1+)
@Component({
  template: `<input [value]="value()" (input)="value.set($event.target.value)" />`
})
export class CustomInputComponent {
  value = model.required<string>();
}

// Utilizare:
// <app-custom-input [(value)]="userName" />
```

### defineExpose()

Controlează ce proprietăți sunt accesibile din **template ref** al părintelui:

```vue
<!-- ChildComponent.vue -->
<script setup lang="ts">
import { ref } from 'vue'

const internalState = ref('privat')
const count = ref(0)

function reset() {
  count.value = 0
}

function increment() {
  count.value++
}

// Doar ce e în defineExpose e accesibil din părinte
defineExpose({
  count,      // ref-ul count
  reset,      // metoda reset
  // internalState NU e expus
  // increment NU e expus
})
</script>
```

```vue
<!-- ParentComponent.vue -->
<script setup lang="ts">
import { useTemplateRef } from 'vue'
import ChildComponent from './ChildComponent.vue'

const childRef = useTemplateRef<InstanceType<typeof ChildComponent>>('child')

function resetChild() {
  // Accesezi doar ce e expus prin defineExpose
  console.log(childRef.value?.count)
  childRef.value?.reset()

  // childRef.value?.internalState  // EROARE - nu e expus!
  // childRef.value?.increment      // EROARE - nu e expus!
}
</script>

<template>
  <ChildComponent ref="child" />
  <button @click="resetChild">Reset Child</button>
</template>
```

### Angular echivalent defineExpose

```typescript
// Angular - totul public pe clasă e accesibil prin ViewChild
@Component({ /* ... */ })
export class ChildComponent {
  count = signal(0);
  private internalState = 'privat';  // private = nu e accesibil

  reset() { this.count.set(0); }
}

// Părinte
@Component({ /* ... */ })
export class ParentComponent {
  child = viewChild.required(ChildComponent);

  resetChild() {
    this.child().reset();
    this.child().count();  // accesibil - e public
  }
}
```

**Diferență:** Vue ascunde totul implicit (cu `<script setup>`) și expui explicit cu `defineExpose()`. Angular expune totul public implicit și ascunzi cu `private`/`protected`.

### Paralela cu Angular - Rezumat Components

| Concept | Vue 3 | Angular | Notă |
|---------|-------|---------|------|
| **Definire component** | SFC (`.vue` file) | Class + Decorator | Vue: un fișier = un component |
| **Props** | `defineProps<T>()` | `input<T>()` / `@Input()` | Ambele type-safe |
| **Props default** | `withDefaults()` | Valoare inițială în `input()` | Angular mai concis |
| **Events** | `defineEmits<T>()` | `output<T>()` / `@Output()` | Vue: multipli params nativ |
| **Slots** | `<slot>` / `<slot name="x">` | `<ng-content>` / `<ng-content select>` | Vue: scoped slots mai puternice |
| **v-model** | `defineModel()` | `model()` signal | Ambele simplifică two-way binding |
| **Expose** | `defineExpose()` | `public` / `private` keywords | Vue: implicit ascuns |
| **Template ref** | `useTemplateRef()` | `viewChild()` | Similar |

---

## 9. defineProps/defineEmits cu TypeScript - Advanced

### Complex Types cu defineProps

```vue
<script setup lang="ts">
import type { Component } from 'vue'

// Union types
interface CardProps {
  variant: 'primary' | 'secondary' | 'danger' | 'success'
  size: 'sm' | 'md' | 'lg' | 'xl'
}

// Nested complex types
interface TableColumn<T = any> {
  key: keyof T
  label: string
  sortable?: boolean
  formatter?: (value: any, row: T) => string
  component?: Component
  width?: string | number
}

interface TableProps<T> {
  columns: TableColumn<T>[]
  data: T[]
  loading?: boolean
  pagination?: {
    page: number
    pageSize: number
    total: number
  }
  selectable?: boolean
  selectedRows?: T[]
}

// Folosire cu interface complexe
const props = withDefaults(defineProps<{
  columns: TableColumn[]
  data: Record<string, any>[]
  loading?: boolean
  emptyText?: string
  striped?: boolean
  bordered?: boolean
}>(), {
  loading: false,
  emptyText: 'Nu există date',
  striped: false,
  bordered: true
})
</script>
```

### Generic Components (Vue 3.3+)

```vue
<!-- GenericList.vue -->
<script setup lang="ts" generic="T extends { id: number | string }">
// generic="T" - declară un type parameter!
// Funcționează similar cu Angular generic components

defineProps<{
  items: T[]
  selected?: T
}>()

defineEmits<{
  select: [item: T]
  delete: [item: T]
}>()

defineSlots<{
  default: (props: { item: T; index: number }) => any
  empty: () => any
}>()
</script>

<template>
  <ul v-if="items.length">
    <li v-for="(item, index) in items" :key="item.id">
      <slot :item="item" :index="index">
        {{ item }}
      </slot>
    </li>
  </ul>
  <div v-else>
    <slot name="empty">
      <p>Niciun element.</p>
    </slot>
  </div>
</template>
```

```vue
<!-- Utilizare cu type inference -->
<script setup lang="ts">
interface User {
  id: number
  name: string
  email: string
}

const users = ref<User[]>([
  { id: 1, name: 'Emanuel', email: 'emanuel@test.com' }
])

function selectUser(user: User) {
  // TypeScript știe că user este User!
  console.log(user.name)
}
</script>

<template>
  <!-- T se inferă ca User din :items -->
  <GenericList :items="users" @select="selectUser">
    <template #default="{ item, index }">
      <!-- item este typed ca User automat! -->
      <span>{{ index + 1 }}. {{ item.name }} ({{ item.email }})</span>
    </template>
  </GenericList>
</template>
```

### Multiple generics

```vue
<script setup lang="ts" generic="TKey extends string | number, TValue">
defineProps<{
  entries: Map<TKey, TValue>
  selectedKey?: TKey
}>()

defineEmits<{
  select: [key: TKey, value: TValue]
}>()
</script>
```

### defineEmits cu overloads

```vue
<script setup lang="ts">
// Pattern pentru emit-uri complexe cu discriminated unions
type FormEvents = {
  submit: [data: FormData]
  validate: [field: string, isValid: boolean]
  reset: []
  'field-change': [field: string, value: unknown, previousValue: unknown]
}

const emit = defineEmits<FormEvents>()

// Toate sunt type-checked:
emit('submit', new FormData())
emit('validate', 'email', true)
emit('reset')
emit('field-change', 'name', 'John', 'Jane')
</script>
```

### ExtractPropTypes și ExtractPublicPropTypes

```typescript
// utils/types.ts
import type { ExtractPropTypes, ExtractPublicPropTypes, PropType } from 'vue'

// Runtime props definition
const buttonProps = {
  variant: {
    type: String as PropType<'primary' | 'secondary'>,
    default: 'primary'
  },
  size: {
    type: String as PropType<'sm' | 'md' | 'lg'>,
    default: 'md'
  },
  disabled: {
    type: Boolean,
    default: false
  },
  loading: {
    type: Boolean,
    default: false
  }
} as const

// Extrage tipul intern (cu Required pe defaults)
type InternalButtonProps = ExtractPropTypes<typeof buttonProps>

// Extrage tipul public (cu Optional pe defaults)
type PublicButtonProps = ExtractPublicPropTypes<typeof buttonProps>
// { variant?: 'primary' | 'secondary', size?: 'sm' | 'md' | 'lg', ... }
```

### ComponentCustomProps - Extending global props

```typescript
// global.d.ts
declare module 'vue' {
  interface ComponentCustomProperties {
    $translate: (key: string) => string
  }

  // Adaugă props globale pe toate componentele
  interface ComponentCustomProps {
    'data-testid'?: string
  }
}
```

### Paralela cu Angular

| Vue TypeScript Feature | Angular Echivalent | Notă |
|----------------------|-------------------|------|
| `defineProps<T>()` | `input<T>()` cu TypeScript | Ambele full type-safe |
| `generic="T"` pe componentă | Generic class `Component<T>` | Vue: la nivel SFC; Angular: la nivel clasă |
| `defineSlots<T>()` | Nu are echivalent typed | Vue 3.3+ avantaj |
| `ExtractPropTypes` | Nu e necesar (clasele au tipuri native) | Vue needs it for runtime props |
| Complex union props | Input cu union types | Identic TypeScript |

---

## 10. Tabel Complet: Angular → Vue Mapping

### Component Basics

| Angular | Vue 3 | Notă |
|---------|-------|------|
| `@Component({ selector, template })` | SFC (`.vue` file) cu `<template>` + `<script setup>` | Vue: un fișier = template + logic + style |
| `@Component({ standalone: true })` | Toate componentele sunt "standalone" | Vue nu are concept de module |
| `class MyComponent` | `<script setup>` (nu e clasă) | Vue: funcțional, nu OOP |
| `imports: [OtherComponent]` | `import OtherComponent from '...'` (auto-registered) | Vue `<script setup>`: import = registered |
| `@Component({ styles })` | `<style scoped>` | Vue: scoped styles built-in |
| `encapsulation: ViewEncapsulation.None` | `<style>` (fără scoped) | Ambele suportă scoped/global |
| `changeDetection: OnPush` | Nu e necesar (fine-grained reactivity) | Vue: reactivity la nivel proprietate |

### Data & State

| Angular | Vue 3 | Notă |
|---------|-------|------|
| `signal(value)` | `ref(value)` | Creare stare reactivă |
| `signal()` (citire) | `ref.value` (citire) | Acces valoare |
| `signal.set(value)` | `ref.value = value` | Setare valoare |
| `signal.update(fn)` | `ref.value = fn(ref.value)` | Update funcțional |
| `computed(() => ...)` | `computed(() => ...)` | Valori derivate |
| `effect(() => ...)` | `watchEffect(() => ...)` | Side effects reactive |
| Nu are echivalent | `reactive({})` | Proxy reactiv (doar obiecte) |
| Nu are echivalent | `watch(source, callback)` | Watch cu acces la oldValue |
| `linkedSignal()` | Writable `computed()` | State derivat writable |

### Component Communication

| Angular | Vue 3 | Notă |
|---------|-------|------|
| `input<T>()` / `@Input()` | `defineProps<T>()` | Props de la părinte |
| `input.required<T>()` | `defineProps<{ prop: T }>()` (required) | Props obligatorii |
| `output<T>()` / `@Output()` | `defineEmits<T>()` | Evenimente către părinte |
| `model<T>()` | `defineModel<T>()` | Two-way binding |
| `viewChild()` | `useTemplateRef()` | Referință la copil |
| `viewChildren()` | `useTemplateRef()` (array) | Referințe multiple |
| `contentChild()` | Scoped slot | Proiecție conținut cu date |
| `contentChildren()` | Multiple scoped slots | Proiecție multiplă |

### Lifecycle

| Angular | Vue 3 | Moment |
|---------|-------|--------|
| `constructor` | `<script setup>` top-level code | Inițializare |
| `ngOnInit` | `onMounted()` (sau top-level setup) | Componenta ready |
| `ngOnChanges` | `watch(() => props.x, ...)` | Prop changed |
| `ngAfterViewInit` | `onMounted()` | DOM gata |
| `ngAfterContentInit` | `onMounted()` | Content projected |
| `ngAfterViewChecked` | `onUpdated()` | DOM actualizat |
| `ngDoCheck` | `onBeforeUpdate()` | Înainte de re-render |
| `ngOnDestroy` | `onUnmounted()` | Cleanup |
| Nu are echivalent | `onActivated()` / `onDeactivated()` | KeepAlive |
| `ErrorHandler` | `onErrorCaptured()` | Error boundary |

### Template Syntax

| Angular | Vue 3 | Exemplu |
|---------|-------|---------|
| `{{ expr }}` | `{{ expr }}` | Interpolation |
| `[property]="expr"` | `:property="expr"` | Property binding |
| `(event)="handler()"` | `@event="handler"` | Event binding |
| `[(ngModel)]="val"` | `v-model="val"` | Two-way binding |
| `@if (cond) { }` | `v-if="cond"` | Conditional rendering |
| `@else { }` | `v-else` | Else branch |
| `@else if (cond) { }` | `v-else-if="cond"` | Else if |
| `@for (x of list; track x.id) { }` | `v-for="x in list" :key="x.id"` | Iteration |
| `@empty { }` | `v-if="list.length === 0"` (separat) | Empty state |
| `@switch (val) { @case ('a') { } }` | `v-if` / `v-else-if` chain | Switch |
| `[ngClass]="{ active: isActive }"` | `:class="{ active: isActive }"` | Class binding |
| `[ngStyle]="{ color: c }"` | `:style="{ color: c }"` | Style binding |
| `[innerHTML]="html"` | `v-html="html"` | Raw HTML |
| `#ref` | `ref="name"` | Template reference |
| `(keyup.enter)="fn()"` | `@keyup.enter="fn"` | Key events |
| `(click)="$event.preventDefault()"` | `@click.prevent="fn"` | Event modifiers |

### Dependency Injection & Services

| Angular | Vue 3 | Notă |
|---------|-------|------|
| `@Injectable() class MyService` | `export function useMyComposable()` | Service vs Composable |
| `inject(MyService)` | `inject(key)` (provide/inject) | DI mechanism |
| `providedIn: 'root'` (singleton) | `app.provide(key, value)` | App-level singleton |
| `providers: [MyService]` (component) | `provide(key, value)` în componentă | Component-level |
| `InjectionToken<T>` | `InjectionKey<T>` (Symbol) | Token tipat |
| `useClass` / `useFactory` | Factory function în `provide()` | Provider config |
| Ierarhie DI automată | Ierarhie provide/inject automată | Ambele: copiii moștenesc |
| `@Optional()` | `inject(key, defaultValue)` | Optional injection |

### Routing

| Angular | Vue 3 (Vue Router) | Notă |
|---------|-------------------|------|
| `provideRouter(routes)` | `createRouter({ routes })` | Config router |
| `{ path, component }` | `{ path, component }` | Route definition - identic! |
| `loadComponent: () => import(...)` | `component: () => import(...)` | Lazy loading |
| `<router-outlet>` | `<RouterView>` | Outlet |
| `routerLink="/path"` | `<RouterLink to="/path">` | Navigation link |
| `ActivatedRoute` | `useRoute()` | Route curentă |
| `Router.navigate()` | `useRouter().push()` | Navigare programatică |
| Route guards (functions) | Navigation guards | Concept identic |
| Route resolvers | `beforeEnter` + async setup | Data loading |
| `{ path: ':id' }` | `{ path: ':id' }` | Params - identic! |
| `?query=value` | `?query=value` | Query params - identic! |

### State Management

| Angular (NgRx/Signals) | Vue (Pinia) | Notă |
|------------------------|-------------|------|
| NgRx Store | Pinia Store | State management centralizat |
| `createFeature()` | `defineStore()` | Definire store |
| `store.select()` | `storeToRefs(store)` | Acces stare |
| Actions | Store actions (methods) | Modificare stare |
| Reducers | Direct mutation (Pinia) | Pinia: fără reducers |
| Selectors | Getters (computed) | Derivare stare |
| Effects (NgRx) | Actions cu async/await | Side effects |
| Signal Store (nou) | Pinia Composition Store | Similar conceptual |

### Forms

| Angular | Vue 3 | Notă |
|---------|-------|------|
| `ReactiveFormsModule` | `v-model` + validare manuală/VeeValidate | Vue: fără form module built-in |
| `FormControl` | `ref()` + `v-model` | Stare câmp |
| `FormGroup` | `reactive({})` sau obiect de ref-uri | Grupare câmpuri |
| `FormArray` | `ref<T[]>([])` | Array de valori |
| `Validators.required` | `required` HTML sau librărie (VeeValidate, Zod) | Validare |
| `formControl.errors` | Gestionare manuală sau VeeValidate | Erori validare |
| `form.valid` | `computed(() => isValid(...))` | Stare formular |
| `AsyncValidator` | Custom cu `watch()` + async | Validare asincronă |
| `ControlValueAccessor` | `defineModel()` | Custom form control |

### Build & Tooling

| Angular | Vue 3 | Notă |
|---------|-------|------|
| Angular CLI | `create-vue` / Vite | Scaffolding |
| Webpack / esbuild | Vite (dev) / Webpack (MFE prod) | Build tools |
| `ng serve` | `vite dev` | Dev server |
| `ng build` | `vite build` | Production build |
| `ng test` (Karma/Jest) | Vitest | Unit testing |
| Protractor / Playwright | Cypress / Playwright | E2E testing |
| Angular ESLint | ESLint + eslint-plugin-vue | Linting |
| `angular.json` | `vite.config.ts` | Config |

### Concepts fără echivalent direct

| Angular (fără echivalent Vue) | Notă |
|------------------------------|------|
| `NgModule` | Vue nu are module - toate componentele sunt standalone |
| Pipes (`\| uppercase`) | Vue: computed sau helper functions |
| `HostListener` / `HostBinding` | Vue: event listeners manuali |
| Interceptors HTTP | Vue: Axios interceptors sau middleware custom |
| Zone.js | Vue: nu folosește Zone.js (Proxy-based reactivity) |
| `APP_INITIALIZER` | Vue: `app.use()` plugin sau top-level `await` în setup |
| Schematics | Nu are echivalent standard |

| Vue (fără echivalent Angular) | Notă |
|------------------------------|------|
| `v-show` | Angular: `[hidden]` (dar nu e identic) |
| `<Transition>` / `<TransitionGroup>` | Angular: `@angular/animations` (mai complex) |
| `<KeepAlive>` | Angular: nu are built-in |
| `<Teleport>` | Angular: CDK Portal |
| `<Suspense>` | Angular: nu are built-in (async pipe parțial) |
| `v-model` modifiers (`.lazy`, `.trim`, `.number`) | Angular: manual |
| Event modifiers (`.prevent`, `.stop`) | Angular: manual `$event.preventDefault()` |

### Paralela cu Angular

**Concluzie pentru tranziție:**

Dacă vii din Angular, cele mai mari ajustări sunt:
1. **Gândire funcțională** vs class-based - Vue Composition API e funcțional
2. **Fără NgModules** - totul se importă direct
3. **Fără DI clasic** - provide/inject e mai simplu dar mai puțin puternic
4. **Fără Pipes** - folosești computed() sau helper functions
5. **Fără RxJS** - reactivitatea e built-in cu ref/computed/watch
6. **Fără FormModule** - v-model + librării externe
7. **Template syntax** - `:` și `@` în loc de `[]` și `()`

Ce e **aproape identic:**
- Component lifecycle
- Template interpolation
- Routing concepts
- Lazy loading
- State management patterns
- TypeScript integration

---

## 11. Întrebări de interviu

### Î1: Ce este Composition API și de ce l-ai folosi în loc de Options API?

**Răspuns:** Composition API este un set de funcții (ref, computed, watch, onMounted, etc.) care permit organizarea logicii componentelor pe funcționalitate, nu pe tip de opțiune. L-aș folosi în loc de Options API din trei motive principale: (1) **organizare mai bună a codului** - într-o componentă cu 500+ linii, logica pentru o funcționalitate e grupată, nu fragmentată între data/methods/computed/watch; (2) **reutilizare logică prin composables** - funcții pure care returnează stare reactivă, mult mai curate decât mixins care aveau probleme cu naming conflicts și sursa datelor neclară; (3) **suport TypeScript excelent** - inferența tipurilor funcționează natural cu funcții, spre deosebire de Options API unde `this` era problematic. Ca și trade-off, Composition API are o curbă de învățare mai abruptă - trebuie să înțelegi sistemul de reactivitate (ref, reactive, Proxy) pentru a-l folosi corect. Pentru componente simple, Options API rămâne perfect valid. Recomandarea mea pentru un proiect enterprise: Composition API cu `<script setup>` peste tot, pentru consistență și scalabilitate.

### Î2: Care e diferența între ref() și reactive()? Când folosești fiecare?

**Răspuns:** `ref()` creează un wrapper reactiv care funcționează cu orice tip (primitive și obiecte), accesat cu `.value` în script și automat unwrapped în template. `reactive()` creează un Proxy direct pe obiect, fără `.value`, dar funcționează doar cu obiecte. Recomandarea oficială și cea pe care o aplic eu este **ref() pentru tot**. Motivele: (1) consistență - mereu `.value`, fără confuzie; (2) poți reasigna complet valoarea (`ref.value = newObject`); (3) destructurarea nu pierde reactivitatea; (4) funcționează cu primitive. Reactive are gotchas serioase: destructurarea pierde reactivitatea (necesită `toRefs()`), nu poți reasigna obiectul întreg, și nu funcționează cu primitive. Singura situație unde reactive e ușor avantajos e un formular local simplu unde nu vrei să scrii `.value` peste tot - dar consistența codebase-ului e mai importantă. Venind din Angular, `ref()` e analogul lui `signal()` - un container reactiv universal.

### Î3: Cum funcționează sistemul de reactivitate Vue 3 sub capotă?

**Răspuns:** Vue 3 folosește **ES6 Proxy** pentru reactivitate (spre deosebire de Vue 2 care folosea `Object.defineProperty`). Când creezi un `reactive()`, Vue creează un Proxy care interceptează `get` și `set`. La `get`, Vue înregistrează dependența (track) - știe că acel computed sau effect depinde de proprietatea citită. La `set`, Vue notifică toți dependenții (trigger). Pentru `ref()`, mecanismul e similar dar wrapped într-un obiect cu proprietatea `.value`. `computed()` este un ref lazy - se calculează prima dată când e citit și se invalidează când dependențele se schimbă (dirty flag). Avantajul Proxy față de Object.defineProperty: (1) detectează adăugarea/ștergerea proprietăților (Vue 2 nu putea); (2) detectează modificări pe array-uri prin index; (3) performanță mai bună pentru obiecte mari (lazy - interceptează doar ce e accesat). Ca și architect, e important să știi: `shallowRef()`/`shallowReactive()` evită deep reactivity pentru obiecte mari, îmbunătățind performanța. Conceptual e similar cu Angular Signals, dar implementarea diferă: Angular folosește un graf de dependențe push-based, Vue folosește Proxy-based tracking.

### Î4: Cum ai migra o aplicație de la Options API la Composition API?

**Răspuns:** Migrarea ar fi **incrementală**, nu big-bang. Strategia mea: (1) Ambele API-uri coexistă în același proiect, deci nu e nevoie de migrare completă. (2) Începi cu **extragerea logicii reutilizabile** din mixins în composables - aceasta oferă cea mai mare valoare imediat. (3) Componentele noi se scriu exclusiv cu `<script setup>`. (4) Componentele existente se migrează la refactoring, nu proactiv. (5) Maparea este directă: `data()` → `ref()`, `computed` → `computed()`, `methods` → funcții, `watch` → `watch()`, lifecycle hooks → `onMounted()` etc. (6) Cel mai complicat: migrarea `this.$refs`, `this.$emit`, `this.$parent` - în Composition API folosești `useTemplateRef()`, `defineEmits()`, `provide/inject`. Ca trade-off, migrarea are cost de code review și potențiale regresi, dar beneficiile pe termen lung (TypeScript, testabilitate, maintainability) justifică investiția. Aș prioriza componentele cu mixins complexe sau cele care vor fi extinse frecvent.

### Î5: Explică lifecycle-ul unui component Vue.

**Răspuns:** Un component Vue trece prin: **Creare** → `<script setup>` se execută (echivalent constructor). **Montare** → `onBeforeMount()` (DOM nu e gata) → `onMounted()` (DOM disponibil - aici faci fetch-uri, setup listeners, interacțiuni DOM). **Actualizare** → `onBeforeUpdate()` → `onUpdated()` (ciclul se repetă la fiecare schimbare de stare care afectează template-ul). **Distrugere** → `onBeforeUnmount()` → `onUnmounted()` (cleanup - clearInterval, abort fetch, remove listeners). Suplimentar: `onErrorCaptured()` prinde erori din componente copil (error boundary), iar `onActivated()`/`onDeactivated()` se aplică componentelor cache-uite cu `<KeepAlive>`. Comparativ cu Angular: Vue are mai puține hook-uri. Nu ai `ngOnChanges` separat (folosești `watch()` pe props) și nu ai separare `AfterViewInit`/`AfterContentInit` (ambele sunt acoperite de `onMounted()`). Cele mai importante hook-uri zi de zi sunt: top-level setup code, `onMounted()` și `onUnmounted()`.

### Î6: Cum faci two-way binding în Vue 3?

**Răspuns:** Sunt trei niveluri. (1) **Pe elemente native**: `v-model="variabila"` care este syntax sugar pentru `:value="variabila" @input="variabila = $event.target.value"`. Suportă modifiers: `.lazy` (change în loc de input), `.number` (conversie automată), `.trim` (trim whitespace). (2) **Pe componente custom (pre-3.4)**: componenta copil declară prop `modelValue` și emit `update:modelValue`. Părintele folosește `v-model="val"`. Pentru v-model-uri multiple: `v-model:first-name="fn"` cu prop `firstName` și emit `update:firstName`. (3) **Cu defineModel() (Vue 3.4+)**: simplifică dramatic - copilul declară `const model = defineModel<string>()` care creează automat prop + emit. `model` e un Ref sincronizat cu v-model-ul părintelui. Aceasta e abordarea recomandată. Echivalentul Angular: `model()` signal input din Angular 17.1+. Ambele au ajuns la o soluție similară - o modalitate declarativă de two-way binding fără boilerplate.

### Î7: Care sunt avantajele `<script setup>` față de setup() clasic?

**Răspuns:** (1) **Zero boilerplate** - nu mai ai `export default { setup() { ... return {} } }`. (2) **Auto-expose** - tot ce declari la top-level e disponibil în template, fără return explicit. (3) **Compiler macros** - `defineProps`, `defineEmits`, `defineModel`, `defineExpose` sunt procesate la build time, zero runtime cost. (4) **Performanță mai bună** - compilatorul poate optimiza template bindings știind exact ce variabile există. (5) **TypeScript** - inferență mai bună deoarece totul e la top-level, nu într-un obiect nested. (6) **Components auto-registered** - importi o componentă și o folosești direct în template, fără secțiunea `components: {}`. Dezavantajul: nu poți folosi `<script setup>` dacă ai nevoie de `inheritAttrs: false` (rezolvat cu `defineOptions()` din Vue 3.3+), sau dacă ai nevoie de exports named (rar). Recomandarea mea: `<script setup>` pentru 99% din cazuri. E direcția clară a framework-ului.

### Î8: Cum organizezi codul în componente mari cu Composition API?

**Răspuns:** Abordarea mea pe niveluri: (1) **În componentă** - grupez codul pe funcționalități cu comentarii secționale. Exemplu: `// === SEARCH ===`, `// === PAGINATION ===`, `// === FILTERS ===`. (2) **Composables** - extrag logica reutilizabilă sau complexă în funcții separate (`useSearch()`, `usePagination()`). Convenția: prefix `use`, returnează ref-uri și funcții. (3) **Folder structure** - composables specifice componentei stau lângă componentă (`ComponentName/composables/useFeature.ts`), cele globale în `src/composables/`. (4) **Principiul Composition**: composables-urile pot compune alte composables. `useProductList()` poate folosi intern `useSearch()` + `usePagination()` + `useFilters()`. Ca trade-off, composables adaugă indirection - nu extrage prematur. Regula mea: dacă logica e folosită în 2+ locuri SAU componenta depășește 200 linii, extragi. Altfel, keep it simple. Venind din Angular, composables înlocuiesc Services dar sunt mai ușoare (nu necesită DI, nu sunt clase, nu au lifecycle propriu).

### Î9: Ce sunt composables și cum diferă de Angular services?

**Răspuns:** **Composables** sunt funcții care encapsulează stare reactivă și logică, returnând ref-uri și metode. Convenția: prefix `use` (useAuth, useApi, useDarkMode). Diferențele față de Angular Services: (1) **Nu sunt clase** - sunt funcții pure, fără constructor, fără decoratori. (2) **Nu folosesc DI** - sunt importate direct. Dacă ai nevoie de "singleton", folosești module-level state (starea declarată în afara funcției). (3) **Nu au lifecycle** - se "activează" când componenta le folosește și se "dezactivează" la unmount (dacă au cleanup). (4) **Fiecare apel creează instanță nouă** implicit (state local), dar poți avea state global folosind variabile la nivel de modul. (5) **Testare mai simplă** - sunt funcții pure, le apelezi direct în teste fără TestBed. Trade-off: Angular DI e mai puternic pentru scenarii complexe (ierarhie, override, testing), dar composables sunt mai simple și mai ușor de înțeles. Pentru state management centralizat, Vue folosește **Pinia** (nu composables simple), care e echivalentul unui NgRx Store sau Signal Store.

### Î10: Cum gestionezi formulare complexe în Vue? (vs Angular Reactive Forms)

**Răspuns:** Vue nu are un modul de formulare built-in precum Angular Reactive Forms. Abordările sunt: (1) **v-model + validare manuală** - pentru formulare simple. Fiecare câmp e un `ref()`, validarea e un `computed()` sau funcție. Funcționează, dar nu scalează. (2) **VeeValidate + Zod/Yup** - soluția enterprise. VeeValidate oferă `useForm()`, `useField()`, integrare cu schema validators (Zod, Yup). Cel mai aproape de Angular Reactive Forms ca developer experience. (3) **FormKit** - librărie all-in-one cu componente UI, validare, generare formulare din schema. Tradeoff: adaugă dependență externă, dar Angular Reactive Forms e tot "extern" (trebuie importat ReactiveFormsModule). Diferența majoră: Angular are un model de forms consistent și official. Vue lasă alegerea la developer. Ca architect, aș standardiza pe **VeeValidate + Zod** pentru type safety end-to-end: schema Zod definește tipul + regulile, VeeValidate conectează la UI, iar TypeScript verifică totul la compile time. E mai flexibil decât Angular Forms dar necesită mai multă configurare inițială.

### Î11: Explică diferența între v-if și v-show. Când folosești fiecare?

**Răspuns:** `v-if` adaugă/elimină elementul din DOM complet (plus lifecycle-ul componentelor copil se declanșează). `v-show` toggle-uiește doar `display: none` - elementul rămâne mereu în DOM. Reguli de decizie: folosește `v-show` pentru **toggle-uri frecvente** (modal visibility, tabs, accordion) deoarece costul e doar CSS. Folosește `v-if` pentru **condiții care se schimbă rar** (user logged in, feature flags, error states) sau când ai componente grele care nu trebuie renderizate dacă nu sunt vizibile. `v-if` suportă `v-else` și `v-else-if`, `v-show` nu. `v-if` poate fi pus pe `<template>`, `v-show` nu. Atenție: `v-if` cu `v-for` pe același element e un anti-pattern - `v-if` are prioritate mai mare și va fi evaluat pentru fiecare iterație. Folosește `<template v-for>` cu `v-if` nested. Angular nu are echivalent pentru `v-show` - cel mai aproape e `[hidden]` dar comportamentul diferă (CSS poate suprascrie `hidden`).

### Î12: Ce sunt compiler macros în Vue? Numește-le pe toate.

**Răspuns:** Compiler macros sunt funcții speciale procesate de compilatorul Vue la **build time**, nu la runtime. Nu trebuie importate - sunt disponibile global în `<script setup>`. Lista completă: (1) `defineProps()` - declarare props; (2) `defineEmits()` - declarare evenimente; (3) `defineModel()` - declarare v-model (Vue 3.4+); (4) `defineExpose()` - expunere publică pentru template refs; (5) `defineOptions()` - opțiuni componentă ca `inheritAttrs`, `name` (Vue 3.3+); (6) `defineSlots()` - typed slots (Vue 3.3+); (7) `withDefaults()` - valori default pentru defineProps type-based. Avantajul major: zero runtime overhead. Compilatorul le transformă în cod optimizat. Dezavantaj: nu pot fi folosite condițional sau într-un block scope - trebuie la top-level. Echivalent Angular: decoratorii (@Input, @Output, @Component) sunt procesați de compilatorul Angular la build time, similar conceptual, dar implementarea diferă fundamental (decoratori TypeScript vs compiler macros Vue).

### Î13: Cum funcționează `<KeepAlive>` și când l-ai folosi?

**Răspuns:** `<KeepAlive>` e un component built-in care **cache-uiește** instanțele componentelor în loc să le distrugă. Când un component wrapped în KeepAlive e "dezactivat" (ex: switch de tab), nu se apelează `onUnmounted()` ci `onDeactivated()`. Când revine, nu se recreează de la zero ci se apelează `onActivated()`. Use case-uri: (1) **Tab-based interfaces** - utilizatorul switch-uiește între tabs, fiecare tab își păstrează starea (scroll position, form data, etc.). (2) **Wizard/stepper** - pașii anteriori își păstrează starea. (3) **Route caching** - cu `<RouterView>` wrapped în `<KeepAlive>` pentru rute vizitate frecvent. Props importante: `include`/`exclude` (component names), `max` (limită cache LRU). Angular nu are echivalent built-in - trebuie implementat manual cu services sau route reuse strategy. Ca architect, e o funcționalitate puternică dar trebuie folosită judicios - cache-ul consumă memorie și poate cauza stale data dacă nu gestionezi corect `onActivated()`.

### Î14: Cum ai structura un proiect Vue mare cu Composition API?

**Răspuns:** Structura pe care o recomand: `src/components/` pentru componente UI reutilizabile (Button, Modal, Table), `src/composables/` pentru logică partajată globală (useAuth, useApi, useNotifications), `src/stores/` pentru Pinia stores, `src/views/` sau `src/pages/` pentru componente asociate rutelor, `src/types/` pentru TypeScript interfaces, `src/utils/` pentru funcții pure fără reactivitate. Fiecare feature complex poate avea structură proprie: `features/products/components/`, `features/products/composables/`, `features/products/stores/`. Principii: (1) **Colocate** - composable-ul specific unui component stă lângă component. (2) **Single responsibility** - un composable face un lucru. (3) **Barrel exports** - `index.ts` în fiecare folder. (4) **Naming conventions** - componente PascalCase, composables `use` prefix, stores `use...Store`. Ca trade-off față de Angular: Vue nu impune o structură (Angular CLI generează convenții). Trebuie să stabilești și documentezi convențiile echipei devreme.

### Î15: Ce sunt Teleport și Suspense? Când le folosești?

**Răspuns:** **Teleport** renderizează conținutul unui component în altă parte a DOM-ului, chiar dacă logic aparține componentei curente. Exemplu: un modal definit într-un component adânc nested se renderizează la `<body>` level. Sintaxă: `<Teleport to="body">...</Teleport>`. Util pentru: modals, tooltips, toasts, dropdown menus - orice care trebuie să "evadeze" din containment-ul CSS al părintelui (overflow, z-index). Echivalent Angular: `CdkPortal` din Angular CDK. **Suspense** e un component experimental care gestionează componente asincrone - afișează un fallback (loading) până când componentele async din interior sunt rezolvate. Funcționează cu `async setup()` și `defineAsyncComponent()`. Util pentru: loading states coordonate, SSR. Ca architect, Teleport e production-ready și extrem de util. Suspense e încă experimental - pentru loading states în producție, recomand pattern-ul clasic cu `v-if="isLoading"` sau librării dedicate (VueQuery/TanStack Query).

### Î16: Cum gestionezi erorile în componentele Vue?

**Răspuns:** Trei niveluri: (1) **Component level** - `onErrorCaptured()` prinde erori din componente copil (propagation, poate fi oprit cu `return false`). Util pentru error boundaries - un component wrapper care afișează fallback UI la eroare. (2) **App level** - `app.config.errorHandler` prinde toate erorile netratate la nivel global. Aici integrez logging services (Sentry, DataDog). (3) **Async errors** - try/catch în `onMounted()` și composables. `watchEffect()` și `watch()` au onCleanup pentru abort. Un pattern pe care îl recomand: composable `useAsyncState()` care gestionează loading/error/data states consistent. Comparativ cu Angular, unde ai `ErrorHandler` global și HTTP interceptors, Vue necesită mai multă disciplină manuală, dar `onErrorCaptured()` pe componentă e mai granular decât Angular ErrorHandler. Pinia stores ar trebui să aibă error handling dedicat în actions, cu pattern-ul Result type (success/error) pentru predictibilitate.


---

**Următor :** [**02 - Reactivitate Vue vs Angular** →](Vue/02-Reactivitate-Vue-vs-Angular.md)