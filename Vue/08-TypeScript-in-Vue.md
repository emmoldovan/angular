# TypeScript în Vue 3 (Interview Prep - Senior Frontend Architect)

> TypeScript integration în Vue 3: script setup lang="ts", defineProps<T>(),
> defineEmits<T>(), generic components, typed composables, PropType.
> Comparație detaliată cu Angular TypeScript patterns.
> Pregătit pentru candidați cu experiență solidă în Angular/TypeScript.

---

## Cuprins

1. [Setup TypeScript în Vue 3](#1-setup-typescript-în-vue-3)
2. [`<script setup lang="ts">` - Basics](#2-script-setup-langts---basics)
3. [defineProps cu TypeScript](#3-defineprops-cu-typescript)
4. [defineEmits cu TypeScript](#4-defineemits-cu-typescript)
5. [Generic Components](#5-generic-components)
6. [defineSlots și defineModel cu TypeScript](#6-defineslots-și-definemodel-cu-typescript)
7. [Typed Composables](#7-typed-composables)
8. [Typing provide/inject](#8-typing-provideinject)
9. [Typing Pinia Stores](#9-typing-pinia-stores)
10. [Typing Template Refs](#10-typing-template-refs)
11. [Utility Types din Vue](#11-utility-types-din-vue)
12. [Paralela: TypeScript în Angular vs Vue](#12-paralela-typescript-în-angular-vs-vue)
13. [Întrebări de interviu](#13-întrebări-de-interviu)

---

## 1. Setup TypeScript în Vue 3

### 1.1 De ce TypeScript în Vue?

**Vue 3 este scris integral în TypeScript.** Asta înseamnă că type definitions sunt first-class citizens, nu un afterthought ca la Vue 2. Suportul TypeScript în Vue 3 este nativ și profund integrat în framework.

**Avantaje TypeScript în Vue:**
- **Type safety** la compile time - erori prinse înainte de runtime
- **Autocompletion** în IDE - productivitate crescută
- **Refactoring sigur** - rename, move, extract funcționează corect
- **Documentație vie** - interfețele servesc drept contract clar
- **Generic components** - reusability la alt nivel (Vue 3.3+)

### 1.2 Configurare proiect nou

Cel mai simplu mod de a crea un proiect Vue cu TypeScript:

```bash
# Vite (recomandat)
npm create vue@latest
# Selectează: TypeScript → Yes

# Sau direct cu Vite template
npm create vite@latest my-app -- --template vue-ts
```

### 1.3 tsconfig.json - Configurare completă

```json
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "jsx": "preserve",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "esModuleInterop": true,
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    },
    "types": ["vite/client"],
    "skipLibCheck": true
  },
  "include": [
    "src/**/*.ts",
    "src/**/*.d.ts",
    "src/**/*.vue"
  ],
  "references": [
    { "path": "./tsconfig.node.json" }
  ]
}
```

**Opțiuni critice:**
- **`strict: true`** - activează toate verificările stricte (strictNullChecks, noImplicitAny, etc.)
- **`moduleResolution: "bundler"`** - necesar pentru Vite (înlocuiește `"node"`)
- **`isolatedModules: true`** - necesar deoarece Vite folosește esbuild per-file transpilation
- **`types: ["vite/client"]`** - permite import-uri de assets (`.svg`, `.png`, etc.)

### 1.4 tsconfig.node.json

```json
// tsconfig.node.json - pentru fișierele de configurare (vite.config.ts, etc.)
{
  "compilerOptions": {
    "composite": true,
    "module": "ESNext",
    "moduleResolution": "bundler",
    "allowSyntheticDefaultImports": true
  },
  "include": ["vite.config.ts"]
}
```

### 1.5 Type checking cu vue-tsc

**Vite nu face type checking** la build - doar transpilă. Trebuie `vue-tsc` separat:

```bash
# Instalare
npm install -D vue-tsc typescript

# Type check (CI/CD)
npx vue-tsc --noEmit

# Type check watch mode (development)
npx vue-tsc --noEmit --watch
```

```json
// package.json
{
  "scripts": {
    "dev": "vite",
    "build": "vue-tsc --noEmit && vite build",
    "type-check": "vue-tsc --noEmit"
  }
}
```

### 1.6 Volar - IDE Extension

**Volar** (acum Vue - Official extension) înlocuiește **Vetur** pentru Vue 3:
- Type checking în `.vue` files direct în editor
- Autocompletion pentru props, emits, slots
- Go to definition funcționează cross-file
- Template type checking (verifică binding-urile din template)

### 1.7 Declarații de tip pentru .vue files

```typescript
// src/env.d.ts sau src/vite-env.d.ts
/// <reference types="vite/client" />

declare module '*.vue' {
  import type { DefineComponent } from 'vue'
  const component: DefineComponent<{}, {}, any>
  export default component
}
```

### Paralela cu Angular

| Aspect | Angular | Vue 3 |
|--------|---------|-------|
| TypeScript | **Obligatoriu** | **Opțional** (dar recomandat) |
| Config file | `tsconfig.json` + `tsconfig.app.json` | `tsconfig.json` + `tsconfig.node.json` |
| Type checking | `ngc` (Angular Compiler) | `vue-tsc` |
| IDE support | Built-in Angular Language Service | Volar extension |
| Template checking | `strictTemplates: true` | `vue-tsc` + Volar |

> **Key insight:** Angular **impune** TypeScript - nu ai opțiunea să nu-l folosești. Vue **permite** JavaScript pur, dar în proiecte enterprise, TypeScript este essential. Diferența e filosofică: Angular e opinionated, Vue e progressive.

---

## 2. `<script setup lang="ts">` - Basics

### 2.1 Sintaxa de bază

`<script setup>` este **syntax sugar** pentru Composition API cu beneficii TypeScript automate:

```vue
<script setup lang="ts">
import { ref, computed, reactive } from 'vue'

// ==========================================
// Type inference funcționează automat
// ==========================================

const count = ref(0)              // Ref<number>
const name = ref('Emanuel')       // Ref<string>
const isActive = ref(true)        // Ref<boolean>
const doubled = computed(() => count.value * 2)  // ComputedRef<number>

// ==========================================
// Explicit typing când inference nu e suficient
// ==========================================

const items = ref<string[]>([])           // Ref<string[]> (nu Ref<never[]>)
const user = ref<User | null>(null)       // Ref<User | null> (nu Ref<null>)
const data = ref<Record<string, number>>({}) // Ref<Record<string, number>>

// ==========================================
// Reactive - tipează obiectul direct
// ==========================================

interface FormState {
  username: string
  email: string
  age: number | null
  preferences: string[]
}

const form = reactive<FormState>({
  username: '',
  email: '',
  age: null,
  preferences: []
})

// form.username → string (autocompletion funcționează)
// form.unknownProp → TypeScript error!
</script>
```

### 2.2 Când e necesar explicit typing

```typescript
// ❌ Fără explicit type - inference greșit
const items = ref([])
// Tip inferat: Ref<never[]> - nu poți adăuga nimic!

// ✅ Cu explicit type
const items = ref<string[]>([])
// Acum items.value.push('test') funcționează

// ❌ Fără explicit type
const selected = ref(null)
// Tip inferat: Ref<null> - mereu null!

// ✅ Cu explicit type
const selected = ref<User | null>(null)
// Acum selected.value = someUser funcționează
```

**Regula de bază:** Folosești explicit typing când valoarea inițială nu reflectă toate tipurile posibile (arrays goale, null, undefined).

### 2.3 Typing funcții în script setup

```vue
<script setup lang="ts">
// Funcții cu tipuri explicite
function handleSubmit(event: Event): void {
  event.preventDefault()
  // ...
}

// Arrow functions tipizate
const formatPrice = (amount: number, currency: string = 'RON'): string => {
  return `${amount.toFixed(2)} ${currency}`
}

// Async functions
async function fetchUser(id: number): Promise<User> {
  const response = await fetch(`/api/users/${id}`)
  if (!response.ok) throw new Error('User not found')
  return response.json()
}

// Event handlers tipizați
function handleInput(event: Event): void {
  const target = event.target as HTMLInputElement
  console.log(target.value)
}

function handleClick(event: MouseEvent): void {
  console.log(event.clientX, event.clientY)
}
</script>
```

### 2.4 Type imports

```vue
<script setup lang="ts">
// Folosește `import type` pentru tipuri
import type { Ref, ComputedRef } from 'vue'
import type { User, Credentials } from '@/types'
import type { RouteLocationNormalized } from 'vue-router'

// `import type` este eliminat la compile time
// Nu ajunge în bundle-ul final
</script>
```

### Paralela cu Angular

```typescript
// Angular - tipuri pe class properties
@Component({ ... })
export class UserComponent {
  count: number = 0                    // type pe property
  name: string = 'Emanuel'
  items: string[] = []
  user: User | null = null
}

// Vue 3 - tipuri pe ref/reactive
const count = ref(0)                   // type inferat sau explicit
const name = ref('Emanuel')
const items = ref<string[]>([])
const user = ref<User | null>(null)
```

> **Key insight:** În Angular tipurile sunt pe proprietățile clasei. În Vue, tipurile sunt pe ref/reactive. Rezultatul final e similar, dar Vue oferă mai multă flexibilitate cu composition functions.

---

## 3. defineProps cu TypeScript

### 3.1 Două moduri de declarare

Vue oferă **două** moduri de a declara props:

```vue
<!-- 1. Runtime declaration (fără TypeScript) -->
<script setup>
const props = defineProps({
  title: { type: String, required: true },
  count: { type: Number, default: 0 }
})
</script>

<!-- 2. Type-based declaration (RECOMANDAT cu TypeScript) -->
<script setup lang="ts">
const props = defineProps<{
  title: string
  count?: number
}>()
</script>
```

**Nu poți combina cele două moduri.** Type-based este recomandat deoarece oferă tipuri mai precise și mai ușor de citit.

### 3.2 Type-based declaration - Detalii

```vue
<script setup lang="ts">
interface Props {
  // Required props
  title: string
  items: string[]
  user: User

  // Optional props (cu ?)
  count?: number
  variant?: 'primary' | 'secondary' | 'danger'
  disabled?: boolean

  // Complex types
  config?: Record<string, unknown>
  onUpdate?: (value: string) => void
  renderItem?: (item: string, index: number) => string
}

const props = defineProps<Props>()

// Type checking funcționează automat:
props.title       // → string
props.count       // → number | undefined
props.variant     // → 'primary' | 'secondary' | 'danger' | undefined
props.user        // → User
props.items       // → string[]
</script>
```

### 3.3 withDefaults - Valori implicite

Problema: Cu type-based declaration, nu poți specifica defaults direct. Soluția: `withDefaults()`.

```vue
<script setup lang="ts">
interface Props {
  title: string
  count?: number
  items?: string[]
  variant?: 'primary' | 'secondary' | 'danger'
  config?: Record<string, boolean>
}

const props = withDefaults(defineProps<Props>(), {
  count: 0,
  items: () => [],                    // Factory function pentru arrays/objects!
  variant: 'primary',
  config: () => ({ visible: true })   // Factory function pentru objects!
})

// Acum tipurile reflectă defaults:
props.count    // → number (nu mai e number | undefined!)
props.items    // → string[] (nu mai e string[] | undefined!)
props.variant  // → 'primary' | 'secondary' | 'danger' (nu mai e undefined!)
```

**Important:** Pentru **non-primitive** defaults (arrays, objects), trebuie să folosești **factory functions** (`() => []`), altfel toate instanțele componentei ar partaja aceeași referință.

### 3.4 Vue 3.5+ Reactive Props Destructure

Începând cu Vue 3.5, poți destructura props **cu reactivitate păstrată** și defaults inline:

```vue
<script setup lang="ts">
interface Props {
  title: string
  count?: number
  items?: string[]
  variant?: 'primary' | 'secondary' | 'danger'
}

// Vue 3.5+ - destructure cu defaults
const {
  title,
  count = 0,
  items = [],
  variant = 'primary'
} = defineProps<Props>()

// ✅ Reactive - funcționează în template și watch
// ✅ Defaults aplicate - nu mai e nevoie de withDefaults
// ✅ Sintaxă JavaScript standard
</script>

<template>
  <!-- title, count, items, variant sunt reactive -->
  <h1>{{ title }}</h1>
  <span>Count: {{ count }}</span>
</template>
```

**Atenție:** În versiuni anterioare Vue 3.5, destructurarea props pierdea reactivitatea. Acum este sigur.

### 3.5 Complex types în props

```vue
<script setup lang="ts">
// Enum-like types cu union literals
type ButtonVariant = 'primary' | 'secondary' | 'danger' | 'ghost'
type Size = 'sm' | 'md' | 'lg' | 'xl'

// Generic-like interface pentru tabele
interface Column<T = any> {
  key: keyof T
  label: string
  sortable?: boolean
  width?: string | number
  align?: 'left' | 'center' | 'right'
  render?: (value: T[keyof T], row: T) => string
}

interface TableProps {
  data: Record<string, any>[]
  columns: Column[]
  loading?: boolean
  selectable?: boolean
  emptyMessage?: string
  rowKey?: string
  striped?: boolean
  bordered?: boolean
}

const props = withDefaults(defineProps<TableProps>(), {
  loading: false,
  selectable: false,
  emptyMessage: 'Nu există date',
  rowKey: 'id',
  striped: false,
  bordered: true
})
</script>
```

### 3.6 Props cu tipuri importate

```vue
<script setup lang="ts">
import type { User, Role, Permission } from '@/types/auth'
import type { PaginationConfig } from '@/types/common'

interface Props {
  user: User
  roles: Role[]
  permissions: Permission[]
  pagination?: PaginationConfig
}

const props = defineProps<Props>()
</script>
```

**Limitare importantă:** Type-based defineProps acceptă **doar** tipuri definite în același fișier sau **importate cu `import type`**. Nu acceptă tipuri complexe din generics externe resolve-uite dinamic.

### Paralela cu Angular

```typescript
// Angular - @Input() pe fiecare proprietate individual
@Component({ ... })
export class UserCardComponent {
  @Input() title!: string
  @Input() count: number = 0
  @Input() items: string[] = []
  @Input() variant: 'primary' | 'secondary' = 'primary'
  @Input({ required: true }) user!: User  // Angular 16+
  @Input({ transform: booleanAttribute }) disabled = false  // Angular 16+
}

// Vue - un singur interface pentru toate props
interface Props {
  title: string
  count?: number
  items?: string[]
  variant?: 'primary' | 'secondary'
  user: User
  disabled?: boolean
}
const props = withDefaults(defineProps<Props>(), {
  count: 0,
  items: () => [],
  variant: 'primary',
  disabled: false
})
```

| Aspect | Angular @Input | Vue defineProps<T> |
|--------|---------------|-------------------|
| Declarare | Pe fiecare proprietate | Un singur interface |
| Required | `!` assertion sau `required: true` | Fără `?` = required |
| Defaults | Valoare directă pe proprietate | `withDefaults()` sau destructure |
| Validare runtime | Nu nativ (trebuie setter) | Runtime validators (mod runtime) |
| Compile-time check | Da (strict templates) | Da (vue-tsc + Volar) |
| Transform | `@Input({ transform })` (v16+) | Manual în computed |

> **Key insight:** Vue defineProps<T> este mai declarativ - un interface descrie toate props-urile. Angular @Input e mai granular - fiecare prop e declarat individual. Abordarea Vue duce la cod mai concis și un contract mai clar.

---

## 4. defineEmits cu TypeScript

### 4.1 Type-based declaration

```vue
<script setup lang="ts">
// Varianta 1: Tuple syntax (Vue 3.3+ - RECOMANDAT)
const emit = defineEmits<{
  change: [value: string]
  update: [id: number, value: string]
  delete: [id: number]
  submit: []                              // Fără argumente
  'update:modelValue': [value: string]    // v-model event
}>()

// Varianta 2: Call signature syntax (Vue 3.0+)
const emit = defineEmits<{
  (e: 'change', value: string): void
  (e: 'update', id: number, value: string): void
  (e: 'delete', id: number): void
  (e: 'submit'): void
  (e: 'update:modelValue', value: string): void
}>()
</script>
```

**Tuple syntax** este preferată deoarece e mai concisă și mai ușor de citit.

### 4.2 Emitere tipizată - Type safety

```vue
<script setup lang="ts">
const emit = defineEmits<{
  change: [value: string]
  update: [id: number, value: string]
  select: [item: User]
}>()

// ✅ Corect - tipuri potrivite
emit('change', 'new value')
emit('update', 42, 'new value')
emit('select', { id: 1, name: 'Emanuel' } as User)

// ❌ TypeScript errors:
emit('change', 123)              // Argument of type 'number' is not assignable to 'string'
emit('update', 'id', 'value')    // Argument of type 'string' is not assignable to 'number'
emit('unknown')                  // Argument '"unknown"' is not assignable
emit('change')                   // Expected 2 arguments, but got 1
</script>
```

### 4.3 Complex emit types

```vue
<script setup lang="ts">
interface PaginationEvent {
  page: number
  pageSize: number
  total: number
}

interface SortEvent {
  column: string
  direction: 'asc' | 'desc'
}

interface FilterEvent {
  field: string
  operator: 'eq' | 'neq' | 'gt' | 'lt' | 'contains'
  value: string | number | boolean
}

const emit = defineEmits<{
  paginate: [event: PaginationEvent]
  sort: [event: SortEvent]
  filter: [events: FilterEvent[]]
  'row-click': [row: Record<string, any>, index: number]
  'row-select': [selectedRows: Record<string, any>[]]
}>()

function handlePageChange(page: number) {
  emit('paginate', {
    page,
    pageSize: 10,
    total: 100
  })
}
</script>
```

### 4.4 Emit cu validare

```vue
<script setup lang="ts">
// Runtime validation (mod runtime, nu type-based)
const emit = defineEmits({
  change: (value: string) => {
    if (value.length === 0) {
      console.warn('Empty value emitted!')
      return false  // Validare eșuată
    }
    return true     // Validare reușită
  }
})
</script>
```

**Notă:** Nu poți combina runtime validation cu type-based declaration. Alege una.

### Paralela cu Angular

```typescript
// Angular - @Output() cu EventEmitter<T>
@Component({ ... })
export class DataTableComponent {
  @Output() change = new EventEmitter<string>()
  @Output() update = new EventEmitter<{ id: number; value: string }>()
  @Output() delete = new EventEmitter<number>()
  @Output() paginate = new EventEmitter<PaginationEvent>()

  onPageChange(page: number) {
    this.paginate.emit({ page, pageSize: 10, total: 100 })
  }
}

// Vue - defineEmits<T>()
const emit = defineEmits<{
  change: [value: string]
  update: [id: number, value: string]
  delete: [id: number]
  paginate: [event: PaginationEvent]
}>()
```

| Aspect | Angular @Output | Vue defineEmits<T> |
|--------|----------------|-------------------|
| Declarare | EventEmitter<T> per output | Un singur interface |
| Type safety pe event name | Da (numele proprietății) | Da (string literal type) |
| Type safety pe payload | Un singur tip generic | Tuple cu multiple args tipizate |
| Multiple argumente | Trebuie wrappuite într-un obiect | Suport nativ cu tuples |
| Validare runtime | Manual | Built-in (mod runtime) |

> **Key insight:** Vue defineEmits permite **multiple argumente tipizate** per event (tuples), pe când Angular EventEmitter acceptă **un singur** tip generic. În Angular trebuie să creezi un obiect wrapper dacă vrei multiple argumente.

---

## 5. Generic Components

### 5.1 Ce sunt Generic Components?

**Vue 3.3+** introduce atributul `generic` pe `<script setup>`, permițând componente parametrizate cu tipuri generice. Aceasta este una dintre funcționalitățile unde Vue e **mai avansat** decât Angular.

### 5.2 Sintaxa de bază

```vue
<!-- GenericList.vue -->
<script setup lang="ts" generic="T">
defineProps<{
  items: T[]
  selected?: T
}>()

defineEmits<{
  select: [item: T]
}>()
</script>

<template>
  <ul>
    <li v-for="(item, index) in items" :key="index"
        @click="$emit('select', item)">
      <slot :item="item" :index="index" />
    </li>
  </ul>
</template>
```

### 5.3 Generic cu constraints

```vue
<!-- GenericList.vue cu constraint -->
<script setup lang="ts" generic="T extends { id: number; label: string }">
const props = defineProps<{
  items: T[]
  selected?: T
  filterFn?: (item: T) => boolean
}>()

const emit = defineEmits<{
  select: [item: T]
  remove: [item: T]
}>()

defineSlots<{
  default: (props: { item: T; index: number }) => any
  empty: () => any
}>()

const filteredItems = computed(() => {
  if (!props.filterFn) return props.items
  return props.items.filter(props.filterFn)
})
</script>

<template>
  <ul v-if="filteredItems.length > 0">
    <li v-for="(item, index) in filteredItems" :key="item.id"
        :class="{ selected: selected?.id === item.id }"
        @click="emit('select', item)">
      <slot :item="item" :index="index">
        {{ item.label }}
      </slot>
    </li>
  </ul>
  <div v-else>
    <slot name="empty">
      <p>Nu există elemente.</p>
    </slot>
  </div>
</template>
```

### 5.4 Multiple generic types

```vue
<!-- GenericSelect.vue -->
<script setup lang="ts" generic="TValue, TOption extends { id: TValue; label: string }">
const props = defineProps<{
  options: TOption[]
  modelValue: TValue | null
  placeholder?: string
  disabled?: boolean
}>()

const emit = defineEmits<{
  'update:modelValue': [value: TValue]
  change: [option: TOption]
}>()

function selectOption(option: TOption) {
  emit('update:modelValue', option.id)
  emit('change', option)
}
</script>

<template>
  <select
    :value="modelValue"
    :disabled="disabled"
    @change="selectOption(options.find(o => String(o.id) === ($event.target as HTMLSelectElement).value)!)"
  >
    <option v-if="placeholder" value="" disabled>{{ placeholder }}</option>
    <option v-for="option in options" :key="String(option.id)" :value="option.id">
      {{ option.label }}
    </option>
  </select>
</template>
```

### 5.5 Usage - Type inference

```vue
<script setup lang="ts">
interface User {
  id: number
  label: string
  name: string
  email: string
  role: 'admin' | 'user'
}

const users = ref<User[]>([
  { id: 1, label: 'Emanuel M.', name: 'Emanuel', email: 'e@test.com', role: 'admin' },
  { id: 2, label: 'Ion P.', name: 'Ion', email: 'i@test.com', role: 'user' }
])

const selectedUser = ref<User | undefined>()

// T este inferat automat ca User din contextul items
function handleSelect(user: User) {
  selectedUser.value = user
}
</script>

<template>
  <!-- T = User (inferat din :items="users") -->
  <GenericList
    :items="users"
    :selected="selectedUser"
    @select="handleSelect"
  >
    <template #default="{ item }">
      <!-- item este tipizat ca User! -->
      <div class="user-card">
        <strong>{{ item.name }}</strong>
        <span>{{ item.email }}</span>
        <badge>{{ item.role }}</badge>
      </div>
    </template>
  </GenericList>
</template>
```

### 5.6 Generic Table Component - Exemplu complet

```vue
<!-- DataTable.vue -->
<script setup lang="ts" generic="T extends Record<string, any>">
interface Column<R> {
  key: keyof R & string
  label: string
  sortable?: boolean
  width?: string
  align?: 'left' | 'center' | 'right'
  formatter?: (value: R[keyof R], row: R) => string
}

const props = withDefaults(defineProps<{
  data: T[]
  columns: Column<T>[]
  loading?: boolean
  rowKey?: keyof T & string
  striped?: boolean
  emptyMessage?: string
}>(), {
  loading: false,
  rowKey: 'id' as any,
  striped: true,
  emptyMessage: 'Nu există date de afișat.'
})

const emit = defineEmits<{
  'row-click': [row: T, index: number]
  'sort': [column: keyof T & string, direction: 'asc' | 'desc']
}>()

defineSlots<{
  header: () => any
  cell: (props: { row: T; column: Column<T>; value: any }) => any
  empty: () => any
  loading: () => any
}>()

const sortColumn = ref<string>('')
const sortDirection = ref<'asc' | 'desc'>('asc')

function handleSort(column: Column<T>) {
  if (!column.sortable) return
  if (sortColumn.value === column.key) {
    sortDirection.value = sortDirection.value === 'asc' ? 'desc' : 'asc'
  } else {
    sortColumn.value = column.key
    sortDirection.value = 'asc'
  }
  emit('sort', column.key, sortDirection.value)
}

function getCellValue(row: T, column: Column<T>): string {
  const value = row[column.key]
  return column.formatter ? column.formatter(value, row) : String(value ?? '')
}
</script>

<template>
  <div class="data-table">
    <slot name="header" />
    <table>
      <thead>
        <tr>
          <th v-for="col in columns" :key="col.key"
              :style="{ width: col.width, textAlign: col.align }"
              :class="{ sortable: col.sortable }"
              @click="handleSort(col)">
            {{ col.label }}
          </th>
        </tr>
      </thead>
      <tbody v-if="!loading && data.length > 0">
        <tr v-for="(row, index) in data" :key="String(row[rowKey])"
            :class="{ striped: striped && index % 2 === 1 }"
            @click="emit('row-click', row, index)">
          <td v-for="col in columns" :key="col.key" :style="{ textAlign: col.align }">
            <slot name="cell" :row="row" :column="col" :value="row[col.key]">
              {{ getCellValue(row, col) }}
            </slot>
          </td>
        </tr>
      </tbody>
    </table>
    <div v-if="loading">
      <slot name="loading"><p>Se încarcă...</p></slot>
    </div>
    <div v-else-if="data.length === 0">
      <slot name="empty"><p>{{ emptyMessage }}</p></slot>
    </div>
  </div>
</template>
```

### Paralela cu Angular

Angular **nu are** generic components la nivel de template. Cel mai apropiat lucru este:

```typescript
// Angular - nu poți parametriza un component cu <T>
@Component({ ... })
export class DataTableComponent {
  @Input() data: any[] = []
  @Input() columns: Column<any>[] = []
  // Pierdere type safety la nivel template!
}

// Poți folosi directive generice (Angular 14+):
@Directive({ ... })
export class TypedDirective<T> {
  // Dar e mai limitat decât Vue generic components
}
```

> **Key insight:** Generic components sunt un **avantaj major** al Vue față de Angular. Permit type safety end-to-end: de la prop definition, prin emits, până la slot props. Angular nu oferă echivalent direct pentru acest pattern.

---

## 6. defineSlots și defineModel cu TypeScript

### 6.1 defineSlots (Vue 3.3+)

`defineSlots` permite tipizarea slot-urilor, oferind type safety atât în componentă cât și la utilizare:

```vue
<!-- Card.vue -->
<script setup lang="ts">
defineSlots<{
  // Slot default - primește props
  default: (props: { message: string; count: number }) => any

  // Named slot - fără props
  header: () => any

  // Named slot - cu props
  footer: (props: { timestamp: Date }) => any

  // Optional slot
  sidebar?: (props: { collapsed: boolean }) => any
}>()
</script>

<template>
  <div class="card">
    <header v-if="$slots.header">
      <slot name="header" />
    </header>

    <main>
      <slot message="Hello" :count="42" />
    </main>

    <aside v-if="$slots.sidebar">
      <slot name="sidebar" :collapsed="false" />
    </aside>

    <footer>
      <slot name="footer" :timestamp="new Date()" />
    </footer>
  </div>
</template>
```

**Usage cu type checking:**

```vue
<template>
  <Card>
    <!-- ✅ props tipizate: message: string, count: number -->
    <template #default="{ message, count }">
      <p>{{ message }} - {{ count }}</p>
    </template>

    <template #header>
      <h2>Titlu Card</h2>
    </template>

    <template #footer="{ timestamp }">
      <!-- timestamp este Date -->
      <span>{{ timestamp.toLocaleDateString() }}</span>
    </template>
  </Card>
</template>
```

### 6.2 defineModel (Vue 3.4+)

`defineModel` simplifică drastic implementarea `v-model` cu TypeScript:

```vue
<!-- TextInput.vue -->
<script setup lang="ts">
// v-model implicit (modelValue)
const modelValue = defineModel<string>()
// Tipul: Ref<string | undefined>
// Generează automat prop: modelValue + emit: update:modelValue

// v-model cu required
const modelValue = defineModel<string>({ required: true })
// Tipul: Ref<string>

// v-model cu default
const modelValue = defineModel<string>({ default: '' })
// Tipul: Ref<string>
</script>

<template>
  <input :value="modelValue" @input="modelValue = ($event.target as HTMLInputElement).value" />
</template>
```

### 6.3 Multiple v-model cu defineModel

```vue
<!-- UserForm.vue -->
<script setup lang="ts">
// Multiple v-model bindings
const firstName = defineModel<string>('firstName', { required: true })
const lastName = defineModel<string>('lastName', { required: true })
const email = defineModel<string>('email', { default: '' })
const age = defineModel<number>('age')
const active = defineModel<boolean>('active', { default: true })
</script>

<template>
  <form>
    <input v-model="firstName" placeholder="Prenume" />
    <input v-model="lastName" placeholder="Nume" />
    <input v-model="email" type="email" placeholder="Email" />
    <input v-model.number="age" type="number" placeholder="Vârstă" />
    <label>
      <input v-model="active" type="checkbox" />
      Activ
    </label>
  </form>
</template>
```

**Usage:**

```vue
<template>
  <UserForm
    v-model:first-name="user.firstName"
    v-model:last-name="user.lastName"
    v-model:email="user.email"
    v-model:age="user.age"
    v-model:active="user.active"
  />
</template>
```

### 6.4 defineModel cu modifiers

```vue
<script setup lang="ts">
const [modelValue, modifiers] = defineModel<string>({
  set(value) {
    // Transform la set
    if (modifiers.capitalize) {
      return value.charAt(0).toUpperCase() + value.slice(1)
    }
    if (modifiers.uppercase) {
      return value.toUpperCase()
    }
    return value
  }
})
</script>

<!-- Usage: <MyInput v-model.capitalize="text" /> -->
```

### Paralela cu Angular

```typescript
// Angular - two-way binding necesită input + output manual
@Component({ ... })
export class UserFormComponent {
  // Angular <= 16: manual pattern
  @Input() firstName: string = ''
  @Output() firstNameChange = new EventEmitter<string>()

  @Input() lastName: string = ''
  @Output() lastNameChange = new EventEmitter<string>()

  // Angular 17.1+: model() signal
  firstName = model.required<string>()
  lastName = model.required<string>()
  email = model<string>('')
}
```

> **Key insight:** `defineModel` în Vue 3.4+ este foarte similar cu `model()` din Angular 17.1+. Ambele framework-uri au evoluat spre simplificarea two-way binding-ului. Vue a avut această funcționalitate ceva mai devreme.

---

## 7. Typed Composables

### 7.1 De ce contează tipizarea composables?

Composable-urile sunt **funcții reutilizabile** care încapsulează logică reactivă. TypeScript le face:
- **Autodocumentate** - interfața definește contractul
- **Sigure** - consumatorii nu pot folosi date greșit
- **Ușor de testat** - return type explicit

### 7.2 Pattern standard - Typed composable

```typescript
// composables/useApi.ts
import { ref, unref, onMounted } from 'vue'
import type { Ref } from 'vue'

// ==========================================
// Types - definite explicit
// ==========================================

interface UseApiOptions<T> {
  immediate?: boolean
  initialData?: T
  transform?: (raw: unknown) => T
  onError?: (error: Error) => void
  onSuccess?: (data: T) => void
  retryCount?: number
  retryDelay?: number
}

interface UseApiReturn<T> {
  data: Ref<T | null>
  loading: Ref<boolean>
  error: Ref<Error | null>
  execute: () => Promise<void>
  reset: () => void
}

// ==========================================
// Implementation
// ==========================================

export function useApi<T>(
  url: string | Ref<string>,
  options: UseApiOptions<T> = {}
): UseApiReturn<T> {
  const {
    immediate = true,
    initialData = null,
    transform,
    onError,
    onSuccess,
    retryCount = 0,
    retryDelay = 1000
  } = options

  const data = ref<T | null>(initialData) as Ref<T | null>
  const loading = ref(false)
  const error = ref<Error | null>(null)

  async function execute(): Promise<void> {
    loading.value = true
    error.value = null

    let attempts = 0
    const maxAttempts = retryCount + 1

    while (attempts < maxAttempts) {
      try {
        const response = await fetch(unref(url))
        if (!response.ok) {
          throw new Error(`HTTP ${response.status}: ${response.statusText}`)
        }
        const raw = await response.json()
        data.value = transform ? transform(raw) : (raw as T)
        onSuccess?.(data.value!)
        return
      } catch (e) {
        attempts++
        if (attempts >= maxAttempts) {
          const err = e instanceof Error ? e : new Error(String(e))
          error.value = err
          onError?.(err)
        } else {
          await new Promise(r => setTimeout(r, retryDelay))
        }
      } finally {
        if (attempts >= maxAttempts || !error.value) {
          loading.value = false
        }
      }
    }
  }

  function reset(): void {
    data.value = initialData
    loading.value = false
    error.value = null
  }

  if (immediate) {
    onMounted(execute)
  }

  return { data, loading, error, execute, reset }
}
```

**Usage:**

```vue
<script setup lang="ts">
interface User {
  id: number
  name: string
  email: string
}

// Fully typed - data este Ref<User[] | null>
const { data: users, loading, error, execute: refresh } = useApi<User[]>(
  '/api/users',
  {
    transform: (raw) => (raw as any).data as User[],
    onError: (err) => console.error('Failed to load users:', err),
    retryCount: 2
  }
)
</script>
```

### 7.3 Composable cu overloads

```typescript
// composables/usePagination.ts
interface PaginationOptions {
  initialPage?: number
  initialPageSize?: number
}

interface PaginationReturn {
  page: Ref<number>
  pageSize: Ref<number>
  offset: ComputedRef<number>
  totalPages: ComputedRef<number>
  hasNextPage: ComputedRef<boolean>
  hasPrevPage: ComputedRef<boolean>
  nextPage: () => void
  prevPage: () => void
  goToPage: (page: number) => void
  setPageSize: (size: number) => void
}

export function usePagination(
  total: Ref<number>,
  options?: PaginationOptions
): PaginationReturn {
  const page = ref(options?.initialPage ?? 1)
  const pageSize = ref(options?.initialPageSize ?? 10)

  const offset = computed(() => (page.value - 1) * pageSize.value)
  const totalPages = computed(() => Math.ceil(total.value / pageSize.value))
  const hasNextPage = computed(() => page.value < totalPages.value)
  const hasPrevPage = computed(() => page.value > 1)

  function nextPage() {
    if (hasNextPage.value) page.value++
  }

  function prevPage() {
    if (hasPrevPage.value) page.value--
  }

  function goToPage(p: number) {
    page.value = Math.max(1, Math.min(p, totalPages.value))
  }

  function setPageSize(size: number) {
    pageSize.value = size
    page.value = 1 // Reset la prima pagină
  }

  return {
    page, pageSize, offset, totalPages,
    hasNextPage, hasPrevPage,
    nextPage, prevPage, goToPage, setPageSize
  }
}
```

### 7.4 Composable cu Generics

```typescript
// composables/useLocalStorage.ts
import { ref, watch } from 'vue'
import type { Ref } from 'vue'

export function useLocalStorage<T>(
  key: string,
  defaultValue: T
): Ref<T> {
  const stored = localStorage.getItem(key)
  const data = ref<T>(
    stored ? (JSON.parse(stored) as T) : defaultValue
  ) as Ref<T>

  watch(data, (newVal) => {
    localStorage.setItem(key, JSON.stringify(newVal))
  }, { deep: true })

  return data
}

// Usage - T inferat din defaultValue
const theme = useLocalStorage('theme', 'light')       // Ref<string>
const count = useLocalStorage('count', 0)              // Ref<number>
const prefs = useLocalStorage('prefs', { dark: false }) // Ref<{ dark: boolean }>
```

### 7.5 Composable care returnează un obiect vs tuple

```typescript
// Obiect - când ai multe proprietăți (pattern standard)
export function useAuth(): {
  user: Ref<User | null>
  isAuthenticated: ComputedRef<boolean>
  login: (credentials: Credentials) => Promise<void>
  logout: () => Promise<void>
} {
  // ...
}

// Tuple - când ai puține valori (similar React hooks)
export function useToggle(
  initial: boolean = false
): [Ref<boolean>, () => void] {
  const state = ref(initial)
  const toggle = () => { state.value = !state.value }
  return [state, toggle]
}

// Usage
const [isOpen, toggleOpen] = useToggle(false)
```

### Paralela cu Angular

```typescript
// Angular - Service cu DI
@Injectable({ providedIn: 'root' })
export class ApiService {
  constructor(private http: HttpClient) {}

  getUsers(): Observable<User[]> {
    return this.http.get<User[]>('/api/users')
  }
}

// Vue - Composable (funcție)
export function useApi<T>(url: string): UseApiReturn<T> {
  // Logica reactivă cu ref, computed, watch
  return { data, loading, error, execute }
}
```

| Aspect | Angular Service | Vue Composable |
|--------|----------------|----------------|
| Paradigmă | Class + DI | Funcție |
| State | Proprietăți pe clasă | ref/reactive |
| Return type | Metode individuale | Obiect cu toate proprietățile |
| Typing | Class types + generics | Interface + generics |
| Testare | TestBed + inject | Apel direct al funcției |
| Reutilizare | Inject în orice component | Import și apel |

> **Key insight:** Typed composables în Vue sunt **mai ușor de tipizat** decât Angular services deoarece sunt funcții pure cu return type explicit. Nu ai nevoie de DI container, TestBed, sau decorators. Tipul returnat este contractul.

---

## 8. Typing provide/inject

### 8.1 Problema fără typing

```typescript
// ❌ Fără InjectionKey - pierzi type safety
provide('auth', { user, login, logout })

const auth = inject('auth')
// auth este: unknown
// Trebuie cast manual, fragil și error-prone
```

### 8.2 InjectionKey<T>

```typescript
// types/injection-keys.ts
import type { InjectionKey, Ref } from 'vue'

// ==========================================
// Definire chei tipizate
// ==========================================

interface User {
  id: number
  name: string
  email: string
  role: 'admin' | 'user'
}

interface Credentials {
  email: string
  password: string
}

interface AuthContext {
  user: Ref<User | null>
  isAuthenticated: ComputedRef<boolean>
  login: (credentials: Credentials) => Promise<void>
  logout: () => Promise<void>
  hasRole: (role: string) => boolean
}

interface ThemeContext {
  current: Ref<'light' | 'dark'>
  toggle: () => void
  set: (theme: 'light' | 'dark') => void
}

interface NotificationContext {
  show: (message: string, type?: 'info' | 'success' | 'warning' | 'error') => void
  clear: () => void
}

// Symbol-based injection keys cu tip generic
export const AuthKey: InjectionKey<AuthContext> = Symbol('auth')
export const ThemeKey: InjectionKey<ThemeContext> = Symbol('theme')
export const NotificationKey: InjectionKey<NotificationContext> = Symbol('notification')
```

### 8.3 Provider cu type safety

```vue
<!-- App.vue - Provider -->
<script setup lang="ts">
import { provide, ref, computed } from 'vue'
import { AuthKey, ThemeKey } from '@/types/injection-keys'

// Auth context
const user = ref<User | null>(null)
const isAuthenticated = computed(() => user.value !== null)

async function login(credentials: Credentials) {
  const response = await fetch('/api/login', {
    method: 'POST',
    body: JSON.stringify(credentials)
  })
  user.value = await response.json()
}

async function logout() {
  await fetch('/api/logout', { method: 'POST' })
  user.value = null
}

function hasRole(role: string): boolean {
  return user.value?.role === role
}

// ✅ TypeScript verifică că obiectul corespunde AuthContext
provide(AuthKey, { user, isAuthenticated, login, logout, hasRole })

// Theme context
const currentTheme = ref<'light' | 'dark'>('light')
provide(ThemeKey, {
  current: currentTheme,
  toggle: () => {
    currentTheme.value = currentTheme.value === 'light' ? 'dark' : 'light'
  },
  set: (theme) => { currentTheme.value = theme }
})
</script>
```

### 8.4 Consumer cu type safety

```vue
<!-- ChildComponent.vue - Consumer -->
<script setup lang="ts">
import { inject } from 'vue'
import { AuthKey, ThemeKey } from '@/types/injection-keys'

// Varianta 1: Cu posibil undefined
const auth = inject(AuthKey)
// auth: AuthContext | undefined
// Trebuie să verifici dacă există

// Varianta 2: Cu default value
const auth = inject(AuthKey, {
  user: ref(null),
  isAuthenticated: computed(() => false),
  login: async () => {},
  logout: async () => {},
  hasRole: () => false
})
// auth: AuthContext (garantat)

// Varianta 3: Non-null assertion (dacă ești sigur că provider-ul există)
const auth = inject(AuthKey)!
// auth: AuthContext (dar risc de runtime error dacă lipsește provider)

// Utilizare
const { user, isAuthenticated, logout, hasRole } = auth

// Theme - cu factory default
const theme = inject(ThemeKey, () => ({
  current: ref<'light' | 'dark'>('light'),
  toggle: () => {},
  set: () => {}
}), true)  // Al treilea argument = true indică factory function
</script>
```

### 8.5 Helper function pattern

```typescript
// composables/useAuth.ts - Wrapper peste inject
import { inject } from 'vue'
import { AuthKey } from '@/types/injection-keys'
import type { AuthContext } from '@/types/injection-keys'

export function useAuth(): AuthContext {
  const auth = inject(AuthKey)
  if (!auth) {
    throw new Error(
      'useAuth() requires AuthProvider. ' +
      'Asigură-te că <AuthProvider> este în tree-ul de componente.'
    )
  }
  return auth
}

// Usage - simplu și sigur
const { user, login, logout } = useAuth()
```

### Paralela cu Angular

```typescript
// Angular - InjectionToken<T>
export const AUTH_TOKEN = new InjectionToken<AuthContext>('auth')

// Provider
@NgModule({
  providers: [
    { provide: AUTH_TOKEN, useFactory: () => ({ ... }) }
  ]
})

// Consumer
@Component({ ... })
export class ChildComponent {
  constructor(@Inject(AUTH_TOKEN) private auth: AuthContext) {}
}
```

| Aspect | Angular InjectionToken | Vue InjectionKey |
|--------|----------------------|------------------|
| Creare | `new InjectionToken<T>('name')` | `Symbol('name') as InjectionKey<T>` |
| Provide | Module/Component providers | `provide(key, value)` |
| Inject | Constructor injection | `inject(key)` |
| Default value | `@Optional()` decorator | Al doilea argument la `inject()` |
| Scope | Hierarchic (Module, Component) | Hierarchic (Component tree) |
| Tree-shaking | Depinde de `providedIn` | Automat (Symbol) |

> **Key insight:** Pattern-ul e foarte similar: Angular InjectionToken<T> ≈ Vue InjectionKey<T>. Ambele folosesc un token/key tipizat pentru a lega provider-ul de consumer. Diferența principală: Angular injectează prin constructor, Vue prin funcția inject().

---

## 9. Typing Pinia Stores

### 9.1 Setup Store - Typing automat

**Setup stores** (composition API style) au cel mai bun TypeScript support deoarece tipurile sunt **inferate automat**:

```typescript
// stores/useUserStore.ts
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'

interface User {
  id: number
  name: string
  email: string
  role: 'admin' | 'editor' | 'viewer'
  preferences: {
    theme: 'light' | 'dark'
    language: string
    notifications: boolean
  }
}

interface LoginCredentials {
  email: string
  password: string
}

export const useUserStore = defineStore('user', () => {
  // State - fully typed automat
  const currentUser = ref<User | null>(null)
  const token = ref<string | null>(null)
  const loading = ref(false)
  const error = ref<string | null>(null)

  // Getters - ComputedRef types inferate
  const isAuthenticated = computed(() => currentUser.value !== null)
  const isAdmin = computed(() => currentUser.value?.role === 'admin')
  const displayName = computed(() => currentUser.value?.name ?? 'Guest')

  // Actions - return types inferate
  async function login(credentials: LoginCredentials): Promise<void> {
    loading.value = true
    error.value = null
    try {
      const response = await fetch('/api/auth/login', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(credentials)
      })
      if (!response.ok) throw new Error('Login failed')
      const data = await response.json()
      currentUser.value = data.user
      token.value = data.token
    } catch (e) {
      error.value = e instanceof Error ? e.message : 'Unknown error'
      throw e
    } finally {
      loading.value = false
    }
  }

  async function logout(): Promise<void> {
    await fetch('/api/auth/logout', { method: 'POST' })
    currentUser.value = null
    token.value = null
  }

  function updatePreferences(prefs: Partial<User['preferences']>): void {
    if (currentUser.value) {
      currentUser.value.preferences = {
        ...currentUser.value.preferences,
        ...prefs
      }
    }
  }

  return {
    // State
    currentUser,
    token,
    loading,
    error,
    // Getters
    isAuthenticated,
    isAdmin,
    displayName,
    // Actions
    login,
    logout,
    updatePreferences
  }
})
```

### 9.2 Usage în componente

```vue
<script setup lang="ts">
import { useUserStore } from '@/stores/useUserStore'
import { storeToRefs } from 'pinia'

const userStore = useUserStore()

// ✅ Destructurare cu storeToRefs - păstrează reactivitatea
const { currentUser, isAuthenticated, isAdmin, displayName } = storeToRefs(userStore)
// currentUser: Ref<User | null>
// isAuthenticated: Ref<boolean>

// Actions - destructurare directă (nu sunt reactive)
const { login, logout, updatePreferences } = userStore

async function handleLogin() {
  try {
    await login({ email: 'test@test.com', password: '123' })
    // currentUser.value este acum User
  } catch (error) {
    // error handling
  }
}
</script>
```

### 9.3 Options Store - Typing explicit

```typescript
// stores/useCartStore.ts
interface CartItem {
  id: number
  name: string
  price: number
  quantity: number
}

interface CartState {
  items: CartItem[]
  coupon: string | null
  discount: number
}

export const useCartStore = defineStore('cart', {
  state: (): CartState => ({
    items: [],
    coupon: null,
    discount: 0
  }),

  getters: {
    totalItems(): number {
      return this.items.reduce((sum, item) => sum + item.quantity, 0)
    },
    subtotal(): number {
      return this.items.reduce((sum, item) => sum + item.price * item.quantity, 0)
    },
    total(): number {
      return this.subtotal * (1 - this.discount)
    },
    isEmpty(): boolean {
      return this.items.length === 0
    }
  },

  actions: {
    addItem(item: Omit<CartItem, 'quantity'>) {
      const existing = this.items.find(i => i.id === item.id)
      if (existing) {
        existing.quantity++
      } else {
        this.items.push({ ...item, quantity: 1 })
      }
    },
    removeItem(id: number) {
      this.items = this.items.filter(item => item.id !== id)
    },
    async applyCoupon(code: string): Promise<boolean> {
      const response = await fetch(`/api/coupons/${code}`)
      if (response.ok) {
        const data = await response.json()
        this.coupon = code
        this.discount = data.discount
        return true
      }
      return false
    },
    clearCart() {
      this.items = []
      this.coupon = null
      this.discount = 0
    }
  }
})
```

### Paralela cu Angular

```typescript
// Angular - NgRx Store cu TypeScript
// State interface
interface UserState {
  user: User | null
  loading: boolean
  error: string | null
}

// Actions - tipizate
export const login = createAction('[Auth] Login', props<{ credentials: LoginCredentials }>())
export const loginSuccess = createAction('[Auth] Login Success', props<{ user: User }>())

// Reducer - tipizat
export const userReducer = createReducer<UserState>(
  initialState,
  on(loginSuccess, (state, { user }) => ({ ...state, user, loading: false }))
)

// Selectors - tipizați
export const selectUser = (state: AppState) => state.user.user
export const selectIsAuthenticated = createSelector(selectUser, user => user !== null)
```

> **Key insight:** Pinia setup stores au TypeScript inference **automat** - nu trebuie să declari tipuri separate pentru state, getters, actions. NgRx în Angular necesită tipuri explicit pe state, actions, reducers, selectors. Pinia e semnificativ mai concis.

---

## 10. Typing Template Refs

### 10.1 DOM Element Refs

```vue
<script setup lang="ts">
import { ref, onMounted } from 'vue'

// Legacy pattern - ref cu același nume
const inputRef = ref<HTMLInputElement | null>(null)
const canvasRef = ref<HTMLCanvasElement | null>(null)
const videoRef = ref<HTMLVideoElement | null>(null)

onMounted(() => {
  // TypeScript știe tipul elementului
  inputRef.value?.focus()
  inputRef.value?.select()

  const ctx = canvasRef.value?.getContext('2d')
  ctx?.fillRect(0, 0, 100, 100)

  videoRef.value?.play()
})
</script>

<template>
  <input ref="inputRef" />
  <canvas ref="canvasRef" />
  <video ref="videoRef" />
</template>
```

### 10.2 useTemplateRef (Vue 3.5+)

```vue
<script setup lang="ts">
import { useTemplateRef, onMounted } from 'vue'

// ✅ Vue 3.5+ - mai explicit, decuplat de numele variabilei
const inputEl = useTemplateRef<HTMLInputElement>('my-input')
const formEl = useTemplateRef<HTMLFormElement>('my-form')

onMounted(() => {
  inputEl.value?.focus()
  formEl.value?.reset()
})
</script>

<template>
  <!-- ref-ul din template corespunde string-ului din useTemplateRef -->
  <form ref="my-form">
    <input ref="my-input" />
  </form>
</template>
```

**Avantajul `useTemplateRef`:** Numele variabilei nu mai trebuie să se potrivească cu ref-ul din template. Poți avea `const input = useTemplateRef('my-custom-ref-name')`.

### 10.3 Component Refs

```vue
<script setup lang="ts">
import { ref, onMounted } from 'vue'
import ChildComponent from './ChildComponent.vue'
import DataTable from './DataTable.vue'

// InstanceType<typeof Component> pentru component refs
const childRef = ref<InstanceType<typeof ChildComponent> | null>(null)
const tableRef = ref<InstanceType<typeof DataTable> | null>(null)

onMounted(() => {
  // Accesezi doar ce e expus cu defineExpose
  childRef.value?.someMethod()
  childRef.value?.publicData

  tableRef.value?.refresh()
  tableRef.value?.scrollToRow(5)
})
</script>

<template>
  <ChildComponent ref="childRef" />
  <DataTable ref="tableRef" />
</template>
```

### 10.4 defineExpose cu TypeScript

```vue
<!-- ChildComponent.vue -->
<script setup lang="ts">
const internalData = ref('secret')
const publicData = ref('visible')

function someMethod(): void {
  console.log('Called from parent')
}

function refresh(): Promise<void> {
  return fetch('/api/data').then(() => {})
}

// Doar ce e în defineExpose e accesibil prin ref
defineExpose({
  publicData,
  someMethod,
  refresh
})
// Parentul NU poate accesa internalData
</script>
```

### 10.5 Ref pe v-for

```vue
<script setup lang="ts">
import { ref, onMounted } from 'vue'

// Array de refs pentru v-for
const itemRefs = ref<HTMLDivElement[]>([])

onMounted(() => {
  itemRefs.value.forEach((el, index) => {
    console.log(`Item ${index}:`, el.textContent)
  })
})
</script>

<template>
  <div v-for="item in items" :key="item.id" ref="itemRefs">
    {{ item.name }}
  </div>
</template>
```

### Paralela cu Angular

```typescript
// Angular - @ViewChild cu ElementRef
@Component({ ... })
export class MyComponent {
  @ViewChild('inputEl') inputEl!: ElementRef<HTMLInputElement>
  @ViewChild(ChildComponent) childRef!: ChildComponent
  @ViewChildren('items') itemRefs!: QueryList<ElementRef<HTMLDivElement>>

  ngAfterViewInit() {
    this.inputEl.nativeElement.focus()
    this.childRef.someMethod()
    this.itemRefs.forEach(ref => console.log(ref.nativeElement))
  }
}

// Vue - ref + useTemplateRef
const inputEl = useTemplateRef<HTMLInputElement>('inputEl')
const childRef = ref<InstanceType<typeof ChildComponent> | null>(null)
```

| Aspect | Angular | Vue 3 |
|--------|---------|-------|
| DOM ref | `@ViewChild` + `ElementRef<T>` | `ref<T \| null>(null)` |
| Vue 3.5+ | - | `useTemplateRef<T>('name')` |
| Component ref | `@ViewChild(Component)` | `ref<InstanceType<typeof C>>` |
| Multiple refs | `@ViewChildren` + `QueryList` | `ref<T[]>([])` + v-for |
| Access timing | `ngAfterViewInit` | `onMounted` |
| Public API | Totul e public by default | `defineExpose` controlează |

> **Key insight:** Vue cu `defineExpose` oferă **encapsulare mai bună** decât Angular, unde toate metodele publice ale unui component sunt accesibile prin `@ViewChild`. În Vue trebuie să expui explicit ce vrei să fie accesibil.

---

## 11. Utility Types din Vue

### 11.1 PropType<T>

Folosit în **runtime declaration** (nu type-based) pentru tipuri complexe:

```typescript
import type { PropType } from 'vue'

interface Book {
  title: string
  author: string
  year: number
}

// Runtime declaration cu PropType
export default defineComponent({
  props: {
    book: {
      type: Object as PropType<Book>,
      required: true
    },
    status: {
      type: String as PropType<'draft' | 'published' | 'archived'>,
      default: 'draft'
    },
    tags: {
      type: Array as PropType<string[]>,
      default: () => []
    },
    callback: {
      type: Function as PropType<(id: number) => Promise<void>>,
      required: false
    }
  }
})
```

**Notă:** `PropType<T>` nu este necesar cu `<script setup lang="ts">` și `defineProps<T>()`. E relevant doar pentru Options API sau când folosești runtime declaration.

### 11.2 ExtractPropTypes și ExtractPublicPropTypes

```typescript
import type { ExtractPropTypes, ExtractPublicPropTypes } from 'vue'

const propsDefinition = {
  title: { type: String, required: true as const },
  count: { type: Number, default: 0 },
  disabled: Boolean
}

// ExtractPropTypes - tipuri interne (ce vede componenta)
type InternalProps = ExtractPropTypes<typeof propsDefinition>
// { title: string; count: number; disabled: boolean }

// ExtractPublicPropTypes - tipuri publice (ce vede parentul)
type PublicProps = ExtractPublicPropTypes<typeof propsDefinition>
// { title: string; count?: number; disabled?: boolean }
// count și disabled sunt opționale deoarece au defaults
```

### 11.3 ComponentPublicInstance

```typescript
import type { ComponentPublicInstance } from 'vue'

// Tipul generic al unei instanțe de component
const instance = ref<ComponentPublicInstance | null>(null)

// Cu props specifice
type MyComponentInstance = ComponentPublicInstance<{
  title: string
  count: number
}>
```

### 11.4 MaybeRef și MaybeRefOrGetter

```typescript
import type { MaybeRef, MaybeRefOrGetter } from 'vue'
import { toValue } from 'vue'

// MaybeRef<T> = T | Ref<T>
function useTitle(title: MaybeRef<string>) {
  // Funcționează cu string simplu SAU Ref<string>
  watchEffect(() => {
    document.title = toValue(title)
  })
}

// MaybeRefOrGetter<T> = T | Ref<T> | (() => T)
function useWidth(width: MaybeRefOrGetter<number>) {
  const resolved = computed(() => toValue(width))
  return resolved
}

// Usage - toate variantele funcționează
useTitle('Static Title')
useTitle(ref('Reactive Title'))

useWidth(100)
useWidth(ref(200))
useWidth(() => window.innerWidth)
```

### 11.5 ShallowRef, TriggerRef, CustomRef

```typescript
import { shallowRef, triggerRef, customRef } from 'vue'
import type { ShallowRef } from 'vue'

// ShallowRef - nu urmărește proprietățile nested
const state = shallowRef<{ nested: { count: number } }>({
  nested: { count: 0 }
})

// Trebuie înlocuit complet pentru trigger
state.value = { nested: { count: 1 } }

// Sau forțat trigger
state.value.nested.count = 2
triggerRef(state)  // Forțează re-render

// CustomRef - ref cu control total
function useDebouncedRef<T>(value: T, delay = 200) {
  let timeout: ReturnType<typeof setTimeout>
  return customRef<T>((track, trigger) => ({
    get() {
      track()
      return value
    },
    set(newValue) {
      clearTimeout(timeout)
      timeout = setTimeout(() => {
        value = newValue
        trigger()
      }, delay)
    }
  }))
}

const search = useDebouncedRef<string>('', 300)
```

### 11.6 Tipuri utile pentru componente

```typescript
import type {
  VNode,
  Slot,
  Slots,
  SetupContext,
  RenderFunction,
  FunctionalComponent,
  DefineComponent
} from 'vue'

// Functional component tipizat
const Badge: FunctionalComponent<{
  text: string
  variant?: 'info' | 'success' | 'warning' | 'error'
}> = (props, { slots }) => {
  return h('span', {
    class: `badge badge-${props.variant ?? 'info'}`
  }, props.text)
}

Badge.props = {
  text: { type: String, required: true },
  variant: String
}
```

---

## 12. Paralela: TypeScript în Angular vs Vue

### 12.1 Tabel comparativ complet

| Aspect | Angular | Vue 3 |
|--------|---------|-------|
| **TS obligatoriu** | Da | Nu (dar recomandat) |
| **Component typing** | Class + Decorators | `defineProps<T>`, `defineEmits<T>` |
| **Input typing** | `@Input() prop: Type` | `defineProps<{ prop: Type }>()` |
| **Output typing** | `@Output() EventEmitter<T>` | `defineEmits<{ event: [T] }>()` |
| **Generic components** | Nu direct | `generic="T"` pe script setup |
| **DI typing** | `InjectionToken<T>` | `InjectionKey<T>` (Symbol) |
| **Template checking** | `strictTemplates: true` | `vue-tsc` + Volar |
| **Form typing** | `FormControl<T>` (v14+) | Manual / Zod / VeeValidate |
| **HTTP typing** | `HttpClient.get<T>()` | Manual pe fetch/axios |
| **Router typing** | Partial (route params as string) | `defineProps` + route meta typing |
| **Store typing** | NgRx - actions, reducers, selectors | Pinia - automat (setup stores) |
| **Enum support** | Nativ (TypeScript enums) | Nativ (TypeScript enums) |
| **Strict mode** | `strictTemplates`, `strict` | `strict: true` + `vue-tsc` |
| **Build-time check** | `ngc` (Angular Compiler) | `vue-tsc` |
| **IDE tooling** | Angular Language Service | Volar (Vue Official) |
| **Decorator usage** | Extensiv (@Component, @Input, etc.) | Niciun decorator |

### 12.2 Unde Angular e mai bun

1. **TypeScript e obligatoriu** - nu poți scrie Angular fără TS, deci nu există proiecte Angular cu JavaScript pur care devin greu de menținut
2. **HttpClient tipizat** - `this.http.get<User[]>('/api/users')` e built-in
3. **Strict templates** - Angular strict template checking e matur și stabil
4. **Forms tipizate** - `FormControl<T>` (Angular 14+) oferă type safety pe reactive forms

### 12.3 Unde Vue e mai bun

1. **Generic components** - Vue suportă `generic="T"`, Angular nu
2. **Pinia type inference** - zero boilerplate pentru tipuri, totul e inferat
3. **defineProps/defineEmits** - contract unic, mai clean decât decorators individuali
4. **defineModel** - two-way binding tipizat cu o linie de cod
5. **Composable typing** - funcții pure, ușor de tipizat și testat
6. **defineSlots** - slot props tipizate (Angular ng-content nu are echivalent)

### 12.4 Code comparison - Același component

```typescript
// ==========================================
// Angular Version
// ==========================================
@Component({
  selector: 'app-user-list',
  template: `
    <div *ngFor="let user of filteredUsers">
      <span>{{ user.name }}</span>
      <button (click)="selectUser(user)">Select</button>
    </div>
  `
})
export class UserListComponent implements OnInit {
  @Input() users: User[] = []
  @Input() filter: string = ''
  @Output() select = new EventEmitter<User>()

  get filteredUsers(): User[] {
    return this.users.filter(u =>
      u.name.toLowerCase().includes(this.filter.toLowerCase())
    )
  }

  selectUser(user: User): void {
    this.select.emit(user)
  }

  ngOnInit(): void {
    console.log('Users loaded:', this.users.length)
  }
}
```

```vue
<!-- ==========================================
     Vue 3 Version
     ========================================== -->
<script setup lang="ts">
import { computed, onMounted } from 'vue'

interface Props {
  users: User[]
  filter?: string
}

const props = withDefaults(defineProps<Props>(), {
  filter: ''
})

const emit = defineEmits<{
  select: [user: User]
}>()

const filteredUsers = computed(() =>
  props.users.filter(u =>
    u.name.toLowerCase().includes(props.filter.toLowerCase())
  )
)

function selectUser(user: User) {
  emit('select', user)
}

onMounted(() => {
  console.log('Users loaded:', props.users.length)
})
</script>

<template>
  <div v-for="user in filteredUsers" :key="user.id">
    <span>{{ user.name }}</span>
    <button @click="selectUser(user)">Select</button>
  </div>
</template>
```

> **Key insight:** Ambele variante oferă type safety comparabil. Vue e mai concis (fără decorators, fără class). Angular e mai familiar pentru dezvoltatori enterprise. Alegerea depinde de preferința echipei și de ecosistemul proiectului.

---

## 13. Întrebări de interviu

### Î1: Cum configurezi TypeScript într-un proiect Vue 3?

**Răspuns:**
Vue 3 are suport nativ TypeScript deoarece framework-ul însuși este scris în TypeScript. Configurarea implică: `npm create vue@latest` cu opțiunea TypeScript activată, un `tsconfig.json` cu `strict: true` și `moduleResolution: "bundler"`, extensia Volar (Vue Official) în IDE, și `vue-tsc` pentru type checking la build. Fișierul `env.d.ts` declară modulele `.vue` pentru TypeScript. Important: Vite nu face type checking la build - doar transpilă. De aceea rulăm `vue-tsc --noEmit` înainte de `vite build` în scriptul de build.

### Î2: Care e diferența între type-based și runtime declaration la defineProps?

**Răspuns:**
**Type-based** (`defineProps<{ title: string }>()`) folosește interfețe TypeScript pentru validare la compile time - mai precis, mai ușor de citit, suportă union types și tipuri complexe. **Runtime declaration** (`defineProps({ title: { type: String, required: true } })`) generează validare la runtime în development mode. Nu le poți combina. Type-based este recomandat în proiecte TypeScript deoarece IDE-ul oferă autocompletion mai bun, dar pierde runtime validation. Dacă ai nevoie de ambele, poți folosi librării externe ca Zod cu `defineProps`.

### Î3: Cum faci un generic component în Vue 3?

**Răspuns:**
Folosești atributul `generic` pe `<script setup>`: `<script setup lang="ts" generic="T extends { id: number }">`. Poți defini constraints cu `extends`, multiple generics cu virgulă (`generic="T, U extends string"`), și tipul T este disponibil în `defineProps`, `defineEmits`, și `defineSlots`. Tipul concret este **inferat automat** la utilizare din props-urile furnizate. Aceasta e o funcționalitate unică Vue - Angular nu are echivalent direct. Generic components sunt ideale pentru componente reutilizabile ca tabele, liste, selecturi.

### Î4: Cum tipezi composables corect pentru reusability maximă?

**Răspuns:**
Definești interfețe separate pentru **opțiuni** (input) și **return type** (output). Folosești generics pentru flexibilitate (`useApi<T>`). Return type-ul trebuie să fie explicit cu `Ref<T>`, `ComputedRef<T>`. Parametrii acceptă `MaybeRef<T>` sau `MaybeRefOrGetter<T>` pentru flexibilitate maximă, iar intern folosești `toValue()` pentru normalizare. Exportezi atât funcția cât și interfețele de tip. Pattern-ul standard: opțiuni obiect cu defaults, return obiect cu proprietăți reactive și funcții de acțiune.

### Î5: Cum tipezi provide/inject corect?

**Răspuns:**
Creezi un `InjectionKey<T>` bazat pe `Symbol` - similar cu Angular `InjectionToken<T>`. Definești interfața contextului (`AuthContext`, `ThemeContext`), creezi cheia (`export const AuthKey: InjectionKey<AuthContext> = Symbol('auth')`), și TypeScript verifică automat că `provide(AuthKey, value)` furnizează tipul corect iar `inject(AuthKey)` returnează `AuthContext | undefined`. Best practice: creezi un composable wrapper (`useAuth()`) care face `inject` și aruncă eroare descriptivă dacă provider-ul lipsește.

### Î6: Ce e vue-tsc și de ce e necesar?

**Răspuns:**
`vue-tsc` este un wrapper peste TypeScript compiler (`tsc`) care înțelege fișierele `.vue`. Vite folosește esbuild pentru transpilare rapidă dar **nu face type checking**. `vue-tsc` verifică tipurile în fișierele `.vue` inclusiv: binding-uri din template (`{{ user.name }}`), props pasate la child components, event handlers, slot props. Se rulează în CI/CD (`vue-tsc --noEmit`) și opțional în watch mode local. Fără vue-tsc, erorile de tip din templates nu sunt detectate decât de Volar în IDE, ceea ce nu e suficient pentru un pipeline de CI robust.

### Î7: Cum tipezi template refs?

**Răspuns:**
Pentru **DOM elements**: `ref<HTMLInputElement | null>(null)` sau Vue 3.5+ `useTemplateRef<HTMLInputElement>('ref-name')`. Pentru **componente**: `ref<InstanceType<typeof ChildComponent> | null>(null)`. `useTemplateRef` e preferabil deoarece decuplează numele variabilei de ref-ul din template. Component refs expun doar ce e în `defineExpose()`. Pentru `v-for`, tipul este array: `ref<HTMLDivElement[]>([])`. Inițializarea cu `null` e importantă deoarece ref-ul e null înainte de mount.

### Î8: Care sunt diferențele principale TypeScript între Angular și Vue?

**Răspuns:**
Angular **impune** TypeScript, Vue îl **suportă** opțional. Angular folosește decorators (`@Input`, `@Output`, `@Component`) cu tipuri pe class properties. Vue folosește `defineProps<T>()` și `defineEmits<T>()` cu interfețe. Vue are **generic components** (`generic="T"`), Angular nu. Angular are HttpClient tipizat built-in, Vue necesită typing manual pe fetch/axios. Pinia (Vue) inferă tipuri automat, NgRx (Angular) necesită tipuri explicite pe actions, reducers, selectors. Angular are Typed Forms (`FormControl<T>`) built-in, Vue se bazează pe librării externe.

### Î9: Cum gestionezi tipuri complexe în props?

**Răspuns:**
Definești interfețe separate în fișiere `.ts` dedicate și le importezi cu `import type`. Union literal types funcționează direct (`variant?: 'primary' | 'secondary'`). Pentru arrays și objects, factory functions în `withDefaults` (`items: () => []`). Pentru props dinamice, `Record<string, T>`. Tipuri utilitare TypeScript (Partial, Pick, Omit) funcționează cu defineProps. În Vue 3.5+, destructurarea props cu defaults inline elimină nevoia de withDefaults. Limitare: defineProps type-based nu suportă tipuri resolve-uite dinamic din generics externe complexe.

### Î10: defineModel - cum funcționează cu TypeScript?

**Răspuns:**
`defineModel<T>()` (Vue 3.4+) creează automat un `Ref<T>` bidirectional. Generează intern prop-ul (`modelValue`) și emit-ul (`update:modelValue`). Suportă modifiers prin destructurare: `const [value, modifiers] = defineModel<string>()`. Multiple v-models: `defineModel<string>('title')` pentru `v-model:title`. Cu `{ required: true }` tipul devine `Ref<T>` (nu `Ref<T | undefined>`). Cu `{ default: value }` oferă valoare implicită. E echivalentul Vue al funcției `model()` din Angular 17.1+, dar a fost disponibil mai devreme.

### Î11: Ce sunt MaybeRef și MaybeRefOrGetter și când le folosești?

**Răspuns:**
`MaybeRef<T>` = `T | Ref<T>` - acceptă valoare simplă sau ref. `MaybeRefOrGetter<T>` = `T | Ref<T> | (() => T)` - acceptă și getter functions. Le folosești în **parametrii composables** pentru flexibilitate maximă. Intern, `toValue()` (Vue 3.3+) normalizează oricare variantă la valoarea raw. Exemplu: `function useTitle(title: MaybeRefOrGetter<string>)` acceptă `useTitle('static')`, `useTitle(ref('reactive'))`, sau `useTitle(() => computedValue)`. Acest pattern face composables-urile mai ergonomice la utilizare fără a sacrifica reactivitatea.

### Î12: Cum asiguri type safety end-to-end într-o aplicație Vue mare?

**Răspuns:**
Activezi `strict: true` în tsconfig. Rulezi `vue-tsc --noEmit` în CI/CD pipeline. Folosești type-based declaration pentru defineProps/defineEmits/defineSlots. Definești InjectionKey<T> pentru provide/inject. Folosești setup stores Pinia pentru type inference automat. Creezi interfețe shared în directorul `types/`. Folosești generic components pentru componente reutilizabile. Implementezi typed composables cu return types explicite. Activezi ESLint cu `@typescript-eslint` rules. Scrii fișiere `.d.ts` pentru module externe netipizate. Acest setup oferă type safety comparabil cu Angular, dar cu mai puțin boilerplate.

---

> **Sumar:** TypeScript în Vue 3 este un first-class citizen. Cu `defineProps<T>`, `defineEmits<T>`,
> generic components, și typed composables, Vue 3 oferă type safety comparabil cu Angular,
> dar cu o abordare mai flexibilă și mai puțin ceremonioasă. Generic components și
> defineModel sunt funcționalități unde Vue **depășește** Angular ca expresivitate TypeScript.


---

**Următor :** [**09 - Testare Vue** →](Vue/09-Testare-Vue.md)